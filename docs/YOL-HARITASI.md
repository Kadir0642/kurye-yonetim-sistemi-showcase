# Yol Haritası

Kapsam, pilot müşteri keşfi sonrasında değişebilir. Her iterasyon bir demo ile kapanır; ayrıca "Portföy çıktısı" ile README için görsel malzeme toplanır. Çalışma düzeni için [GELISTIRME-SURECI.md](GELISTIRME-SURECI.md).

| İterasyon | Odak | Durum |
|---|---|---|
| 0 | Planlama ve araştırma | 🚧 Devam ediyor |
| 1 | Çekirdek | 🚧 Spesifikasyon hazır, uygulama başlıyor |
| 2 | **Ana hedef: rota uyumlu atama** (duraklar, rota optimizasyonu, öneri / otomatik atama, ret ve devir); push, iOS, OTA, RLS | ⏳ |
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

## İterasyon 1 — Çekirdek 🚧

**Hedef:** Bir yönetici paket girer, kurye havuzdan alır, navigasyonla teslim eder, yönetici canlı haritada izler. Demo **ücretli anahtar olmadan** çalışır.

Teknik spesifikasyon: iterasyon-1/SPEC.md. API sözleşmesi `packages/contracts` altındadır. Üç paralel görev: A (API + altyapı), B (yönetim paneli), C (kurye mobil uygulaması).

- [ ] Monorepo (npm workspaces), ortak sözleşme (OpenAPI 3.1 + WebSocket protokolü), docker compose altyapısı ([ADR-0007](kararlar/0007-backend-ve-web-yigini.md))
- [ ] Kimlik doğrulama (JWT + rotasyonlu refresh) ve tenant altyapısı (uygulama katmanı + bileşik FK; RLS İterasyon 2'de)
- [ ] Kiracı hizmet bölgesi yapılandırması (`service_area`: poligon / bbox, şehir, varsayılan harita merkezi) ([ADR-0006](kararlar/0006-hizmet-bolgesi-eskisehir.md))
- [ ] İşletme yönetimi (ekleme, düzenleme, arşivleme)
- [ ] Kurye yönetimi (panelden hesap açma, PIN sıfırlama, vardiyayı kapatma)
- [ ] Paket ekleme; adres otomatik tamamlama API üzerinden (Photon, anahtar varsa Google Places) ([ADR-0008](kararlar/0008-acik-kaynak-harita.md))
- [ ] Paket durum makinesi ve olay geçmişi (audit)
- [ ] Sipariş havuzu, kurye yarıçap filtresi (PostGIS `ST_DWithin`), yarışa dayanıklı alma
- [ ] Elle atama, yeniden atama, havuza geri alma
- [ ] Kurye uygulaması (React Native + Expo, Android'de doğrulanır; iOS yapılandırması hazır) ([ADR-0002](kararlar/0002-kurye-uygulamasi-react-native-expo.md)): giriş, KVKK onayları, izinler ve pil optimizasyonu ekranı
- [ ] Kurye uygulaması: vardiya aç / kapat, arka plan konum servisi (kalıcı kuyruk, toplu gönderim)
- [ ] Kurye uygulaması: havuzdan paket alma, harita (MapLibre) ve tek dokunuş navigasyon, teslim etme
- [ ] Panel: canlı harita (MapLibre + WebSocket)
- [ ] Konum kesilme uyarısı (heartbeat)
- [ ] Demo seed verisi (kurgusal Eskişehir firması, işletmeler, kuryeler, paketler) ve kurye konum simülatörü

**Portföy çıktısı:** Paket oluşturma, havuzdan alma ve navigasyon başlatmayı gösteren GIF; panelde canlı harita ekran görüntüsü.

---

## İterasyon 2 — ana hedef: Rota uyumlu atama ⏳

**Hedef:** Birden fazla paket taşıyan kuryenin paketlerini haritada **optimal sırayla durak** olarak görmesi ve yeni paketin kuryenin **rotasına en az sapma ekleyen** kuryeye atanması. Pilot müşterinin sahadaki sorunu: mevcut yazılım yoldaki kuryeye yeni paketi yalnızca yakınlığa göre atıyor, paket çoğu zaman rota dışında veya ters yönde kalıyor, kuryeler yanlış atanmış paketi bırakıyor, yönetici haritaya bakıp elle yeniden dağıtıyor. Rota uyumlu atama ürünün ayırt edici özelliğidir.

Tasarım: iterasyon-2/ROTA-UYUMLU-ATAMA.md. Karar: [ADR-0009](kararlar/0009-rota-uyumlu-atama.md).

Önerilen sıra: (a) altyapı ve rota, (b) atama, (c) panel ve kurye uygulaması, (d) benchmark; platform maddeleri paralel yürür.

**Rota uyumlu atama — çekirdek**
- [ ] OSRM + VROOM konteynerleri (docker compose) ve Eskişehir OSM çıkarımı boru hattı (osmium, `osrm-extract/partition/customize`) ([ADR-0006](kararlar/0006-hizmet-bolgesi-eskisehir.md))
- [ ] Adres kalitesi testi: ~30 örnek adreste Google geocode ile OSRM snapping karşılaştırması (gerçek müşteri verisi olmadan)
- [ ] Durak modeli ve kurye başına canlı rota: `courier_routes`, `route_stops`; paketleri duraklara gruplama (`PICKUP` / `DROPOFF`)
- [ ] Rota optimizasyonu (VROOM, pickup-delivery): paket ekle/çıkar/tamamla ile yeniden sıralama; kilitli sıradaki durak ve kazanç eşiği (kararlılık kuralları)
- [ ] Aday süzme (PostGIS) + en ucuz yerleştirme + uygulanabilirlik + skor; gerekçeli öneri (ör. "Rotası üzerinde, +2 dk"); performans hedefi: 25 kurye için < 300 ms
- [ ] Üç atama modu: `ONERI` (varsayılan), `AKILLI_HAVUZ`, `OTOMATIK` (+ bekleyenler kuyruğu); kiracı ayarları (azami paket 4, sapma eşiği 5 dk, azami süre 35 dk, kazanç eşiği 2 dk, ön süzme 5 km)
- [ ] Kontrollü ret (alıştan önce, gerekçe enum) ve devir isteği (alıştan sonra, yönetici onayı); `package_rejections`, `handover_requests`, denetim kaydı. Devir isteği, alıştan sonraki iki devir yolundan biridir; diğeri aşağıdaki kuryeler arası devirdir
- [ ] Panel: rota çizgileri ve numaralı duraklar, öneri paneli, sürükle-bırak atama ve büyük sapma uyarısı ("+14 dk, 2 paket gecikir"), bekleyenler ve devir listeleri
- [ ] Kurye uygulaması "Rotam": numaralı duraklar (liste + harita), sıradaki durağa tek dokunuş navigasyon, rota değişince bildirim ("Yeni paket rotana eklendi: +3 dk"), ret ve devir isteği
- [ ] Metrikler: ortalama sapma, elle yeniden atama sayısı, ret oranı, kötü atama oranı, zamanında teslim oranı
- [ ] Simülasyon benchmark'ı: sentetik Eskişehir günü (~20 kurye, ~400 paket), "en yakın kurye" ile rota uyumlu atama karşılaştırması (toplam km, ortalama teslimat süresi, geç teslim %, geri dönüş)

**Kuryeler arası devir** (pilot müşteri isteği; karar: [ADR-0010](kararlar/0010-kuryeler-arasi-devir.md), tasarım: iterasyon-2/KURYE-DEVIR.md)
- [ ] Sözleşme (contract-first): OpenAPI uçları (`handover-candidates`, `handover-offers`, `accept | decline | confirm`, yönetici uçları) ve WebSocket olayları (`handover.offered`, `handover.updated`)
- [ ] Veri modeli: `handover_offers` tablosu (kısmi benzersiz indeks: pakette tek `PENDING`), kiracı ayarları (`handover_radius_m` 1500, `handover_offer_timeout_s` 60, `handover_requires_dispatcher_approval` false, `handover_meet_radius_m` 150, `max_handovers_per_package` 2, `max_handovers_per_courier_per_day` 10), yeni `package_events` tipleri
- [ ] Aday listesi: PostGIS `ST_DWithin` ile son konumlar, vardiya / STALE / kapasite süzgeci, mesafeye göre sıralama; rota skoru varsa rotaya sapma etiketi
- [ ] Alıştan önce (`ASSIGNED`): teklif, kabul / ret / zaman aşımı, koşullu UPDATE ile atomik yeniden atama
- [ ] Alıştan sonra (`PICKED_UP`): QR / 4 haneli kod, buluşma yarıçapı doğrulaması (kod + konum), taze kalma süresinin orijinal alıştan sürmesi, `CASH` pakette tahsilat sorumluluğu devri ve kasa için olay kaydı
- [ ] Yönetici: canlı WS olayı ve denetim kaydı, isteğe bağlı onay, iptal ve elle onay; yönetici onaylı devir isteğiyle (ADR-0009) aynı buluşma adımlarının paylaşılması
- [ ] Kötüye kullanım sınırları (paket başı / günlük), zaman aşımı işi ve devir metrikleri ("en çok devreden / devralan")
- [ ] Uç durum testleri: devreden STALE olur, devralan vardiyayı kapatır, paket iptal edilir, iki eşzamanlı teklif, kabul ile zaman aşımı yarışı
- [ ] Kurye uygulaması: "Devret" aday listesi, teklif kartı, kod / QR ekranı ve QR tarama; panel: devirler sekmesi, onay kutusu ve devir ayarları
- [ ] KVKK: aydınlatma metni güncellemesi (meslektaşa yakınlık / buluşma konumu), açık sorular 24-27'nin pilotla kapatılması

**Platform ve altyapı (İterasyon 2'ye bırakılanlar)**
- [ ] Push bildirim (FCM / APNs): yeni paket, atama ve rota değişikliği bildirimi
- [ ] PostgreSQL RLS ile ikinci savunma hattı; yeni rota tablolarına da uygulanır ([ADR-0004](kararlar/0004-cok-kiracili-mimari.md))
- [ ] iOS derleme ve TestFlight dağıtımı (EAS Build, Apple Developer hesabı) ([ADR-0003](kararlar/0003-dagitim-apk-self-update.md))
- [ ] OTA güncelleme kanalı (EAS Update veya kendi güncelleme sunucumuz) ve uygulama içi APK güncellemesi (PackageInstaller)
- [ ] OEM'e özel pil optimizasyonu yönlendirme ekranları (Xiaomi, Huawei, Samsung ayrıntılı)
- [ ] Panel oturumunda httpOnly çerez; demo HTTP yerine HTTPS

**Pilotla doğrulanacak varsayımlar:** kurye başına azami paket, kabul edilebilir sapma, azami teslim süresi, restoran hazırlık süresi bilgisi, otomatik atama tercihi, ret politikası, çoklu restoran alışı ([MIMARI açık sorular 17-23](MIMARI.md#8-açık-sorular)).

**Portföy çıktısı:**
- Benchmark sonuç grafiği ("en yakın kurye" ile rota uyumlu atama), README'de yayınlanır.
- Rota haritası GIF'i: numaralı duraklar, yeni paketin sapma dakikasıyla önerilmesi ve atanması.
- Kuryeler arası devir GIF'i: aday listesi, teklif kartı, kod / QR ile devir tamamlanması ve panelde canlı olay.
- Optimize edilmiş rota ve "Rotam" ekran görüntüleri, iOS uygulama ekran görüntüleri, push bildirim akışı.

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
