# ADR-0016: Eskişehir'e özel, kendi barındırdığımız harita platformu

- **Durum:** Kabul edildi (ürün sahibi onayı: 2026-11-03)
- **Tarih:** 2026-11-03
- **Kısmen yerine geçtiği:** [ADR-0008](0008-acik-kaynak-harita.md) madde 1 (karo sağlayıcısı) ve madde 2 (genel Photon sunucusu); madde 3 (navigasyonu uygulamaya devir) geçerlidir
- **İlgili:** [ADR-0001](0001-hibrit-harita-mimarisi.md), [ADR-0005](0005-konum-takibi-ve-kvkk.md), [ADR-0006](0006-hizmet-bolgesi-eskisehir.md), [ADR-0011](0011-osrm-motosiklet-profili-ve-rota-altyapisi.md), [ADR-0017](0017-saha-verisinden-ogrenen-harita.md), İterasyon 3 planı

## Bağlam

Bugünkü (İterasyon 1-2) harita bağımlılıkları:

| Parça | Bugün | Sorun |
|---|---|---|
| Harita karoları | OpenFreeMap herkese açık karoları (`MAP_STYLE_URL`, `GET /tenant` yanıtında `mapStyleUrl`) | SLA yok; her harita açılışında kuryenin IP'si ve görüntülenen bölge üçüncü tarafa gider; çevrimdışı çalışmaz |
| Adres arama | `photon.komoot.io` genel sunucusu (`PHOTON_BASE_URL`), yalnızca geliştirme / demo için adil kullanım | Canlıda kullanılamaz ([ADR-0008](0008-acik-kaynak-harita.md) bunu zaten "yalnızca demo" olarak işaretledi); sorgu metinleri (müşteri adresleri) üçüncü tarafa gider |
| Rota / matris | Kendi OSRM (MLD, motosiklet profili) + VROOM, Eskişehir çıkarımı ([ADR-0011](0011-osrm-motosiklet-profili-ve-rota-altyapisi.md)) | Zaten kendimizde; ancak OSM güncelleme "aylık zamanlanmış iş" olarak not düşülmüş, otomatikleştirilmemiş |
| Navigasyon | Kuryenin telefonundaki Google Maps / Apple Haritalar'a devir ([ADR-0008](0008-acik-kaynak-harita.md) madde 3) | Uygulama düzeyinde devir; bu ADR'nin kapsamı dışında (bkz. Sonuçlar) |

Hizmet bölgesi tektir: Eskişehir ([ADR-0006](0006-hizmet-bolgesi-eskisehir.md)); Odunpazarı ve Tepebaşı çekirdek, çevre ilçeler isteğe bağlı. OSM çıkarımı (bbox `[30.34, 39.655, 30.71, 39.895]`) ölçüldü: PBF 3,8 MiB, `osrm-*` dosyaları toplam 30,4 MB, tüm hat yaklaşık 10 dk (çoğu Türkiye PBF indirmesi) (infra/routing/README.md). Yani **tüm harita verisi tek bir küçük çıkarımdan** üretilebilir; bunu yalnızca rota için değil, görüntüleme ve adres arama için de yapmak ek bir veri kaynağı gerektirmez.

Ürün sahibi yönü onayladı: **Eskişehir'e özel bir harita platformu**. Tüm harita, rota ve adres verisi kendi sistemimizin içinde çalışır; çalışma zamanında üçüncü taraf harita servisine bağımlılık yoktur. Hedef, bizim kullanım alanımızda (motosikletli kurye, tek şehir, tekrar eden adresler) genel amaçlı haritaları geçmektir.

## Karar

**1. Kendi vektör karolarımız: planetiler + OpenMapTiles şeması, PMTiles çıktısı.**
- Aynı Eskişehir OSM çıkarımından ([ADR-0011](0011-osrm-motosiklet-profili-ve-rota-altyapisi.md) boru hattının çıktısı olan `eskisehir.osm.pbf`) `planetiler` ile tek bir `eskisehir.pmtiles` dosyası üretilir (düşük yakınlaştırmalar için Natural Earth, su poligonları planetiler'in `--download` adımıyla; indirilenler önbelleğe alınıp sürümlenir, çalışma zamanı bağımlılığı değil **derleme zamanı** bağımlılığıdır).
- **Neden planetiler (tilemaker değil):** (a) OpenMapTiles şemasını hazır üretir; bugün kullandığımız OpenFreeMap "liberty" stili aynı şemayı kullandığı için ilk sürümde stil neredeyse değişmeden çalışır, ardından kurye stiline sadeleştirilir; (b) doğrudan PMTiles yazar, ara dönüşüm yoktur; (c) küçük bir çıkarım için dakikalar ve birkaç GB bellekle biter (JVM yalnızca derleme anında çalışır); (d) Apache-2.0, Docker imajı hazır, tekrar üretilebilir. tilemaker (C++, Lua profili) daha az bellek kullanır ve esnektir, ancak OpenMapTiles şemasını kendimiz Lua ile bakımla uyumlu tutmamız gerekir; bellek farkı bu ölçekte (30 MB'lık PBF) önemsizdir. Şema bakım yükü, bellek tasarrufundan ağır basar.
- Yakınlaştırma: z0-z14 üretilir, istemci z17'ye kadar büyütür (overzoom); mahalle ve sokak adları, kapı numaraları (`housenumber` katmanı), tek yön okları yeterlidir. Dosya boyutu **tahmini 15-40 MB**; ilk derlemede ölçülüp bu ADR'ye ve plana yazılır (varsayım, ölçülmedi).
- **Sunum:** tek dosya statik olarak sunulur; HTTP range desteği olan bir sunucu (Caddy / nginx) veya `go-pmtiles serve` (ZXY uç noktası da verir). İstemci türüne göre: panel `pmtiles` protokol eklentisiyle doğrudan dosyayı okur; kurye uygulaması (MapLibre Native) için **iki yol** İterasyon 3a başında kısa bir spike ile (en çok 2 gün) cihazda doğrulanır: (i) MapLibre Native'in `pmtiles://` desteği, (ii) olmazsa `go-pmtiles` ZXY uç noktasından MapLibre `OfflineManager` paketi indirme. Karar spike sonucuyla bu ADR'ye not edilir; API sözleşmesi iki durumda da aynıdır (`mapStyleUrl`).
- **Stil sürümlü ve kuryeye özel:** `infra/map/style/` altında iki stil: `courier` (hafif: yaklaşık 40 katman, 3B bina yok, güneş altında yüksek kontrast, büyük sokak etiketleri, tek yön okları, gece varyantı, POI sınırlı; düşük uçlu telefonda çizim süresi için) ve `admin` (daha zengin). Stil, sprite ve glyph (Noto Sans, Türkçe karakter aralıkları) aynı sunucudan gelir.

**2. Çevrimdışı harita: aynı PMTiles dosyası telefona indirilir.**
- Kurye uygulaması `GET /map/manifest` ile `{ version, sha256, sizeBytes, bbox, minZoom, maxZoom, styleVersion, publishedAt }` alır; dosya yeniden başlatılabilir (HTTP range) indirilir, `sha256` doğrulanır, cihaz depolama alanı kontrol edilir.
- **Sürüm ve yeniden indirme politikası** (ayrıntı: PLAN 3a-4): her haftalık yayın yeni sürüm üretir; yollar nadiren değiştiği için eski karolar çalışmaya devam eder. Uygulama sürümü vardiya açılışında denetler; yerel sürüm 2 sürümden (yaklaşık 14 gün) eskiyse veya dosya yok / bozuksa Wi-Fi'de ve şarjdayken (kullanıcı onayıyla) tam dosya yeniden indirilir. Gerçek delta güncellemesi yapılmaz: haftalık değişiklik dosyaya dağınık yayılır, PMTiles'ta yerel delta yoktur ve 15-40 MB dosya için karmaşıklığa değmez.

**3. Kendi adres arama: Photon, kendi Eskişehir çıkarımımızdan üretilmiş dizinle.**
- **Neden Photon (Nominatim değil):** Photon yazarken arama (search-as-you-type) için tasarlandı; API'de `PhotonProvider` ve `GeoProvider` arayüzü zaten var ([ADR-0008](0008-acik-kaynak-harita.md)), geçiş yalnızca `PHOTON_BASE_URL` ayarıdır, istemci ve sözleşme değişmez. Nominatim yazarken aramaya uygun değil (ADR-0008 alternatifler); bu karar korunur.
- **Üretim yolu:** Eskişehir çıkarımı geçici bir Nominatim (PostgreSQL) içine alınır, Photon bu veritabanından dizin üretir (`-nominatim-import`), dizin kalıcı Photon servisine yüklenir; Nominatim yalnızca **derleme zamanında** çalışır ve sonra kapatılır. Böylece sorgu hizmeti küçük kalır, dizin her hafta aynı OSM sürümüyle (rota ve karolarla tutarlı) yenilenir. Yedek yol: hazır Türkiye Photon dökümü (boyut ve güncellik doğrulanacak); kendi çıkarımımız tercih edilir çünkü diğer parçalarla aynı veri sürümündedir.
- Sorgular `bbox` = hizmet bölgesi ile sınırlanır, Redis önbelleği korunur. `GEO_PROVIDER=auto` sırası: kendi Photon, ardından (anahtar ve ürün sahibi izniyle) Google; **üretim varsayılanı Google'sız**. `GooglePlacesProvider` kodu, adres kalitesi testindeki karşılaştırma ve olası ileriki tercih için isteğe bağlı kalır; çalışma zamanı bağımlılığı değildir.
- **Dürüst sınır:** OSM'de Türkiye'de kapı numarası kapsamı Google'dan belirgin zayıftır (araştırma raporu: doğrulanamadı, bkz. teknik araştırma). Bunu iki şeyle telafi ederiz: haritada iğne sürükleme (mevcut) ve **saha verisinden öğrenilen teslim noktaları** ([ADR-0017](0017-saha-verisinden-ogrenen-harita.md)). Kalite ölçümü: adres kalitesi testi (PLAN 3a-8).

**4. Haftalık otomatik OSM güncelleme boru hattı, kesintisiz geçişle.**
- Zamanlama: haftada bir, trafik azken (Pazar 03:00 yerel). Adımlar: Geofabrik Türkiye PBF + md5 -> `osmium extract` -> **yeni, sürüm numaralı bir dizinde** (`releases/<yıl>-w<hafta>/`) paralel olarak: `osrm-extract/partition/customize` (ve ADR-0017 hız dosyası varsa), `planetiler` karoları, Nominatim -> Photon dizini.
- **Kapılar (geçiş öncesi otomatik kontroller):** rota test seti (N sabit başlangıç-bitiş çifti; süre farkı önceki sürüme göre eşik içinde), PMTiles açılıyor ve boyutu önceki sürümün ±%20'sinde, altın sorgu seti (sabit adres sorguları) Photon'da geçiyor. Kapı kalırsa geçiş yapılmaz, uyarı gider, eski sürüm sürer.
- **Geçiş (sıfır kesinti):** OSRM ve Photon için ikinci örnek yeni sürümle başlatılır, sağlık denetimi geçince ters vekil sunucudaki (Caddy) hedef değiştirilir; eski örnek 24 saat açık kalır (geri dönüş), son 2 sürüm saklanır. Karolar statik dosyadır: yeni dosya yüklenir, `current` işaretçisi atomik olarak yeniden adlandırılır, `GET /map/manifest` yeni sürümü duyurur. API tek bir sabit iç adrese konuşur; OSRM adresinin değişmesi API yeniden başlatması gerektirmez.
- Optimizasyon (sonra): tam Türkiye PBF'i (~650 MB) yerine Geofabrik günlük farklarını (`osmium` / `pyosmium-up-to-date`) uygulamak. Başlangıçta tam indirme yeterlidir (ölçülen yaklaşık 7 dk).

**5. Dürüst kapsam: Google'ın üstün kaldığı yerler.** Bu platform şunlarda Google'ı geçmeyi **hedeflemez**: canlı trafik, kapı düzeyinde adres verisi, genel POI (işletme, çalışma saati, yorum), sokak görüntüleri. Biz şunlarda geçmeyi hedefleriz: motosiklet profiline uyumlu rota ve süreler, **kendi sahamızın teslim noktaları** (ADR-0017), çevrimdışı çalışma, kurye için sadeleştirilmiş harita, rota uyumlu atama, kişisel verinin Türkiye'deki kendi sunucumuzda kalması.

## Sonuçlar

- (+) Çalışma zamanında üçüncü taraf harita servisi yok; kesinti ve adil kullanım riski kalkar. Harita karoları ve adres sorguları yüzünden kuryenin IP'si, bölgesi ve müşteri adres metni yurt dışı üçüncü taraflara gitmez (KVKK açısından sınır ötesi aktarım yüzeyi küçülür; hukuk onayı PLAN P-9).
- (+) Karo, rota ve geocoder **aynı OSM sürümünden** gelir; harita ile rota arasında tutarsızlık (haritada yol var, rotada yok) azalır.
- (+) Çevrimdışı harita mümkün olur (yer altı otoparkı, bina içi, zayıf çekim).
- (+) Tek yönetim noktası: stil, sürüm ve güncelleme sıklığı bizim elimizde.
- (-) **İşletim yükü:** iki yeni çalışma zamanı servisi (karo sunucusu, Photon) ve derleme zamanı araçları (planetiler, Nominatim); haftalık boru hattının izlenmesi ve arıza yönetimi. Hafifletme: tek `docker compose` profili, kapılar ve uyarılar, son 2 sürümün saklanması.
- (-) **Depolama (tahmin, ilk derlemede ölçülecek):** her sürüm için karolar 15-40 MB, OSRM 30 MB, Photon dizini yüzlerce MB; Türkiye PBF 650 MB geçici; son 2 sürüm saklanınca toplam yaklaşık 2-4 GB. Telefonda 15-40 MB.
- (-) Derleme sırasında geçici bellek tepeleri: planetiler 2-4 GB (JVM), Nominatim içe aktarma 2-4 GB (tahmin). Üretim VPS boyutlandırması PLAN P-1.
- (-) Bölge dışı: karo yalnızca çıkarım sınırı içinde ayrıntılıdır (z0-z6 dünya taban haritası dışında); yeni şehir eklemek yeni çıkarım ve karo derlemesi demektir ([ADR-0006](0006-hizmet-bolgesi-eskisehir.md) ile uyumlu: bölge veridir).
- (-) Navigasyon (dönüş dönüş yönlendirme) hâlâ kuryenin telefonundaki Google Maps / Apple Haritalar'a devredilir ([ADR-0008](0008-acik-kaynak-harita.md) madde 3): bu, kuryenin kendi cihazındaki uygulamadır, sistemimizin çalışma zamanı bağımlılığı değildir; uygulama içi sesli yönlendirme kapsam dışıdır. Uygulama içi navigasyon API'si ayrı bir çalışma kolunda ele alınmaktadır (ADR-0015, bu PR'ın kapsamı dışı).
- **Atıf (zorunlu):** harita üzerinde ve uygulama / panel "hakkında" ekranında "© OpenStreetMap katkıcıları" ([ODbL](https://www.openstreetmap.org/copyright)) ve "© OpenMapTiles" (CC-BY 4.0, şema) görünür. Geocoder dizini OSM'den türetilmiş bir veritabanıdır; ODbL paylaşım koşulu **herkese açık dağıtımda** doğar, dizin yalnızca dahili çalışır ve dağıtılmaz; öğrenilmiş katmanlar ayrı bir veritabanıdır, OSM verisine karıştırılmaz (hukuk görüşünde teyit edilir). Yazı tipleri Noto Sans (OFL), planetiler Apache-2.0.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Herkese açık servisleri korumak (OpenFreeMap, genel Photon) | SLA ve adil kullanım sınırı; çevrimdışı yok; sorgu ve konum verisi üçüncü tarafa gider; genel Photon canlıda zaten yasak ([ADR-0008](0008-acik-kaynak-harita.md)) |
| Google Maps Platform veya Mapbox | Anahtar, faturalama ve kullanım şartı kilidi; Google verisi Google dışı haritada kısıtlı ([ADR-0001](0001-hibrit-harita-mimarisi.md) açık riski); Mapbox GL v2+ özgür değil; ürünün "veri bizde kalır" hedefine aykırı |
| tilemaker | Bellek ve esneklik avantajı bu ölçekte küçük; OpenMapTiles şemasını kendimiz sürdürmemiz gerekir (yukarı bakınız) |
| Nominatim'i canlı geocoder yapmak | Yazarken arama için uygun değil; kalıcı PostgreSQL ve daha ağır işletim |
| Cihaz üstü rota hesaplama (çevrimdışı yönlendirme motoru) | Düşük uçlu Android telefonlarda bellek / pil / uygulama boyutu yükü; şimdilik önbelleğe alınmış rota + "bağlantı gerekli" deneyimi yeterli (PLAN 3a-5). Pilot verisi gerektirirse sonra yeniden değerlendirilir |
| Raster karolar önceden indirip paketlemek | Vektöre göre 10 kat büyük, stil değişmez, gece modu yok |
