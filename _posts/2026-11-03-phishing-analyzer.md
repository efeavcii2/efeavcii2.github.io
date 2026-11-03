---
title: "phishing-analyzer — Kanıta Dayalı Phishing Risk Analiz Aracı"
date: 2026-11-03 18:00:00 +0300
categories: [Projeler]
tags: [python, phishing, email-security, spf-dkim-dmarc, threat-intel, cli]
image: /assets/img/posts/phishing-analyzer.svg
---

> Kişisel Proje · Python 3 · CLI Tool
{: .prompt-info }

Bir `.eml` dosyasını alıp "bu phishing mi?" sorusuna kanıta dayalı, ağırlıklı bir risk skoruyla cevap veren bir komut satırı aracı. SOC analistinin günlük phishing triyaj işini — başlık uyumsuzluğu, SPF/DKIM/DMARC sonucu, link gizleme, şüpheli domain, bilinen kötü amaçlı URL/hash kontrolü — tek bir çalıştırmada birleştiriyor.

| Dil | Girdi | Tehdit İstihbaratı | Repo |
|---|---|---|---|
| Python 3 (stdlib ağırlıklı) | `.eml` dosyası | URLhaus + VirusTotal | [GitHub ↗](https://github.com/efeavcii2/phishing-analyzer) |

**Akış:** `.eml` oku → başlık/link/dil/domain kontrolleri → URLhaus + VirusTotal sorgusu → ağırlıklı risk skoru (DÜŞÜK/ORTA/YÜKSEK) → terminal raporu

## 01 — Çekirdek fikir: görünen ile gerçek arasındaki fark

Bir phishing e-postasının **görünen** kısmı (From adı, link metni) ile **gerçek** kısmı (Return-Path, href, SPF/DKIM/DMARC sonucu) arasında hemen her zaman bir tutarsızlık var — çünkü saldırgan görüneni istediği gibi yazabilir ama altyapısını (DNS, SMTP zarf adresi) taklit etmesi çok daha zor. Katman 1'deki dört kontrolün hepsi aynı soruyu farklı açılardan soruyor: *"görünen ile gerçek aynı mı?"*

- `sender_mismatch` / `reply_to_mismatch`: From domaini ile Return-Path/Reply-To domaini
- `spf_fail` / `dkim_fail` / `dmarc_fail`: `Authentication-Results` başlığının yorumlanması
- `link_mismatch`: linkin görünen metni ("paypal.com" gibi görünen) ile gerçek `href`'in gittiği domain

## 02 — Katman 2: zenginleştirme + harici tehdit istihbaratı

- Şüpheli dil kalıpları (aciliyet, hesap tehdidi, sahte ödül — TR+EN regex)
- IP tabanlı URL tespiti (`ipaddress` stdlib modülü)
- Punycode (`xn--`) ve leetspeak/typosquat marka taklidi tespiti (`paypa1-verify.com` → `paypal` eşleşmesi)
- **URLhaus** (abuse.ch) ile link sorgusu, **VirusTotal** ile ek hash sorgusu — ikisi de `.env`'den okunan Auth-Key gerektiriyor, key yoksa ağ çağrısı hiç yapılmadan BİLGİ seviyesinde "atlandı" dönüyor
- Her bulgu ağırlıklandırılıp (YÜKSEK=10, ORTA=4, DÜŞÜK=1) tek bir risk skoruna indirgeniyor

## 03 — Gerçek dünya testi: bir false positive bulmak ve düzeltmek

Sentetik test örnekleriyle (kurgusal phishing maili) her şey yolundaydı — gerçek sorun, gerçek bir e-postayı test edince çıktı. Kendi GitHub bildirim mailimi (`notifications@github.com`, `Reply-To: reply.github.com`) analiz ettiğimde araç 1 yanlış bulgu üretti: `reply_to_mismatch`.

Sebep: GitHub, yanıtları işlemek için `reply.github.com` alt-domainini kullanıyor — From ise `github.com`. İlk versiyondaki **tam domain karşılaştırması** bunu "farklı domain" sayıyordu.

```python
def _base_domain(domain: str) -> str:
    """Son iki etiketi alir: reply.github.com -> github.com"""
    parts = domain.split(".")
    if len(parts) <= 2:
        return domain
    return ".".join(parts[-2:])
```

Düzeltme, domain karşılaştırmasını "temel domain"e indirgemekti — böylece `reply.github.com` ile `github.com` aynı sayılıyor, ama `totally-different-domain.ru` ile `paypa1-verify.com` hâlâ farklı sayılıyor. Aynı fix, link karşılaştırmasına da uygulandı (örneğin `accounts.google.com` gibi meşru alt-domain kullanımlarını da false positive saymasın diye).

> Gerçek veriyle test etmeden bu hata hiç fark edilmezdi — sentetik örnekler sadece "beklediğim senaryoyu" doğruluyor, gerçek dünyanın meşru ama alışılmadık kalıplarını (alt-domain reply adresleri gibi) değil.
{: .prompt-tip }

## 04 — Örnek çıktı

```console
$ python3 analyze.py samples/sample_phishing.eml

RISK SEVIYESI: YUKSEK (puan: 82)

BULGULAR (10)
  [YUKSEK] sender_mismatch: From domaini (paypa1-verify.com) ile Return-Path
           domaini (totally-different-domain.ru) uyusmuyor...
  [YUKSEK] spf_fail / dkim_fail / dmarc_fail: ucu de basarisiz
  [YUKSEK] link_mismatch: paypal.com gosteriyor, gercekte
           paypa1-verify-secure.ru'e gidiyor
  [YUKSEK] lookalike_domain: iki domain de "paypal" markasini taklit ediyor
  [ORTA]   reply_to_mismatch + 2x supheli dil kalibi

Ozet: YUKSEK: 7, ORTA: 3
```

Aynı araç, gerçek bir GitHub bildirimi ve gerçek bir pazarlama maili (Albion Online, SendGrid click-tracking linkleriyle) için **0 bulgu, DÜŞÜK risk** döndürüyor — yani false positive üretmiyor.

## Özet — Bu projede öğrendiklerim

- **"Görünen vs gerçek" tek bir kontrol değil, tekrar eden bir desen.** From/Return-Path, link metni/href, hatta domain/marka adı karşılaştırması — hepsi aynı ilkenin farklı uygulamaları.
- **Tam string eşleştirme, gerçek dünyada kırılgan.** Alt-domain kullanımı (reply.github.com) gibi meşru kalıplar, naif karşılaştırmaları false positive'e sürüklüyor — "temel domain"e indirgemek gerekiyor.
- **Harici API'ler bile artık ücretsiz kullanımda key istiyor.** URLhaus, yakın zamanda (abuse.ch politika değişikliğiyle) zorunlu Auth-Key'e geçti — araç bunu zarifçe (crash etmeden, BİLGİ seviyesinde) ele alacak şekilde tasarlandı.
- **Ağırlıklı skorlama, tek bir zayıf sinyali ezmeden toplu sinyali yakalıyor.** Tek başına düşük siddetli bir bulgu (eksik Authentication-Results gibi) risk seviyesini tek başına değiştirmiyor, ama birkaç sinyal üst üste binince (sender mismatch + SPF fail + lookalike domain) skor hızla YÜKSEK'e çıkıyor — gerçek SOC triyajının da çalışma mantığı.

---

Kişisel eğitim projesi — kaynak kod: [github.com/efeavcii2/phishing-analyzer](https://github.com/efeavcii2/phishing-analyzer). Araç e-postaları yalnızca pasif analiz eder, hiçbir ek/link açmaz veya takip etmez.
