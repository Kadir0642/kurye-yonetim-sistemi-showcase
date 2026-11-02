# ADR-0008: Açık kaynak harita gösterimi (MapLibre + OpenFreeMap) ve sağlayıcı arkasında adres arama

- **Durum:** Kabul edildi
- **Tarih:** 2026-10-30
- **Değiştirdiği karar:** [ADR-0001](0001-hibrit-harita-mimarisi.md) madde 2 (Maps SDK / Maps JavaScript API ile gösterim) ve madde 1'in uygulama biçimi

## Bağlam

- Demo aşamasında ücretli anahtar olmadan çalışmak gerekiyor: ürün yalnızca pilot müşteri beğenirse canlıya alınacak.
- Google Maps JavaScript API dinamik harita yüklemeleri belirli bir hacimden sonra ücretlidir ve anahtar ister. Maps SDK ücretsizdir ama anahtar ve Google hesabı ister. Google verisinin Google dışı haritada gösterimi ise şartlarla kısıtlanabilir ([ADR-0001](0001-hibrit-harita-mimarisi.md) açık riski).
- Kurye uygulaması artık Android ve iOS'ta çalışan React Native + Expo uygulamasıdır ([ADR-0002](0002-kurye-uygulamasi-react-native-expo.md)); harita kütüphanesi iki platformu da desteklemelidir.
- Türkiye'de kapı numarası düzeyinde adres kalitesinde Google belirgin üstündür. Açık kaynak geocoder'lar (Photon, Nominatim) daha zayıf beklenir ama demo için yeterlidir.

## Karar

**1. Harita gösterimi: MapLibre + OpenFreeMap vektör karoları (ücretsiz, anahtarsız).**
- Panel: MapLibre GL JS. Kurye uygulaması: `@maplibre/maplibre-react-native` (Android ve iOS'ta MapLibre Native).
- Stil adresi tek yerden gelir: API yapılandırması `MAP_STYLE_URL` (varsayılan `https://tiles.openfreemap.org/styles/liberty`), istemcilere `GET /tenant` yanıtında `mapStyleUrl` olarak verilir. Karo sağlayıcısını değiştirmek (ör. ileride kendi karo sunucumuz) istemci değişikliği gerektirmez.
- OSM / OpenFreeMap atıf metni haritada her zaman görünür kalır.

**2. Adres arama ve otomatik tamamlama yalnızca API üzerinden (`/geo/...`), bir sağlayıcı arayüzü arkasında.**
- **GooglePlacesProvider:** `GOOGLE_MAPS_API_KEY` tanımlıysa kullanılır. Places API (New) Autocomplete, **session token** ile çağrılır. İstek parametreleri: `includedRegionCodes: ["tr"]`, `languageCode: "tr"`, `locationRestriction.rectangle` (kiracının `service_area` sınırlayıcı kutusu, [ADR-0006](0006-hizmet-bolgesi-eskisehir.md)). Oturum, Place Details (Essentials alan maskesi: `id,formattedAddress,location`) ile kapatılır. Ters geocode için Geocoding API kullanılır.
- **PhotonProvider:** anahtar yoksa kullanılır. Adres `photon.komoot.io` üzerinden, `bbox` = hizmet bölgesi ile aranır. Genel Photon sunucusu **yalnızca geliştirme ve demo** içindir (adil kullanım): istemci tarafında 300 ms debounce ve en az 3 karakter, sunucu tarafında kullanıcı başına hız sınırı ve Redis önbelleği uygulanır, uygulamayı tanımlayan `User-Agent` gönderilir. Canlıda ya Google anahtarı kullanılır ya da kendi Photon örneğimiz barındırılır.
- Seçim: `GEO_PROVIDER=auto|google|photon` (varsayılan `auto`).
- İstemciler hiçbir harici geocoding servisini doğrudan çağırmaz ve anahtar taşımaz.

**3. Navigasyona devir uygulamadan yapılır** (harita uygulaması kuryenin telefonundaki uygulamadır):
- Android: `google.navigation:q=lat,lng&mode=l` (iki tekerlekli mod; araç tipine göre `l`/`b`/`d`/`w`). Açılamazsa `geo:lat,lng?q=lat,lng` URI'si kullanılır.
- iOS: Google Maps kuruluysa `comgooglemaps://?daddr=lat,lng&directionsmode=...`, değilse Apple Haritalar `maps://?daddr=lat,lng`.

**4. Rota ve optimizasyon (OSRM + VROOM) İterasyon 2'dedir**; İterasyon 1'de mesafeler kuş uçuşu (PostGIS `geography`) hesaplanır.

## Sonuçlar

- (+) Demo sıfır harita maliyetiyle ve anahtarsız çalışır; Google anahtarı eklendiğinde kod değişmeden daha kaliteli adres önerilerine geçilir.
- (+) Google verisi Google dışı haritada gösterilse bile kalıcı olarak yalnızca `place_id` ve paketin operasyonel koordinatı saklanır; Google yanıtları önbelleğe alınmaz (ToS riski azaltılır, tamamen kapanmaz: ADR-0001 açık riski sürer).
- (+) Anahtar yalnızca sunucuda tutulur; istemcide sızdırılamaz.
- (+) Panel ve iki mobil platform aynı harita stilini kullanır.
- (-) OpenFreeMap üçüncü taraf, ücretsiz bir hizmettir; SLA yoktur. Canlıda kesinti riski için kendi karo barındırma (OpenFreeMap kendi barındırılabilir) yedek plandır.
- (-) Photon'un Türkçe ve kapı numarası kalitesi Google'dan zayıftır. Panelde haritada iğneyi sürükleyerek konum düzeltme (`/geo/reverse`) bu zayıflığı telafi eder.
- (-) Her adres araması API'den geçtiği için API'ye küçük bir gecikme eklenir (önbellekle azaltılır).

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Google Maps JS API + Maps SDK (ADR-0001 madde 2) | Anahtar ve faturalama hesabı gerekir; demo aşamasında gereksiz maliyet ve kurulum |
| Mapbox | Ücretli, anahtar gerekir; GL JS v2+ özgür lisanslı değil |
| Raster OSM karoları (tile.openstreetmap.org) | Ticari ve yoğun kullanım politikası kısıtlı; vektör karo daha hızlı ve hafif |
| İstemciden doğrudan Places çağrısı | Anahtar istemciye gömülür; sağlayıcı değiştirmek istemci sürümü gerektirir |
| Nominatim | Yazarken arama (autocomplete) için uygun değil; Photon bu iş için tasarlanmış |
