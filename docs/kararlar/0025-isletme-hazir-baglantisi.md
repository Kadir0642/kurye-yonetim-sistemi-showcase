# ADR-0025: İşletme "paket hazır" bağlantısı: girişsiz, kısa ömürlü, opak jeton (URL parçasında), yalnızca o işletmenin açık paketleri

- **Durum:** Önerildi (İterasyon 3b spesifikasyonuyla birlikte onaya sunuldu)
- **Tarih:** 2026-11-06
- **İlgili:** [ADR-0004](0004-cok-kiracili-mimari.md), [ADR-0009](0009-rota-uyumlu-atama.md) (`ready_at`), [ADR-0020](0020-cevrimdisi-eylem-kuyrugu-ve-idempotency.md), [ADR-0022](0022-uretim-dagitimi-ucretsiz-katman.md) (HTTPS, sırlar), PLAN 3b-2, SPEC-3b §7.5

## Bağlam

Motorun `readyAt`'i bilmesi kuryenin restoranda beklemesini ve geç teslimi azaltır (benchmark duyarlılık koşusu). Restoranların bugün sisteme hesabı yoktur; işletme paneli İterasyon 5+'dadır. PLAN 3b-2 üçüncü sinyal kaynağı olarak "işletmeye özel imzalı, kısa ömürlü jeton, giriş yok, tek dokunuş 'Hazır'" istedi; güvenlik koşulu: jeton yalnızca o işletmenin açık paketlerini kapsar, hız sınırı vardır.

Kısıtlar: hiçbir ücretli servis yok (SMS / WhatsApp Business API kullanılamaz); restoran personeli telefondan, giriş yapmadan kullanır; bağlantı yanlış kişiye geçerse zarar "yanlış hazır sinyali" ile sınırlı kalmalı; jeton log ve tarayıcı geçmişinde iz bırakmamalı; müşteri kişisel verisi işletmeye fazladan açılmamalı.

## Karar

1. **Opak rastgele jeton**, imzalı (durumsuz) jeton değil. 32 bayt kriptografik rastgele, base64url (43 karakter). Veritabanında yalnızca `sha256(jeton)` (`business_ready_links.token_hash`, UNIQUE) saklanır; jeton yalnızca oluşturma yanıtında bir kez döner.
2. **Ömür ve iptal:** varsayılan 14 sa (kiracı ayarı `preparation.businessLinkTtlH`, 1-24), istekte en çok 24 sa. Yönetici anında iptal eder; işletme arşivlenirse veya kiracıda özellik kapatılırsa (`businessLinkEnabled = false`) tüm bağlantılar geçersizdir. İşletme başına en çok 3 etkin bağlantı.
3. **Taşıma:** bağlantı `<panel kökü>/isletme#t=<jeton>`; jeton **URL parçasında** (fragment) durur: sunucuya, ters vekil loglarına ve Referer'a gitmez. Sayfa jetonu `sessionStorage`'a alır, adres çubuğundan siler ve API'ye `X-Business-Link-Token` başlığıyla gönderir; başlık loglanmaz.
4. **Kapsam (en az yetki):** yalnızca iki girişsiz uç: o işletmenin son 12 saatteki açık paketlerini listelemek ve bir paketi "hazır" işaretlemek. Gösterilen alanlar: paket numarası, sipariş saati, aşama, hazır / tahmini varış saatleri, tutar ve ödeme türü, teslim adresinin ilk parçası (≤ 40 karakter). Müşteri adı / telefonu, adres ayrıntısı, not ve kurye adı yoktur. Başka işletmenin paketi 404.
5. **Kötüye kullanım sınırı:** jeton başına 30 / dk, IP başına 60 / dk; geçersiz / süresi dolmuş / iptal edilmiş tüm durumlar **tek tip** 401 `BUSINESS_LINK_INVALID`; `Cache-Control: no-store`, `Referrer-Policy: no-referrer`; her kullanım `last_used_at`, `use_count` ile izlenir. Olay `READY_SIGNALED`, aktör `SYSTEM`, `data.source = BUSINESS_LINK`, `data.linkId` (denetim kaydı).
6. **Dağıtım ücretsiz:** yönetici bağlantıyı panelde kopyalar veya QR kodunu gösterir (WhatsApp vb. ile elle). Otomatik SMS / WhatsApp gönderimi ücretli API gerektirir ve yapılmaz (ürün sahibi kararı, SPEC-3b §19).
7. **Üretimde yalnızca HTTPS** (P-2; ücretsiz Let's Encrypt). Geliştirmede HTTP kabul edilir.

## Sonuçlar

- (+) Restoran için sıfır sürtünme: hesap, parola, uygulama yok; tek dokunuş.
- (+) İptal ve süre doğal (her istekte veritabanı zaten okunuyor); yeni imza sırrı ve döndürme prosedürü gerekmez (P-7 envanteri küçük kalır).
- (+) Jeton sızıntısının etkisi sınırlı: en kötü hâlde yanlış "hazır" sinyali (motor kuryeyi biraz erken gönderir); paket verisi en aza indirilmiş.
- (−) Bağlantıyı elinde tutan herkes işletme adına "hazır" diyebilir (kimlik doğrulama yoktur); kullanım sayacı ve kısa ömür bunu sınırlar, tam çözüm işletme hesabıdır (İterasyon 5+).
- (−) Her gün (veya vardiyada) yeni bağlantı üretmek yöneticiye küçük bir iş yükü getirir.
- (−) `dropoffHint` müşteri adresinin bir parçasıdır; işletme siparişi zaten bilse de hukuk görüşüne bağlıdır (alan isteğe bağlıdır, kaldırılabilir).

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| HMAC imzalı durumsuz jeton (JWT benzeri) | İptal için yine sunucu tarafı liste gerekir; yeni sır ve döndürme prosedürü; her istekte paket listesi için veritabanı zaten okunuyor, durumsuzluğun kazancı yok |
| Jeton sorgu dizesinde veya yolda | Ters vekil / API erişim loglarına, tarayıcı geçmişine ve Referer'a düşer |
| İşletmeye kullanıcı hesabı (rol `BUSINESS`) | İşletme paneli kapsamı (İterasyon 5+); parola yönetimi restoran için sürtünme |
| SMS / WhatsApp Business API ile otomatik gönderim | Ücretli; ürün sahibinin kuralı |
| Paket başına ayrı bağlantı | Restoran her sipariş için yeni bağlantı açmak zorunda kalır; işletme başına tek sayfa daha kullanışlı |
| Süresiz bağlantı | Sızıntı etkisi zamanla büyür; kısa ömür + kolay yenileme daha güvenli |

## Uygulama notları (İterasyon 3b-2, Görev Q3b)

- **Kod:** `services/api/src/business-link/` (servis, koruyucu, panel ve girişsiz denetleyiciler). Jeton `crypto.randomBytes(32)` base64url (43 karakter); veritabanında yalnızca `sha256` özeti; `X-Business-Link-Token` başlığı hiçbir günlüğe yazılmaz (istek serileştiricisi yalnızca `authorization` ve `user-agent` taşır; `business-link.e2e-spec.ts` günlük testi).
- **Hız sınırı:** jeton başına `BUSINESS_LINK_RATE_PER_MIN` (30) / dk koruyucuda Redis sayacıyla (`RATE_LIMITED`, 429, `Retry-After`); IP başına ×2 genel hız sınırlayıcıda (`app.factory.ts` `bl-ip` kuralı). Tüm `/business-link/*` yanıtları (hata ve 429 dahil) `Cache-Control: no-store` ve `Referrer-Policy: no-referrer` taşır.
- **Kapsam:** liste son 12 saat CREATED / IN_POOL / ASSIGNED + son 30 dk PICKED_UP; `dropoffHint` adres metninin ilk virgülüne kadar, <= 40 karakter (hukuk görüşüne bağlı, alan kolayca kaldırılabilir). Başka işletmenin paketi 404, alınmış 409.
- **Sapma:** "hazır" çağrısı `ready_at_source = SIGNAL` iken idempotenttir (SPEC'teki "ready_signaled_at doluysa" yerine); kurye "Hazır değil" dedikten sonra "Hazır oldu" / panel "Hazır" yeniden `readyAt = şimdi` yapar ve `ready_signaled_at` ilk değerini korur.
- CORS: `allowedHeaders` listesine `X-Business-Link-Token` eklendi (önkontrol testi). Saklama: bitiş / iptalden 30 gün sonra `JobsService.runRetention` siler.
