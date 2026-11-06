# ADR-0020: Çevrimdışı eylem kuyruğu ve sunucu tarafı idempotency (Idempotency-Key, cihaz olay zamanı, çakışma sözleşmesi)

- **Durum:** Önerildi (İterasyon 3a spesifikasyonuyla birlikte onaya sunuldu)
- **Tarih:** 2026-11-05
- **İlgili:** [ADR-0005](0005-konum-takibi-ve-kvkk.md), [ADR-0010](0010-kuryeler-arasi-devir.md), [ADR-0017](0017-saha-verisinden-ogrenen-harita.md) (öğrenilmiş noktalar cihaz zamanı ve konumu ister), [ADR-0019](0019-kurye-cevrimdisi-harita-ve-navigasyon.md), PLAN 3a-6, SPEC-3a §7

## Bağlam

Kodla doğrulanan bugünkü durum (2026-11-05, `main` 564863c):

| Konu | Bugün |
|---|---|
| Konum kuyruğu | Kalıcı (SQLite `location-queue.db`), seyreltme, batch, üstel geri çekilme; sunucuda `ON CONFLICT (courier_id, recorded_at) DO NOTHING`. **Yeterli, değişmeyecek** |
| Kurye eylemleri (`pickup`, `deliver`, `fail`, `reject`, `release`, havuzdan alma, devir adımları, vardiya) | Doğrudan çevrimiçi REST çağrısı (`useStopActions`); başarısızsa yalnızca hata şeridi, **kalıcı kuyruk ve yeniden deneme yok** |
| `Idempotency-Key` | Sözleşmede ve API'de yok |
| Olay zamanı | Sunucu zamanı (`package_events.created_at`, `picked_up_at = now()`) |
| Doğal idempotentlik | `pickup` / `deliver` / `fail`: koşullu UPDATE başarısızsa ve paket **zaten hedef durumdaysa** 200 (ör. ikinci `pickup` PICKED_UP iken). `reject` / `release`: paket havuzda ve son ret / bırakma olayı bu kuryeninse 200. `claim`: paket zaten bu kuryedeyse 200 |
| Boşluk | (1) Bir sonraki adım olduysa tekrar 409 olur: `pickup` yanıtı kaybolup kurye `deliver` yaptıktan sonra `pickup` tekrar gelirse 409 INVALID_TRANSITION; yeniden deneme ile gerçek çakışma ayırt edilemez. (2) Paket başka kuryeye verildiyse kurye **404** alır (çakışma olarak tanınmaz). (3) Çevrimdışı yapılan eylemin gerçek zamanı kaybolur (bağlanınca sunucu zamanı yazılır) |

Sahada çevrimdışı eylem kaybı güveni bozar; 3b (öğrenilmiş teslim noktaları, hazırlık süresi) cihaz olay zamanı ve olay anı konumu ister.

## Karar

**1. İstemci: kalıcı, tek sıralı eylem kuyruğu.** Kuyruğa giren eylemler: `pickup`, `deliver`, `fail`, `reject`, vardiya bitirme ve kendi devir isteğini iptal (`handover-requests/{id}/cancel`). Her eylem anında istemci bir UUID `Idempotency-Key`, sunucu saatine göre düzeltilmiş `occurredAt` ve olay anı konumunu (`ActionPosition`: lat, lng, accuracyM, isMock, recordedAt) üretir, kaydı SQLite `action_queue`'ya yazar, **sonra** gönderir. Gönderici global FIFO'dur (paket başına sıra ve "vardiya bitirme en son" kendiliğinden korunur); yalnızca yeniden denenebilir hatalarda (ağ, 5xx, 429, 401 yenileme, 409 IDEMPOTENCY_IN_PROGRESS) baştaki kayıtta bekler, üstel geri çekilme 2-60 sn. Arayüz iyimserdir: eylem yerel durumda hemen uygulanır, "Senkron bekliyor" rozeti ve bekleyen sayısı gösterilir.

**2. Kuyruğa girmeyenler (ağ gerektirir):** havuzdan alma (yarış), devir teklifi / kabulü / kod / onayı (karşı taraf canlı, kod süresi 120 sn), devir isteği açma, vardiya başlatma, navigasyon durak kilidi (`/navigate`; çevrimdışıyken atlanır, mevcut kural). Çevrimdışıyken bu düğmeler devre dışıdır ve nedeni yazar. `release` (kullanımdan kalkıyor) uygulamada kullanılmaz; sunucu yine de anahtarı destekler.

**3. Sunucu: `idempotency_keys` tablosu + Nest kesicisi (interceptor).**
- Anahtar kapsamı (`tenant_id`, `user_id`, `key`). Saklanan: işlem kimliği (`method` + yol şablonu + yol parametreleri), istek gövdesi özeti (kanonik JSON sha256), durum (`IN_PROGRESS` / `DONE`), yanıt durum kodu ve gövdesi, zamanlar. Saklama `IDEMPOTENCY_TTL_HOURS` (varsayılan **72 sa**; PLAN'daki 7 gün kişisel veri en aza indirme gereği kısaltıldı: saklanan yanıt müşteri bilgisi içerir ve 24 sa'ten eski `occurredAt` zaten sunucu zamanına düşer).
- Akış: `INSERT ... ON CONFLICT DO NOTHING`. Kayıt varsa: aynı işlem + aynı özet + `DONE` -> saklanan yanıt aynen, `Idempotent-Replayed: true`; farklı özet veya işlem -> 422 IDEMPOTENCY_KEY_REUSED; `IN_PROGRESS` ve 30 sn'den genç -> 409 IDEMPOTENCY_IN_PROGRESS (`Retry-After: 2`); daha yaşlı `IN_PROGRESS` (çökme) -> devralınır. İşlem bitince 2xx, 404, 409, 422 yanıtları saklanır (deterministik); 5xx, 429, 401, 403 saklanmaz ve kayıt silinir (yeniden denenebilir).
- Çökme penceresi (işlem commit edildi, yanıt saklanamadı): yeniden denemede mevcut doğal idempotentlik (hedef durumdaysa 200, son olay bu kuryenin reddiyse 200) aynı sonucu verir; çift olay yazılmaz. Bu, anahtar tablosunu iş işlemiyle aynı veritabanı işlemine sokmadan yeterli güvenceyi verir.

**4. Cihaz olay zamanı (`occurredAt`).** `package_events.occurred_at` (yeni, NOT NULL, varsayılan `created_at`) ve paket zaman damgaları (`picked_up_at`, `delivered_at`, `failed_at`) `occurredAt` ile yazılır; `created_at` sunucu kayıt zamanı olarak kalır. Sınırlar: şimdi + 120 sn'den ileri veya 24 sa'ten eski -> sunucu zamanı (`data.occurredAtAdjusted = FUTURE | TOO_OLD`); paketin bir önceki olayından eski -> önceki olay zamanı (`BEFORE_PREVIOUS`). Olayın `data.source = OFFLINE_QUEUE` ve `queuedForS` alanları ölçüm içindir. Vardiya bitiminde `shifts.ended_at` aynı kuralla.

**5. Çakışma sözleşmesi (sessiz kayıp yok).**
- Paket bu kuryeye daha önce atanmış ama artık değilse **409 PACKAGE_NOT_YOURS** (bugünkü 404 yerine; hiç atanmamışsa veya başka kiracınınsa 404 kalır).
- Paket iptal edildiyse 409 INVALID_TRANSITION (`currentStatus = CANCELLED`); başka uyumsuz durumda 409 INVALID_TRANSITION + `currentStatus`.
- Paket zaten hedef durumdaysa (ör. yönetici "teslim edildi" işaretlemiş) 200, yeni olay yazılmaz.
- `Idempotency-Key` taşıyan bir istek 409 / 404 ile reddedilirse sunucu pakete `OFFLINE_ACTION_REJECTED` olayı yazar (durum değişmez; `data`: action, code, occurredAt, queuedForS) ve panele `courier.sync_conflict` yayınlar. Uygulama kaydı kuyruktan çıkarır, iyimser durumu geri alır, kuryeye açık "Senkron çakışması" iletisi gösterir ("Paket #1042 başka kuryeye verildi; 'Aldım' uygulanmadı").

**6. Hesap ve oturum.** Kuyruk kayıtları `user_id` taşır; başka kullanıcı giriş yaparsa önceki kullanıcının kayıtları gönderilmez (karantina), aynı kullanıcı dönünce gönderilir. Bekleyen eylem varken çıkış uyarı ister. 401'de token yenilenir; yenileme başarısızsa kuyruk korunur ve giriş istenir. 403 ACCOUNT_DISABLED'da gönderim durur.

## Sonuçlar

- (+) Aynı eylem N kez gönderilse de tek olay; yeniden deneme ile gerçek çakışma ayrışır (replay 200 + başlık, çakışma 409 + kod).
- (+) Cihaz olay zamanı tazelik, gecikme ve 3b ölçümlerini doğru yapar; düzeltme her zaman olayda işaretlidir.
- (+) Çakışmalar hem kuryeye hem yöneticiye görünür; denetim kaydı kalır.
- (+) Sunucu değişikliği geriye uyumludur: başlık ve alanlar isteğe bağlıdır; İterasyon 2 APK'sı aynen çalışır (yalnızca başka kuryeye geçmiş paket için 404 yerine 409 görür; mevcut uygulama 409'u genel hata iletisiyle gösterir).
- (-) Yeni tablo ve günlük temizleme işi; saklanan yanıtlar 72 sa kişisel veri taşır (TTL ile sınırlı).
- (-) İstemci saati kötü ayarlıysa ve sunucu saati alınamamışsa `occurredAt` sınırlara takılabilir (işaretlenir, kayıp değil).
- (-) İyimser arayüz, çakışmada geri alınan bir durum gösterir (açık iletiyle).

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Yalnızca doğal idempotentlik (bugün) | Bir sonraki adım olduktan sonraki tekrar 409 olur; çakışma ile tekrar ayırt edilemez; olay zamanı kaybolur |
| Anahtarı iş işlemiyle aynı DB işlemine yazmak | Her servis metoduna işlem nesnesi geçirmek gerekir (geniş değişiklik); çökme penceresini doğal idempotentlik zaten kapatıyor |
| Redis'te anahtar saklamak | Redis kalıcı veri taşımaz kabul edildi (PLAN P-3); yeniden başlatmada anahtar kaybı çift olay riski |
| Toplu senkron ucu (tek istekte tüm kuyruk) | Kısmi başarı ve sıralı hata yönetimi karmaşık; tek tek istek mevcut uçları ve kuralları aynen kullanır |
| Havuzdan almayı da kuyruğa almak | Yarış: çevrimdışı "aldım" başka kuryenin aldığı paketi göstermeye devam eder; yanıltıcı |
| Anahtar TTL 7 gün (PLAN taslağı) | Saklanan yanıtlarda müşteri bilgisi; 24 sa sonrası zaten sunucu zamanı; doğal idempotentlik TTL sonrasını da kapatır |

## Uygulama notları

Sunucu tarafı Görev Q (İterasyon 3a) ile uygulandı: `services/api/src/common/idempotency/`, `src/packages/occurred-at.ts`, `drizzle/0003_iterasyon3a.sql`.

- **Kesici.** `@Idempotent({ action, target })` işaretli uçlarda (`pickup`, `deliver`, `fail`, `reject`, `release`, `shifts/end`, `handover-requests/{id}/cancel`) ve yalnızca `Idempotency-Key` varsa çalışan `IdempotencyInterceptor` (APP_INTERCEPTOR). Başlık geçerli bir UUID olmalıdır (aksi 400). `operation` = `POST /courier/packages/{packageId}/pickup#<packageId>`; özet, `req.body`'nin (doğrulamadan önceki ham gövde) kanonik JSON sha256'sıdır; gövdesiz, `null` ve `{}` aynı sayılır. Replay'de saklanan 2xx gövde `of(...)` ile, saklanan hata (404 / 409 / 422) `StoredProblemError` ile aynen döner; ikisinde de `Idempotent-Replayed: true`.
- **Saklama kararları (spec'ten küçük sapma).** 2xx, 404, 409, 422 saklanır; 5xx, 429, 401, 403 saklanmaz. **400 de saklanmaz** (kayıt silinir): doğrulama hatası hiçbir şey çalıştırmadı ve istemci aynı anahtarı düzeltilmiş gövdeyle kullanabilmelidir; saklansaydı düzeltilmiş istek 422 `IDEMPOTENCY_KEY_REUSED` alırdı. `IN_PROGRESS` kaydı 30 sn'den eskiyse devralınır; TTL'i (`IDEMPOTENCY_TTL_HOURS`) geçmiş kayıt talep anında silinip yeni sayılır, saklama işi de ayrıca temizler.
- **`occurredAt`.** `effectiveOccurredAt` saf fonksiyondur (birim test). `courierAction` / `returnToPool` işleminde paket satırı `FOR UPDATE` ile kilitlenir (rota kilidinden sonra: mevcut kilit sırası korunur), sınırlama paketin `occurred_at`'i en büyük olayına (`OFFLINE_ACTION_REJECTED` hariç) göre yapılır. `occurredAt` hiç verilmediyse veritabanı saati kullanılır ve sınır denetimi yapılmaz (API ve veritabanı saatleri arasındaki kayma yüzünden sahte `BEFORE_PREVIOUS` oluşmasın). `source = OFFLINE_QUEUE` ve `queuedForS` yalnızca `Idempotency-Key` taşıyan isteklerde yazılır. `route_stops.completed_at` ve `shifts.ended_at` aynı etkin zamanla yazılır; vardiya bitişi vardiya başlangıcından önceye çekilmez (başlangıca sabitlenir).
- **Çakışma kodu.** "Bu kuryeye daha önce atanmış" tanımı: pakette bu kuryenin `courier_id`'sini taşıyan, `OFFLINE_ACTION_REJECTED` olmayan herhangi bir olay. Hiç atanmamış paket ve başka kiracı 404 kalır. Anahtarlı istek 404 / 409 ile bitince `OFFLINE_ACTION_REJECTED` (durum değişmez, `from` / `to` null) ve `courier.sync_conflict` yazılır; istisna: `IDEMPOTENCY_IN_PROGRESS` (kesicinin kendi yanıtı) ve `SHIFT_NOT_OPEN` (istemci başarı sayar). Paket kiracıya ait değilse veya vardiya eylemiyse olay yazılmaz, WS'te `packageId: null` gider (başka kiracıya bilgi sızmaz). Replay'de ikinci olay yazılmaz.
- **Bilinen sınır.** Anahtar tablosu iş işlemiyle aynı işlemde değildir (ADR kararı): işlem commit edilip yanıt saklanamazsa yeniden deneme doğal idempotentlik sayesinde aynı sonucu verir; `IN_PROGRESS` devralma penceresinde (30 sn) çökme sonrası aynı çakışma olayı iki kez yazılabilir (nadir, zararsız).
- **Doğrulama.** `test/idempotency.e2e-spec.ts` (19 senaryo): aynı eylem 5 kez -> 1 olay + 5 aynı yanıt (4'ünde `Idempotent-Replayed`), farklı gövde / işlem / paket 422, kullanıcı yalıtımı, devralma, TTL, `occurredAt` (30 dk önce, FUTURE, TOO_OLD, BEFORE_PREVIOUS), reassign / iptal / hedef durumda çakışmaları, vardiya ve devir isteği iptali, saklama temizliği.
