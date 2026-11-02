# ADR-0013: Rota motoru sınırı, yaklaşık (düşüş) modu ve süreç içi kesin sıralayıcı

- **Durum:** Önerildi (İterasyon 2 spesifikasyonuyla birlikte onaya sunuldu)
- **Tarih:** 2026-11-02
- **İlgili:** [ADR-0009](0009-rota-uyumlu-atama.md) madde 8 (saf fonksiyonlar + `TravelTimeProvider`), [ADR-0011](0011-osrm-motosiklet-profili-ve-rota-altyapisi.md), İterasyon 2 SPEC §6, §7, §11

## Bağlam

- ADR-0009, yerleştirme ve skor mantığının G/Ç içermeyen saf fonksiyonlar olmasını ve benchmark ile API'nin aynı kodu çalıştırmasını istiyor.
- İterasyon 2 paralel geliştirilecek: rota motoru (Görev R) ile API alan modülleri (Görev A2, H2) aynı anda yazılacak. İkisinin buluştuğu sınır kesin olmalı ki biri diğerini beklemesin.
- ROTA-UYUMLU-ATAMA taslağı OSRM erişilemezse öneri ucunun `422 ROUTING_UNAVAILABLE` dönmesini öneriyordu. Ürün sahibi ise "OSRM/VROOM düşerse İterasyon 1 yarıçap / elle atama davranışına açık bir işaretle dönülmesini" istiyor.
- Taslak, VROOM'da kilitli durağı araç `steps` alanıyla sabitlemeyi öneriyordu. VROOM'da `steps` esas olarak plan modu (`-c`) için tanımlıdır ve sürümler arasında çözme modundaki anlamı değişmiştir.
- Taslak durak-durak süre bloklarını kurye başına Redis'te tutmayı öneriyordu.

## Karar

**1. Sınır: motor saf bir kütüphanedir; arayüz sözleşme paketindedir.**
- Arayüz `packages/contracts/src/routing-engine.d.ts` (yalnızca tip, `@kurye/contracts/routing-engine`). İstemcilere açık değildir.
- Uygulama `services/api/src/routing/` altındadır ve **Nest, veritabanı, Redis veya WebSocket bilmez**: girdi olarak kurye anlık görüntüleri (`CourierSnapshot`) alır, sonuç döner. Fabrika: `createRoutingEngine(config, deps)`.
- Nest bağlama, veritabanı okuma/yazma, kilitler, WS yayını ve iş kuralları (modlar, ret, devir) API alan modüllerinindir (A2, H2). A2, R birleşene kadar aynı arayüzü uygulayan `ScriptedRoutingEngine` ile çalışır.
- Benchmark (`services/api/bench/`) motoru doğrudan, önceden hesaplanmış bir matris sağlayıcısıyla çalıştırır.

**2. Süre önbelleği süreç içidir; bacak süreleri rota kaydıyla saklanır.**
- Mevcut dizinin değerlendirilmesi için gereken ardışık bacak süreleri `route_stops.leg_duration_s` olarak kalıcıdır (her plan yazımında güncellenir).
- Yeni paket için gereken süreler iki ince OSRM `table` çağrısıyla (düğümler -> {R, D, ilk duraklar} ve {R, D} -> düğümler) alınır.
- Motor içinde nokta çifti (5 ondalık haneye yuvarlanmış koordinatlar) anahtarlı bir LRU önbellek bulunur (varsayılan 200.000 kayıt, 6 saat). Redis'te kurye başına matris bloğu **tutulmaz**: bacaklar zaten kalıcı, LRU aynı kuryeler için tekrar eden çağrıları keser, çok örnekli API'de tutarlılık sorunu çıkmaz.

**3. Yaklaşık mod (düşüş davranışı) hata değil, işaretli sonuçtur.**
- Motor OSRM veya VROOM'a erişemezse (zaman aşımı veya ağ hatası; ardışık 3 hata sonrası 10 sn devre kesici) hata fırlatmaz; **Haversine sağlayıcısına** düşer (kuş uçuşu x 1,35 / 22 km/sa) ve her sonuçta `routingMode = APPROXIMATE` döner. Kiracıya `routing.status = DEGRADED` yayınlanır.
- Yaklaşık modda iş kuralları İterasyon 1 davranışına döner: OTOMATIK atama askıya alınır ve yeni paketler havuza açılır (yarıçap süzgeci), AKILLI_HAVUZ yarıçap süzgecine düşer, öneriler "(yaklaşık)" etiketiyle gösterilmeye devam eder, yeni duraklar rotanın sonuna eklenir (`APPEND_FALLBACK`), panelde ve uygulamada şerit görünür. Elle atama hiçbir zaman engellenmez.
- `ROUTING_ENABLED=false` aynı yoldan `DISABLED` durumunu üretir. Böylece demo rota verisi olmadan da çalışır.

**4. Nihai sıralayıcı iki uygulamalıdır.**
- **VROOM** (üretim): pickup-delivery `shipments`, alınmış paketler için `jobs` (`delivery`), kapasite, `readyAt` zaman pencereleri, alınmış paketler için tazelik son tarihi (teslim zaman penceresi).
- **EXACT** (süreç içi): öncelik kuralına uyan tüm dizileri derinlik öncelikli arama ve dal-sınır ile tarar; en çok 10 durak (5 paket). VROOM erişilemezken ve benchmark'ta kullanılır; deterministiktir ve testlerde VROOM sonucunun doğrulanmasına yarar.
- Kilitli durak sıralama problemine **verilmez**: araç başlangıcı kilitli durağın konumu ve tahmini bitiş anı, başlangıç yükü kilitli durak sonrası yük olarak verilir; sonuç kilitli durakla birleştirilir. Bu, VROOM'un `steps` davranışına bağımlılığı kaldırır.
- VROOM'un yerel olarak zorlamadığı "alıştan teslime azami süre" kısıtı motorun kendi değerlendirmesiyle doğrulanır; ihlal varsa sonuç atılır.

**5. Kayıt biçimleri.**
- Kurye başına **tek** `courier_routes` satırı (birincil anahtar `courier_id`), `version` sürekli artar; tarihsel rota satırı tutulmaz (geçmiş `route_stops` `DONE` / `SKIPPED` satırlarındadır).
- Öneri hesabı **tek satır + JSONB aday listesi** olarak saklanır (`assignment_suggestions.candidates`): okuma her zaman kümenin tamamıdır, metrikler için `best_courier_id` ve `best_detour_s` ayrı kolondur.

## Sonuçlar

- (+) R, A2 ve H2 aynı gün paralel başlayabilir; tek ortak nokta tip dosyasıdır ve `tsc` ile doğrulanır.
- (+) Benchmark üretim koduyla birebir aynı algoritmayı ölçer.
- (+) Rota motoru düştüğünde operasyon durmaz; ekip ne olduğunu şeritten görür. Elle atama her zaman mümkündür.
- (+) Kilitli durağın problem dışına alınması VROOM sürüm farklarından bağımsızdır.
- (-) Yaklaşık mod süreleri kaba tahmindir; bu sırada verilen elle kararların kalitesi düşer (kabul edilen risk).
- (-) EXACT sıralayıcı 10 durağın üzerinde kullanılamaz; kapasite 5'in üzerine çıkarılırsa VROOM yokken sıra eklenme sırasında kalır.
- (-) Süreç içi LRU her API örneğinde ayrıdır; çok örnekte OSRM çağrı sayısı örnek sayısıyla artar (bu ölçekte önemsiz).

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Motoru ayrı bir mikroservis olarak çalıştırmak | Ağ sıçraması 300 ms bütçesini yer, ek dağıtım birimi; pilot ölçeğinde gereksiz |
| Motoru ayrı bir workspace paketi (`packages/routing-engine`) yapmak | API Nest derlemesi workspace TS kaynağını derlemez; ayrı derleme adımı geliştirme akışını yavaşlatır. Arayüz zaten sözleşme paketinde, uygulama API içinde yeterli |
| OSRM yokken `422 ROUTING_UNAVAILABLE` (taslak) | Paneli ve modları durdurur; ürün sahibinin istediği "İterasyon 1 davranışına işaretle dönüş" değil |
| Kurye başına Redis matris bloğu (taslak) | Sürüm geçersizleştirme karmaşası; bacak süreleri zaten kalıcı, LRU yeterli |
| Kilitli durağı VROOM `steps` ile sabitlemek (taslak) | Plan moduna özgü, sürüme bağlı davranış |
| Öneri adayları için ayrı satırlar (taslak) | Okuma her zaman tüm küme; satır başına yazım maliyeti ve şema karmaşası |
