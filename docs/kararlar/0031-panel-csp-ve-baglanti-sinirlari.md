# ADR-0031: Panel içerik güvenlik politikası (CSP) ve bağlantı / istek sınırları

- **Durum:** Kabul edildi
- **Tarih:** 2026-11-07
- **İlgili:** [ADR-0022](0022-uretim-dagitimi-ucretsiz-katman.md) (üretim dağıtımı, Caddy), [ADR-0018](0018-harita-karo-ve-stil-sunumu.md) (harita aynı kökenden); güvenlik denetimi 2026-11 B-06 (P0 kısmı) ve B-07

## Bağlam

Denetim iki açık buldu. (B-06) Panel yanıtlarında CSP yok; panel refresh token'ı `localStorage`'da tutar, yani tek bir XSS 30 günlük yönetici oturumunu çaldırır. (B-07) `/ws`'e IP başına sınırsız bağlantı açılabilir (kimliksiz bağlantı 5 sn yaşar, her biri bellek ve zamanlayıcı tutar), `/health` her çağrıda veritabanına gider ve Caddy'de başlık / gövde sınırı yoktur; 1 GB bellekli tek VPS'te tek makine tüm firmaları etkileyebilir. Kapsam dışı: refresh token'ın HttpOnly çereze taşınması (B-06 P1, sözleşme değişikliği, ayrı karar).

## Karar

**1. Panel CSP (Caddy, yalnızca panel / SPA yanıtları).** `(panel_csp)` parçacığı, denetimin önerdiği politikayı **hiç sapma olmadan** yayar:

`default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; font-src 'self' data:; connect-src 'self'; worker-src 'self' blob:; child-src blob:; object-src 'none'; base-uri 'self'; form-action 'self'; frame-ancestors 'none'`

| Yönerge | Neden |
|---|---|
| `script-src 'self'` | Vite derlemesi satır içi betik üretmez (`index.html` yalnızca `<script type="module" src>`); `eval` yok (zorunlu kipte `eval('1')` engelleniyor, denendi). Betik enjeksiyonu (XSS'in yolu) kapanır |
| `style-src ... 'unsafe-inline'` | Mantine ve MapLibre satır içi `style` yazar. `'unsafe-inline'` yalnızca stilde (testle sabit) |
| `worker-src/child-src blob:` | MapLibre çalışanları `blob:` ile açılır (denendi) |
| `img-src data: blob:`, `font-src data:` | QR kodu (`data:`), harita görselleri; yazı tipleri kendi kökenimizden (`@fontsource`) |
| `connect-src 'self'` | API, harita stili / karo / glyph / sprite (`/api/v1/map/...`, PMTiles Range) ve `/ws` aynı kökenden. Modern tarayıcılarda `'self'` `ws:`/`wss:` aynı kökeni de kapsar (WebSocket denendi) |
| `frame-ancestors 'none'`, `object-src 'none'`, `base-uri`, `form-action` | Tıklama hırsızlığı, eklenti, `<base>` ve form yönlendirme kapatılır |

CSP **yalnızca panel bloğuna** eklenir; API (JSON) ve `/indir/` (APK sayfası, kendi satır içi betiği olabilir) almaz (testle sabit). Başlık adı `import panel_csp <başlık>` satırındaki tek parametredir: bugün `Content-Security-Policy-Report-Only`; pilot haftası temizse **aynı satırda** `Content-Security-Policy` yapılır (CALISMA-KITABI §14). Rapor uç noktası (`report-uri`) **eklenmedi**: Caddy gövdeleri loglamaz, ücretsiz yerel bir alıcı yoktur; ihlaller pilot haftasında tarayıcı konsolunda (`[Report Only]`) okunur.

*Doğrulama:* gerçek `apps/admin-web` derlemesi, gerçek `Caddyfile` ile (yalnızca bu denemede yerel API'ye yönlendirilmiş geçici Caddy konteyneri) **zorunlu kipte** gezildi: giriş, Canlı harita, Paketler, Havuz, Devirler, İşletmeler, Adres defteri, Kuryeler, Rota geri bildirimleri, Metrikler, Ayarlar ve `/isletme` sayfası; WebSocket "Bağlı". Tek ihlal geliştirme yedek stili `https://tiles.openfreemap.org/...` idi: yalnızca yayınlanmış harita sürümü **yokken** ve **geliştirmede** kullanılır (üretimde API yedeğe düşmez, stil ucu 503 verir; `MAP_STYLE_URL=self` zorunlu). Üretimde bu yedeğe izin vermek için CSP gevşetilmez. Başka ihlal yok; HTML'e dokunmak gerekmedi.

**2. WebSocket bağlantı sınırları (`RealtimeGateway`).**

- **IP başına 20 eşzamanlı kimliksiz bağlantı.** 21. bağlantı sayaca girmeden `4029 TOO_MANY_CONNECTIONS` ile kapatılır (zamanlayıcı kurulmaz). Bağlantı kapanınca veya `auth` olunca sayaç düşer. IP: `TRUST_PROXY=true` iken `X-Forwarded-For` başı (Caddy istemciden gelen değeri ezip kendi gördüğü adresi yazar; Fastify ile aynı kural), aksi halde soket adresi.
- **Kullanıcı başına 10 kimlikli bağlantı.** 11. `auth.ok`'ta **en eski** bağlantı `4030 CONNECTION_LIMIT` ile kapatılır, yenisi kalır (ele geçirilmiş jeton / hatalı yeniden bağlanma döngüsü bellek biriktiremez; yarı kopuk eski bağlantı yenisini engellemez). Sayaçlar `handleDisconnect`'te temizlenir.
- **Çok örnekli not:** sayaçlar API örneği başına bellektedir (Redis'e taşınmadı). Tek VPS / tek API konteyneri bugünkü dağıtımdır; ikinci örnekte sınırlar örnek başına geçerli olur (en kötü durumda örnek sayısı kadar). Sıkı genel sınır gerekirse Redis sayacına geçilir (geri uyumlu).
- **İstemci davranışı ve sözleşme:** `4029` / `4030` `packages/contracts/websocket-events.md` §3 tablosuna eklendi: istemci **en az 60 sn bekleyip** yeniden bağlanır (hemen bağlanmak başka bir bağlantıyı düşürür; `auth.ok` geri çekilme sayacını sıfırladığı için normal geri çekilme sekmeler arasında 1-2 sn'lik düşürme döngüsü yaratırdı). Panel `wsClient.ts` bunu uygular (`limitRetryMs`, varsayılan 60 sn) ve 4030'da "başka sekmede / cihazda açık" uyarısı gösterir. **Mobil istemci** (`courier-mobile/src/realtime/socket.ts`) bu PR'da değiştirilmedi (başka çalışma alanı); 4029/4030'da şimdilik normal geri çekilmeyle yeniden bağlanır, aynı davranışı sonraki PR'da alır. Bir kurye = bir bağlantı olduğundan sınıra pratikte yaklaşmaz.
- **Neden 10:** kullanıcı başına 5 sınırında 6 panel sekmesi birbirini sürekli düşürürdü; 10, makul çok sekmeli kullanımın üstünde, tek bir jetonun bellek tüketimini ise sınırlı tutar.

**3. `GET /health`.** IP başına 60/dk hız sınırı kuralı (`ruleFor`, anahtar `health`). Bağımlılık denetimi (DB + Redis) 5 sn önbelleklidir (eşzamanlı çağrılar tek sorguyu paylaşır; sağlıksız sonuç yalnızca 1 sn tutulur ki toparlanma geç görünmesin). **Muafiyet:** `TRUST_PROXY=true` iken `X-Forwarded-For` taşımayan sağlık çağrısı sınırlanmaz: bu, Caddy'nin kendi etkin sağlık denetimidir (`health_uri`, 2 sn'de bir = 30/dk); internetten gelen her istek Caddy'den geçip başlığı taşır, istemci bunu atlayamaz. Sınırlansaydı Caddy API'yi "sağlıksız" sayıp tüm trafiğe 503 verebilirdi. (Uptime Kuma 60 sn aralıkla çağırır, kuralın çok altındadır.)

**4. Caddy sınırları.** `servers { max_header_size 16KB }` (varsayılan 1 MB; aşan istek 431). `request_body max_size`: `/api/v1/*` için **1 MB** (API'nin kendi sınırı 256 KB; Caddy daha erken keser, 413), `/ws` için 1 KB, panel ve `/indir/` için 8 KB (yalnızca GET). Bugün dosya yükleme yoktur; eklenirse yalnızca ilgili yola ayrı `request_body` verilir.

## Sonuçlar

- XSS'in betik yürütme yolu kapanır (zorunlu kipte); refresh token hâlâ `localStorage`'dadır, yani bu karar riski **azaltır**, kapatmaz (P1 açık).
- Report-Only süresince koruma yoktur, yalnızca görünürlük vardır; zorunlu geçiş ürün sahibinin bir haftalık temiz konsol gözleminden sonra tek satırlık değişikliktir.
- Yan etki: yeni bir dış kaynak (CDN yazı tipi, üçüncü taraf harita) eklenirse zorunlu kipte engellenir; bilinçli olarak politika değişikliği ve bu ADR'nin güncellenmesini gerektirir.
- Sayaçlar örnek başınadır; çok örnekli kurulumda gevşer.
- Testler: `services/api/test/security-limits.e2e-spec.ts` (9 test), `apps/admin-web/src/security/securityHeaders.test.ts` (7 test), `npm run security` içinde. `caddy validate` ve `caddy adapt` geçti; canlı Caddy ile 413 (gövde), 431 (başlık) ve `/ws` 101 el sıkışması doğrulandı.

## Alternatifler

- **Doğrudan zorunlu CSP:** gerçek kullanıcıda gözlemlenmemiş politika paneli sessizce bozabilir; önce Report-Only (denetimin önerisi).
- **CSP'de nonce / hash ile `style-src`'den `'unsafe-inline'`'ı kaldırmak:** Mantine ve MapLibre çalışma anında stil yazar; nonce taşımak sunucu tarafı HTML üretimi gerektirir (panel statik). Betik kapalı olduğu sürece stil enjeksiyonunun etkisi sınırlıdır; ileride değerlendirilir.
- **Rapor alıcısı (`report-uri`):** Caddy gövde loglamaz; ücretsiz alıcı için ek servis gerekir (Kural 10). Konsol gözlemi pilot ölçeğinde yeterli.
- **Bağlantı sayaçlarını Redis'te tutmak:** doğru çok örnekli sınır, ama her bağlantıda Redis gidiş-dönüşü ve kapanışta temizleme hatalarında sızıntı; tek örnekli bugünkü dağıtım için gereksiz.
- **`/health` için sınır yerine yalnızca önbellek:** önbellek DB yükünü çözer ama CPU / günlük gürültüsünü değil; ikisi birlikte ucuzdur.
