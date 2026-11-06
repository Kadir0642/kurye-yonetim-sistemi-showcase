# ADR-0026: Teslim kanıtı geofence politikası: ölçüm kipiyle başlangıç, etkin teslim noktasına tolerans, gerekçeli geçersiz kılma, öğrenmeden dışlama

- **Durum:** Önerildi (İterasyon 3b spesifikasyonuyla birlikte onaya sunuldu)
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

_(Görev Q3b doldurur: sunucu tarafı uygulama, ölçülen davranış, sapmalar.)_

## Uygulama notları — Mobil

_(Görev M3b doldurur: cihazda ön uyarı, geçersiz kılma akışı, emülatör / cihaz gözlemleri.)_
