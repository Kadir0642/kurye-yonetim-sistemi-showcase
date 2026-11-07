# Yol Haritası

Kapsam, pilot müşteri keşfi sonrasında değişebilir. Her iterasyon bir demo ile kapanır; ayrıca "Portföy çıktısı" ile README için görsel malzeme toplanır. Çalışma düzeni için [GELISTIRME-SURECI.md](GELISTIRME-SURECI.md).

| İterasyon | Odak | Durum |
|---|---|---|
| 0 | Planlama ve araştırma | 🚧 Devam ediyor |
| 1 | Çekirdek | ✅ Uygulandı ve birleştirildi |
| 2 | **Ana hedef: dinamik, rota uyumlu dağıtım** (tek akıllı rota, en iyi uyan kuryeye tekil ve toplu atama, kararlı yeniden sıralama, öneri / otomatik atama, ret ve devir, uygulama içi navigasyon, benchmark) | ✅ Uygulandı ve birleştirildi (SPEC) |
| 2b | Platform ve sağlamlaştırma: RLS, push (FCM), iOS / TestFlight, OTA, aynı restoran gruplaması, gerçek cihaz testi | ⏳ |
| 3 | **Eskişehir'e özel harita platformu:** kendi karo / geocoder / OSM güncelleme hattı, çevrimdışı harita ve eylem kuyruğu, saha verisinden öğrenen harita, üretim hazırlığı | **3a ✅ tamamlandı** (SPEC-3a; entegrasyon turunda gerçek yığınla doğrulandı, açık maddeler aşağıda); **3b ✅ tamamlandı** (SPEC-3b, sözleşme 0.4.0; PR #31-#38 2026-11-07'de birleşti, entegrasyon kaydı; açık: gerçek cihaz, pilot kararları); 3c ⏳ (PLAN) |
| MVP+ 4a | **Dağıtım pilot kipi:** havuz kuryelerden gizli (kiracı ayarı), atama kabulü ve süre aşımı (pilot 90 sn), panelde atama gerekçesi ve havuz bekleme verisi (SPEC-4a, [ADR-0028](kararlar/0028-havuz-gorunurlugu-ve-atama-kabulu.md)) | 🚧 API ve sözleşme 0.5.0 PR #45; mobil (M4a) PR #47; **panel (W4a): havuz görünümü, atama gerekçesi, kabul durumu, pilot ayarları, canlı harita yumuşatma, işletme sayfası tahmini varış** PR açıldı (panel notu) |
| MVP+ 4b | **Konum kademeleri ve "Rota yanlış":** navigasyonda GPS 2,5 sn / gönderim 5 sn, duran kuryede GPS 30 sn / kalp atışı 60 sn, panel `courier.position` kurye başına <= 1 olay / 3 sn; navigasyon menüsünden tek dokunuşlu rota geri bildirimi, panel listesi ve OpenStreetMap için kimliksiz GeoJSON dışa aktarma (SPEC-4b, [ADR-0029](kararlar/0029-konum-kademeleri-ve-rota-geri-bildirimi.md)) | 🚧 API, mobil ve sözleşme 0.6.0 PR #49 birleşti; **panel: "Rota geri bildirimleri" sayfası** (menü rozeti, süzgeçler, durum değişimi, OWNER GeoJSON indirme) PR açıldı |
| Güvenlik (P0, CSP + sınırlar) | **Pilot öncesi güvenlik:** panel içerik güvenlik politikası (CSP, önce Report-Only), WebSocket bağlantı sınırları (IP başına 20 kimliksiz, kullanıcı başına 10), `/health` sınırı ve önbelleği, Caddy başlık / gövde sınırı (denetim B-06 P0, B-07, [ADR-0031](kararlar/0031-panel-csp-ve-baglanti-sinirlari.md)) | 🚧 PR açıldı; CSP zorunlu geçişi pilot haftası sonrası |
| Güvenlik (P0, kimlik) | **Pilot öncesi kimlik sertleştirmesi:** giriş kilidi kimlik + IP (yabancı kuryeyi dışarıda bırakamaz), kimlik başına yüksek eşik ve artan gecikme; kurye / panel şifresi en az 8 karakter + yaygın şifre listesi + telefon içermez; panelin verdiği şifre geçici, kurye uygulamasında ilk girişte şifre değiştirme ekranı ([ADR-0030](kararlar/0030-kimlik-sertlestirme.md), denetim B-04 / B-05, sözleşme 0.6.1) | 🚧 PR #55 açıldı |
| 4 | Hakediş, kasa ve atama (eski İterasyon 3) | ⏳ |
| 5+ | Genişleme (eski İterasyon 4+) | ⏳ |

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

## İterasyon 4 — Hakediş, kasa ve atama ⏳

> Numaralandırma: eskiden "İterasyon 3" olan bu iş, harita platformunun ([İterasyon 3](#iterasyon-3-harita)) öne alınmasıyla İterasyon 4'e kaydı. Eski belgelerdeki "İterasyon 3 (hakediş / kasa)" atıfları İterasyon 4 olarak okunur.

**Hedef:** Kurye güvenini artıran şeffaf hakediş ve finansal takip.

- [ ] Paket bazlı hakediş hesabı ve kuryenin hakediş ekranı
- [ ] Yönetici raporları (kurye, işletme, dönem bazlı)
- [ ] Kasa / nakit mutabakatı (kapıda ödeme takibi)
- [ ] ETA hesaplama ve müşteriye gösterimi (rota ETA'ları İterasyon 2'de iç kullanım için üretilir)
- [ ] Adil atama ağırlıklarının hakedişle birlikte ayarı (temel adalet terimi ve otomatik / akıllı atama İterasyon 2'de gelir)

**Portföy çıktısı:** Kurye hakediş ekranı ve panel rapor ekran görüntüleri; mutabakat akışı GIF'i.

---

## İterasyon 5+ — Genişleme ⏳

**Hedef:** Ürünü ölçeklenebilir ve satılabilir hâle getirmek.

- [ ] İşletme paneli (restoranlar kendi siparişlerini girer)
- [ ] POS entegrasyonu, ilk hedef Adisyo (herkese açık API)
- [ ] Platform entegrasyonları (restoranın kendi kimlik bilgileriyle ya da aracı üzerinden)
- [ ] SoftPOS ile kapıda kartlı ödeme
- [ ] Faturalama (paket başı)

**Portföy çıktısı:** İşletme paneli ekran görüntüleri, entegrasyon akış diyagramı.
