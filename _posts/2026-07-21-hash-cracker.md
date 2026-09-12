---
title: "hash-cracker — CLI Password Cracking Tool"
date: 2026-07-21 05:44:00 +0300
categories: [Projeler]
tags: [python, multiprocessing, hash-cracking]
image: /assets/img/posts/hash-cracker.svg
---

> Kişisel Proje · Python 3 · CLI Tool
{: .prompt-info }

Offline ele geçirilmiş hash'leri (dictionary ve brute-force yöntemleriyle) kırmayı simüle eden, `multiprocessing` ile hızlandırılmış bir komut satırı aracı. Pentest/CTF pratiklerinde kendi hash'lerimi test etmek için yazdım.

| Dil | Yöntem | Hızlandırma | Repo |
|---|---|---|---|
| Python 3 | Dictionary + Bruteforce | Multiprocessing | [GitHub ↗](https://github.com/efeavcii2/hash-cracker) |

**Akış:** Wordlist / Charset → Worker havuzuna böl → Paralel hash karşılaştırma → Bulunca erken çıkış

## 01 — Ne yapıyor: Offline hash kırma simülasyonu

Aracı iki modla çalıştırıyorsun: `--mode dict` bir wordlist'i dener, `--mode bruteforce` verilen bir karakter kümesinden olası tüm kombinasyonları üretip dener. Hash tipi (`md5`, `sha1`, `sha256`, `sha512`, `bcrypt`) uzunluk/format bakılarak otomatik tespit ediliyor, istersen `--hash-type` ile elle de verebiliyorsun.

Amaç canlı bir sisteme saldırmak değil — kendi ürettiğin ya da izinli bir CTF/lab ortamından aldığın hash'leri offline test etmek.

## 02 — Mimari: Wordlist / kombinasyon uzayı işçiler arasında bölünüyor

`modules/hash_utils.py` hash tipini tespit ediyor ve bir aday şifreyi hedef hash ile karşılaştırıyor (bcrypt için `bcrypt.checkpw`, diğerleri için `hashlib`). `modules/cracker.py` ise asıl işi yapıyor:

- **Dictionary attack**: wordlist dosyası okunup satırlar `--workers` kadar process'e eşit parçalar halinde dağıtılıyor.
- **Bruteforce attack**: charset<sup>uzunluk</sup> kombinasyon uzayı, adayların *ilk karakterine* göre process'ler arasında bölüşülüyor (`charset[worker_id::num_workers]`). Her process kendi ilk karakteri için geri kalanını `itertools.product` ile üretiyor — böylece hiçbir process başkasına ait kombinasyonu tekrar üretmiyor.

Bir process doğru şifreyi bulunca `multiprocessing.Value` ile paylaşılan bir flag set ediyor; diğer process'ler bir sonraki kontrol noktasında bunu görüp duruyor, hepsinin işi bitirmesini beklemiyoruz.

> **Neden multiprocessing, threading değil:** Hash hesaplama CPU-bound bir iş, ve CPython'daki GIL yüzünden `threading` bu tür işlerde gerçek paralellik sağlamıyor. Ayrı process'ler GIL'i tamamen bypass ediyor.
{: .prompt-info }

## 03 — Kullanım: Üç mod da test edildi

```console
$ python3 main.py --hash cc03e747a6afbbcbf8be7668acfebee5 --mode dict --workers 4
[+] Tespit edilen hash tipi: md5
[+] SIFRE BULUNDU: test123
[i] Geçen süre       : 0.03 saniye
[i] Denenen aday     : 68
[i] Hız (hashes/sec) : 2186.14
```

```console
$ python3 main.py --hash 749d7048edd2de31c2c7a88d4d196254 \
    --mode bruteforce --charset abcdefghijklmnopqrstuvwxyz0123456789 \
    --min-len 1 --max-len 4 --workers 4
[+] Tespit edilen hash tipi: md5
[+] SIFRE BULUNDU: ab12
[i] Geçen süre       : 0.06 saniye
[i] Denenen aday     : 58401
[i] Hız (hashes/sec) : 921216.80
```

```console
$ python3 main.py --hash '$2b$04$VFlMnPX7xLe.vmZJiAgoDetkn5yj5U2FVgjju8fGKoerxjjh3DGgm' --mode dict
[+] Tespit edilen hash tipi: bcrypt
[+] SIFRE BULUNDU: merhaba123
[i] Geçen süre       : 0.12 saniye
[i] Denenen aday     : 346
[i] Hız (hashes/sec) : 2879.12
```

> ✅ smoke test geçti — MD5(test123) dictionary attack ile bulundu
{: .prompt-tip }

## 04 — Performans: 6 fiziksel çekirdek, 12 mantıksal thread

Kendi makinemde (Ryzen 5 4600H) 62 milyonluk bir bruteforce uzayında (36 karakterlik charset, 1-5 uzunluk, MD5, hash uzayda yok — tam tarama) worker sayısını değiştirerek ölçtüm:

| Workers | Süre | Hız |
|---|---|---|
| 1 | 69.9 sn | ~890K hash/sn |
| 6 | 33.4 sn | ~1.86M hash/sn |
| 12 | 49.7 sn | ~1.25M hash/sn |

12 worker, 6 worker'dan daha yavaş çıktı. Sebebi: makinede fiziksel çekirdek sayısı 6, geri kalan 6 "thread" aynı çekirdekleri paylaşan SMT (hyperthreading) hatları. Hash hesaplama gibi CPU'yu tamamen dolduran bir işte SMT neredeyse fayda getirmiyor — `--workers` için mantıksal thread sayısı yerine fiziksel çekirdek sayısını vermek burada daha hızlı sonuç verdi.

bcrypt'in neden bu kadar yavaş olduğunu da izole ölçtüm — aynı makinede, aynı anda:

| Algoritma | Hız |
|---|---|
| MD5 | ~1.54M hash/sn |
| bcrypt (cost=4, minimum) | ~1.1K hash/sn |

Yani en düşük maliyet ayarında bile bcrypt, MD5'ten ~1350 kat yavaş — ve bu kasıtlı, bcrypt'in var oluş amacı brute-force'u pahalılaştırmak. Gerçek sistemlerde cost genelde 10-12 kullanılır, her +1 maliyeti ikiye katladığı için bu fark katbekat büyür.

## Özet — Bu projede öğrendiklerim

- GIL'in CPU-bound işlerde neden `threading`'i işe yaramaz kıldığı ve `multiprocessing`'in bunu nasıl bypass ettiği.
- İş bölüşümünün naif bir şekilde yapılması (tüm uzayı üretip `idx % num_workers` ile filtrelemek) gizli bir performans maliyeti getirebiliyor — kombinasyon uzayını doğru bölüştürmek (ilk karaktere göre) gerçek bir hızlanma farkı yarattı.
- SMT/hyperthreading'in CPU-bound iş yüklerinde fiziksel çekirdek sayısının ötesinde pek fayda sağlamadığı — ölçümle doğruladım.
- bcrypt gibi kasıtlı yavaş tasarlanmış hash fonksiyonlarının parola saklama için neden MD5/SHA'dan çok daha güvenli olduğu.

---

Kişisel eğitim/CTF projesi — kaynak kod: [github.com/efeavcii2/hash-cracker](https://github.com/efeavcii2/hash-cracker). Yalnızca kendi sahip olduğun veya izinli hash'lerde kullan; araç tamamen offline çalışır.
