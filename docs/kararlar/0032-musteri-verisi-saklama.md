# ADR-0032: Müşteri verisi saklama süresi ve anonimleştirme (KVKK)

- **Durum:** Kabul edildi (ürün sahibi, 2026-11-07: "teslimden 30 gün sonra müşteri ad/telefon/adresini anonimleştiren kiracı ayarı, temkinli varsayılan; hukuki görüş gelince süre değişir")
- **Tarih:** 2026-11-07
- **İlgili:** Güvenlik denetimi Kasım 2026 B-08; [ADR-0005](0005-konum-takibi-ve-kvkk.md) (konum saklama), [ADR-0017](0017-saha-verisinden-ogrenen-harita.md) A (öğrenilmiş teslim noktaları), [ADR-0026](0026-teslim-geofence-politikasi.md) (geofence, öğrenme dışlaması), [ADR-0022](0022-uretim-dagitimi-ucretsiz-katman.md) (yedekler); LOG-SAKLAMA.md; sözleşme 0.6.2

## Bağlam

Saklama işi konum pinglerini (30 gün), refresh token'ları, bağlantıları ve gözlemleri siliyor; `packages` tablosundaki müşteri adı, telefonu ve açık adres metni hiç silinmiyordu. Panel araması bunları yıllar sonra da buluyordu. KVKK "amaçla sınırlı ve gerektiği kadar saklama" ilkesi için bir süre gerekir; bunu belirleyecek hukuki görüş henüz yok. Denetim 90 gün önermişti; ürün sahibi **temkinli** davranıp 30 gün istedi ve süreyi bir ayar yaptı: hukuki görüş gelince kod değişmeden süre değişir.

Kısıtlar: ücretli servis yok; paket kaydı, tutar, zamanlar ve istatistikler (işletme bazlı raporlar, geofence, atama metrikleri) bozulmamalı; öğrenilmiş teslim noktaları (adres defteri) müşteri kimliği taşımaz, anonimleştirmeden etkilenmemeli; kiracılar birbirinin verisine dokunmamalı.

## Karar

**1. Kiracı ayarı `settings.privacy.customerRetentionDays`.** Tam sayı, **0 = kapalı** (süresiz saklama), aksi hâlde **7-365 gün**; **yeni kiracılar için varsayılan 30**. Yalnızca OWNER (`PATCH /tenant`); aralık dışı değer 400 `VALIDATION_FAILED` (alan hatası `settings.privacy.customerRetentionDays`). Veritabanında `tenants.privacy_customer_retention_days` + CHECK. **Mevcut kiracılar migration ile 0 (kapalı) alır** (şema varsayılanı 30 kalır, migration `UPDATE tenants SET ... = 0` ile geri doldurur): geri alınamaz silme, sahibin bilinçli açmasıyla başlar; dağıtım hiçbir veriyi kendiliğinden silmez. Yeni oluşturulan kiracı 30 ile başlar. COURIER rolü de `GET /tenant` ile değeri okur (gizli değildir).

**2. Anonimleştirme işi (`services/api/src/privacy/`, yeni modül).** Saatlik (`setInterval`, açılışta 45 sn sonra bir kez), `SET NX PX` iş kilidi (`lock:job:privacy-anonymize`, çok örnekte tek çalıştırır) ve **kiracı başına** ikinci kilit (`lock:job:privacy:<tenantId>`; elle çalıştırmayla çakışmaz). Saklama süresi 0 olan kiracı atlanır. Tur başına 500 paket, tek işlemde (`FOR UPDATE ... SKIP LOCKED`), kiracı başına çalıştırma başına en çok 40 tur (ilk açılışta büyük birikim sonraki saate kalır). Hata bir kiracıyı sonuncuya taşımaz; **yalnızca sayı** loglanır (kiracı kimliği + adet), ad / telefon / adres asla.

**3. Uygunluk.** Paket **terminal durumda** (`DELIVERED`, `CANCELLED`, `FAILED`), terminal zamanı (`COALESCE(delivered_at, cancelled_at, failed_at, updated_at)`) `now() - N gün`'den eski ve `customer_anonymized_at IS NULL`. Aktif paket (`CREATED`, `IN_POOL`, `ASSIGNED`, `PICKED_UP`) ne kadar eski olursa olsun **asla** dokunulmaz. Tarama için kısmi dizin `packages_anonymize_scan_idx (tenant_id, terminal_zamanı) WHERE customer_anonymized_at IS NULL AND status IN (...)`.

**4. Ne yapılır (aynı işlemde):**

| Alan | Sonuç |
|---|---|
| `customer_name` | `Anonim` |
| `customer_phone` | `NULL` (sütun artık NULL olabilir; CHECK: yalnızca `customer_anonymized_at` doluysa) |
| `dropoff_address_text` | Yalnızca **mahalle / ilçe / il düzeyi**: virgülle ayrılmış parçalardan sondan en çok iki "alan" parçası; ilk parça (açık adres satırı / yer adı) asla, rakam içeren veya cadde-sokak-daire belirten parça görülünce durulur, posta kodu öneki ve ülke atılır. Hiçbir şey kalmazsa `Anonim adres`. (`area-label.ts`, birim testli) |
| `dropoff_address_detail`, `dropoff_place_id`, `note`, `fail_note`, `cancel_reason` | `NULL` (serbest metinler kişisel veri taşıyabilir; "Müşteri X aradı" gibi) |
| `dropoff_location`, `dropoff_target_location` | **~100 m'ye yuvarlanır** (`ST_SnapToGrid` 0,001°): ilçe / mahalle ısı haritaları ve mesafe istatistikleri kalır, kapı ayırt edilmez |
| Olay verisi (`package_events.data`) | `note` silinir; `CANCELLED`/`PICKED_UP`/`DELIVERED`/`FAILED` olaylarında serbest `reason` silinir; `DELIVERED`/`FAILED` olayındaki `position` ~100 m'ye yuvarlanır; `geofence.override.note` silinir. `geofence` sonucu, mesafe, kaynak ve `failReason` **kalır** |
| `handover_requests.note/decision_note`, `handover_offers.note`, `package_rejections.note` | `NULL` (kurye / yönetici serbest metni müşteri telefonu içerebilir) |
| `customer_anonymized_at` | `now()` |

Tutar, ödeme tipi, işletme, kurye, tüm zaman damgaları, `status`, `fail_reason`, `pickup_*` (işletmenin kendi adresi) ve olayların kendisi **kalır**. `version` ve `updated_at` değişmez.

**5. Neyi etkilemez (ADR-0017 / ADR-0026).** Adres defteri (`address_book_entries`) müşteri kimliği ve adres metni taşımaz (adres yalnızca HMAC anahtarıdır); öğrenilmiş / sabitlenmiş noktalar, güven ve örnek sayıları **olduğu gibi kalır** (e2e: satır birebir aynı). Teslim gözlemleri paket koordinatına değil teslim anı konumuna dayanır ve zaten kurye kimliği olmadan ayrı tutulur. Tek görünür fark: adres defteri listesi etiketini son paketin adresinden alır (`last_package_id`); anonimleştirmeden sonra etiket mahalle / ilçe düzeyine iner (örn. "Odunpazarı, Eskişehir"), nokta ve harita konumu değişmez. Kasıtlı: etiket olarak tam adresi saklamak bu kararın amacına aykırı olurdu.

**6. API.** `GET /privacy/status[?days=N]` → `{ retentionDays, lastRunAt, anonymizedTotal, eligibleNow, pendingCount }` (`pendingCount`: `days` verilmişse o süreye göre, kapalıyken de, açma / kısaltma onayındaki "ilk çalıştırmada X paket etkilenir" önizlemesi; verilmezse `eligibleNow` ile aynı); `POST /privacy/anonymize-run` (elle çalıştır, anında; kullanıcı başına **5/dk** hız sınırı, `x-roles: [OWNER]`). Süre 0 iken veya aynı kiracıda çalışma sürerken 409 `CONFLICT`. Yanıt: `{ ranAt, retentionDays, anonymizedPackages }`. `Package` yanıtına `customerAnonymizedAt` eklendi; `Package.customer` artık `PackageCustomer` (`phone` `string | null`); istek gövdesindeki `Customer` değişmedi. Güvenlik matrisi (`security.e2e-spec.ts`) yeni uçları `x-roles`'tan otomatik kapsar.

**7. Panel.** Ayarlar > **Gizlilik (KVKK)** sekmesi (yalnızca OWNER): açma / kapama anahtarı, gün alanı (7-365, anlık doğrulama), açıklama metni, durum (son çalışma, anonimleştirilen toplam, sırada bekleyen) ve **Şimdi çalıştır**. Kapalıyken "Kapalı — açmak için gün girin" gösterilir ve gün alanı boş başlar. Süreyi kısaltmak veya kapalıdan açmak onay penceresi ister: "N günden eski teslim edilmiş, iptal edilmiş veya başarısız olmuş paketlerdeki müşteri verisi kalıcı olarak silinecek; ilk çalıştırmada X paket etkilenir" (X = `pendingCount`). Elle çalıştırma da onay ister (geri alınamaz). Paket ayrıntısında anonimleştirilmiş müşteri "Anonim" / telefon "Silindi" ve açıklayıcı not gösterilir. İşletme "hazır" sayfasında anonimleştirilmiş pakette adres ipucu gösterilmez (null). Bu sayfa yalnızca aktif paketleri listelediğinden pratikte etkilenmez, savunma amaçlıdır.

## Sonuçlar

- (+) Denetim B-08 kapanır: ayar + iş + elle çalıştırma + durum; istatistikler, rotalar ve öğrenilmiş noktalar çalışmaya devam eder.
- (+) Süre hukuki görüşe göre tek ayarla değişir; 0 ile kapatılabilir (acil geri dönüş ve demo için).
- (−) **Geri alınamaz.** Süreyi kısaltmak (ör. 90 → 30) bir sonraki saatlik çalışmada o süreden eski tüm paketleri kalıcı olarak anonimleştirir. Panel onay ister; bu belge ve CALISMA-KITABI §16 uyarır. Anonimleştirilen veri geri getirilemez (yalnızca eski bir yedekten, aşağıya bakın).
- (+) **Mevcut kurulumlarda sürpriz silme yok:** migration mevcut kiracıları 0 (kapalı) yapar; sahip kartta süreyi girip etkilenecek paket sayısını görerek onaylar. Yeni kiracı 30 ile açılır (temkinli varsayılan korunur). e2e: migration ifadesi çalıştırıldıktan sonra iş hiçbir şeyi değiştirmez.
- (−) **Yedekler:** anonimleştirme canlı veritabanında olur; `pg_dump` yedekleri (48 saatlik + 35 günlük, YEDEKLEME.md) eski veriyi süresi dolana kadar taşır. Fiili üst sınır **N + 35 gün**; yedekten geri yükleme sonrası anonimleştirme işi yeniden çalışır ve veriyi tekrar siler (iş idempotenttir). Bu, aydınlatma metnine yazılır.
- (−) Cihazdaki çevrimdışı önbellek (kurye uygulaması) bu kararın dışındadır; yalnızca aktif paketlerin müşteri verisini taşır ve şifresiz SQLite sorunu B-17 altında izlenir.
- (−) Serbest metin alanları (`note`, `cancel_reason`, `fail_note`) anonimleştirmede bütün olarak silinir; bu alanlara bakarak yapılan sonradan analiz (iptal nedeni sözlüğü) mümkün olmaz. Raporlar bu alanları kullanmıyor (kontrol edildi).
- (−) Adres defteri etiketi yıllık görünümde yalnızca mahalle / ilçe düzeyinde kalır (madde 5).

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Önerilen 90 gün varsayılan (denetim) | Ürün sahibi temkinli (daha kısa) saklamak istedi; hukuki görüşte uzatılabilir, kısaltma geri alınamaz olduğundan başlangıç 30 daha güvenli yöndür |
| Paketi tamamen silmek | Tutar, zaman, işletme ve kurye istatistikleri, hakediş / kasa (İterasyon 4) ve denetim kaydı kaybolur; KVKK'nın istediği kimliksizleştirmedir, kayıt silme değil |
| Telefonu kısmen maskelemek (son 4 hane) | Hâlâ kişisel veri sayılabilir; ayrıca işlevi yok |
| Tam adres satırını korumak, yalnızca adı silmek | Adres tek başına kişiyi belirler (konut); amacı karşılamaz |
| Koordinatı tamamen silmek | Mesafe / ısı haritası / geofence istatistikleri ve öğrenme doğrulaması bozulur; ~100 m yuvarlama ev ayırt etmeden bu işlevi korur |
| Kiracı yerine sunucu ortam değişkeni | Firma bazında hukuki fark (sözleşme) ve acil kapatma için ayar kiracıda olmalı |
| Dış anonimleştirme servisi / araç | Ücretli servis kuralı; iş zaten tek SQL grubudur |

## Uygulama notları

- **Kod:** `services/api/src/privacy/` (`privacy.service.ts`, `privacy-jobs.service.ts`, `privacy.controller.ts`, `area-label.ts`); migration `0009_musteri_verisi_saklama.sql`; panel `apps/admin-web/src/pages/settings/PrivacySettings.tsx`.
- **Test:** `services/api/test/privacy.e2e-spec.ts` (ayar doğrulaması ve roller; iki kiracıda yalnızca uygun paketler ve kiracının kendi süresi; olay / devir notu temizliği; adres defteri birebir aynı; özet ve geofence metrikleri değişmez; arama bulamaz; log kişisel veri içermez; kilit; işletme sayfası; 6. istekte 429), `area-label.spec.ts`, panel `privacy.test.tsx`, `privacy.test.ts`.
- **Hukuki açık nokta:** gerçek süre (30 gün korunacak mı?), yedek süresinin aydınlatma metnine yazılması ve VERBİS kaydı hukuki görüşe bağlıdır ([MIMARI.md](../MIMARI.md) açık soru 38).
