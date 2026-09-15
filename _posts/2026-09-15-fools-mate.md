---
title: "Fool's Mate — TryHackMe Client-Side Validation Bypass"
date: 2026-09-15 20:15:00 +0300
categories: [TryHackMe]
tags: [client-side-validation, api-abuse, request-forgery, web, tryhackme]
image: /assets/img/posts/fools-mate.png
---

> TryHackMe · Web Exploitation / Client-Side Bypass · Easy
{: .prompt-info }

Şah mat bir hamle uzakta, tahta bunu biliyor, motor bunu biliyor — ama uygulama seni o hamleyi oynamaktan fiziksel olarak alıkoyuyor. Sahte bir "güvenlik" katmanı devreye giriyor ve tarayıcıda bir uyarı fırlatıp taşı eski yerine geri koyuyor. Sorun şu ki bu kontrol tamamen client-side; sunucu hamlenin nereden geldiğine hiç bakmıyor.

| Hedef | Zorluk | Kategori |
|---|---|---|
| Fool's Mate (EndgameTrainer) | Easy | Web Exploitation / Client-Side Bypass |

**Zincir:** Tahtayı oku → tek kazanan hamle Ra1-a8 (back-rank mat) → hamleyi denediğinde tarayıcı içi bir modal engelliyor ve taşı geri alıyor → engel network gecikmesi olmadan anında tetiklendiği için doğrulamanın local/JS tarafında koştuğu belli → sayfa kaynağından `js/app.js` bootstrapper'ını bul → `curl` ile dosyayı çek → `preMoveCheck()` fonksiyonu hamleyi simüle edip mat oluyorsa engelliyor, ama gerçek hamleyi sunucuya gönderen ayrı bir `sendMove()` fonksiyonu `/api/move` adında korumasız bir REST endpoint'ine POST atıyor → `type="module"` olduğu için konsoldan fonksiyona direkt erişilemiyor, ama endpoint'e tarayıcıyı hiç kullanmadan `curl` ile aynı JSON gövdesini atmak yeterli → sunucu hamleyi kör güveniyle işliyor, mat + flag.

## 01 — Tahtayı okumak

Hedefte (`http://10.82.174.183`) çalışan **EndgameTrainer** uygulaması "Mate-in-one · White to move" etiketiyle bir satranç pozisyonu sunuyor:

![EndgameTrainer arayüzü — beyaz sırada, a1'deki kale a8'e back-rank mat için hazır](/assets/img/posts/foolsmate-board.png)

Taş dizilimi net:

- **Siyah Şah:** g8'de, kendi piyonları (f7, g7, h7) tarafından kuşatılmış — kaçış karesi yok.
- **Beyaz Kale:** açık a-sütununda, a1'de.

Kaleyi a1'den a8'e sürmek klasik bir back-rank mat veriyor. Ama tahtada bu hamleyi sürüklemeye ya da tıklamaya çalıştığında uygulama hamleyi tamamlatmıyor: bir sistem modal'ı beliriyor —

> "I'll shut down your PC if you play that."

— ve uyarı kapatıldığında kale otomatik olarak a1'e geri dönüyor. Engel hiçbir network gecikmesi olmadan anında geldiği için doğrulamanın sunucuya sormadan tarayıcı içinde, yerel olarak yapıldığı açık.

## 02 — Client-side kaynağı incelemek

Sayfa kaynağında (`view-source` / DevTools) oyun mantığını süren script görülüyor:

```html
<script type="module" src="js/app.js"></script>
```

Tarayıcıya indirilen her JS dosyası gibi bu da public bir asset — doğrudan çekilebilir:

```console
$ curl -s http://10.82.174.183/js/app.js
```

Dosyada engelleyici fonksiyon net şekilde ortaya çıkıyor:

```javascript
function preMoveCheck(from, to, promotion) {
  const probe = new Chess(game.fen());
  let result;
  try {
    result = probe.move({ from, to, promotion: promotion || undefined });
  } catch (e) {
    result = null;
  }
  if (result && probe.isCheckmate()) {
    showSystemNotice("I'll shut down your PC if you play that.");
    return false;
  }
  return true;
}
```

`preMoveCheck`, hamleni gizli bir board kopyasında simüle ediyor; sonuç mat ise `false` dönüp uyarıyı tetikliyor ve hamleyi sunucuya hiç göndermiyor. Yani engel, ağa hiç çıkmadan tamamen tarayıcı içinde durduruluyor.

## 03 — Gerçek hamlenin gittiği yer: `/api/move`

`app.js` içinde biraz daha aşağıda, gerçek hamleleri sunucuya ileten fonksiyon var:

```javascript
async function sendMove(from, to, promotion) {
  locked = true;
  let data;
  try {
    const res = await fetch('/api/move', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ from, to, promotion: promotion || undefined })
    });
    data = await res.json();
  } catch (e) { /* ... */ }

  finalize(data);
}
```

Buradan iki şey çıkıyor:

- **Konsoldan direkt çağrı yok:** Script `type="module"` ile yüklendiği için `sendMove` global scope'ta değil — DevTools konsoluna `sendMove('a1','a8')` yazmak işe yaramıyor.
- **Korumasız API:** Gerçek hamle mantığı `/api/move` adında düz bir REST endpoint'inde. Endpoint sadece `from`/`to` içeren basit bir JSON gövdesi bekliyor.

> Sunucu, isteğin tarayıcıdaki `preMoveCheck` engelinden geçip geçmediğini hiç sormuyor — gelen her `/api/move` isteğine kör güveniyor. Client-side kontrol UX içindir, güvenlik sınırı değildir.
{: .prompt-tip }

## 04 — Tarayıcıyı atlayıp API'ye direkt yazmak

Tarayıcıyı tamamen devre dışı bırakıp, kazanan hamleyi `curl` ile doğrudan endpoint'e gönderdim:

```console
$ curl -X POST http://10.82.174.183/api/move \
     -H "Content-Type: application/json" \
     -d '{"from":"a1","to":"a8"}'
```

Sunucu, `preMoveCheck` engelini hiç görmediği için hamleyi normal şekilde işliyor, mat durumunu kaydediyor ve flag'i dönüyor:

![curl ile /api/move'a doğrudan POST — checkmate durumu ve flag (blurlanmış)](/assets/img/posts/foolsmate-flag.png)

## Özet — Öğrenilen teknikler

- **Client-side doğrulama bir güvenlik sınırı değildir.** `preMoveCheck` sadece UX için var; sunucu aynı kontrolü tekrar etmediği sürece saldırgan onu hiç görmeden geçebilir.
- **`type="module"` konsol erişimini engeller ama korumaz.** Fonksiyon scope'u kapalı olsa da dosyanın kendisi public bir asset — `curl` ile okunup analiz edilebilir.
- **Gerçek işi yapan endpoint'i bulmak yeterli.** UI'ı hiç kullanmadan, sadece istemcinin gönderdiği isteği taklit ederek (`/api/move` + `{from, to}`) tüm iş mantığı bypass edildi.
- **Sunucu her zaman kendi state'ini doğrulamalı.** İstek "meşru" bir kullanıcı arayüzünden mi yoksa `curl`'den mi geldiğini ayırt edemiyorsa, iş kuralları (burada: hangi hamlenin "izinli" olduğu) hiç var olmamış demektir.

---

Kişisel TryHackMe lab notu — Fool's Mate, eğitim/lab ortamı. Flag değeri platform politikası gereği bu belgede paylaşılmamıştır.
