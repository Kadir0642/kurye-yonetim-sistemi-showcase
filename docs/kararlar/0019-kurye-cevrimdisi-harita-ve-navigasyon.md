# ADR-0019: Kurye uygulamasında çevrimdışı harita ve çevrimdışı navigasyon (yerel PMTiles, manifest + sha256, rota bacağı önbelleği)

- **Durum:** Önerildi (İterasyon 3a spesifikasyonuyla birlikte onaya sunuldu; madde 1'in son hâli cihaz spike'ıyla kesinleşir)
- **Tarih:** 2026-11-05
- **Ayrıntılandırdığı karar:** [ADR-0016](0016-eskisehire-ozel-kendi-barindirdigimiz-harita-platformu.md) madde 2 (çevrimdışı harita) ve "spike" notu; [ADR-0015](0015-uygulama-ici-navigasyon.md) (uygulama içi navigasyonun çevrimdışı davranışı)
- **İlgili:** [ADR-0002](0002-kurye-uygulamasi-react-native-expo.md), [ADR-0018](0018-harita-karo-ve-stil-sunumu.md), [ADR-0020](0020-cevrimdisi-eylem-kuyrugu-ve-idempotency.md), SPEC-3a §7.2, §8

## Bağlam

- Kurye uygulaması Expo SDK 58 + `@maplibre/maplibre-react-native` 11.x kullanır (Android'de MapLibre Native Android 11.x, iOS'ta MapLibre Native iOS 6.x). Bugün harita, rota, paket listesi ve navigasyon tamamen çevrimiçidir; yalnızca konum kuyruğu kalıcıdır (PLAN §2).
- Eskişehir karo dosyası küçüktür (tahmin 15-40 MB, z0-z14; ilk derlemede ölçülür). Tek şehirde tüm haritayı telefona koymak gerçekçidir.
- MapLibre Native'de yerel karo okumak için üç yol vardır: (a) `pmtiles://` kaynak adresi (MapLibre Native çekirdeğine 2024-2025'te eklenen PMTiles desteği; `pmtiles://https://...` ve `pmtiles://file://...`), (b) `mbtiles://` yerel dosya kaynağı (uzun süredir desteklenir), (c) `OfflineManager` ile ZXY uç noktasından çevrimdışı paket indirmek (MapLibre'nin kendi önbellek veritabanına). Hangisinin RN 11 köprüsünde, Android ve iOS'ta sorunsuz çalıştığı **cihazda doğrulanmamıştır**.
- Uygulama içi navigasyon ([ADR-0015](0015-uygulama-ici-navigasyon.md)) rotayı her seferinde sunucudan ister; ağ yoksa veya OSRM düşmüşse Google Maps devrine düşer. Çekim dışı bölgelerde (otopark, bina içi, ilçe kenarı) bu, kuryenin uygulamadan çıkması demektir.
- Cihaz üstü rota motoru (OSRM / Valhalla mobil) düşük uçlu telefonlarda bellek, pil ve APK boyutu yüzünden kapsam dışıdır ([ADR-0016](0016-eskisehire-ozel-kendi-barindirdigimiz-harita-platformu.md) alternatifler).

## Karar

**1. Harita dosyası: tek PMTiles dosyası, uygulamanın belge dizininde, `pmtiles://file://` ile.**
- Birincil yol (a). Yedek (b): aynı derlemeden MBTiles üretilir (planetiler MBTiles yazar, PMTiles'a dönüştürülür; ikisi aynı karoları taşır), manifest `format: mbtiles` duyurur, uygulama `mbtiles://` kullanır; **sözleşme değişmez**. Son yedek (c): `OfflineManager` + koşullu `GET /map/tiles/...` ZXY ucu ([ADR-0018](0018-harita-karo-ve-stil-sunumu.md) madde 7); bu yolda sha256 dosya bazında doğrulanamaz, sürüm paketin meta verisinde tutulur.
- **Spike (Görev M'nin ilk adımı, en çok 1 gün):** Android emülatör + en az bir gerçek Android cihazda (a) yerel `file://` PMTiles, (a') uzak `pmtiles://https://` ve (b) `mbtiles://` denenir; ölçüt: z10-z17 çizim, sokak adları (glyph `file://`), sprite, uçak modunda soğuk açılış. iOS derlemesi bu iterasyonda yapılmaz ([ADR-0003](0003-dagitim-apk-self-update.md)); seçilen yolun iOS'ta da MapLibre Native iOS tarafından desteklendiği belgeden doğrulanıp not edilir. Sonuç bu ADR'nin "Spike sonucu" bölümüne yazılır.

**2. Paket = manifest'teki dosya listesi; her dosya sha256 ile doğrulanır; sürüm atomik etkinleşir.**
- Paket: karo dosyası + stil şablonları (`style.<id>.json`) + sprite + glyph alt kümesi (`OfflineMapPackage.files[]`, her biri `path`, `url`, `sizeBytes`, `sha256`). Zip yoktur (RN'de yerel zip modülü gerekmesin).
- İndirme yeniden başlatılabilir (HTTP Range; `expo-file-system` yeniden başlatılabilir indirme API'si), `.download/<sürüm>/` altında; boş alan kontrolü (paket boyutu x 2 + 50 MB); her dosya sha256 ile doğrulanır; hepsi geçince dizin `map/<sürüm>/` olarak yeniden adlandırılır ve `map/state.json` geçici dosya + yeniden adlandırma ile güncellenir. Eski sürüm, yeni sürüm bir kez başarıyla çizilene kadar silinmez (geri dönüş).
- Çevrimiçiyken de yerel dosya varsa yerel dosya kullanılır (veri tasarrufu); yoksa çevrimiçi stil (`Tenant.mapStyles`), o da yoksa `Tenant.mapStyleUrl`.

**3. Yeniden indirme politikası (ADR-0016 madde 2'nin uygulanışı).** Vardiya açılışında ve ön planda en sık 6 saatte bir `GET /map/manifest` (`ETag`). Yeni sürüm varsa: Wi-Fi + şarjda sessiz indirme; değilse bekler. Yerel sürüm `updatePolicy.staleAfterVersions` (2) sürüm veya `staleAfterDays` (14) gün eskiyse ya da dosya yok / bozuksa "Harita güncellemesi (xx MB)" istemi; "Yalnızca Wi-Fi" (varsayılan açık) kapalıysa veya kullanıcı onaylarsa mobil veride indirilir. Delta yoktur.

**4. Çevrimdışı navigasyon: sunucunun ürettiği rota bacakları önceden saklanır.**
- Yeni uç `POST /courier/route/legs`: güncel rotanın her PLANNED durağı için `NavigationRoute` (ilk bacak kuryenin konumundan, sonrakiler önceki duraktan). Aynı OSRM çağrısı, aynı Türkçe talimat ve sesli anons üreticisi; uygulama metin üretmez ([ADR-0015](0015-uygulama-ici-navigasyon.md) ilkesi korunur).
- Uygulama, rota sürümü değiştiğinde (WS `route.updated` veya `GET /courier/route`) 5 sn erteleyerek bacakları ister ve yerel SQLite'a yazar; navigasyon çevrimiçiyken yine `POST /courier/navigation/route` kullanır (taze konumdan en iyi rota).
- **Ağ yoksa veya 503 / zaman aşımı:** navigasyon önbellekteki bacakla başlar / sürer. Bandın altında kalıcı "Çevrimdışı" şeridi; yoldan çıkınca **yeniden rota yapılmaz**: önbellekteki çizgide ileriye doğru en yakın noktaya kesikli "rotaya dön" kılavuzu ve mesafe gösterilir, bir kez "Bağlantı yok, rotaya geri dönün" anonsu okunur; kurye çizgiye dönünce adımlar kaldığı yerden sürer. Bacak yoksa (veya kurye çizgiden > 1 km uzaktaysa) hedefe kuş uçuşu kesikli çizgi + mesafe + yön; Google Maps düğmesi görünür kalır (kuryenin Maps'i kendi çevrimdışı verisini kullanabilir).
- Bağlantı gelince: otomatik yeniden rota (mevcut akış), "Bağlantı geri geldi, rota güncellendi" şeridi; eylem kuyruğu ([ADR-0020](0020-cevrimdisi-eylem-kuyrugu-ve-idempotency.md)) ve rota sürümü tek turda senkronlanır. Daha eski rota sürümü daha yenisinin üzerine asla yazılmaz (`pickNewerRoute` kuralı).

**5. Yerel veri ve gizlilik.** Rota, paket listesi (müşteri adı / telefonu / adresi dahil) ve bacaklar yalnızca uygulamanın özel deposunda (SQLite) tutulur; teslim edilen / iptal edilen paketlerin kayıtları en geç 12 saat sonra, tümü çıkışta ve vardiya bitiminden 12 saat sonra silinir. Android `allowBackup = false` (kişisel veri bulut yedeğine gitmez). Harita dosyası kişisel veri içermez.

## Sonuçlar

- (+) Uçak modunda harita, Rotam, durak ayrıntısı ve navigasyon (önbellekteki rota üzerinde) çalışır; eylemler kuyrukta kaybolmaz.
- (+) Aynı önbellek OSRM kesintisinde de işe yarar (503'te Google Maps'e atmak yerine önbellekteki bacak).
- (+) Dosya tabanlı paket: sha256, sürüm, atomik geçiş ve geri dönüş bizim denetimimizde; MapLibre'nin iç önbelleğine bağımlı değiliz.
- (+) Ek ücret yok: MapLibre, Expo modülleri ve kendi sunucumuz.
- (-) Çevrimdışıyken gerçek yeniden rota yok; kurye rotadan çok saparsa yalnızca kuş uçuşu kılavuz alır (dürüstçe gösterilir).
- (-) İlk bacak, bacaklar alındığı andaki konumdan hesaplanır; kurye o arada yer değiştirdiyse ilk bacak kısmen eskidir (çizgiye dönüş kılavuzu bunu karşılar).
- (-) PMTiles yerel okuma desteği RN köprüsünde doğrulanmadıysa yedek yollar ek iş getirir (MBTiles: derlemede ikinci çıktı; OfflineManager: koşullu ZXY ucu).
- (-) Telefonda 15-40 MB (tahmin) + bacak önbelleği (< 1 MB) depolama.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Yalnızca `OfflineManager` (MapLibre iç önbelleği) | Dosya bazında bütünlük ve atomik sürüm geçişi yok; paket boyutu ve sürüm denetimi dolaylı; yedek olarak tutulur |
| Raster karo paketi | Vektörün yaklaşık 10 katı boyut; gece stili ve etiket ölçeği yok |
| Cihaz üstü rota motoru (OSRM / Valhalla mobil, GraphHopper Android) | Bellek, pil, APK boyutu; motosiklet profilinin iki yerde bakımı; düşük uçlu cihazda risk |
| Bacakları uygulamada OSRM yanıtından üretmek | Türkçe talimat ve anons üreticisi iki yerde olur ([ADR-0015](0015-uygulama-ici-navigasyon.md) ilkesine aykırı) |
| Çevrimdışıyken otomatik Google Maps devri (bugünkü davranış) | Kurye uygulamadan çıkar, varış kartı ve eylem akışı kaybolur; önbellek varken gereksiz. Önbellek yoksa yedek olarak kalır |
| Zip paketi | RN'de yerel zip modülü (ek yerel bağımlılık) gerekir; dosya listesi aynı işi görür |

## Spike sonucu

**Tarih:** 2026-11-06. **Sonuç: yol A (yerel `pmtiles://file://`) çalışıyor; B (MBTiles) ve C (OfflineManager) gerekmedi. Sözleşme değişmez (`format: pmtiles`).**

| Konu | Değer |
|---|---|
| Ortam | Android emülatörü `Medium_Phone_API_36.1` (Android 16, x86_64, `swiftshader_indirect`), development build (Gradle debug) |
| Sürümler | `@maplibre/maplibre-react-native` 11.5.1 (bu sürüm **MapLibre Native Android 13.6.1**, `opengl` varyantı; ADR'deki "11.x" ifadesi eskiydi), Expo SDK 58, React Native 0.88 |
| Karo dosyası | planetiler 0.10.2 (Docker, OpenMapTiles profili), Eskişehir çıkarımı, z0-14, **2,8 MB** PMTiles (v3, gzip karolar, 389 karo). Yalnızca spike içindi; üretim dosyasını T üretir |
| A: `pmtiles://file:///data/user/0/com.kuryeyonetim.courier/files/map/<sürüm>/eskisehir.pmtiles` | **Geçti.** Yol, bina ve park alanları çizildi; **uçak modunda** (`cmd connectivity airplane-mode enable`) açılış ve kaydırma çalıştı. Dosya `documentDirectory/map/<sürüm>/` altındaydı; adres biçimi `pmtiles://` + `file://` URI'si (üç eğik çizgi) |
| A': `pmtiles://http://10.0.2.2:4100/data/eskisehir.pmtiles` | **Geçti.** MapLibre yalnızca `Range` istekleriyle (başlık, dizin, karo aralıkları) çizdi; sunucu 206 verdi |
| Metin etiketleri (glyph `file://`) | **Geçti** (emülatörde `-gpu host` ile): yol adları (`Yurt Sokak`), mahalle ve POI adları, Türkçe karakterler (İ, ş, ğ) yerel `glyphs/Noto Sans Regular/0-255.pbf` ve `256-511.pbf` dosyalarından, uçak modunda çizildi; yazı tipi yolundaki boşluklar sorun çıkarmadı. **Uyarı:** emülatör `-gpu swiftshader_indirect` ile açıldığında MapLibre metni hiç çizmiyor (`SurfaceView`) veya düz bloklar çiziyor (`TextureView`); çevrimiçi OpenFreeMap stili de aynı davranır, yani PMTiles / glyph ile ilgisizdir. Doğrulama emülatörleri `-gpu host` ile açılmalıdır |
| Gerçek cihaz | Bu ortamda yok; ürün sahibinin cihazıyla P'nin cihaz test protokolünde (`CIHAZ-TEST-PROTOKOLU.md`) doğrulanır |
| iOS | Derleme kapsam dışı. PMTiles okuma MapLibre Native çekirdeğindedir (platformdan bağımsız C++); iOS'ta aynı `pmtiles://` kaynak şeması beklenir, **belgeden / cihazdan doğrulanmadı** |

Uygulama notları:

- Karo kaynağı olarak tek PMTiles dosyası ve `file://` URI'si yeterli; `mbtiles://` ve `OfflineManager` yollarına gerek kalmadı: T'nin `--with-mbtiles` yayını ve Q'nun koşullu ZXY ucu **gerekmiyor**.
- Spike sırasında planetiler'in `osmdata.openstreetmap.de` (su poligonları) kaynağına bu makineden ulaşılamadı; spike için `--exclude-layers=water` ve boş bir yer tutucu zip kullanıldı (su katmanı yok). T ve G için risk: o kaynağın yansısı veya önbelleği gerekebilir.
- Windows'ta Metro dosya izleyicisi bu çalışma dizininde değişiklikleri kaçırdı; JS değişikliğinden sonra Metro yeniden başlatıldı.
- Emülatörde metin doğrulaması için GPU modu: `emulator -avd <ad> -gpu host`. Uygulamada `EXPO_PUBLIC_MAP_VIEW=texture` anahtarı yalnızca `swiftshader` gibi sorunlu GPU yığınları içindir; varsayılan `surface`.

- **Karar notu (3a entegrasyonu):** Spike birincil yolu (a) doğruladığı için Q'nun koşullu ZXY ucu (`GET /map/tiles/{version}/{z}/{x}/{y}`) **sözleşmeden ve `x-planned` işaretinden kaldırıldı** ([ADR-0018](0018-harita-karo-ve-stil-sunumu.md) madde 7). Entegrasyon turunda gerçek API + gerçek paketle (4,38 MB, sha256 cihazda eşleşti) çevrimdışı harita, çevrimdışı navigasyon ve eylem kuyruğu emülatörde uçtan uca doğrulandı; ayrıntı `docs/iterasyon-3/ENTEGRASYON-DURUM.md` ve PR açıklaması. Dev client uçak modunda Metro'ya bağlanamadığı için uygulama yeniden başlatma denemesi uçak modunda değil, API kapalıyken yapıldı; sürüm derlemesiyle (JS gömülü) gerçek uçak modu açılışı cihaz test protokolünde kalır.
