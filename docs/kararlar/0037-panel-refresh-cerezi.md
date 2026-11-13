# ADR-0037: Panel refresh token'ı HttpOnly çerezde (B-06 P1)

- **Durum:** Kabul edildi
- **Tarih:** 2026-11-13
- **İlgili:** [ADR-0030](0030-kimlik-sertlestirme.md) (giriş kilidi, şifre kuralı, zorunlu şifre değişikliği), [ADR-0031](0031-panel-csp-ve-baglanti-sinirlari.md) (panel CSP, B-06 P0), [ADR-0022](0022-uretim-dagitimi-ucretsiz-katman.md) (Caddy, tek köken); güvenlik denetimi 2026-11 B-06 (P1 kısmı); sözleşme **0.7.0**

## Bağlam

Panel 30 günlük refresh token'ı `localStorage`'da tutuyordu (`apps/admin-web/src/api/tokens.ts`). Panelde tek bir XSS (bağımlılık açığı, ileride eklenecek bir `dangerouslySetInnerHTML`) bu anahtarı okuyup **başka bir makineye taşıyabilir** ve 30 gün boyunca yönetici gibi davranabilirdi (tüm müşteri / kurye verisi, ayarlar). ADR-0031'deki CSP (bugün Report-Only) XSS'i zorlaştırır ama anahtarın JS'e açık olması sorununu kapatmaz.

Kısıtlar: kurye uygulaması (Expo, SecureStore) gövde akışını kullanır ve **değişmemeli**; rotasyon + 30 sn tolerans + yeniden kullanım tespiti (aile iptali) aynen sürmeli; ADR-0030 kilidi ve zorunlu şifre değişikliği bozulmamalı; geliştirmede Vite proxy (aynı köken, http), üretimde Caddy (aynı köken, HTTPS) çalışmalı; ücretli servis yok.

## Karar

**1. Ayrı web uçları (kip seçimi açık, kazara karışmaz).** Panel yalnızca yeni uçları kullanır:

| Uç | Gövde | Yanıt |
|---|---|---|
| `POST /auth/web/login` | `LoginRequest` (e-posta + parola, yalnızca OWNER / DISPATCHER) | `WebLoginResponse { session, user }` + `Set-Cookie` |
| `POST /auth/web/refresh` | yok (çerez) | `WebSession` + yeni `Set-Cookie` (rotasyon); 401'de çerez silinir |
| `POST /auth/web/logout` | yok (çerez) | 204; aile sunucuda iptal, çerez silinir |

`WebSession` = `{ tokenType, accessToken, accessTokenExpiresAt, refreshTokenExpiresAt }`: **gövdede refresh token yoktur** (`additionalProperties: false`, testle sabit). Gövde uçları (`/auth/login`, `/auth/courier/login`, `/auth/refresh`, `/auth/logout`) **çerez okumaz ve yazmaz**; web uçları gövdedeki refresh token'ı okumaz. Böylece çerezi taşıyan bir tarayıcı, hiçbir uçtan refresh token'ı JS'e açık biçimde alamaz. Kimlik doğrulama, kilit, rotasyon ve yeniden kullanım tespiti tek yerdedir (`AuthService`); web uçları yalnızca taşıma katmanıdır.

**2. Çerez.** `__Secure-kurye_rt=<opak>; Path=/api/v1/auth/web; Max-Age=<refresh bitişine kalan sn>; HttpOnly; SameSite=Strict; Secure`.

- `HttpOnly`: JS okuyamaz. `Path=/api/v1/auth/web`: çerez yalnızca giriş / yenileme / çıkışa gider, diğer API isteklerine ve loglara binmez. `Domain` yok: yalnızca paneli sunan ana makine (alt alan adlarına sızmaz).
- `Secure` + `__Secure-` öneki üretimde zorunlu: `AUTH_COOKIE_SECURE` boşsa `NODE_ENV=production`'da açık, diğer ortamlarda kapalı (yerel http; ad `kurye_rt`). Üretimde `false` verilirse API başlamaz. Sunucu yalnızca yapılandırılmış adı okur.
- `Max-Age` refresh token satırının `expires_at`'i ile aynıdır (`REFRESH_TOKEN_TTL_DAYS`, 30 gün); her rotasyonda yeniden yazılır (kayan pencere bugünkü davranışla aynı).
- **`SameSite=Strict` (Lax değil):** panel ve API aynı kökendedir (Caddy, Vite proxy); panele dışarıdan bağlantıyla gelinse bile oturum çerezle değil, sayfa açıldıktan sonra yapılan `fetch` ile yüklenir ve o istek aynı sitedendir. Strict'in bilinen bedeli (dış bağlantıyla gelen ilk gezinmede çerez gitmemesi) burada yoktur. Lax yalnızca API farklı bir **siteye** taşınırsa gerekir, o da ayrı karar.

**3. CSRF.** Çerezle kimlik doğrulanan her web ucu (giriş dahil: oturum sabitleme / "kurbanı saldırganın hesabıyla girdirme" de kapanır):

1. `X-Kurye-Client: web` başlığı zorunlu. HTML formu, `<img>`, basit `fetch` bu başlığı gönderemez; başka köken için tarayıcı önce CORS önkontrolü yapar ve API önkontrolü yalnızca `CORS_ORIGINS` kökenlerine geçirir.
2. `Origin` varsa ya `CORS_ORIGINS` içinde ya da isteğin kendi kökeni (`protocol://host`, `TRUST_PROXY` ile Caddy'nin `X-Forwarded-*`'ı) olmalı; `null` reddedilir.
3. `Sec-Fetch-Site: cross-site` ancak izinli `Origin` ile geçer (Origin göndermeyen eski tarayıcılar için ek savunma).

Hepsi 403 `CSRF_REJECTED` (yeni hata kodu). `SameSite=Strict` zaten başka siteden çerez göndermez; özel başlık + Origin, aynı sitedeki başka bir alt alan adından (ör. ele geçirilmiş `*.example.com`) gelecek isteğe karşı ikinci katmandır. Çift gönderimli (double-submit) CSRF jetonu **seçilmedi**: JS'e açık bir jeton saklamayı gerektirir ve aynı kökenli API'de bu iki katmana bir şey eklemez.

**4. CORS.** `credentials: true`, köken bir fonksiyonla yalnızca `CORS_ORIGINS` listesinden yansıtılır; izinsiz kökene (ve `Origin`'siz isteğe) hiçbir CORS başlığı, `Access-Control-Allow-Credentials` dahil, yazılmaz. `CORS_ORIGINS` içinde `*` olursa API başlamaz. `X-Kurye-Client` izinli başlıklara eklendi.

**5. Panel (admin-web).** Access token yalnızca bellekte (`tokenStore`); `localStorage` / `sessionStorage`'a hiçbir belirteç yazılmaz. Açılışta `POST /auth/web/refresh` sessizce denenir (çerez yoksa uyarısız giriş ekranı), sonra `/me`. 401 `TOKEN_EXPIRED` → tek uçuşlu yenileme → isteği tekrar; jetonsuz gönderilmiş isteğin 401'i yenileme tetiklemez. Oturum uçları `credentials: 'include'` ile çağrılır (API ayrı alt alan adına taşınırsa da çalışsın). **Çok sekme:** çerez ortak, access token sekme başına. Eşzamanlı yenilemeler sunucudaki 30 sn toleransla çakışmaz. `BroadcastChannel('kurye.auth')` ile çıkış diğer sekmeleri hemen giriş ekranına döndürür, başka sekmede giriş anonim sekmeyi çerezden yükler, **farklı hesapla** giriş açık sekmeyi kapatır (aksi halde ortak çerez o sekmeyi sessizce başka hesaba geçirirdi). Kanalda belirteç taşınmaz.

**6. Mevcut panel oturumlarının geçişi: temiz çıkış (taşıma yok).** Açılışta `localStorage` (ve olası `sessionStorage`) `kurye.refreshToken` okunur, **hemen silinir** ve gövde akışıyla `POST /auth/logout` çağrılarak ailesi sunucuda iptal edilir; çereze **çevrilmez**. Kullanıcı bir kez yeniden girer ("Güvenlik güncellemesi nedeniyle lütfen bir kez yeniden giriş yapın."). Neden: bu anahtar 30 güne kadar JS'e açıktı; çalınmışsa onu çerez oturumuna taşımak, tam da kapatmak istediğimiz açığı yeni oturuma devretmek olurdu. Bedel: birkaç yönetici bir kez yeniden giriş yapar. Aynı tarayıcıda zaten çerez oturumu varsa (ör. diğer sekmede yeniden girilmiş) eski anahtar yine iptal edilir, çerez oturumu sürer.

**7. Mobil ve ADR-0030.** Kurye uygulaması ve jest testleri değişmedi (`/auth/courier/login`, `/auth/refresh`, `/auth/logout`, SecureStore). Giriş kilidi web girişinde de aynı sayaçla çalışır (testli); hız sınırı `/auth/web/login` → `login`, `/auth/web/refresh` → `refresh` kuralı. `POST /me/password` yalnızca kurye içindir, değişmedi. WebSocket kimliği access token ile, değişmedi.

**8. Altyapı.** Caddy değişmedi (aynı köken, HTTPS; `Cookie` / `Set-Cookie` zaten logdan siliniyor; 16 KB başlık sınırı ~70 baytlık çerez için yeterli). Vite proxy değişmedi (`changeOrigin` yalnızca `Host`'u değiştirir; `Origin` = panel kökeni, `CORS_ORIGINS`'te olmalı). Çalışma notları: CALISMA-KITABI §19.

## Sonuçlar

- XSS artık 30 günlük oturumu **çalıp dışarı taşıyamaz**. Kalan risk: sayfa açıkken XSS, panel adına istek atabilir ve bellekteki access token'ı (en çok 15 dk) okuyabilir; CSP (ADR-0031, zorunlu kipe geçince) bunu daraltır.
- Sözleşme 0.7.0: yeni uçlar, `WebSession`, `WebLoginResponse`, `refreshCookie` güvenlik şeması, `X-Kurye-Client` parametresi, `RefreshCookie` başlığı, `CSRF_REJECTED`. Gövde uçları değişmedi; eski istemciler etkilenmez.
- Dağıtımda açık panel oturumları bir kez kapanır (yeniden giriş); kuryeler etkilenmez.
- Üretimde HTTPS zorunludur (zaten ADR-0022); düz http ile panelde oturum yenilemede düşer.
- Panel, API'den farklı bir **siteye** taşınırsa (başka kayıtlı alan adı) Strict yüzünden çalışmaz; aynı site / alt alan adı çalışır (`VITE_API_BASE_URL` + `CORS_ORIGINS`).
- Testler: `services/api/test/auth-web.e2e-spec.ts` (21), `src/auth/web-session.spec.ts` (7), `env.spec.ts` (+1), `security.e2e-spec.ts` (+1, herkese açık uç listesi), panel `src/security/sessionStorage.test.tsx` (9) ve güncellenen `app.test.tsx`; `npm run security` hepsini koşar.

**İleride:** ek sertleştirme adımı güvenlik denetimi belgesinde izlenir.

**Geri dönüş:** panel kodunu önceki sürüme almak yeterlidir (gövde uçları duruyor); eski panel `localStorage` akışına döner, web uçları kullanılmadan kalır. Kullanıcılar bir kez daha yeniden girer. API tarafı geri alınmak istenirse web uçları ve `CSRF_REJECTED` kaldırılır (veritabanı değişikliği yok, migration yok).

### Ürün sahibi kararı bekleyen

| Konu | Güvenli varsayılan (uygulanan) | Alternatif | Nerede değişir |
|---|---|---|---|
| Eski `localStorage` oturumları | Temiz çıkış: silinir + sunucuda iptal, bir kez yeniden giriş | Bir kerelik çereze taşıma (kullanıcı fark etmez; çalınmış anahtar da taşınır) | `apps/admin-web/src/auth/AuthProvider.tsx` açılış etkisi |
| `SameSite` | `Strict` | `Lax` (yalnızca API ayrı siteye taşınırsa gerekir; CSRF başlığı yine korur) | `services/api/src/auth/web-session.ts` `serializeRefreshCookie` |
| Refresh ömrü / çerez Max-Age | 30 gün (`REFRESH_TOKEN_TTL_DAYS`, değişmedi) | Panel için daha kısa (ör. 7 gün; ayrı ortam değişkeni gerekir) | `.env` / `issueTokens` |
| Farklı hesapla başka sekmede giriş | Açık sekme kapanır, yeniden giriş ister | Sekmeyi yeni hesaba sessizce geçirmek | `AuthProvider.tsx` `onAuthBroadcast` |
| Çerezin `Secure` olmadığı ortam | Yalnızca `NODE_ENV≠production` (yerel http) | Geliştirmede de `AUTH_COOKIE_SECURE=true` (localhost tarayıcıda güvenli sayılır) | `.env` `AUTH_COOKIE_SECURE` |

## Alternatifler

- **Refresh token'ı bellekte tutmak (depolamamak):** XSS'e karşı güvenli ama her sayfa yenilemesinde yeniden giriş ister; yönetici paneli için kabul edilemez kullanılabilirlik.
- **Aynı uçlarda istemci türü alanı (`clientType: web`) / başlığa göre kip:** tek bir uçta iki davranış; alan unutulursa ya da sahte gönderilirse tarayıcı gövdede refresh token alabilir. Ayrı uçlar kipi URL'de açık ve testle sabit kılar.
- **Gövde uçlarını çereze de duyarlı yapmak (çerez varsa çerez, yoksa gövde):** gövde ucu çerezi okursa, çerezi taşıyan tarayıcıdaki XSS onu gövdede refresh token'a çevirebilirdi. Reddedildi.
- **Çift gönderimli CSRF jetonu:** JS'e açık ikinci bir sır gerektirir; aynı kökenli düzende `SameSite=Strict` + özel başlık + Origin ile aynı korumayı daha az parçayla verir.
- **BFF / sunucu tarafı oturum (access token da çerezde, tüm API çerezle):** tüm uçlarda CSRF ve WS kimliği yeniden tasarımı; bu bulgu için orantısız. İleride gerekirse bu ADR'nin üstüne kurulabilir.
- **Eski token'ı bir kerelik çereze taşıma uç noktası:** sorunsuz geçiş verir ama muhtemelen açıkta kalmış anahtarı yeni oturuma devreder ve yeni bir gövde-kabul eden web ucu açar. Temiz çıkış seçildi (bkz. ürün sahibi tablosu).
