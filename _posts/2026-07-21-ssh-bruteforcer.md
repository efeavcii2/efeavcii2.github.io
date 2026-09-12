---
title: "ssh-bruteforcer — SSH Credential Brute-Force Tool"
date: 2026-07-21 07:35:00 +0300
categories: [Projeler]
tags: [python, threading, ssh, bruteforce]
---

> Kişisel Proje · Python 3 · CLI Tool
{: .prompt-info }

Yetkili pentest/CTF ortamları için SSH kimlik bilgisi brute-force aracı. Kullanıcı adı/parola listelerini bir thread havuzuyla paralel dener; hedefin rate-limit/lockout savunmasını tetiklediğini fark edip kendini durdurabilir.

| Dil | Yöntem | Hızlandırma | Repo |
|---|---|---|---|
| Python 3 | Credential Brute-Force | Threading (I/O-bound) | [GitHub ↗](https://github.com/efeavcii2/ssh-bruteforcer) |

**Akış:** Userlist × Wordlist → Thread havuzuna dağıt → SSH auth denemesi → Bulunca / lockout görünce dur

## 01 — Ne yapıyor: SSH kimlik bilgisi brute-force

Bir kullanıcı adı listesi ile bir parola listesini (ya da tek bir kullanıcı adı/parola) çarpraz deneyip hedef SSH sunucusuna karşı test ediyor. `--workers` kadar thread eşzamanlı çalışıyor, `--delay` ile her denemeden sonra bekleme eklenebiliyor.

[hash-cracker]({% post_url 2026-07-21-hash-cracker %})'ın devamı niteliğinde bir proje — ikisi de kimlik bilgisi kırma mantığında ama farklı bir eşzamanlılık modeliyle.

## 02 — Mimari: Neden threading, multiprocessing değil

hash-cracker'da multiprocessing kullanmıştım çünkü hash hesaplama CPU-bound bir işti ve GIL yüzünden threading paralellik sağlamıyordu. Burada durum tam tersi: SSH denemesinin maliyeti CPU'da değil, ağ round-trip'inde ve TCP/SSH handshake'inde (key exchange, vb.) — yani I/O-bound. I/O beklerken Python zaten GIL'i serbest bırakıyor, dolayısıyla `threading` burada hem yeterli hem de `multiprocessing`'den daha hafif.

- `modules/ssh_client.py` → `try_login()`: paramiko ile tek bir bağlantı dener, sonucu dört duruma eşler: `SUCCESS`, `FAILED`, `RATE_LIMITED` (banner okunamadı/bağlantı koptu — hedef bir savunma tetikledi demek), `UNREACHABLE`.
- `modules/bruteforcer.py` → kombinasyonları bir `queue.Queue`'ya doldurup N thread'e dağıtıyor, `threading.Event` ile erken çıkış yapıyor. Art arda `RATE_LIMITED` belirli bir eşiği geçerse aracı durduruyor — kör kör denemeye devam edip hedefte gürültü yaratmıyor.

## 03 — Gerçek test: Kendi kurduğum SSH hedefine karşı

Localhost'ta 2222 portunda, tek kullanımlık bir test kullanıcısıyla (`sshtestuser` / `test1234`) kendi SSH sunucumu ayağa kaldırıp araca karşı gerçekten çalıştırdım.

```console
$ python3 main.py --target 127.0.0.1 --port 2222 --username sshtestuser --workers 4

[+] KIMLIK BILGISI BULUNDU: sshtestuser : test1234
[i] Durum            : kimlik bilgisi bulundu
[i] Geçen süre       : 18.09 saniye
[i] Denenen kombinasyon: 20
[i] Hız (attempt/sec): 1.11
```

~1.1 attempt/sn — hash-cracker'daki milyonlarca hash/sn ile karşılaştırınca çok düşük ama beklenen: her deneme gerçek bir TCP bağlantısı + SSH key exchange + auth round-trip'i gerektiriyor, tek bir hash hesaplamaktan çok daha pahalı.

> **Beklenmedik bulgu — sshd'nin kendi savunması:** Testin ilk turunda (varsayılan sshd ayarlarıyla) araç sürekli "rate-limit/lockout tespit edildi" diyerek duruyordu. Araştırınca öğrendim: OpenSSH 9.8+ ile gelen `PerSourcePenalties` özelliği varsayılan olarak açık — `authfail:5`, yani bir authentication hatasından sonra o kaynak IP 5 saniyeliğine cezalandırılıp yeni bağlantıları reddediliyor (fail2ban'a bile gerek yok). Aracın `RATE_LIMITED` tespiti bunu doğru yakaladı.
{: .prompt-warning }

```console
$ python3 main.py --target 127.0.0.1 --port 2222 --username sshtestuser --workers 4

[-] Kimlik bilgisi bulunamadı.
[i] Durum            : rate-limit/lockout tespit edildi, durduruldu
[i] Geçen süre       : 7.05 saniye
[i] Denenen kombinasyon: 14
[i] Hız (attempt/sec): 1.99
```

> ✅ her iki senaryo da (bulma + lockout tespiti) gerçek bir SSH hedefine karşı doğrulandı
{: .prompt-tip }

## Özet — Bu projede öğrendiklerim

- I/O-bound bir işte `threading`'in `multiprocessing`'den neden hem yeterli hem daha hafif olduğu — GIL sadece CPU-bound işlerde paralelliği engelliyor.
- Modern OpenSSH'ın (9.8+) `PerSourcePenalties` ile fail2ban'a ihtiyaç duymadan brute-force'a karşı yerleşik bir savunması olduğu.
- Gerçek bir hedefe karşı brute-force yaparken `--workers`/`--delay` ayarının sadece hız için değil, hedefi gereksiz kilitlememek için de önemli olduğu.

---

Kişisel eğitim/CTF projesi — kaynak kod: [github.com/efeavcii2/ssh-bruteforcer](https://github.com/efeavcii2/ssh-bruteforcer). Yalnızca kendi sahip olduğun veya izinli sistemlerde kullan.
