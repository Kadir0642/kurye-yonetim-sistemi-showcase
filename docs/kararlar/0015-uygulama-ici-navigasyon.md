# ADR-0015: Uygulama içi adım adım navigasyon (kendi hafif navigasyonumuz: OSRM adımları + MapLibre + Türkçe sesli yönlendirme)

- **Durum:** Kabul edildi (ürün sahibi onayı, 2026-11-03)
- **Tarih:** 2026-11-03
- **İterasyon:** İterasyon 2 (ek kapsam; "Rotam" ekranının "Navigasyon" düğmesinin yerine geçer, rota uyumlu atamanın devamıdır)
- **Değiştirdiği karar:** [ADR-0008](0008-acik-kaynak-harita.md) madde 3 (navigasyona devir). Devir **yedek yol** olarak kalır, birincil yol artık uygulama içi navigasyondur.
- **İlgili:** [ADR-0002](0002-kurye-uygulamasi-react-native-expo.md), [ADR-0005](0005-konum-takibi-ve-kvkk.md), [ADR-0011](0011-osrm-motosiklet-profili-ve-rota-altyapisi.md), [ADR-0013](0013-rota-motoru-siniri-ve-yaklasik-mod.md), N2 görev brief'i

## Bağlam

- Bugün "Navigasyon" düğmesi kuryeyi Google Maps'e (veya Apple Haritalar'a) atar ([ADR-0008](0008-acik-kaynak-harita.md) madde 3). Kurye uygulamadan çıkar; uygulama durağa varıp varmadığını, kuryenin yoldan sapıp sapmadığını ve ne zaman "Aldım / Teslim ettim" kartının açılacağını bilemez. Pilot müşterinin kuryeleri her durakta uygulamaya dönüp kartı elle bulmak zorunda.
- Ürün sahibi, "Navigasyonu başlat"ın Google Maps gibi çalışan **uygulama içi** bir ekran açmasını onayladı: üstte yeşil bir bant (manevra oku, manevraya kalan mesafe, sokak adı, sonraki manevranın önizlemesi), kuryeyi takip eden eğik ve yön-yukarı kamera, altta kalan süre / km / varış saati / "Çıkış" çubuğu, Türkçe sesli yönlendirme ve yoldan çıkınca otomatik yeniden rota. Varışta durak kartı açılır; işlemden sonra sonraki durağa navigasyon kendiliğinden başlar.
- Rota altyapısı zaten var: kendi barındırdığımız OSRM, depodaki motosiklet profiliyle ([ADR-0011](0011-osrm-motosiklet-profili-ve-rota-altyapisi.md)). OSRM `route` ucu `steps=true` ile manevra listesini (tür, yön, konum, sokak adı, mesafe, süre, dönel kavşak çıkış numarası) verir. Eksik olan: bu listeyi Türkçe, kısa, doğal cümlelere ve zamanlı sesli anonslara çevirmek ve uygulamada tüketmek.
- Kısıtlar: ücretli anahtar olmadan çalışmalı (demo ilkesi, [ADR-0008](0008-acik-kaynak-harita.md)); Expo ile tek kod Android + iOS ([ADR-0002](0002-kurye-uygulamasi-react-native-expo.md)); düşük donanımlı telefonlarda ve düşük pil ile akıcı kalmalı.

## Karar

**1. Kendi hafif navigasyonumuzu yazarız: sunucuda adım üretimi, uygulamada takip.**
- **Sunucu (`services/api/src/navigation`):** `POST /courier/navigation/route` kuryenin anlık konumundan hedef durağa OSRM `route` (`steps=true`, `overview=full`, motosiklet profili) çağırır, sonucu sözleşmedeki `NavigationRoute` biçimine çevirir: tam geometri, adımlar, her adım için Türkçe talimat (`instructionTr`) ve zamanlı sesli anonslar (`voiceAnnouncements`). Türkçe üretimi sunucudadır (tek yerde, testli; uygulama metin üretmez, SPEC §7.6 ilkesiyle aynı) ve uygulama sürümünden bağımsız düzeltilebilir.
- **Uygulama:** MapLibre (rota çizgisi, kamera: `bearing` + `pitch`), konumu rotaya oturtma (snap), adım ilerlemesi, yoldan çıkma tespiti (>40 m süresince >5 sn), `expo-speech` `tr-TR` ile anonsları bir kez okuma, varış akışı. Ayrıntı: N2 brief'i.
- **Yeniden rota** aynı uçla yapılır (yeni konum + aynı hedef); yalnızca sunucu hesaplar, uygulama yerel rota hesaplamaz. Hız sınırı kurye başına dakikada 30 istektir.

**2. Sözleşme kararları (contract-first, `packages/contracts/openapi.yaml`).**
- **Geometri GeoJSON `LineString`** (polyline6 değil). Gerekçe: (a) sözleşmedeki mevcut `CourierRoute.geometry` ve MapLibre `ShapeSource` ile aynı biçim, uygulamada çözücü / ek bağımlılık yok; (b) adımların `geometryRange` dizinleri doğrudan `coordinates[i]` ile okunur; (c) şehir içi tipik rota (3-10 km) 200-800 noktadır, yani yaklaşık 10-40 KB; polyline6'nın yaklaşık 4 kat küçük olması bu ölçekte yeniden rotada hissedilir fark yaratmaz. Yük ileride sorun olursa aynı alan için sıkıştırma (gzip) veya `geometryEncoding` seçeneği geriye uyumlu eklenir.
- **Adım anlamı:** adım = "manevra + o manevradan sonraki yol". `steps[i].maneuver` adımın **başında** yapılacak işlemdir; `distanceM` / `durationS` o manevradan bir sonraki manevraya kadardır. Dolayısıyla sıradaki manevra `steps[i + 1].maneuver`, ona kalan mesafe `steps[i].distanceM` eksi o adımda alınan yoldur.
- **Sesli anonslar adımın içinde taşınır** (Mapbox `voiceInstructions` biçimi): `steps[i].voiceAnnouncements[].atDistanceM`, "i. adımın sonuna (= sıradaki manevraya) kalan mesafe bu değere indiğinde oku" demektir; metin sıradaki manevradır ("500 metre sonra sağa dönün"). İlk adımın ilk anonsu kalkış cümlesidir ve hemen okunur; son (varış) adımının tek anonsu `atDistanceM = 0` "Hedefinize ulaştınız"dır. Eşikler 500 / 150 / 30 m'dir ("manevrada" anonsu GPS gecikmesi nedeniyle 30 m'de, kısa buyruk cümlesiyle); kısa adımlarda fazla anons atlanır.
- **`routingMode`:** `EXACT` = konum ve hedef yol ağına 100 m'den yakın oturdu; `APPROXIMATE` = biri 100-3000 m uzağa oturdu (kampüs içi, özel yol), rota yol ağındaki en yakın noktadan hesaplandı, uygulama "Rota yaklaşık" ipucu gösterir ve yoldan çıkma tespitini gevşetir. 3000 m'den uzak oturma `422 NAVIGATION_ROUTE_NOT_FOUND` olur. Aynı alan, ileride OSRM dışı yedek üretici gelirse (ör. kuş uçuşu) onu da işaretler.
- **Hata sözleşmesi:** OSRM erişilemez / zaman aşımı / `ROUTING_ENABLED=false` => `503 NAVIGATION_UNAVAILABLE` (`Retry-After: 10`); OSRM rota bulamazsa (`NoRoute`, `NoSegment`) veya konum harita kapsamı dışındaysa `422 NAVIGATION_ROUTE_NOT_FOUND`. **İkisinde de uygulama Google Maps devrine düşer.** Başkasının durağı, başka kiracının durağı, olmayan durak: `404 NOT_FOUND`; `next: true` ve rotada durak yok: `404 NOT_FOUND`.
- Yalnızca `COURIER` rolü; hedef kuryenin **kendi** `PLANNED` durağıdır ya da `next: true` (rotasının sıradaki durağı). Kiracı yalnızca JWT'den gelir.
- Navigasyon sunucuda **durum tutmaz** (rota kaydedilmez); `routeId` günlük korelasyonu içindir. Süreler `ROUTING_DURATION_FACTOR` ile ölçeklenir (ADR-0011 madde 3), böylece "Rotam" ETA'sı ile navigasyon ETA'sı tutarlıdır.

**3. Pil ve kaynak.**
- Yüksek doğruluklu GPS (`Location.Accuracy.BestForNavigation`, 1 sn) ve ekranı açık tutma (keep-awake) yalnızca navigasyon ekranı açıkken; çıkışta eski arka plan izleme ayarlarına dönülür ([ADR-0005](0005-konum-takibi-ve-kvkk.md): sunucuya konum gönderimi aynı kalır, navigasyon kendi yerel konum akışını kullanır).
- Harita kamerası ve konum işaretçisi animasyonları sınırlıdır (düşük donanım için kare hızı bütçesi N2 brief'indedir); sesli anons metinleri sunucudan hazır geldiği için uygulamada metin üretimi / ağ çağrısı yoktur.

**4. Yedek: Google Maps devri korunur.** Navigasyon ekranı bir hata durumunda (503, 422, ağ yok) mevcut `handoff.ts` ile Google Maps / Apple Haritalar'ı açar; kurye ekranda "Uygulama içi navigasyon şu an kullanılamıyor, Google Maps açılıyor" bilgisini görür. Navigasyon ekranında ayrıca "Google Maps'te aç" ikincil eylemi bulunur; kurye uygulama içi navigasyonu beğenmezse istediği an devre geçer (N2 brief'i).

**5. Sınırlar (v1).**
- **Canlı trafik yok** ([ADR-0006](0006-hizmet-bolgesi-eskisehir.md)): süreler OSM hızlarına ve motosiklet profiline dayanır; `ROUTING_DURATION_FACTOR` ortalamayı düzeltir, yoğun saati düzeltmez. Google Maps ile karşılaştırıldığında ETA farkı beklenir.
- **Şerit bilgisi (lanes) v1 dışıdır.** OSRM `intersections[].lanes` verisini yalnızca OSM'de şerit etiketi olan kavşaklarda üretir ve Eskişehir'de seyrektir; yarım veri yanlış güven verir.
- **Hız sınırı v1 dışıdır (gelecek iş).** OSRM `annotations=speed` hesaplanan profil hızıdır, yasal sınır değildir; `maxspeed` etiketini yanıta taşımak profilde (`motorcycle.lua`) ek değişiklik ve veri kapsama ölçümü gerektirir. Pilotta talep edilirse ADR-0011 profiliyle birlikte ele alınır.
- Kavşak adı / çıkış tabelası (`destinations`, `exits`) v1'de kullanılmaz.
- Çok duraklı tek rota yoktur: her seferinde tek hedef (sıradaki durak); varıştan sonra sonraki durak için yeni istek yapılır.
- Dönel kavşaklarda OSRM'in ayrı verdiği `exit roundabout` / `exit rotary` adımları sunucuda önceki `roundabout` / `rotary` adımına katlanır (aynı talimat iki kez okunmasın diye); sözleşme enum'unda kalır ama yanıtta görünmez.

## Sonuçlar

- (+) Kurye uygulamadan çıkmaz: varış, sapma ve "Aldım / Teslim ettim" akışı uygulamanın kontrolündedir; durak takibi ve sonraki durağa otomatik geçiş mümkündür.
- (+) Sıfır lisans ve harita maliyeti: self-hosted OSRM + MapLibre + cihaz üstü TTS. Motosiklet profili ve süre çarpanı navigasyona da yansır.
- (+) Türkçe tek yerde üretilir ve birim testlidir; yanlış telaffuz veya dilbilgisi düzeltmesi uygulama güncellemesi gerektirmez.
- (+) Altyapı düşerse zarar kontrollüdür: 503 / 422 açık bir sözleşmeyle Google Maps devrine döner; operasyon durmaz.
- (-) Google Maps kalitesinde trafik, şerit ve hız sınırı yok; kurye bazı durumlarda Google Maps'i tercih edebilir (devir yolu bu yüzden durur).
- (-) Türkçe ek uyumu (Atatürk Bulvarı'na) sezgisel kurallarla üretilir; nadir adlarda hatalı ek çıkabilir. Rakamla biten adlarda ek yerine "yoluna" kalıbı kullanılır. Pilotta görülen hatalar birim testine eklenir.
- (-) TTS kalitesi cihaza bağlıdır (`tr-TR` ses paketi yoksa sesli anons sessiz kalır; uygulama bunu algılayıp bandı yine gösterir).
- (-) Kendi navigasyon durumunu (adım ilerlemesi, yoldan çıkma) uygulamada tutmak ek test yükü getirir; N2 emülatörde GPS rota oynatmasıyla doğrular.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Google Navigation SDK | Ticari sözleşme ve kullanım başına ücret; Expo'da yerel modül bağlama işi; Google Navigation yalnızca Google haritasıyla birlikte gösterilebilir ([ADR-0008](0008-acik-kaynak-harita.md) MapLibre kararıyla çelişir). Demo ilkesi: anahtar / fatura yok |
| Mapbox Navigation SDK | Ücretli (aylık aktif kullanıcı / yolculuk başına) ve kapalı kaynak; Mapbox hesabı ve anahtarı ister; MapLibre kararını bozar |
| MapLibre Navigation Android (`maplibre-navigation-android`) | Yalnızca yerel Android; Expo ile tek kod Android + iOS ([ADR-0002](0002-kurye-uygulamasi-react-native-expo.md)) yerine yerel modül, iOS karşılığı yok, bakımı sınırlı; yönlendirme motoru olarak yine bir OSRM benzeri sunucu ister |
| Yalnızca devir (mevcut durum) | Kurye uygulamadan çıkar; durak takibi, otomatik varış kartı ve sonraki durağa geçiş kaybolur. Yedek yol olarak korunur |
| Valhalla / GraphHopper'ın kendi yönlendirme motoru (Türkçe talimat dahil) | Ek altyapı ve ikinci rota motoru; Valhalla'nın Türkçe talimatı vardır ama ADR-0011'deki motosiklet profili ve ETA tutarlılığı bozulur. OSRM adımları + kendi üreticimiz tek motoru korur |
| Talimatları uygulamada üretmek | Her istemci sürümünde Türkçe düzeltmesi; test ve tutarlılık dağınık. Sunucuda üretim tek kaynak |
