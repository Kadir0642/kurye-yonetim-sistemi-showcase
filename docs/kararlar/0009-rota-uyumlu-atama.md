# ADR-0009: Rota uyumlu atama (rota başına canlı durak listesi, sapma süresine dayalı uygunluk)

- **Durum:** Kabul edildi (varsayılan eşikler pilot müşteri ile doğrulanacak)
- **Tarih:** 2026-10-30
- **Güncelleme:** 2026-10-30, madde 4 [ADR-0010](0010-kuryeler-arasi-devir.md) ile genişletildi (kuryeler arası devir eklendi, yönetici onaylı devir isteği iki devir yolundan biri oldu).

## Bağlam

Pilot müşteri (bir kurye firması) kullandığı mevcut yazılımla ilgili sahadan şu geri bildirimi verdi:

- Birden fazla paket taşıyan kurye, paketlerini **harita üzerinde durak olarak ve en uygun teslimat sırasıyla** görmek istiyor.
- Mevcut yazılım, yoldaki bir kuryeye yeni paketi yalnızca **yakınlığa göre** atıyor. Yeni paket çoğu zaman kuryenin rotası üzerinde değil, hatta **ters yönde** çıkıyor; kurye geri dönmek zorunda kalıyor.
- Kuryeler yanlış atanmış paketleri **bırakabiliyor** (kontrolsüz).
- Kurye yöneticisi haritayı izleyip kötü atanmış paketleri **elle, daha uygun konumdaki kuryelere yeniden dağıtıyor**.

Bu, pilotun en somut operasyonel acısıdır ve rakip yazılımlardan ayrışabileceğimiz alandır. İterasyon 1 yalnızca havuz (kuş uçuşu yarıçap filtresi) ve elle atama sunar (SPEC §7); durak ve rota kavramı yoktur. Altyapı kararı zaten verilmiştir: self-hosted OSRM + VROOM, yalnızca Eskişehir OSM çıkarımı ([ADR-0001](0001-hibrit-harita-mimarisi.md), [ADR-0006](0006-hizmet-bolgesi-eskisehir.md), [ADR-0008](0008-acik-kaynak-harita.md) madde 4). Bu ADR, o altyapının üzerine kurulan **atama davranışını** kararlaştırır.

## Karar

**Atama, "kurye pakete ne kadar yakın" sorusuyla değil, "paket kuryenin rotasına ne kadar sapma ekler" sorusuyla yapılır.** Ayrıntılı tasarım: iterasyon-2/ROTA-UYUMLU-ATAMA.md.

**1. Kurye başına canlı rota.**
- Kuryenin paketleri sıralı bir **durak listesi** oluşturur: işletmede `PICKUP`, müşteride `DROPOFF`. Alış her zaman teslimden önce gelir.
- Duraklar VROOM ile (pickup-delivery "shipments" problemi; kapasite ve zaman penceresi) OSRM üzerinde optimize edilir. Paket eklenince, çıkarılınca veya tamamlanınca yeniden optimize edilir.
- **Kararlılık kuralları:** kuryenin yöneldiği bir sonraki durak kilitlenir; sıra, yalnızca kazanç eşiği (varsayılan 2 dk) aşılırsa değişir. Böylece rota sürekli oynamaz.

**2. Uygunluk sapma süresiyle ölçülür.**
- Yeni paketin alış noktası R, teslim noktası D olsun. Önce PostGIS ile aday kuryeler süzülür: vardiyada, kapasitesi dolmamış, R'ye veya kendi rotasına belirli bir mesafe içinde.
- Her aday için, (R, D) çiftinin mevcut durak dizisine **en ucuz yerleştirmesi** hesaplanır (kuryenin konumu, mevcut duraklar, R ve D üzerinde OSRM `table` matrisi).
- Uygulanabilirlik kontrolleri: alınmış hiçbir paket "alıştan teslime azami dakika" sınırını aşmamalı (yemek tazeliği), kapasite aşılmamalı, istenirse işletmenin hazır olma zamanı gözetilmeli.
- **Skor = sapma süresi + α·(mevcut paketlere eklenen gecikme toplamı) + β·(kuryenin alış noktasında bekleme süresi) + küçük bir adalet terimi (kuryenin bugünkü paket sayısı).** Düşük skor iyidir.
- En iyi k aday, insan okunur gerekçeyle döner (ör. "Rotası üzerinde, +2 dk", "Ters yön, +14 dk"). Atamadan sonra nihai sıra VROOM ile belirlenir.

**3. Üç atama modu (kiracı ayarı), varsayılan `ONERI`.**
- `ONERI`: yönetici paketi açar, haritada en iyi 3 kuryeyi sapma dakikalarıyla görür; tek tıkla atar.
- `AKILLI_HAVUZ`: havuz, her kuryeye yalnızca sapması eşiğin altında olan paketleri gösterir; boştaki kuryelere yarıçap filtresi uygulanır.
- `OTOMATIK`: sapma eşiğin altındaysa otomatik atanır; değilse paket yöneticinin "bekleyenler" kuyruğuna düşer.

**4. Kontrollü ret ve devir.**
- **Alıştan önce** kurye, nedenini (enum) seçerek paketi reddedebilir. Paket öncelikle havuza döner ve anında yeni öneri üretilir. Kurye isterse paketi yakındaki bir meslektaşına devir teklifi olarak da gönderebilir ([ADR-0010](0010-kuryeler-arasi-devir.md)).
- **Alıştan sonra** paket kuryenin kendi başına bırakabileceği bir şey değildir; kurye paketi yalnızca **iki devir yolundan biriyle** başka kuryeye geçirebilir:
  - **A) Kurye-kurye devir** ([ADR-0010](0010-kuryeler-arasi-devir.md)): kurye yakındaki, vardiyadaki bir meslektaşa teklif gönderir; iki kurye fiziksel olarak buluşur (kod + konum kanıtı). Yönetici her zaman canlı görür; kiracı ayarıyla onay da zorunlu kılınabilir.
  - **B) Yönetici onaylı devir isteği:** kurye gerekçe (arıza, kaza, vardiya bitişi…) ile **devir isteği** gönderir; yönetici yeni kuryeyi seçip onaylamadan paket kuryeden alınmaz. Onaylanınca buluşma ve kanıt adımları A yolundakiyle aynıdır (kaza gibi durumlarda yönetici elle onay verebilir).
- Her şey kayıt altındadır. Ölçülen göstergeler: kurye başına ret sayısı, elle yeniden atama sayısı, kötü atama oranı.

**5. Panel ve kurye uygulaması.**
- Panel: kurye başına rota çizgisi ve numaralı duraklar, öneri paneli, paketi kuryenin üzerine sürükleyerek atama; büyük sapma veya gecikme getiren elle atamadan önce uyarı ("+14 dk, 2 paket gecikir").
- Kurye uygulaması "Rotam" ekranı: numaralı duraklar liste ve harita olarak, sıradaki durağa tek dokunuşla navigasyon, rota değişince bildirim ("Yeni paket rotana eklendi: +3 dk").

**6. Kiracı ayarları** (varsayılanlar pilotla doğrulanacak varsayımlardır):

| Ayar | Varsayılan |
|---|---|
| Kurye başına azami paket | 4 |
| Sapma eşiği | 5 dk |
| Alıştan teslime azami süre | 35 dk |
| Yeniden sıralama kazanç eşiği | 2 dk |
| Ön süzme mesafesi | 5 km |
| Atama modu | ONERI |

**7. Ölçüm ve portföy çıktısı.** Sentetik bir Eskişehir günü (yaklaşık 20 kurye, 400 paket, gerçekçi işletme kümeleri ve yoğun saatler) üzerinde simülasyon kıyaslaması yapılır: "en yakın kurye" ataması ile rota uyumlu atama; toplam km, ortalama teslimat süresi, geç teslim yüzdesi ve geri dönüş (backtracking) karşılaştırılır. Sonuç grafiği README'de yayımlanır. Canlıda KPI'lar: ortalama sapma, elle yeniden atama sayısı, ret oranı, zamanında teslim oranı.

**8. Mimari ilke.** Yerleştirme ve skor mantığı **G/Ç içermeyen saf fonksiyonlardır**; mesafeleri bir `TravelTimeProvider` arayüzünden alır (OSRM uygulaması, test ve simülasyon için sahte uygulama). Böylece aynı kod hem API'de hem benchmark simülatöründe çalışır.

## Sonuçlar

- (+) Sahadaki asıl sorun (ters yön ve geri dönüş) doğrudan hedeflenir; yönetici elle düzeltme yükü azalır.
- (+) Adayların değerlendirilmesi, yerleştirme sezgiseli (O(n²) küçük n) ve önbelleğe alınmış matrisle hızlıdır; 25 kurye için hedef 300 ms altındadır. VROOM yalnızca nihai sıralamada çalıştırılır.
- (+) Öneri modu varsayılan olduğundan karar yöneticide kalır; otomasyon, güven oluştukça kiracı ayarıyla açılır.
- (+) Ek maliyet yok: self-hosted OSRM + VROOM, Google Route Optimization maliyetinden (araştırma raporu) kaçınır.
- (+) Simülasyon benchmark'ı hem ürün doğrulaması hem portföy kanıtıdır.
- (-) OSRM canlı trafik bilmez; süreler yaklaşıktır. Sapma eşiklerinin anlamlı olması için süreler ölçülüp kalibre edilecek.
- (-) OSRM'de motosiklet profili yoktur; başlangıçta `car` profili kullanılır ve motosiklete uygun sürat ayarı yapılır. Kurye aracı türüne göre ayrı profil sonraya kalır.
- (-) Ek altyapı ve işletim yükü: iki yeni konteyner, OSM çıkarımı ve güncelleme boru hattı.
- (-) Skor ağırlıkları (α, β, adalet) sezgiseldir; benchmark ve pilot verisiyle ayarlanmalıdır. Pilotun kabul edilebilir sapma, kapasite ve ret politikası henüz bilinmiyor ([MIMARI açık sorular](../MIMARI.md#8-açık-sorular)).
- (-) VROOM "alıştan teslime azami süre" kısıtını yerel olarak zorlamaz; bu kısıt sonuçta doğrulanır, ihlal halinde yerleştirme sırası korunur.
- (-) Kalıcı veri modeline yeni tablolar eklenir (rota, durak, öneri, ret, devir isteği, kurye-kurye devir teklifi); İterasyon 1 durum makinesine dokunulmaz, yalnızca `release` yerine gerekçeli `reject` uç noktası gelir.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Yalnızca en yakın kurye (mevcut yazılımın davranışı) | Sahadaki sorunun kendisidir: rota dışı veya ters yöndeki paketler geri dönüşe yol açar. Benchmark'ta taban çizgisi olarak kullanılacak. |
| Google Route Optimization API | Paket başına faturalanır (araştırma raporuna göre yaklaşık 400 USD/ay ve üzeri) ve her yeniden optimizasyon maliyeti çarpar; self-hosted VROOM aynı işi sunucu kirasıyla yapar ([ADR-0001](0001-hibrit-harita-mimarisi.md)). |
| Her aday için VROOM ile tam yeniden çözüm | Küçük rotalarda doğru ama adaylar × çözüm çağrısı 300 ms bütçesini aşar; yerleştirme sezgiseli çok daha hızlıdır. VROOM, atamadan sonra nihai sıralama için kullanılır. |
| Makine öğrenmesi tabanlı atama | Erken: etiketli veri yok, açıklanabilirlik zayıf ("neden bu kurye?" sorusu yöneticiye yanıtlanamaz). Pilot verisi birikince ileride değerlendirilir. |
| Yalnızca ret yasaklamak / yalnızca yöneticinin elle dağıtması | Sorunu çözmez, yükü yöneticide bırakır; kontrollü ret ve devir hem esnekliği hem denetimi korur. |
