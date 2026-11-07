# ADR-0030: Kimlik sertleştirmesi: kimlik + IP giriş kilidi, şifre kuralı ve ilk girişte değiştirme

- **Durum:** Kabul edildi
- **Tarih:** 2026-11-07
- **İlgili:** Güvenlik denetimi Kasım 2026 B-01, B-04, B-05; [ADR-0012](0012-rls-iterasyon-2b-ye-ertelendi.md) (izolasyon); sözleşme 0.6.1

## Bağlam

Kasım 2026 güvenlik denetimi iki orta önemde bulgu buldu ve ürün sahibi önerileri olduğu gibi onayladı:

- **B-04:** Hatalı giriş sayacı yalnızca kimliğe (telefon / e-posta) bağlıydı: 15 dk'da 5 hata hesabı 15 dk kilitliyordu. Kuryenin telefonunu bilen biri (işletme, müşteri) döngüyle kuryeyi vardiya başında süresiz dışarıda bırakabilirdi. Açık oturum refresh ile sürdüğü için para / veri kaybı yoktu, iş sürekliliği riskiydi.
- **B-05:** Kurye şifresi en az 6 karakterdi, yaygın şifre engeli ve ilk girişte değiştirme yoktu; tohum ve fikstürlerde `123456`. B-01 (kodlanmış yolla hız sınırı atlama) kapandıktan sonra bile çok IP'li püskürtme `123456`, `111111` gibi şifreleri bulur; kurye hesabı paket adreslerini ve müşteri telefonlarını görür.

Kısıtlar: ücretli servis yok (dış şifre sızıntı API'si, CAPTCHA yok); kuryeler eldivenle, hızlı giriş yapar (kullanılabilirlik); var olan hesaplar zorla sıfırlanmamalı (pilot öncesi kesinti istenmiyor).

## Karar

**1. Giriş kilidi üç katman (Redis, `services/api/src/auth/auth.service.ts` `LOGIN_LOCK`).**

| Katman | Anahtar | Kural |
|---|---|---|
| Kimlik + IP | `rl:login-fail:<kimlik>\|ip:<ip>` | 15 dk içinde 5 hata -> o IP'den o hesaba 15 dk 429 `RATE_LIMITED` (`Retry-After`). Başka IP'den doğru şifre girer. Başarılı girişte bu sayaç sıfırlanır. |
| Kimlik başına yüksek eşik | `rl:login-fail-id:<kimlik>` (1 sa) + `rl:login-lock:<kimlik>` (15 dk) | Tüm IP'lerden 1 saatte 50 hata -> hesaba 15 dk kilit; kilit konunca sayaç sıfırlanır (her kilit yeniden 50 hata ister). Başarılı giriş bu sayacı sıfırlamaz (saldırgan doğru şifreli bir girişi beklemesin). |
| Artan gecikme | kimlik + IP sayacı | 3. hatadan sonra her hatalı yanıt +1 sn geç döner, en çok 5 sn. Doğru şifreli giriş gecikmez. |

- Sayaçlar `MULTI` ile artırılır ve süre `EXPIRE ... NX` ile yalnızca ilk kez konur (yarış yok, süresiz anahtar kalmaz). Redis erişilemezse eski davranış: kilit denetlenemez, giriş çalışır.
- IP, Fastify `req.ip`'dir: üretimde `TRUST_PROXY=true` ve Caddy `X-Forwarded-For`'u ezer (B-20 kurulum denetimi). IP okunamazsa tek kova (`unknown`).
- Kilit yalnızca girişi etkiler; refresh, çıkış ve açık oturum sürer. Günlük / denetim olayı değişmedi (yeni kişisel veri yok; Redis anahtarında telefon zaten vardı).
- `POST /me/password` mevcut şifre denemesini aynı `kimlik + IP` sayacına yazar (`user:<id>`): çalınmış erişim jetonuyla şifre tahmini de 5 denemede durur.

**2. Şifre kuralı (`services/api/src/auth/password-policy.ts`).** Yeni belirlenen her şifre (kurye oluşturma, panelden sıfırlama, kuryenin kendi değişikliği, panel kullanıcısı oluşturma / güncelleme):

- en az 8, en çok 128 karakter;
- gömülü yaygın şifre listesinde değil (`common-passwords.txt`, 1918 girdi, ASCII, depoda; kamuya açık sızıntı derlemelerindeki en sık şifrelerin 8+ karakterli kalıpları + Türkiye'ye özgü kökler: kulüp + kuruluş yılı, şehir + plaka, `sifre` / `parola` türevleri; elle derlendi, dosya indirilmedi). Karşılaştırma büyük / küçük harf ve Türkçe harf duyarsız; sondaki noktalama (`Galatasaray1905!`) ve kısa tekrarlar (`aaaaaaaa`, `12121212`, `abcabcabc`) da reddedilir;
- kuryenin telefon numarasını (10 haneli ulusal kısım; boşluk / tire ile yazılmış hali dahil) içermez;
- kendi değişikliğinde mevcut şifreden farklı.

İhlal 400 `VALIDATION_FAILED`, alan bazlı Türkçe mesaj (`password` / `newPassword` / `currentPassword`). **Var olan şifreler girişte denetlenmez**: kısa şifreli eski kuryeler girmeye devam eder, zorla sıfırlama yoktur.

**3. Geçici şifre ve ilk girişte değiştirme.** Panelin verdiği her şifre (oluşturma ve sıfırlama) geçicidir: `users.must_change_password = true` (migration 0008, varsayılan false; eski hesaplar etkilenmez). `Me.mustChangePassword` (isteğe bağlı alan, API her zaman gönderir) true iken `AuthGuard` yalnızca `@AllowPendingPasswordChange()` işaretli uçlara izin verir: `GET /me`, `POST /me/password`; `/auth/*` zaten herkese açıktır. Diğer tüm uçlar 403 `PASSWORD_CHANGE_REQUIRED`. Bayrak kullanıcı durumu önbelleğinde (30 sn) taşınır; sıfırlama / değişiklik önbelleği siler.

`POST /me/password` (yalnızca COURIER): mevcut + yeni şifre; başarıda bayrak kalkar, kullanıcının **tüm** refresh token'ları iptal edilir ve yeni bir token çifti döner (`LoginResponse`).

**4. Kurye uygulaması.** Oturum durumu `password`: girişte ya da açılışta `mustChangePassword` görülürse ya da herhangi bir uç 403 `PASSWORD_CHANGE_REQUIRED` dönerse uygulama yalnızca **"Yeni şifre belirleyin"** ekranını açar (büyük alanlar, kısa A2 düzeyi metin; geçici şifre, yeni şifre, tekrar). Bu durumda hiçbir sorgu ve WebSocket açılmaz (`useLoggedIn` yalnızca `in`). Cihaz uzunluk / tekrar / telefon / eşleşmeyi önceden denetler; yaygın liste yalnızca sunucudadır, mesajı alanın altında gösterilir. Çevrimdışı eylem kuyruğu bu hatayı alırsa kaydı atmaz, 30 sn sonra yeniden dener.

**5. Panel.** Kurye oluşturma ve şifre sıfırlama formları 8 karakteri ister, kuralı ve "Kurye bu şifreyle ilk girişte kendi şifresini belirleyecek" notunu gösterir; sunucunun kural hatası alanın altında görünür.

**6. Demo verisi.** Kurgusal demo kurye şifresi `KuryeDemo26` (`SEED_COURIER_PASSWORD`, en az 8); tohum kuryeleri bayraksızdır (demo akışı değişmez). Panel demo şifresi `Demo1234!` değişmedi (tohum kuralı atlar; yaygın listede olduğu için panelden aynı şifre yeniden verilemez).

## Sonuçlar

- (+) Telefon numarasını bilen yabancı kuryeyi dışarıda bırakamaz: kendi IP'sini kilitler; hesabı kilitlemek için saatte 50 farklı denemesi gerekir (IP başına hız sınırı 30/dk ile birlikte birden çok IP).
- (+) Dağıtık püskürtme yavaşlar ve yaygın şifreler yeni hesaplarda yoktur; panel geçici şifreyi bilse bile kuryenin kalıcı şifresini bilmez.
- (+) Sözleşme eklemelidir (0.6.1): eski uygulama sürümleri bayrağı tanımaz: yeni / sıfırlanan hesapla girince "Profil yüklenemedi" görür (403), veri açılmaz; mevcut hesaplar etkilenmez.
- (−) Kimlik başına eşik yine bir kilit yoludur (dağıtık, 50 deneme / 15 dk): risk kabul edildi, açık oturum etkilenmez. Tam çözüm (cihaz bağlama, "bilinen cihaz" istisnası) ayrı iş.
- (−) Yaygın liste sabit ve küçüktür (sızıntı API'si ücretli / dış servis); yılda bir denetimde gözden geçirilir.
- (+) WebSocket de aynı kuralı uygular: `mustChangePassword` true iken kimlik doğrulaması `4005 PASSWORD_CHANGE_REQUIRED` ile kapanır (`websocket-events.md`); uygulama yeniden bağlanmaz, şifre ekranına geçer.
- (−) Eski APK'lar `POST /me/password` ekranına sahip değildir: geçici şifreli yeni kurye yeni APK'yı kurmalıdır (pilotta tek APK dağıtımı).

## Alternatifler

- **Yalnızca IP'ye göre kilit:** botnet her denemeyi başka IP'den yapar; hesap başına koruma kalmaz. Reddedildi (yüksek eşik eklendi).
- **CAPTCHA / SMS doğrulama:** ücretli veya dış servis; eldivenli kullanımda zor. Reddedildi (ürün kuralı).
- **Have I Been Pwned k-anonimlik API'si:** ücretsiz ama dış servise bağımlılık ve üretimde dış ağ çağrısı; gömülü liste yeterli görüldü.
- **Tüm hesaplarda zorunlu sıfırlama:** pilot öncesi kesinti; ürün sahibi istemedi. Yalnızca panelin yeni verdiği şifreler geçici sayıldı.
- **Sabit gecikme yerine üstel gecikme:** 5 hatada IP kilidi zaten devreye girer; üstel gecikme bağlantıları uzun süre tutar (VPS kaynak riski, B-07). Doğrusal + tavan seçildi.
