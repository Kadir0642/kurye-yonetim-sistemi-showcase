# ADR-0042: Dış sipariş girişi iskeleti (platformdan bağımsız)

- **Durum:** Önerildi (ürün sahibi, 2026-11-13: "iskelet hazırlık olarak onaylandı; gerçek platform entegrasyonu ve platform hesabı şimdi yok")
- **Tarih:** 2026-11-14
- **İlgili:** [ADR-0025](0025-isletme-hazir-baglantisi.md) (işletme başına girişsiz bağlantı, jeton deseni), [ADR-0032](0032-musteri-verisi-saklama.md) (KVKK saklama süresi), [ADR-0030](0030-kimlik-sertlestirme.md), [ADR-0031](0031-panel-csp-ve-baglanti-sinirlari.md); sözleşme 0.7.3; migration 0016

## Bağlam

Pilot müşterinin (bir kurye firması) restoranlarından gelen siparişler bugün yöneticinin elle girdiği paketlerdir. Rakip ürünler ve kurye yazılımları siparişi şu yollarla alır: (a) restoranın kendi kasa / adisyon programından **webhook** (restoran başına kimlik, JSON gövde, durum geri yazma), (b) restoranın platform hesabından platform API'sine bağlanma (yoklama ya da webhook), (c) telefonla gelen siparişin elle girilmesi, (d) dosya (CSV) aktarımı. Hangi platform ya da kasa programının ilk bağlanacağı ve kimliğin kimde olacağı pilot müşteriyle görüşülmeden belli olmaz; gerçek bir platform bağlantısı için ayrıca hesap ve sözleşme gerekir (ürün sahibi kuralı: onaysız dış hesap yok).

İhtiyaç: hesap gerektirmeden, **platformdan bağımsız** bir giriş katmanı kurmak; böylece bir platform ya da kasa programı bağlanacağı gün yalnızca "o platformun biçimini bizim şemaya çeviren" küçük bir bağdaştırıcı yazılsın. Aynı giriş hizmeti elle giriş ve CSV için de kullanılsın (tek doğru, tek test).

## Karar

**1. Üç yeni tablo ve paket bağlantısı (migration 0016).**
- `intake_keys`: işletme başına giriş anahtarı. Anahtarın kendisi saklanmaz; yalnızca sha256 özeti (`key_hash`) ve tanımak için ilk 11 karakter (`key_hint`). Ad, açılan paketin dağıtım kipi (`AUTO` / `POOL` / `HOLD`), isteğe bağlı durum bildirimi adresi (`webhook_url`, varsayılan boş), imza sırrı sürümü, iptal / son kullanım / sayaç.
- `intake_orders`: alınan her sipariş. Tekrar anahtarı `UNIQUE (kiracı, işletme, kaynak, dış kimlik)`. Durum: `RECEIVED` (işleniyor) → `PACKAGE_CREATED` | `NEEDS_ADDRESS` → `DISMISSED` | `EXPIRED`. `payload` (adres, ad, telefon) yalnızca `RECEIVED` ve `NEEDS_ADDRESS` iken dolu olabilir (CHECK ile zorlanır).
- `intake_webhook_outbox`: durum bildirimi giden kutusu (olay başına bir satır; `UNIQUE (sipariş, olay)`).
- `packages.intake_order_id` (FK'siz; paketi açan sipariş). `packages_customer_phone_chk` gevşetildi: müşteri telefonu boş olabilir, **yalnızca** anonimleştirilmiş ya da dış siparişle açılmış pakette.

**2. Anahtar.** `ik_` + 32 rastgele bayt (base64url). Panelde yalnızca üretim yanıtında görünür. İşletme başına en çok 5 etkin anahtar (409). Anahtar kiracı **ve** işletmeye bağlıdır: paket her zaman anahtarın işletmesi için açılır, gövdedeki `businessId` / `tenantId` yok sayılır. Kimlik `X-Intake-Key` başlığıdır; başlık yok / biçim hatalı / bilinmeyen / iptal edilmiş / işletme arşivli: **tek tip** 401 `INTAKE_KEY_INVALID` (neden ayrıştırılmaz). Anahtar asla loglanmaz; yanıtlar `Cache-Control: no-store`. Oluşturma ve iptal OWNER + DISPATCHER; ad, kip ve bildirim adresi ayarı ile imza sırrı yalnızca OWNER.

**3. Hız sınırı.** Anahtar başına dakikada `INTAKE_RATE_PER_MIN` (varsayılan 120; Redis, anahtar özeti sayaç anahtarı) ve IP başına bunun iki katı (genel hız sınırlayıcı; geçersiz anahtarla kaba kuvvet denemesini durdurur). Aşımda 429 `RATE_LIMITED` + `Retry-After`.

**4. Giriş ucu `POST /api/v1/intake/orders` ve şema.** Sürüm yol önekindedir (`/api/v1`). Gövde `IntakeOrderInput`: zorunlu `externalId` ve `dropoff.addressText`; isteğe bağlı `source` (varsayılan `generic`), `dropoff.location`, `dropoff.addressDetail`, `customer.name` / `customer.phone` (**varsayılan olarak gönderilmez**), `items` (içerik özeti; pakete not olarak eklenir), `note`, `paymentType`, `amountKurus`, `readyAt`. Bilinmeyen alanlar yok sayılır. Kurallar:
- Ödeme türü verilmezse `amountKurus > 0` ise `CASH`, değilse `PAID_ONLINE`; `PAID_ONLINE` + tutar > 0 → 400.
- Müşteri adı yoksa pakette "Müşteri"; telefon yoksa boş (kurye uygulaması müşteri bloğunu göstermez, işletme telefonu kalır).
- `readyAt` 5 dk geçmiş / 3 sa ileri aralığının dışındaysa **reddedilmez**, yok sayılır ve yanıtta `warnings: ['READY_AT_IGNORED']` döner.
- **Idempotent:** aynı (işletme, `source`, `externalId`) ikinci kez gelirse yeni paket açılmaz; ilk kaydın sonucu `duplicate: true` ve 200 ile döner (ilk kayıt 201). Eşzamanlı çift istekte tek sipariş / tek paket (veritabanı `ON CONFLICT`). İşlenirken süreç çökerse sipariş `RECEIVED` kalır; 60 sn sonra bir sonraki istek ya da bakım işi onu sahiplenip tamamlar, pakette `intake_order_id` olduğundan ikinci paket açılmaz.

**5. Adres çözümü ve "adres bekliyor" yolu.** Koordinat verilmişse olduğu gibi kullanılır (`locationSource: PROVIDED`). Yoksa `GeoService.geocode` (otomatik tamamlamanın ilk önerisi + ayrıntı; mevcut sağlayıcı ve önbellek) çağrılır (`GEOCODED`). Çözülemezse sipariş **reddedilmez**: `NEEDS_ADDRESS` olarak kaydedilir; nedenler `GEOCODE_NOT_FOUND`, `GEOCODE_UNAVAILABLE` (sağlayıcı hatası), `OUT_OF_SERVICE_AREA` (hizmet bölgesi `REJECT` politikasında). Paket **adres çözülene kadar açılmaz** (bkz. Alternatifler: `packages.dropoff_location` zorunludur; konumsuz paket rota, havuz, atama ve kurye uygulamasında tanımsız davranış üretirdi). Yönetici panelde sırayı görür (menü rozeti + "Adres bekleyen" süzgeci), `POST /intake/orders/{id}/resolve` ile adres seçici sonucunu verir (ya da gövdesiz çağırıp otomatik çözümü yeniden dener) ya da `.../dismiss` ile vazgeçer. 48 saat içinde tamamlanmayan sipariş bakım işinde `EXPIRED` olur.

**6. Durum geri bildirimi (isteğe bağlı).** Anahtarın `webhook_url` adresi boşsa hiçbir şey gönderilmez (varsayılan). Doluysa olaylar `picked_up`, `delivered`, `undeliverable` (teslim edilemedi ya da iptal) olarak, gövde `IntakeWebhookPayload` ile gönderilir: `{ id, event, occurredAt, order: { externalId, source, packageNumber } }`; **kişisel veri ve kurye bilgisi yoktur**. Başlıklar `X-Kurye-Event`, `X-Kurye-Delivery` (= `id`; alıcı tekrarı buna göre ayıklar), `X-Kurye-Signature: t=<unix sn>,v1=<hex>`; `v1 = HMAC-SHA256(secret, "<t>.<ham gövde>")`, alıcı `t` değeri 5 dakikadan eskiyse reddeder.
- **İmza sırrı saklanmaz:** `INTAKE_SIGNING_SECRET` (boşsa `JWT_ACCESS_SECRET`'tan türetilen ayrı bir bağlam) + anahtar kimliği + sürümden HKDF-SHA256 ile türetilir (`whsec_…`). OWNER panelden gösterir ya da yeniler (sürüm + 1; eski sır anında geçersiz).
- **Giden kutusu ve yeniden deneme:** olaylar paket servisine dokunulmadan, paketin zaman damgalarından (alındı / teslim / başarısız / iptal) üretilir (5 sn'lik tarama; olay başına tek kuyruk satırı). Gönderim satır kirasıyla (`FOR UPDATE SKIP LOCKED`; çok örnekte güvenli), 5 sn zaman aşımı, **yönlendirmeler izlenmez**, yanıt gövdesi okunmaz / saklanmaz. 2xx = teslim edildi; diğerleri 30 sn, 2 dk, 10 dk, 30 dk, 1 sa, 2 sa, 4 sa aralıklarla en çok **8 deneme**, sonra `FAILED` (anahtar kartında "N vazgeçildi"). Anahtar iptal edilince bekleyenler gönderilmez.
- **SSRF koruması:** kayıt anında üretimde yalnızca `https`, kullanıcı bilgisi / `#` yok, `localhost` / `.local` / `.internal` ve özel IP yok; gönderim anında üretimde ad çözülür ve hiçbir adres özel / yerel / bağlantı-yerel olamaz (DNS ile yerel ağa dönüş). Geliştirme ve testte `http` ve yerel adres serbesttir (yerel alıcıyla deneme).
- Testler gerçek bir dış adrese çağrı yapmaz: enjekte edilen sahte `fetch` alıcıyı temsil eder.

**7. Elle giriş ve CSV (panel).** Aynı `IntakeService` kullanılır (`channel: MANUAL` / `CSV`, `source: manual` / `csv`). Panel CSV'yi tarayıcıda ayrıştırır (Excel noktalı virgül + UTF-8 BOM, virgül, sekme, tırnaklı hücre; Türkçe ve İngilizce başlıklar; `185,50` ₺ → kuruş), `POST /intake/orders/import` ile **önizleme** (`dryRun`: yalnızca doğrulama + tekrar denetimi; kayıt ve adres çözümü yok) ve onaydan sonra kayıt yapar. En çok 100 satır (gövde sınırı 256 KB); hatalı satır diğerlerini engellemez.

**8. Test aracı.** (a) Panel > Dış siparişler > Bağlantı anahtarları > "Test siparişi gönder" (`POST /intake/keys/{id}/test-order`): anahtarla gelmiş gibi işlenen kurgusal sipariş (`source: test`). (b) `npm run intake:simulate -w @kurye/api -- --key ik_… [--count 5] [--secret whsec_… --listen 4010]`: çalışan API'ye platform gibi sipariş iter (bir tekrar ve bir koordinatsız adres dahil), isteğe bağlı yerel alıcı açıp imzaları doğrular. Gerçek bir platforma çağrı yapılmaz.

**9. Bağdaştırıcı arayüzü.** `IntakeAdapter { id; parse(raw): IntakeOrderInput[]; verifyRequest?() }`; yalnızca `generic` uygulanmıştır (gövde zaten şemadadır). Çekirdek (anahtar, tekrar, adres, paket, bildirim) bağdaştırıcıdan bağımsızdır.

**10. KVKK ([ADR-0032](0032-musteri-verisi-saklama.md) ile uyum).**
- Kişisel veri (ad, telefon, açık adres) paket açıldığı anda **yalnızca pakette** kalır ve ADR-0032 saklama süresine (anonimleştirme) tabidir; `intake_orders.payload` paket açılınca, vazgeçilince ya da süre dolunca silinir (veritabanı CHECK'i bunu zorlar).
- Adres bekleyen sipariş en çok 48 saat saklanır (sürenin anlamı kalmaz); süre dolunca `EXPIRED` + `payload` silinir.
- Durum bildirimi gövdesi, paket olayı verisi (`intake: { source, externalId }`) ve günlükler kişisel veri taşımaz; adres metni, ad, telefon ve anahtar loglanmaz (e2e: günlükte yok).
- Telefon varsayılan olarak istenmez; gönderen yalnızca teslimat için gerekeni göndermelidir (sözleşme açıklaması).

## Sonuçlar

- (+) Platform ya da kasa programı bağlanacağı gün iş yalnızca bağdaştırıcıdır; anahtar, tekrar, adres, KVKK, bildirim, panel hazırdır.
- (+) Pilot müşteri bugünden telefon siparişini elle ve toplu CSV ile girebilir; çift tıklama / çift gönderim paket çoğaltmaz.
- (+) Adres çözülemese de sipariş kaybolmaz; yönetici tek yerden görür.
- (−) Adres bekleyen sipariş **henüz paket değildir**: atanamaz, haritada görünmez; yönetici adresi tamamlayana kadar bekler (48 saat).
- (−) İlk önerinin otomatik seçilmesi kötü eşleşme üretebilir; çözülen adres pakette görünür ve gönderenin metni "adres tarifi" olarak kalır (kurye ikisini de görür). Eşleşme güveni ölçülmedi; pilot verisiyle gözden geçirilecek.
- (−) Müşteri telefonu yoksa kurye uygulaması müşteri bloğunu (ad dahil) göstermez; işletme telefonu kalır. Mobil uygulama bu iskelette değiştirilmedi.
- (−) Durum bildirimi taramaya dayandığından olay gecikmesi en çok ~5 sn'dir; paket servisinde yeni bağımlılık yoktur.
- (−) Platformdan iptal (müşteri vazgeçti) ve sipariş güncelleme alma yoktur (aşağıdaki yapılacaklar).

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Çözülemeyen adresli paketi `NEEDS_ADDRESS` durumuyla `packages` içinde açmak | `dropoff_location` zorunlu ve durum makinesi / rota / havuz / kurye uygulaması konumsuz paketi bilmiyor; yeni paket durumu mobili ve atamayı etkilerdi. Ayrı tablo + adres çözülünce paket açmak aynı görünürlüğü (yönetici listesi, rozet) risksiz verir |
| Anahtarı düz saklamak ya da imza sırrını veritabanında tutmak | Anahtar yalnızca özetle doğrulanır; imza sırrı türetilir (ana sır dışında saklanacak sır yok), sızıntıda tek sütun yeterli olmaz |
| Webhook'u paket servisinin her geçişinden tetiklemek | Paket servisi büyük ve paralel işlerde değişiyor; zaman damgası taraması bağımsız, kayıpsız (idempotent) ve test edilebilir |
| Gövdede `businessId` kabul etmek (tek anahtar, çok işletme) | Sızan anahtar tüm işletmelere sipariş açardı; işletme başına anahtar etkiyi sınırlar |
| Platform API'lerine doğrudan bağlanmak (yoklama) | Hesap, sözleşme ve gizli anahtar gerekir; ürün sahibi kuralı gereği şimdi yok |
| Tarayıcı eklentisi / panel okuma | Kırılgan, platform kullanım şartlarını ihlal riski ve KVKK riski yüksek |
| Ücretli SMS / kuyruk servisi | Ücretli servis kuralı; giden kutusu zaten veritabanında |

## Entegratör için özet

```
POST /api/v1/intake/orders          X-Intake-Key: ik_…      Content-Type: application/json
{ "externalId": "SIP-1001", "source": "pos-a",
  "dropoff": { "addressText": "Vişnelik Mah. Demo Sok. No:12", "addressDetail": "Kat 2" },
  "items": "2 pide, 1 ayran", "paymentType": "CASH", "amountKurus": 18550 }
→ 201 { "id": "…", "status": "PACKAGE_CREATED", "packageNumber": 412, "duplicate": false, … }   (tekrar: 200, duplicate: true)
→ 201 { "status": "NEEDS_ADDRESS", "reason": "GEOCODE_NOT_FOUND", … }                          (adres yöneticide bekler)
```

Bildirim doğrulama (Node, sözde kod):

```js
const [, t, v1] = header.match(/t=([0-9]+),v1=([0-9a-f]{64})/);
const expected = createHmac('sha256', secret).update(t + '.' + rawBody).digest('hex');
const ok = timingSafeEqual(Buffer.from(expected), Buffer.from(v1)) && Math.abs(Date.now() / 1000 - Number(t)) <= 300;
```

## Yapılacaklar (bu iskelette yok)

- **Platform ve POS bağdaştırıcıları** (her biri ayrı PR; hesap / sözleşme gerektirir): kasa / adisyon programları (webhook ile kurye firmasına yönlendirme belgeli olanlar önce), Getir, Yemeksepeti, Trendyol Go, Migros. Sıra, kimlik bilgisinin kimde olacağı ve ücret **pilot müşteri görüşmesine** bağlıdır ([MIMARI §8](../MIMARI.md) açık soru 39).
- Platformdan iptal / güncelleme alma (`POST .../cancel`), durum olaylarına "atandı / yolda" eklenmesi, bildirim günlüğü ekranı, ham yükün (telefonsuz) denetim için kısa süreli saklanması (KVKK görüşü sonrası).
- Ad çözümlemede eşleşme güveni (ilk öneri yerine skorlu seçim) ve toplu adres tamamlama.

## Uygulama notları

- **Kod:** `services/api/src/intake/` (`intake.service.ts`, `intake-keys.service.ts`, `intake-key.guard.ts`, `intake-webhook.service.ts`, `intake-jobs.service.ts`, `intake-maintenance.service.ts`, `signing.ts`, `webhook-url.ts`, `adapters/`), `GeoService.geocode`, `PackagesService.create` (isteğe bağlı `intake` + boş telefon); şema `db/schema/intake.ts`, migration `0016_dis_siparis_girisi.sql`; panel `apps/admin-web/src/pages/IntakePage.tsx` + `pages/intake/*`, `lib/intakeCsv.ts`.
- **Test:** `services/api/test/intake.e2e-spec.ts` (kimlik, kiracı izolasyonu, idempotency + eşzamanlılık + çökme sonrası, adres çözümü ve bekleyen yol, KVKK süre dolumu ve günlük, elle / CSV, imza + yeniden deneme + vazgeçme, hız sınırı), `security.e2e-spec.ts` (matris `intakeKey` türü), birim: `signing.spec.ts`, `webhook-url.spec.ts`, `intake-order.spec.ts`; panel `src/test/intake.test.tsx`, `lib/intakeCsv.test.ts`.
- **Yapılandırma:** `INTAKE_RATE_PER_MIN` (120), `INTAKE_SIGNING_SECRET` (isteğe bağlı; üretimde bildirim kullanılmadan önce ayrı bir değer verilmesi önerilir, sonradan değiştirmek tüm alıcı sırlarını değiştirir), `INTAKE_WEBHOOK_INTERVAL_S` (5).
