# ADR-0026: Teslim kanıtı geofence politikası: ölçüm kipiyle başlangıç, etkin teslim noktasına tolerans, gerekçeli geçersiz kılma, öğrenmeden dışlama

- **Durum:** Kabul edildi (ürün sahibi, 2026-11-06; uygulandı PR #34 + #37)
- **Tarih:** 2026-11-06
- **İlgili:** [ADR-0010](0010-kuryeler-arasi-devir.md) (buluşma yarıçapı 150 m), [ADR-0017](0017-saha-verisinden-ogrenen-harita.md) A, [ADR-0020](0020-cevrimdisi-eylem-kuyrugu-ve-idempotency.md), PLAN 3b-4, SPEC-3b §9, İterasyon 2 SPEC §13 (`DeliveryVerifier`)

## Bağlam

"Teslim ettim" bugün konumdan bağımsız kabul edilir (`AlwaysOkDeliveryVerifier`). Uzaktan basılan teslimler hem müşteri şikâyetinde kanıtı zayıflatır hem de öğrenilmiş teslim noktalarını (ADR-0017 A) kirletir. Öte yandan şehir içi GPS hatası, site / kampüs girişleri ve "müşteri aşağı indi" durumları gerçektir; kuryeyi kilitleyen bir kural sahada güveni bozar. Teslim çevrimdışı kuyruktan da gelebilir (ADR-0020) ve sahadaki 3a APK'ları geçersiz kılma alanını bilmez.

## Karar

1. **Üç kip (kiracı ayarı `delivery.geofenceMode`):** `OFF`, `OBSERVE` (varsayılan: ölç ve kaydet, engelleme), `ENFORCE`. ENFORCE'a geçiş, tüm kuryeler 3b uygulamasına geçtikten ve OBSERVE'deki olası geçersiz kılma oranı görüldükten sonra ürün sahibinin kararıdır.
2. **Hedef ve tolerans:** hedef paketin **etkin teslim noktasıdır** (öğrenilmiş / sabitlenmiş nokta, yoksa geocoder noktası). Hedef öğrenilmiş veya sabitlenmişse tolerans `geofenceM` (150 m, 50-500), geocoder noktasıysa `lowConfidenceToleranceM` (250 m, ≥ geofenceM). Mesafeden GPS doğruluğu (en çok 50 m) düşülür.
3. **Kuryeyi kilitlememe:** GPS doğruluğu > 50 m ise engellenmez (yalnızca uyarı ve işaret). Tolerans dışında veya konum yoksa (ENFORCE) kurye **gerekçeyle** teslim eder: `CUSTOMER_MET_ELSEWHERE`, `GPS_ERROR`, `OTHER` (+ zorunlu not). Geçersiz kılma her zaman kabul edilir; arayüzde iki ayrı bilinçli adımdır (uyarı, sonra gerekçe + kaydırarak onay).
4. **Denetim ve öğrenme:** sonuç DELIVERED olayının `data.geofence` alanına yazılır (kip, sonuç, mesafe, tolerans, hedef kaynağı, doğruluk, sahte konum işareti, gerekçe). Geçersiz kılınan, OBSERVE kipinde tolerans dışında kalan (`OUTSIDE_OBSERVED`), düşük doğruluklu, sahte konumlu veya konumsuz teslimler öğrenilmiş teslim noktasına **gözlem olarak alınmaz**.
5. **Çevrimdışı:** cihaz aynı kuralı olay anı konumu ve önbellekteki hedefle önceden uygular (geçersiz kılma kuyruğa yazmadan önce istenir); kesin karar sunucudadır. Sunucu 422 `OUTSIDE_DELIVERY_GEOFENCE` dönerse kayıt sessizce atılmaz: kurye gerekçe seçer, kayıt yeni `Idempotency-Key`, özgün `occurredAt` ve konumla yeniden gönderilir.
6. **Metrik adres kalitesi içindir:** geçersiz kılma / dışarıda oranı işletme ve adres (adres defteri girdisi) bazında raporlanır; **kurye bazlı rapor üretilmez** (amaç sınırlılığı; ihtiyaç olursa ürün sahibi + hukuk kararı). Pilotun ilk iki haftasında oran > %15 ise eşik ve öğrenilmiş nokta mantığı gözden geçirilir.
7. Geofence yalnızca DELIVER'a uygulanır; PICKUP serbesttir. Paket zaten hedef durumdaysa (yeniden deneme) geofence uygulanmaz.

## Sonuçlar

- (+) Kanıt: her teslimin hedefe uzaklığı ve gerekçesi kayıtlı; müşteri şikâyetinde ve adres düzeltmede kullanılır.
- (+) Öğrenme verisi temiz kalır; yüksek geçersiz kılma oranı yanlış adres noktalarını panelde görünür kılar.
- (+) OBSERVE varsayılanı eski APK'ları kilitlemez ve ENFORCE kararı ölçümle verilir.
- (−) Kurye gerekçeyle her zaman teslim edebilir; geofence bir hile önleyici değil, kanıt ve veri kalitesi aracıdır. `accuracyM`'i yüksek gönderen değiştirilmiş istemci denetimi atlatabilir (işaretlenir, sayılır).
- (−) ENFORCE'ta dışarıdaki her teslim iki ek dokunuş ister.
- (−) Ölçüm ile gerçek arasında kayma: geocoder noktası yanlışsa ilk teslimlerde gereksiz uyarı çıkar (250 m tolerans bunu azaltır).

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Doğrudan ENFORCE ile açmak | 3a APK'ları geçersiz kılma gönderemez, kilitlenir; eşik ölçülmeden dayatılır |
| Dışarıdaki teslimi tamamen reddetmek (geçersiz kılma yok) | GPS hatası ve "müşteri aşağı indi" durumlarında kurye kilitlenir; PLAN kabulüne aykırı |
| Düşük doğrulukta da engellemek | Kötü GPS yüzünden kurye kilitlenir (PLAN kabul c) |
| Tek sabit tolerans (hedef kaynağından bağımsız) | Geocoder noktası yanlışsa gereksiz uyarı çoğalır; öğrenilmiş noktada 250 m gereksiz gevşek |
| Kurye bazlı geçersiz kılma raporu | Performans izlemesine dönüşür (KVKK, ADR-0017 amaç sınırlılığı) |
| Fotoğraf / imza / kod kanıtı | Pilot ihtiyacı belirsiz (MIMARI açık soru 8); ayrı iş |

## Uygulama notları — API

- **Kod:** saf `decideGeofence` (`services/api/src/packages/geofence.ts`) ve `GeofenceDeliveryVerifier` (`delivery-verifier.ts`). `courierAction` DELIVER akışı: rota kilidi -> paket satırı kilidi -> yalnızca paket bu kuryede `PICKED_UP` ise geofence -> bugünkü teslim. Hedef durumdaki yeniden deneme ve çakışmalar 422 almaz; bugünkü 400 VALIDATION_FAILED dalı kaldırıldı, PICKUP serbesttir.
- **422:** `OUTSIDE_DELIVERY_GEOFENCE` (`distanceM` 10 m yuvarlanmış, konum yoksa `null`; `requiredM`; `targetSource`). Kesici 422'yi sakladığı için aynı anahtar aynı yanıtı döner; geçersiz kılma yeni anahtarla gönderilir (`geofence.e2e-spec.ts`). Gövde farklıysa (ör. `recordedAt` değişmişse) kesici `IDEMPOTENCY_KEY_REUSED` döner; istemci yeniden gönderimde özgün `position` ve `occurredAt`'i aynen kullanmalıdır (M3b'ye not).
- **Olay verisi:** her DELIVER'da `data.geofence` (OFF dahil: `DISABLED`, `distanceM: null`). `Package.deliveryCheck` son DELIVERED olayından okunur. `GET /metrics/delivery-geofence` kurye kırılımı içermez; ölçülen teslim = INSIDE + dışarıda + konumsuz (DISABLED ve LOW_ACCURACY ölçüm dışı).
- **Öğrenme ilişkisi:** geçersiz kılma, `OUTSIDE_OBSERVED`, düşük doğruluk, sahte konum, konumsuz ve bayat konumlu teslimler gözlem olmaz (`observation-filter.ts`). Ortak test vektörleri: `services/api/test/fixtures/geofence-vectors.json` (31 durum; mobil kopyalar).
- Varsayılan `OBSERVE`; ENFORCE'a geçiş çalışma kitabı adımıdır (tüm kuryeler 3b APK'sına geçince).

## Uygulama notları — Mobil

İterasyon 3b-4, kurye uygulaması (M3b). Ölçüm kaynağı: `Medium_Phone_API_36.1` emülatörü (release APK, x86_64) + yerel mock sunucu (`MOCK_GEOFENCE_MODE=ENFORCE`, `/__mock/geofence?mode=`); gerçek API (PR #34) birleşmediği için bu doğrulama mock ile yapıldı.

- **Cihazda karar:** `src/lib/geofence.ts` `decideGeofence`, sunucuyla ortak vektör dosyasını (`geofence-vectors.json`) geçer. Hedef sırası: paketin `deliveryTarget` (konum + `toleranceM`), yoksa rota durağının etkin konumu ve `locationSource`'u, yoksa geocoder noktası (`lowConfidenceToleranceM`). Ayarlar önbellekteki `tenant.settings.delivery`'den okunur, bu yüzden aynı karar **çevrimdışıyken de** verilir.
- **Tek yol:** rota kartı, navigasyon varış kartı ve paket ayrıntısındaki "Teslim ettim" aynı `deliverWithGeofence` akışını kullanır. ENFORCE + dışarıda (ya da konumsuz): tam ekran sayfa ("Teslim noktasına git" birincil, "Vazgeç", "Yine de teslim ettim"), sonra üç büyük gerekçe ve kaydırarak onay. Gerekçe seçilmeden kaydırma etkinleşmez; "Vazgeç" hiçbir şeyi kuyruğa yazmaz. OBSERVE, OFF, INSIDE ve düşük doğruluk (> 50 m) engellemez: yalnızca bilgi şeridi (düşük doğrulukta uyarı) gösterilir.
- **Kuyruk:** `classify` 422 `OUTSIDE_DELIVERY_GEOFENCE` yanıtını `NEEDS_OVERRIDE` yapar (`action_queue.state` genişledi; `distanceM`, `requiredM` kayda yazılır). Gönderici bu kaydı atlar, sıradaki kayıtları sürdürür. Kalıcı kart ("Teslim onay bekliyor") kapatılamaz; tek çıkış gerekçe seçmek ya da "Vazgeç" ile bilinçli iptaldir (paket yolda kalır). Gerekçeyle yeniden gönderim **yeni `Idempotency-Key`**, özgün `occurredAt` ve özgün `position` ile yapılır (emülatörde doğrulandı: sunucu günlüğünde özgün olay zamanı, yeni anahtar).
- **Çevrimdışı gerekçe:** uçak modunda 1,9 km uzaktaki teslimde önbellekteki ayarla uyarı sayfası açıldı, gerekçe + kaydırma sonrası "Kaydedildi, bağlanınca gönderilecek"; bağlantı gelince kayıt `geofenceOverride` ile tek seferde kabul edildi.
- **Sunucu 422'si (cihaz eski ayarla OBSERVE sanıyordu):** mock'ta kip çalışırken ENFORCE'a çevrildi; teslim 422 aldı, gerekçe sayfası doğrudan ikinci adımda açıldı, kapatınca kalıcı kart kaldı, gerekçeyle gönderim kabul edildi.
- **Hazır durumu:** `ready-status` kuyruğa girmez; çevrimdışıyken düğmeler devre dışı ve "Bağlantı gerekli" yazar. "Hazır değil" etiketi kalan süreyi yuvarlar (yukarı yuvarlama bildirimden hemen sonra "~6 dk" gösteriyordu).
- **Gözlem:** hiç açılmamış bir paketin ayrıntısı çevrimdışıyken önbellekte yoktur ("Sunucuya ulaşılamıyor"); rota kartı çevrimdışı çalışır. Bu, 3a'dan gelen davranıştır ve M3b kapsamında değiştirilmedi.

