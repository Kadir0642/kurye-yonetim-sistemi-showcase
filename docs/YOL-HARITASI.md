# Yol Haritası

Kapsam, pilot müşteri keşfi sonrasında değişebilir. Her iterasyon bir demo ile kapanır; ayrıca "Portföy çıktısı" ile README için görsel malzeme toplanır. Çalışma düzeni için [GELISTIRME-SURECI.md](GELISTIRME-SURECI.md).

| İterasyon | Odak | Durum |
|---|---|---|
| 0 | Planlama ve araştırma | 🚧 Devam ediyor |
| 1 | Çekirdek | ✅ Uygulandı ve birleştirildi |
| 2 | **Ana hedef: dinamik, rota uyumlu dağıtım** (tek akıllı rota, en iyi uyan kuryeye tekil ve toplu atama, kararlı yeniden sıralama, öneri / otomatik atama, ret ve devir, benchmark) | 🚧 Spesifikasyon hazır (SPEC) |
| 2b | Platform ve sağlamlaştırma: RLS, push, iOS, OTA, adres kalitesi testi | ⏳ |
| 3 | Hakediş, kasa ve atama | ⏳ |
| 4+ | Genişleme | ⏳ |

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

## İterasyon 2 — ana hedef: dinamik, rota uyumlu dağıtım 🚧

**Hedef:** Birden fazla paket taşıyan kurye paketlerini **tek bir akıllı rota** olarak, haritada numaralı duraklarla ve uygulanabilir en kısa sırayla görür. Yeni paket (veya aynı anda gelen paket**ler**) kuryenin **mevcut rotasına en iyi uyduğu** kuryeye, mevcut paketleri gecikme ve tazelik sınırları içinde tutarak atanır ve rota durak olarak yeniden kurulur. Pilot müşterinin sahadaki sorunu: mevcut yazılım yoldaki kuryeye yeni paketi yalnızca yakınlığa göre atıyor, paket çoğu zaman rota dışında veya ters yönde kalıyor, kuryeler yanlış atanmış paketi bırakıyor, yönetici haritaya bakıp elle yeniden dağıtıyor. Rota uyumlu dağıtım ürünün ayırt edici özelliği ve müşterinin **en yüksek önceliğidir**.

**Teknik spesifikasyon: iterasyon-2/SPEC.md** (sözleşme `packages/contracts` 0.2.0). Tasarım dayanakları: ROTA-UYUMLU-ATAMA.md, KURYE-DEVIR.md. Kararlar: [ADR-0009](kararlar/0009-rota-uyumlu-atama.md), [ADR-0010](kararlar/0010-kuryeler-arasi-devir.md), [ADR-0011](kararlar/0011-osrm-motosiklet-profili-ve-rota-altyapisi.md), [ADR-0012](kararlar/0012-rls-iterasyon-2b-ye-ertelendi.md), [ADR-0013](kararlar/0013-rota-motoru-siniri-ve-yaklasik-mod.md), [ADR-0014](kararlar/0014-toplu-atama-regret-ekleme.md).

Beş paralel görev (gorevler/): **R** rota altyapısı + motor + benchmark, **A2** API rota ve atama alanı, **H2** API devir alanı (A2'nin M1 dönüm noktasından sonra), **B2** panel, **C2** kurye uygulaması. Birleştirme sırası: R → A2 → H2 → B2 → C2 → entegrasyon turu (SPEC §20.5).

**Çekirdek: dinamik rota uyumlu dağıtım** (kabul kriterleri SPEC §19.0 (a)-(f))
- [ ] OSRM (MLD, motosiklet profili) + VROOM konteynerleri (compose `routing` profili) ve tekrar üretilebilir Eskişehir çıkarımı (osmium, PowerShell + Bash) ([ADR-0011](kararlar/0011-osrm-motosiklet-profili-ve-rota-altyapisi.md))
- [ ] Kurye başına canlı rota ve duraklar (`courier_routes`, `route_stops`; `PICKUP` / `DROPOFF` / `HANDOVER`)
- [ ] En ucuz yerleştirme + uygulanabilirlik (kapasite, tazelik 35 dk, **mevcut paket başına gecikme sınırı 8 dk**) + skor + gerekçeli öneri ("Rotası üzerinde, +2 dk"); 25 kurye için < 300 ms
- [ ] **Toplu atama:** birden çok yeni paketin tüm kuryelerin mevcut rotalarına birlikte yerleştirilmesi (regret-2), aynı restoran siparişlerinin gruplanması ([ADR-0014](kararlar/0014-toplu-atama-regret-ekleme.md))
- [ ] Nihai sıra (VROOM; yokken süreç içi kesin sıralayıcı) ve kararlılık kuralları: kilitli sıradaki durak, 2 dk kazanç eşiği, soğuma, sessiz değişiklik yok
- [ ] Üç atama modu: `ONERI` (varsayılan), `AKILLI_HAVUZ`, `OTOMATIK` (+ toplu pencere) ve bekleyenler kuyruğu; kiracı ayarları
- [ ] Rota motoru düşerse açık işaretli yaklaşık mod ve İterasyon 1 davranışına dönüş ([ADR-0013](kararlar/0013-rota-motoru-siniri-ve-yaklasik-mod.md))
- [ ] Panel: rota çizgileri ve numaralı duraklar (yeni paketle canlı değişim), öneri paneli (ayrıntı + oluşturma), sürükle-bırak atama ve büyük sapma / gecikme uyarısı, bekleyenler ve toplu öneri
- [ ] Kurye uygulaması "Rotam": numaralı duraklar (liste + harita), sıradaki durağa tek dokunuş navigasyon, rota değişince şerit ("Yeni paket rotana eklendi: +3 dk")
- [ ] Simülasyon benchmark'ı (`npm run benchmark`): sentetik Eskişehir günü (~20 kurye, ~400 paket, yoğun saatler, aynı restoran siparişleri), "en yakın kurye" ile rota uyumlu atama karşılaştırması (toplam km, ortalama teslim süresi, geç teslim %, geri dönüş); grafik + rapor `docs/benchmark/`

**Kontrollü ret ve devir**
- [ ] Gerekçeli ret (alıştan önce) ve yeniden dağıtım; `release` uyumluluğu; ret uyarısı; metrikler (ortalama sapma, elle yeniden atama, ret oranı, kötü atama oranı, zamanında teslim)
- [ ] Yönetici onaylı devir isteği (yol B) ve kuryeler arası devir (yol A, [ADR-0010](kararlar/0010-kuryeler-arasi-devir.md)): aday listesi, teklif, iki aşama, QR / 4 haneli kod, buluşma yarıçapı, tazeliğin orijinal alıştan sürmesi, nakit sorumluluğu devri, sınırlar, yönetici onayı ve elle onay, metrikler
- [ ] Kurye uygulaması devir ekranları (aday listesi, teklif kartı, buluşma, kod / QR, QR tarama, nakit onayı); panel devirler sekmesi, onaylar, devir ayarları
- [ ] KVKK: aydınlatma metni güncellemesi (meslektaşa yakınlık / buluşma konumu, rota motoru), yeni `consentVersion`; açık sorular 24-27'nin pilotla kapatılması

**Pilotla doğrulanacak varsayımlar:** kurye başına azami paket, kabul edilebilir sapma, azami teslim süresi, mevcut paket gecikme sınırı, restoran hazırlık süresi bilgisi, otomatik atama tercihi, ret politikası, çoklu restoran alışı, `ROUTING_DURATION_FACTOR` kalibrasyonu ([MIMARI açık sorular 17-29](MIMARI.md#8-açık-sorular)).

**Portföy çıktısı:**
- Benchmark sonuç grafiği ("en yakın kurye" ile rota uyumlu atama), README'de yayınlanır (sonuç ne ise).
- Rota haritası GIF'i: numaralı duraklar, yeni paketin sapma dakikasıyla önerilmesi, atanması ve rotanın canlı yeniden kurulması; aynı restorandan üç siparişin toplu atanması.
- Kuryeler arası devir GIF'i: aday listesi, teklif kartı, kod / QR ile devir tamamlanması ve panelde canlı olay.
- "Rotam" ekran görüntüleri.

---

## İterasyon 2b — Platform ve sağlamlaştırma ⏳

İterasyon 2'den bilinçli olarak ayrılan maddeler (SPEC §1.3). Ön koşulu ürün sahibinin hesap / anahtar sağlaması olanlar işaretlidir.

- [ ] PostgreSQL RLS (ikinci savunma hattı; `withTenant`, iki DB rolü) — **ikinci kiracı canlıya alınmadan önce zorunlu** ([ADR-0012](kararlar/0012-rls-iterasyon-2b-ye-ertelendi.md))
- [ ] Push bildirim (FCM / APNs): yeni paket, atama, rota değişikliği, devir teklifi — *Firebase projesi gerekir* (`CourierNotifier` bağlantı noktası hazır)
- [ ] iOS derleme ve TestFlight dağıtımı (EAS Build) — *Apple Developer hesabı gerekir* ([ADR-0003](kararlar/0003-dagitim-apk-self-update.md))
- [ ] Adres kalitesi testi: ~30 örnek adreste Google geocode ile OSRM snapping karşılaştırması — *Google Maps anahtarı gerekir*
- [ ] Panelde durağı sürükleyerek elle sıralama (kilit altyapısı İterasyon 2'de hazır)
- [ ] OTA güncelleme kanalı (EAS Update veya kendi sunucumuz) ve uygulama içi APK güncellemesi (PackageInstaller)
- [ ] OEM'e özel pil optimizasyonu yönlendirme ekranları (Xiaomi, Huawei, Samsung ayrıntılı)
- [ ] Panel oturumunda httpOnly çerez; demo HTTP yerine HTTPS
- [ ] Ürün sahibinin ekleyebileceği, bağlantı noktaları hazır özellikler: restoran "hazır" sinyali (`ready_at`), teslim kanıtı geofence (`DeliveryVerifier`) (SPEC §13)

---

## İterasyon 3 — Hakediş, kasa ve atama ⏳

**Hedef:** Kurye güvenini artıran şeffaf hakediş ve finansal takip.

- [ ] Paket bazlı hakediş hesabı ve kuryenin hakediş ekranı
- [ ] Yönetici raporları (kurye, işletme, dönem bazlı)
- [ ] Kasa / nakit mutabakatı (kapıda ödeme takibi)
- [ ] ETA hesaplama ve müşteriye gösterimi (rota ETA'ları İterasyon 2'de iç kullanım için üretilir)
- [ ] Adil atama ağırlıklarının hakedişle birlikte ayarı (temel adalet terimi ve otomatik / akıllı atama İterasyon 2'de gelir)

**Portföy çıktısı:** Kurye hakediş ekranı ve panel rapor ekran görüntüleri; mutabakat akışı GIF'i.

---

## İterasyon 4+ — Genişleme ⏳

**Hedef:** Ürünü ölçeklenebilir ve satılabilir hâle getirmek.

- [ ] İşletme paneli (restoranlar kendi siparişlerini girer)
- [ ] POS entegrasyonu, ilk hedef Adisyo (herkese açık API)
- [ ] Platform entegrasyonları (restoranın kendi kimlik bilgileriyle ya da aracı üzerinden)
- [ ] SoftPOS ile kapıda kartlı ödeme
- [ ] Faturalama (paket başı)

**Portföy çıktısı:** İşletme paneli ekran görüntüleri, entegrasyon akış diyagramı.
