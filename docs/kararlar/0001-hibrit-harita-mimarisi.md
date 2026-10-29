# ADR-0001: Hibrit harita mimarisi

- **Durum:** Önerildi (pilot müşteri cevapları ve Google şartları doğrulaması bekliyor)
- **Tarih:** 2026-10-30

## Bağlam

Sistemin dört harita ihtiyacı var: adres girişi, kurye haritası, rota matrisi / durak optimizasyonu ve navigasyon başlatma. Araştırmaya göre (teknik rapor), günde yaklaşık 1500 paket (~45 bin paket/ay) için yalnızca Google kullanılırsa aylık maliyet yaklaşık 0,6-0,8 bin USD civarındadır; bunun büyük kısmı Route Optimization (~400 USD) ve Place Details (~175 USD) kalemlerinden gelir. Mesafe matrisini Google ile kurmak ise aylık binlerce dolara çıkar ve uygulanabilir değildir. Türkiye adres kalitesinde Google belirgin avantajlıdır; açık kaynak geocoder'lar kapı numarası düzeyinde daha zayıf olabilir.

## Karar

Aşağıdaki hibrit yapı önerilir:

1. **Adres girişi:** Google Places Autocomplete, **session token** ile; oturum bir **Place Details (Essentials)** çağrısıyla kapatılır. Autocomplete oturum kullanımı ücretsizdir; Place Details Essentials 10 bin istekten sonra ücretlidir.
2. **Kurye haritası:** Maps SDK for Android (ücretsiz). Yönetim panelinde Maps JavaScript API (belirli hacme kadar ücretsiz).
3. **Rota matrisi ve durak optimizasyonu:** Self-hosted **OSRM + VROOM**. Google Route Optimization (~400 USD/ay ve üzeri) maliyetli bulunduğundan seçilmedi.
4. **Navigasyon başlatma:** Durak başına `google.navigation:` intent'i (tek hedef; uygulama durak durumunu kendisi izleyebilir). İsteğe bağlı olarak çok duraklı Google Maps URL'si (en fazla 9 ara nokta, parçalı gönderim).
5. **ETA:** Önce OSRM tahmini; trafik duyarlı ETA gerekirse yalnızca bu amaçla Google Routes.

## Sonuçlar

- (+) Aylık harita maliyeti yaklaşık 175 USD seviyesine (Place Details) iner; matris / optimizasyon maliyeti sunucu kirasıyla (4-8 GB RAM VPS, ~20-40 USD/ay) sınırlıdır.
- (+) Adres kalitesi Google ile korunur.
- (-) Tek hedefli navigasyon, kuryenin "her durakta" uygulamaya dönmesini gerektirir; arada bir dokunuş ekstra olur.
- (-) OSRM canlı trafik bilmez; ETA'lar yaklaşık olur.
- (-) Ek altyapı bakımı: OSM verisi (Geofabrik Türkiye) güncellemesi, OSRM / VROOM servisleri.
- **Açık risk:** Google şartları Places / Geocoding koordinatlarının önbelleğini sınırlar (yaklaşık 30 gün; yalnızca `place_id` süresiz saklanabilir) ve Google verisinin Google dışı harita ile kullanımını kısıtlayabilir. OSRM'ye giren koordinatların kaynağı ve saklama şekli hukuken netleştirilmeli. Yedek plan: yalnızca Google ile Single Vehicle Routing (maliyet artışı kabul edilirse).
- **Açık risk:** Türkiye OSM'de kapı numarası ve kırsal yol kalitesi doğrulanmadı; pilot bölgede ölçülmeli.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Yalnızca Google (Places + Route Optimization) | Aylık yaklaşık 400 USD ve üzeri; her yeniden optimizasyon maliyeti çarpar |
| Google Route Matrix ile kendi optimizasyonu | 25 kurye x 60 durak için ~2,7 milyon eleman/ay, yaklaşık 6 bin USD |
| Tamamen açık kaynak (Nominatim / Photon + OSRM) | Türkçe adres kalitesi Google'dan zayıf bekleniyor, doğrulanamadı |
| Valhalla + VROOM | Uygun alternatif; OSRM matris hızı nedeniyle önce OSRM tercih edildi |
| GraphHopper (barındırılan optimizasyon) | Ücretsiz plan ticari kullanıma kapalı; ücretli planlar araç sayısıyla sınırlı |
| Yandex Navi / Waze ile çoklu durak | Waze'de ara durak yok; Yandex'te uygulama tanıtım koşulları ve düşük pazar payı |
