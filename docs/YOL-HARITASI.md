# Yol Haritası

Kapsam, pilot müşteri keşfi sonrasında değişebilir. Her iterasyon bir demo ile kapanır; ayrıca "Portföy çıktısı" ile README için görsel malzeme toplanır. Çalışma düzeni için [GELISTIRME-SURECI.md](GELISTIRME-SURECI.md).

| İterasyon | Odak | Durum |
|---|---|---|
| 0 | Planlama ve araştırma | 🚧 Devam ediyor |
| 1 | Çekirdek | ✅ Uygulandı ve birleştirildi |
| 2 | **Ana hedef: dinamik, rota uyumlu dağıtım** (tek akıllı rota, en iyi uyan kuryeye tekil ve toplu atama, kararlı yeniden sıralama, öneri / otomatik atama, ret ve devir, uygulama içi navigasyon, benchmark) | ✅ Uygulandı ve birleştirildi (SPEC) |
| 2b | Platform ve sağlamlaştırma: RLS, push (FCM), iOS / TestFlight, OTA, aynı restoran gruplaması, gerçek cihaz testi | ⏳ |
| 3 | **Eskişehir'e özel harita platformu:** kendi karo / geocoder / OSM güncelleme hattı, çevrimdışı harita ve eylem kuyruğu, saha verisinden öğrenen harita, üretim hazırlığı | **3a ✅ tamamlandı** (SPEC-3a; entegrasyon turunda gerçek yığınla doğrulandı, açık maddeler aşağıda); **3b ✅ tamamlandı** (SPEC-3b, sözleşme 0.4.0; PR #31-#38 2026-11-07'de birleşti, entegrasyon kaydı; açık: gerçek cihaz, pilot kararları); 3c ⏳ (PLAN) |
| 4 | **Pilot hazırlığı:** dağıtım pilot kipi (havuz gizli, atama kabulü), konum kademeleri ve "Rota yanlış", güvenlik denetimi (P0 ve küçük P1 maddeleri kapatıldı, panel oturumu HttpOnly çerezde), KVKK saklama süresi, yük testi, kapıda tahsilat mutabakatı, kurye molası, gecikme ve ret oranı uyarıları, kurye uygulamasında arama / süzgeç ([ADR-0028](kararlar/0028-havuz-gorunurlugu-ve-atama-kabulu.md)-[ADR-0037](kararlar/0037-panel-refresh-cerezi.md), sözleşme 0.7.0) | ✅ Tamamlandı (2026-11-13, [ayrıntı](#iterasyon-4-pilot)); açık: ürün sahibi kararları ve gerçek cihaz testi |
| 5 | Hakediş, kasa raporu ve atama ağırlıkları (eski "İterasyon 4"; **öneri**, ürün sahibi kararlarına bağlı) | ⏳ |
| 6+ | Genişleme: işletme paneli, POS / platform entegrasyonları, SoftPOS, faturalama (eski "İterasyon 5+") | ⏳ |

---

## İterasyon 0 — Planlama ve araştırma 🚧

**Hedef:** Ne yapılacağını, hangi teknolojiyle ve hangi riskle yapılacağını belirlemek.

- [x] Teknik fizibilite araştırması (rapor)
- [x] Pazar araştırması (rapor)
- [x] Taslak mimari ([MIMARI.md](MIMARI.md))
- [x] İlk ADR'ler (önerildi) ([kararlar](kararlar/README.md))
- [x] Depo iskeleti ve AI ajanları için `CLAUDE.md`
- [ ] Pilot müşteri keşif görüşmesi ve açık soruların kapatılması
- [x] ADR'lerin onaylanması / revize edilmesi (ADR-0001..0008 kabul edildi; kurye uygulaması React Native + Expo, açık kaynak harita)

**Portföy çıktısı:** Mimari diyagramı ve karar kayıtları (README'de mevcut).

---

## İterasyon 1 — Çekirdek ✅

**Hedef:** Bir yönetici paket girer, kurye havuzdan alır, navigasyonla teslim eder, yönetici canlı haritada izler. Demo **ücretli anahtar olmadan** çalışır.

Teknik spesifikasyon: iterasyon-1/SPEC.md. API sözleşmesi `packages/contracts` altındadır. Üç paralel görev: A (API + altyapı), B (yönetim paneli), C (kurye mobil uygulaması).

- [x] Monorepo (npm workspaces), ortak sözleşme (OpenAPI 3.1 + WebSocket protokolü), docker compose altyapısı ([ADR-0007](kararlar/0007-backend-ve-web-yigini.md))
- [x] Kimlik doğrulama (JWT + rotasyonlu refresh) ve tenant altyapısı (uygulama katmanı + bileşik FK; RLS İterasyon 2b'de, [ADR-0012](kararlar/0012-rls-iterasyon-2b-ye-ertelendi.md))
- [x] Kiracı hizmet bölgesi yapılandırması (`service_area`: poligon / bbox, şehir, varsayılan harita merkezi) ([ADR-0006](kararlar/0006-hizmet-bolgesi-eskisehir.md))
- [x] İşletme yönetimi (ekleme, düzenleme, arşivleme)
- [x] Kurye yönetimi (panelden hesap açma, PIN sıfırlama, vardiyayı kapatma)
- [x] Paket ekleme; adres otomatik tamamlama API üzerinden (Photon, anahtar varsa Google Places) ([ADR-0008](kararlar/0008-acik-kaynak-harita.md))
- [x] Paket durum makinesi ve olay geçmişi (audit)
- [x] Sipariş havuzu, kurye yarıçap filtresi (PostGIS `ST_DWithin`), yarışa dayanıklı alma
- [x] Elle atama, yeniden atama, havuza geri alma
- [x] Kurye uygulaması (React Native + Expo, Android'de doğrulanır; iOS yapılandırması hazır) ([ADR-0002](kararlar/0002-kurye-uygulamasi-react-native-expo.md)): giriş, KVKK onayları, izinler ve pil optimizasyonu ekranı
- [x] Kurye uygulaması: vardiya aç / kapat, arka plan konum servisi (kalıcı kuyruk, toplu gönderim)
- [x] Kurye uygulaması: havuzdan paket alma, harita (MapLibre) ve tek dokunuş navigasyon, teslim etme
- [x] Panel: canlı harita (MapLibre + WebSocket)
- [x] Konum kesilme uyarısı (heartbeat)
- [x] Demo seed verisi (kurgusal Eskişehir firması, işletmeler, kuryeler, paketler) ve kurye konum simülatörü

**Portföy çıktısı:** Paket oluşturma, havuzdan alma ve navigasyon başlatmayı gösteren GIF; panelde canlı harita ekran görüntüsü.

---

## İterasyon 2 — ana hedef: dinamik, rota uyumlu dağıtım ✅

**Hedef:** Birden fazla paket taşıyan kurye paketlerini **tek bir akıllı rota** olarak, haritada numaralı duraklarla ve uygulanabilir en kısa sırayla görür. Yeni paket (veya aynı anda gelen paket**ler**) kuryenin **mevcut rotasına en iyi uyduğu** kuryeye, mevcut paketleri gecikme ve tazelik sınırları içinde tutarak atanır ve rota durak olarak yeniden kurulur. Pilot müşterinin sahadaki sorunu: mevcut yazılım yoldaki kuryeye yeni paketi yalnızca yakınlığa göre atıyor, paket çoğu zaman rota dışında veya ters yönde kalıyor, kuryeler yanlış atanmış paketi bırakıyor, yönetici haritaya bakıp elle yeniden dağıtıyor. Rota uyumlu dağıtım ürünün ayırt edici özelliği ve müşterinin **en yüksek önceliğidir**.

**Teknik spesifikasyon: iterasyon-2/SPEC.md** (sözleşme `packages/contracts` 0.2.0). Tasarım dayanakları: ROTA-UYUMLU-ATAMA.md, KURYE-DEVIR.md. Kararlar: [ADR-0009](kararlar/0009-rota-uyumlu-atama.md), [ADR-0010](kararlar/0010-kuryeler-arasi-devir.md), [ADR-0011](kararlar/0011-osrm-motosiklet-profili-ve-rota-altyapisi.md), [ADR-0012](kararlar/0012-rls-iterasyon-2b-ye-ertelendi.md), [ADR-0013](kararlar/0013-rota-motoru-siniri-ve-yaklasik-mod.md), [ADR-0014](kararlar/0014-toplu-atama-regret-ekleme.md), [ADR-0015](kararlar/0015-uygulama-ici-navigasyon.md) (uygulama içi navigasyon, ek kapsam).

Beş paralel görev (gorevler/): **R** rota altyapısı + motor + benchmark, **A2** API rota ve atama alanı, **H2** API devir alanı (A2'nin M1 dönüm noktasından sonra), **B2** panel, **C2** kurye uygulaması. Birleştirme sırası: R → A2 → H2 → B2 → C2 → entegrasyon turu (SPEC §20.5).

**Çekirdek: dinamik rota uyumlu dağıtım** (kabul kriterleri SPEC §19.0 (a)-(f))
- [x] OSRM (MLD, motosiklet profili) + VROOM konteynerleri (compose `routing` profili) ve tekrar üretilebilir Eskişehir çıkarımı (osmium, PowerShell + Bash) ([ADR-0011](kararlar/0011-osrm-motosiklet-profili-ve-rota-altyapisi.md))
- [x] Kurye başına canlı rota ve duraklar (`courier_routes`, `route_stops`; `PICKUP` / `DROPOFF` / `HANDOVER`)
- [x] En ucuz yerleştirme + uygulanabilirlik (kapasite, tazelik 35 dk, **mevcut paket başına gecikme sınırı 8 dk**) + skor + gerekçeli öneri ("Rotası üzerinde, +2 dk"); 25 kurye için < 300 ms
- [x] **Toplu atama:** birden çok yeni paketin tüm kuryelerin mevcut rotalarına birlikte yerleştirilmesi (regret-2), aynı restoran siparişlerinin gruplanması ([ADR-0014](kararlar/0014-toplu-atama-regret-ekleme.md))
- [x] Nihai sıra (VROOM; yokken süreç içi kesin sıralayıcı) ve kararlılık kuralları: kilitli sıradaki durak, 2 dk kazanç eşiği, soğuma, sessiz değişiklik yok
- [x] Üç atama modu: `ONERI` (varsayılan), `AKILLI_HAVUZ`, `OTOMATIK` (+ toplu pencere) ve bekleyenler kuyruğu; kiracı ayarları
- [x] Rota motoru düşerse açık işaretli yaklaşık mod ve İterasyon 1 davranışına dönüş ([ADR-0013](kararlar/0013-rota-motoru-siniri-ve-yaklasik-mod.md))
- [x] Panel: rota çizgileri ve numaralı duraklar (yeni paketle canlı değişim), öneri paneli (ayrıntı + oluşturma), sürükle-bırak atama ve büyük sapma / gecikme uyarısı, bekleyenler ve toplu öneri
- [x] Kurye uygulaması "Rotam": numaralı duraklar (liste + harita), sıradaki durağa tek dokunuş navigasyon (Google Maps devri; uygulama içi navigasyon aşağıda), rota değişince şerit ("Yeni paket rotana eklendi: +3 dk")
- [x] **Uygulama içi adım adım navigasyon** ([ADR-0015](kararlar/0015-uygulama-ici-navigasyon.md), görevler N1 API + N2 uygulama): "Navigasyonu başlat" Google Maps'e atmak yerine uygulama içinde yeşil bantlı, eğik / yön-yukarı kameralı, Türkçe sesli yönlendirmeli, yoldan çıkınca otomatik yeniden rotalı ekran açar; varışta durak kartı, işlemden sonra sonraki durağa otomatik navigasyon. OSRM kapalıysa Google Maps devri yedek. Canlı trafik, şerit ve hız sınırı yok
- [x] Simülasyon benchmark'ı (`npm run benchmark`): sentetik Eskişehir günü (~20 kurye, ~400 paket, yoğun saatler, aynı restoran siparişleri), "en yakın kurye" ile rota uyumlu atama karşılaştırması (toplam km, ortalama teslim süresi, geç teslim %, geri dönüş); grafik + rapor `docs/benchmark/`

**Kontrollü ret ve devir**
- [x] Gerekçeli ret (alıştan önce) ve yeniden dağıtım; `release` uyumluluğu; ret uyarısı; metrikler (ortalama sapma, elle yeniden atama, ret oranı, kötü atama oranı, zamanında teslim)
- [x] Yönetici onaylı devir isteği (yol B) ve kuryeler arası devir (yol A, [ADR-0010](kararlar/0010-kuryeler-arasi-devir.md)): aday listesi, teklif, iki aşama, QR / 4 haneli kod, buluşma yarıçapı, tazeliğin orijinal alıştan sürmesi, nakit sorumluluğu devri, sınırlar, yönetici onayı ve elle onay, metrikler
- [x] Kurye uygulaması devir ekranları (aday listesi, teklif kartı, buluşma, kod / QR, QR tarama, nakit onayı); panel devirler sekmesi, onaylar, devir ayarları
- [x] KVKK: aydınlatma metni güncellemesi (TASLAK metin; hukuki onay ve açık soruların pilotla kapatılması açık) (meslektaşa yakınlık / buluşma konumu, rota motoru), yeni `consentVersion`

**Teslim edilen (özet).** Gerçek rota motoru (OSRM motosiklet profili + VROOM; Eskişehir çıkarımı `npm run routing:build` ile tekrar üretilebilir) ve saf TypeScript atama motoru (en ucuz yerleştirme, regret-2 toplu atama, kararlılık kuralları) çalışır; motor düşerse açık işaretli yaklaşık moda iner. Panel: gerçek yolları izleyen rota çizgileri ve numaralı duraklar, öneri paneli, sürükle-bırak atama ve sapma uyarısı, bekleyenler, toplu öneri, devirler, ayarlar ve metrikler. Kurye uygulaması: "Rotam", uygulama içi adım adım navigasyon (gece teması sisteme / saate uyar), gerekçeli ret ve kuryeler arası devir (kod / QR). Benchmark ([docs/benchmark](benchmark/README.md)): sentetik Eskişehir gününde "en yakın kurye"ye göre toplam km -%30 ve geç teslim %19,9 → %3,2 (**simülasyon**, saha sonucu değil). Demo rehberi DEMO.md İterasyon 2 için güncellendi.

**Açık kalanlar (takip işleri):**
- Aynı restoranın siparişlerini **kuryenin mevcut rotasındaki** paketle gruplama (şimdi yalnızca toplu pencerede gruplanır)
- **Gerçek cihaz saha testi** (uygulama içi navigasyon ve devir yalnızca emülatörde ve mock / gerçek API ile doğrulandı); Xiaomi / Samsung pil davranışı
- iOS derlemesi ve TestFlight (Apple Developer hesabı gerekir)
- Push bildirim (FCM; Firebase projesi gerekir): `CourierNotifier` bağlantı noktası hazır
- PostgreSQL RLS (ikinci kiracıdan önce zorunlu)
- Panelde atama önizlemesi (kesikli çizgi) düz çizgidir; gerçek yol geometrisi ile çizmek için öneri yanıtına geometri eklenmeli
- Sentetik benchmark'ın pilotun gerçek siparişleriyle doğrulanması; `ROUTING_DURATION_FACTOR` kalibrasyonu

**Pilotla doğrulanacak varsayımlar:** kurye başına azami paket, kabul edilebilir sapma, azami teslim süresi, mevcut paket gecikme sınırı, restoran hazırlık süresi bilgisi, otomatik atama tercihi, ret politikası, çoklu restoran alışı, `ROUTING_DURATION_FACTOR` kalibrasyonu ([MIMARI açık sorular 17-29](MIMARI.md#8-açık-sorular)).

**Portföy çıktısı:**
- Benchmark sonuç grafiği ("en yakın kurye" ile rota uyumlu atama), README'de yayınlanır (sonuç ne ise).
- Rota haritası GIF'i: numaralı duraklar, yeni paketin sapma dakikasıyla önerilmesi, atanması ve rotanın canlı yeniden kurulması; aynı restorandan üç siparişin toplu atanması.
- Kuryeler arası devir GIF'i: aday listesi, teklif kartı, kod / QR ile devir tamamlanması ve panelde canlı olay.
- "Rotam" ekran görüntüleri.

---

## İterasyon 2b — Platform ve sağlamlaştırma ⏳

İterasyon 2'den bilinçli olarak ayrılan maddeler (SPEC §1.3). Ön koşulu ürün sahibinin hesap / anahtar sağlaması olanlar işaretlidir.

- [ ] Aynı restoranın siparişlerini kuryenin **mevcut rotasındaki** paketle gruplama (İterasyon 2'de yalnızca toplu pencerede)
- [ ] Gerçek cihazda saha testi (uygulama içi navigasyon, devir, arka plan konumu; Xiaomi / Samsung)
- [ ] PostgreSQL RLS (ikinci savunma hattı; `withTenant`, iki DB rolü) — **ikinci kiracı canlıya alınmadan önce zorunlu** ([ADR-0012](kararlar/0012-rls-iterasyon-2b-ye-ertelendi.md))
- [ ] Push bildirim (FCM / APNs): yeni paket, atama, rota değişikliği, devir teklifi — *Firebase projesi gerekir* (`CourierNotifier` bağlantı noktası hazır)
- [ ] iOS derleme ve TestFlight dağıtımı (EAS Build) — *Apple Developer hesabı gerekir* ([ADR-0003](kararlar/0003-dagitim-apk-self-update.md))
- [ ] Adres kalitesi testi: ~30 örnek adreste Google geocode ile OSRM snapping karşılaştırması — *Google Maps anahtarı gerekir* — **İterasyon 3'e taşındı (3a-8), Google'sız**
- [ ] Panelde durağı sürükleyerek elle sıralama (kilit altyapısı İterasyon 2'de hazır)
- [ ] OTA güncelleme kanalı (EAS Update veya kendi sunucumuz) ve uygulama içi APK güncellemesi (PackageInstaller)
- [ ] OEM'e özel pil optimizasyonu yönlendirme ekranları (Xiaomi, Huawei, Samsung ayrıntılı)
- [ ] Panel oturumunda httpOnly çerez; demo HTTP yerine HTTPS
- [ ] Ürün sahibinin ekleyebileceği, bağlantı noktaları hazır özellikler: restoran "hazır" sinyali (`ready_at`), teslim kanıtı geofence (`DeliveryVerifier`) (SPEC §13) — **İterasyon 3'e taşındı (3b-2, 3b-4)**

---

## İterasyon 3 — Eskişehir'e özel harita platformu (3a ✅, 3b ✅, 3c ⏳) <a id="iterasyon-3-harita"></a>

**Hedef:** Tüm harita, rota ve adres verisi kendi sistemimizin içinde çalışır; çalışma zamanında üçüncü taraf harita servisine bağımlılık yoktur. Genel amaçlı haritaları **bizim kullanım alanımızda** (tek şehir, motosikletli kurye, tekrar eden adresler) geçmek; canlı trafik, kapı düzeyi adres, genel POI ve sokak görüntülerinde Google'ın üstün kaldığını kabul etmek.

**Plan: iterasyon-3/PLAN.md.** Kararlar: [ADR-0016](kararlar/0016-eskisehire-ozel-kendi-barindirdigimiz-harita-platformu.md) (kendi karolarımız: planetiler + PMTiles; kendi Photon; haftalık kesintisiz OSM güncelleme hattı; çevrimdışı harita), [ADR-0017](kararlar/0017-saha-verisinden-ogrenen-harita.md) (öğrenilmiş teslim noktaları ve öğrenilmiş hızlar). Mimari: [MIMARI §9](MIMARI.md#9-harita-platformu-iterasyon-3).

**3a (hızlı, yüksek değerli)** — teknik spesifikasyon: iterasyon-3/SPEC-3a.md (sözleşme 0.3.0; kararlar [ADR-0018](kararlar/0018-harita-karo-ve-stil-sunumu.md) ... [ADR-0022](kararlar/0022-uretim-dagitimi-ucretsiz-katman.md)). Beş paralel görev (gorevler/): **Q** API (idempotency, harita uçları, rota bacakları, ping toplama), **T** karolar + stiller + panel, **G** Photon + haftalık hat + adres testi, **M** kurye uygulaması çevrimdışı, **P** üretim hazırlığı (ücretsiz). Birleştirme sırası: Q -> T -> G -> M -> P -> entegrasyon turu. Ücretli kalemler yalnızca ürün sahibi kararı (SPEC-3a §18).
- [x] Kendi vektör karoları ve kurye stili (3a-1), kendi Photon geocoder'ı (3a-2), haftalık OSM güncelleme hattı (3a-3)
- [x] Çevrimdışı harita indirme + sürüm denetimi (3a-4), çevrimdışı rota / durak önbelleği (3a-5)
- [x] Çevrimdışı eylem kuyruğu + idempotency (3a-6)
- [x] Ping özetleme, **yalnızca toplama**, pilotun ilk gününden (3a-7; iş varsayılan kapalı, KVKK P-9 onayına bağlı)
- [x] Adres kalitesi testi, Google'sız (3a-8); Photon sorgu normalleştirmesi API'de (<= 50 m isabet %71 -> %83, [rapor](benchmark/adres-kalitesi/README.md))
- [x] Üretim hazırlığı (ücretsiz kısım): prod compose + Caddy, yedek + geri yükleme tatbikatı, izleme, CI, bootstrap komutu (ilk kiracı + OWNER), `postgis/postgis:16-3.6`
- [x] Entegrasyon turu: gerçek yığınla (kendi karolar + Photon + OSRM/VROOM + API + panel + emülatör) uçtan uca doğrulama; bulunan hatalar düzeltildi (ENTEGRASYON-DURUM.md)

**3a sonrası açık maddeler (follow-up):**
- [ ] Gerçek cihazda saha testi (CIHAZ-TEST-PROTOKOLU.md); sürüm derlemesi (`KURYE_ALLOW_CLEARTEXT=0`, düz HTTP kapalı) ve JS gömülü APK ile **uçak modunda uygulamayı yeniden başlatma** denemesi (dev client uçak modunda Metro'ya bağlanamıyor)
- [ ] iOS derlemesi ve cihaz testi; push bildirimleri (FCM / APNs)
- [ ] Satır düzeyinde güvenlik (RLS) — ikinci kiracıdan önce ([ADR-0012](kararlar/0012-rls-iterasyon-2b-ye-ertelendi.md))
- [ ] Çevrimdışıyken ETA / varış saatlerinin soluklaştırılması (şu an son değer olduğu gibi görünür)
- [ ] "Çevrimdışı" göstergesini yalnızca cihaz ağına değil sunucuya erişilememesine de bağlamak (zayıf kapsamada ağ "bağlı" görünür; uygulama 4 sn sonra önbellekle açılır)
- [ ] Panelde `courier.sync_conflict` bildiriminin gerçek akışta ekran görüntüsü; ikinci şehir poligonu için bootstrap seçeneği
- [ ] Adres kalitesi: site / kampüs kapısı ve belediye binası gibi yerlerde konum düzeltme (öğrenilmiş nokta, 3b-1)
- [x] Navigasyon ekranı akıcılığı ve course-up düzeltmesi (harita rota yönüne döner; yerel öngörülü kamera, konum başına harita ağacı çizimi yok, düşük donanım kademesi; emülatörde ortalama kare hızı 23,2 -> 27-34 fps, >= 100 ms takılma 71 -> 5-10, 60 fps elde edilmedi; [ADR-0024](kararlar/0024-navigasyon-render-performansi.md), ölçüm)
- [ ] Navigasyon kare ölçümü gerçek ucuz Android cihazda (protokol); emülatörde MapLibre GL iş parçacığı doygun, 60 fps görülemedi
- [x] Navigasyon ekranı B tasarımı ve marka kimliği: koyu krom + neon yeşil, gündüz açık "gümüş" harita, manevra ikon seti, Inter / Barlow Condensed (ürün sahibi B'yi seçti; maketler, belirteçler ve uygulama; [ADR-0027](kararlar/0027-marka-kimligi-ve-navigasyon-gorsel-dili.md), [tasarım](tasarim/navigasyon/README.md), [marka](tasarim/marka/README.md), uygulama notu). Diğer kurye ekranları ve yönetim paneli geçişi ayrı görev

**3b ✅** — teknik spesifikasyon: iterasyon-3/SPEC-3b.md (sözleşme 0.4.0; kararlar [ADR-0025](kararlar/0025-isletme-hazir-baglantisi.md), [ADR-0026](kararlar/0026-teslim-geofence-politikasi.md); görev brief'leri: Q3b, R3b, W3b, M3b). Sekiz PR (#31-#38) 2026-11-07'de birleşti; birleşik ağaç çakışmasız ve yeşil (API birim 626 / e2e 341 / panel 135 / mobil 456 / harita 36). Kayıt: ENTEGRASYON-3B-DURUM.md.
- [x] Öğrenilmiş teslim noktaları (3b-1): adres defteri, HMAC adres anahtarı, ağırlıklı medyan ve güven, sabitle / sıfırla; `DELIVERY_POINT_LEARNING_ENABLED=false` ile kapalı başlar
- [x] Restoran "paket hazır" sinyali ve hazırlık süresi öğrenme (3b-2): panel, kurye ve girişsiz işletme bağlantısı (QR); hazırlık tahmini
- [x] Aynı restoran gruplamayı iyileştirme (3b-3): motor + 30 tohumlu benchmark; kiracı ayarı **varsayılan kapalı** (önerilen 150 / 240 / 0,5; [rapor](benchmark/v2/gruplama.md))
- [x] Teslim geofence'i (3b-4): `OFF | OBSERVE | ENFORCE` (varsayılan OBSERVE), gerekçeli geçersiz kılma, çevrimdışı kuyruk uyumu
- [x] Dağıtım v2 ayarları için panel ekranı (3b-5) (SPEC-3b §10)

**3b sonrası açık maddeler:**
- [ ] Gerçek cihaz saha testi (emülatör kabul testi sürüyor; sonucu ayrıca eklenecek)
- [ ] Gerçek `prod.env` için `ADDRESS_KEY_SECRET` (aksi hâlde 3b dağıtımı API'yi başlatmaz)
- [ ] Karar: öğrenilmiş nokta güven eşiği (k = 3 gözlemde yalnızca ~%61 kullanılabilir; N0 gevşetilsin mi)
- [ ] Karar: gruplama (T) ön ayarı ve pilotta firma bazında açılış; hazır zamanı bilinince km +%25-40 incelemesi
- [ ] Karar: geofence ENFORCE zamanı (tüm kuryeler 3b APK'sında, OBSERVE'de geçersiz kılma < %15)

**3c**
- [ ] Öğrenilmiş hızların OSRM'ye uygulanması, ETA hatası (MAE) önce / sonra, benchmark güncellemesi (pilot verisi >= 4 hafta gerekir)

**Algoritma v2 kolu (paralel)**
- [x] Faz 1: araştırma ve benchmark — kayan ufuk toplu atama, kuryeler arası ALNS, CBBA karşılaştırması (DAGITIM-ALGORITMASI-V2.md, [benchmark v2](benchmark/v2/README.md), [ADR-0023](kararlar/0023-dagitim-algoritmasi-v2.md))
- [x] Faz 2: kayan ufuk + gecikmeli atama `DISPATCH_V2` ve kiracı ayarı arkasında üretimde (varsayılan kapalı), ALNS gölge kipte (`DISPATCH_V2_ALNS_SHADOW`), `GET /metrics/dispatch-v2`; benchmark üretim yolu (RH60-PROD) Faz 1 ile birebir aynı ([ADR-0023](kararlar/0023-dagitim-algoritmasi-v2.md) kabul edildi, [MIMARI §6.1.1](MIMARI.md#611-dağıtım-v2-ayarları-adr-0023-faz-2))
- [ ] Faz 3: pilotta firma bazında açılış ve gölge ALNS verisiyle gerçek ALNS taşımalarına (`DISPATCH_V2_ALNS`) karar

**Pilot öncesi, paralel: üretim hazırlığı** (P-1 ... P-9): Türkiye'de VPS ve boyutlandırma, HTTPS, yedek ve geri yükleme, izleme ve uyarı, log saklama, CI/CD, gizli bilgi politikası, Xiaomi / Samsung gerçek cihaz testi, KVKK hukuki onayı.

**Not:** 2b'deki "adres kalitesi testi" ve "restoran hazır sinyali / teslim geofence" maddeleri bu iterasyona taşındı (3a-8, 3b-2, 3b-4); "Google anahtarı gerekir" ön koşulu kalktı (Google isteğe bağlı sütundur).

**Portföy çıktısı:** Kendi karomuzun ve çevrimdışı çalışmanın GIF'leri, adres kalitesi grafiği, kesintisiz güncelleme grafiği, öğrenilmiş teslim noktası önce / sonra haritası, benchmark güncellemeleri (hazır zamanı, gruplama, ETA).

---

## İterasyon 4 — Pilot hazırlığı ✅ <a id="iterasyon-4-pilot"></a>

> Numaralandırma: eskiden "İterasyon 4" olan hakediş / kasa işi [İterasyon 5](#iterasyon-5-hakedis)'e, eski "İterasyon 5+" genişleme işi İterasyon 6+'ya kaydı; bu iterasyon, harita platformundan (İterasyon 3) sonra pilot müşteriyle sahaya çıkmadan önce gereken dağıtım, güvenlik, saha operasyonu ve ölçüm işlerini toplar. Yol haritasındaki eski "MVP+ 4a ... MVP+ 12" etiketleri bu iterasyonun parçalarıdır. Eski SPEC ve ADR belgelerindeki "İterasyon 4 (hakediş)" (ve daha eskisi "İterasyon 3 (hakediş / kasa)") atıfları önceki numaralandırmaya göredir ve İterasyon 5 olarak okunur; eski "İterasyon 5+" atıfları İterasyon 6+ demektir.

**Tamamlandı: 2026-11-13.** PR #43-#66 `main`'de. Sözleşme `packages/contracts` **0.7.0**, veritabanı geçişleri 0014'e kadar (15 dosya), karar kayıtları ADR-0028 ... ADR-0037. Teknik spesifikasyonlar: SPEC-4a (dağıtım pilot kipi), SPEC-4b (konum kademeleri, rota geri bildirimi), SPEC-4c (kapıda tahsilat). Sahada açma, izleme ve geri dönüş adımları: çalışma kitabı.

**Hedef:** pilot müşteriye "havuzu kuryeye gösterme, sistem atasın, kurye kabul etsin" düzenini, pilot öncesi güvenlik ve KVKK işlerini, menajerin ihtiyaç duyduğu uyarı / kasa / mola araçlarını ve pilot ölçeğinin kaldırılıp kaldırılamayacağı ölçümünü tamamlamak.

### Teslim edilenler

| Konu | PR | ADR / belge |
|---|---|---|
| **Dağıtım pilot kipi:** havuz kuryelerden gizli (kiracı ayarı), atama kabulü ve süre aşımı (pilot ayarı 90 sn, süre dolunca sistem reddi ve yeniden dağıtım), panelde Havuz sayfası, atama gerekçesi, kabul geri sayımı, işletme sayfasında tahmini varış | #45 (API), #47 (mobil), #50 (panel) | [ADR-0028](kararlar/0028-havuz-gorunurlugu-ve-atama-kabulu.md), SPEC-4a |
| **Konum kademeleri:** navigasyonda sık, duran kuryede seyrek konum; panel yayını kurye başına <= 1 olay / 3 sn. **"Rota yanlış":** kurye navigasyondan tek dokunuşla bildirir; panel sayfası, durum yönetimi, OpenStreetMap düzeltmesi için kimliksiz GeoJSON dışa aktarma | #49, #52 | [ADR-0029](kararlar/0029-konum-kademeleri-ve-rota-geri-bildirimi.md), SPEC-4b |
| **Rota doğruluğu ölçümü** (30 + 300 çift, OSM çapraz kontrolü) ve telefonla saha test protokolü | #48 | rapor, protokol |
| **Telefon testi (APK) kiti** ve kurye uygulamasında WebSocket bağlantı sınırı kodlarına bekleme + bildirim | #51, #56 | telefon protokolü, APK dağıtımı |
| Navigasyon hedef işareti uygulama içi durak işaretiyle aynı aileden | #43 | [ADR-0027](kararlar/0027-marka-kimligi-ve-navigasyon-gorsel-dili.md) |
| **Panel marka geçişi (P2):** koyu kenar çubuğu + açık içerik, kurye uygulamasıyla aynı dil | #46 | [panel tasarımı](tasarim/panel/README.md) |
| **Güvenlik denetimi ve regresyon paketi** (`npm run security`: kimlik / rol matrisi, kiracılar arası erişim, jeton kurcalama, hız sınırı, üretim başlıkları; Definition of Done'a girdi) | #44 | [GELISTIRME-SURECI](GELISTIRME-SURECI.md#definition-of-done), denetim |
| **Güvenlik P0 maddeleri kapatıldı:** panel içerik güvenlik politikası (önce izleme kipinde) ve bağlantı sınırları (#54); giriş kilidi, güçlü şifre kuralı ve zorunlu ilk şifre değişimi (#55) | #54, #55 | [ADR-0031](kararlar/0031-panel-csp-ve-baglanti-sinirlari.md), [ADR-0030](kararlar/0030-kimlik-sertlestirme.md) |
| **Güvenlik P1 maddeleri:** küçük işler (hız sınırı kapsamı, sayaç atomikliği, sürüm derleme denetimleri, geliştirme altyapısı sıkılaştırması, bağımlılık denetimi CI işi) (#63) ve **panel oturum belirtecinin HttpOnly çerezde tutulması** (#66; panel için ayrı `/auth/web/*` uçları, `SameSite=Strict` çerez + CSRF başlığı / köken denetimi, erişim belirteci yalnızca bellekte, sözleşme 0.7.0; dağıtımda panel kullanıcıları bir kez yeniden girer, kurye uygulaması değişmedi) | #63, #66 | [ADR-0037](kararlar/0037-panel-refresh-cerezi.md) |
| **Atama isabeti ve harita verisi tarihi:** Metrikler sayfasında "Atama isabeti" kartı (tek, açıklanabilir oran; isabetsizlik nedenleri ve günlük eğilim; kurye sıralaması yok), Ayarlar > Hizmet bölgesi'nde "Harita verisi: tarih" ve eskime uyarısı, zamanlanmadan hazır haftalık güncelleme betiği, pilot ön ayarında kurye başına 5 aktif paket (sözleşme 0.7.1) | bu PR | [ADR-0038](kararlar/0038-atama-isabeti-ve-harita-verisi-tarihi.md) |
| **KVKK müşteri verisi saklama süresi:** kiracı ayarı, saatlik anonimleştirme işi, panel Ayarlar > Gizlilik kartı; yeni kiracıda varsayılan 30 gün, **mevcut kiracılarda kapalı** | #57 | [ADR-0032](kararlar/0032-musteri-verisi-saklama.md) |
| **Yük testi ve darboğaz düzeltmeleri:** pilot ölçeği senaryoları (kurye, panel, sipariş dalgaları), panel liste yenilemelerinin olay güdümlü olması, bekleyenler yeniden deneme aralığı ayarı, konum eklemesi tuning'i; navigasyon karışımıyla 25 / 50 / 100 kurye (ikişer koşu) ve bir keşif koşusu olarak 200 kurye ölçümü | #53, #58, #68 | [ADR-0033](kararlar/0033-yuk-testi-darbogazlari.md), rapor |
| **Kapıda tahsilat mutabakatı:** teslimde nakit / kart / alınamadı, "Bugün üzerimde" kartı; panelde Tahsilat sayfası, kasa teslim alma, yalnızca-ekle kayıt, CSV | #59 | [ADR-0034](kararlar/0034-kapida-tahsilat-mutabakati.md), SPEC-4c |
| **Kurye molası:** uygulamada "Mola ver" + geri sayım; molada atama aday saymaz, süre kiracı ayarı, panelde "Molada" çipi ve geçmiş | #60 | [ADR-0035](kararlar/0035-mola-durumu.md) |
| **Gecikme uyarıları:** geç alım, geç teslim, konumu kesilen ve yerinde duran kurye; zil + çekmece + Uyarılar sayfası, kiracı eşikleri, işletme sayfasında "Gecikiyor" rozeti (panel içi; FCM / SMS yok) | #61 | [ADR-0036](kararlar/0036-gecikme-uyarisi.md) |
| **Ret oranı uyarısı:** vardiyadaki kurye son 24 saatte tekliflerin en az %40'ını reddederse "Yüksek ret" uyarısı (en az 5 teklif; kabul süresi aşımları orana girmez) | #65 | [ADR-0036 eki](kararlar/0036-gecikme-uyarisi.md) |
| **Takip paketi:** moladaki kurye devir adayı sayılmaz, dağıtım v2 pencere kararı ile mola uyumu, KVKK onay metni, mobil soket bekleme, kabul süresi dolduğu bilgisi | #62 | [ADR-0035](kararlar/0035-mola-durumu.md) |
| **Kurye uygulamasında paket arama ve süzgeç:** işletme / adres / paket kodu ile arama (Türkçe İ-ı ve aksan duyarsız), durum ve ödeme türü çipleri, boş sonuç durumu; çevrimdışı çalışır | #64 | çalışma kitabı |
| **Yandex Navi ile aç:** dış navigasyon devrinde Google Maps'in yanına Yandex Navi seçeneği; Profil'de cihazda saklanan tercih, düğme etiketi tercihi yansıtır, paket detayında uzun basışla tek seferlik diğer uygulama; Yandex Navi yoksa Yandex Haritalar, Google Maps ve `geo:` yedek zinciri (uygulama içi navigasyon değişmez) | #70 | mobil README, telefon protokolü |
| **Ters atama ölçütü ve ölçümü:** "ters atama"nın geometri + süre tanımı (eşikler veriden), saf sınıflandırıcı ve 8 senaryolu (25 / 50 kurye, 2-6 sipariş/dk, karma / karşıt akış, kapasite 5) tekrar üretilebilir benchmark; OTOMATİK v1 pilot ayarlarıyla sistem atamalarında %10,3 ters (en yakın kurye %17,3), ana neden toplu pencere kararı; benchmark'ta yön koruması %3,1'e indiriyor (uygulanması ürün sahibi kararı; üretim davranışı değişmedi) | bu PR | [ADR-0039](kararlar/0039-ters-atama-olcutu.md), [rapor](benchmark/TERS-ATAMA-2026-11.md) |

**Yük testi:** ayrıntılı sonuçlar YUK-TESTI-2026-11.md raporundadır (bölüm 10). Navigasyon profiliyle 25, 50 ve 100 kuryede ikişer (10 dakikalık) koşu ve bir keşif koşusu olarak 200 kuryede toplam 56.363 istekte sunucu hatası 0 çıktı. Canlı kurye listesi p95: 25 / 50 / 100 kuryede sırasıyla 62-109 / 108-114 / 136-137 ms; konum yazımı p99: 41-49 / 65-67 / 89-94 ms (karışık genel p95 yanıltıcıdır: hızlı konum istekleri karışımın yaklaşık %20'sinden %41'ine çıkar). Sistem atamasında p95 35-48 sn'dir (25 kuryede 47-48 sn); uzun atamalar kuyrukta 30 sn'lik yeniden denemeyi bekleyen paketlerdir ve kurye sayısıyla artmaz. **Sonuç: 100 kuryeye kadar, dakikada 3 siparişle hatasız.** Sipariş hızı kurye sayısıyla ölçeklenmedi; gerçekçi sipariş hızında dağıtım yükü denenmedi (orantılı sipariş hızıyla koşu takip işidir).

### Teslim edilmeyenler / sonraya taşınanlar

Ürün sahibinin kararı veya hesabı gereken maddeler (hiçbiri ücretli servis açılmadan ilerletilmez):
- **Hakediş tarifesi:** paket başı / kurye bazlı kural ve ödeme dönemi (İterasyon 5'i başlatır)
- **Push bildirim (FCM) ve Firebase hesabı:** bugün uyarılar ve atama bildirimleri panel / uygulama açıkken WebSocket ile gelir
- **Teslim fotoğrafı:** depolama yeri ve saklama süresi kararı
- **İşletme paneli** ve **platform siparişi için fiş / tablet** akışı
- **OSM düzeltmeleri:** "Rota yanlış" listesinden çıkan düzeltmelerin kimin, hangi ritimle yapacağı
- **KVKK saklama süresinin pilot kiracıda açılması** (hukuki görüşe göre süre; mevcut kiracıda bilinçli olarak kapalı)
- **Panel CSP'nin zorunlu kipe alınması** (pilot haftasından sonra, tek satırlık değişiklik)
- **Telefon numarası tekilliği kuralı** (ürün kararı)
- **Üretim sunucusu (VPS) seçimi ve kurulumu**
- **Gerçek telefonda saha testi** (arka plan konum, pil optimizasyonu, navigasyon akıcılığı; protokol)
- **Kurye geçmiş sorgusu:** "Bugün" listesi yalnızca bugünü döndürür; tarih çipleri (dün / son 7 gün) arama alt yapısında hazır ama sunucu tarafı geçmiş ucu yok

Küçük teknik takip işleri iç takip belgesinde izlenir. İterasyon 2b ve [İterasyon 3](#iterasyon-3-harita) açık maddeleri (RLS, iOS, FCM, gerçek cihaz, 3c) geçerliliğini korur.

**Portföy çıktısı:** dağıtım pilot kipi panel ekran görüntüleri (Havuz, kabul bekliyor, atama gerekçesi), uyarı çekmecesi, tahsilat / mola / arama ekranları ([README](../README.md#ekran-görüntüleri)); yük testi raporu.

---

## İterasyon 5 — Hakediş, kasa raporu ve atama (öneri) ⏳ <a id="iterasyon-5-hakedis"></a>

> Bu bölüm bir **öneridir, taahhüt değildir**: kapsam ve sıra pilot geri bildirimine ve yukarıdaki ürün sahibi kararlarına (özellikle hakediş tarifesi) göre değişir. Eski "İterasyon 4 — Hakediş, kasa ve atama" bölümünün devamıdır.

**Hedef:** Kurye güvenini artıran şeffaf hakediş ve finansal takip.

- [ ] Paket bazlı hakediş hesabı ve kuryenin hakediş ekranı (tarife kararı gerekir)
- [ ] Yönetici raporları (kurye, işletme, dönem bazlı)
- [x] Kasa / nakit mutabakatı (kapıda ödeme takibi): İterasyon 4'te teslim edildi ([ADR-0034](kararlar/0034-kapida-tahsilat-mutabakati.md)); hakediş ile birleşik rapor hâlâ açık
- [ ] ETA hesaplama ve müşteriye gösterimi (rota ETA'ları İterasyon 2'de iç kullanım için üretilir; işletme sayfasında tahmini varış İterasyon 4'te geldi)
- [ ] Adil atama ağırlıklarının hakedişle birlikte ayarı (temel adalet terimi ve otomatik / akıllı atama İterasyon 2'de gelir)
- [ ] Pilot geri bildirimiyle çıkan düzeltmeler: eşik ayarları (uyarılar, kabul süresi), kurye geçmiş sorgusu, gerçek cihaz bulguları

**Portföy çıktısı:** Kurye hakediş ekranı ve panel rapor ekran görüntüleri; mutabakat akışı GIF'i.

---

## İterasyon 6+ — Genişleme ⏳

**Hedef:** Ürünü ölçeklenebilir ve satılabilir hâle getirmek.

- [ ] İşletme paneli (restoranlar kendi siparişlerini girer)
- [ ] POS entegrasyonu, ilk hedef Adisyo (herkese açık API)
- [ ] Platform entegrasyonları (restoranın kendi kimlik bilgileriyle ya da aracı üzerinden)
- [ ] SoftPOS ile kapıda kartlı ödeme
- [ ] Faturalama (paket başı)

**Portföy çıktısı:** İşletme paneli ekran görüntüleri, entegrasyon akış diyagramı.
