# Yol Haritası

Kapsam, pilot müşteri keşfi sonrasında değişebilir. Her iterasyon bir demo ile kapanır; ayrıca "Portföy çıktısı" ile README için görsel malzeme toplanır. Çalışma düzeni için [GELISTIRME-SURECI.md](GELISTIRME-SURECI.md).

| İterasyon | Odak | Durum |
|---|---|---|
| 0 | Planlama ve araştırma | 🚧 Devam ediyor |
| 1 | Çekirdek | ⏳ |
| 2 | Duraklar ve rota optimizasyonu | ⏳ |
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
- [ ] ADR'lerin onaylanması / revize edilmesi

**Portföy çıktısı:** Mimari diyagramı ve karar kayıtları (README'de mevcut).

---

## İterasyon 1 — Çekirdek ⏳

**Hedef:** Bir işletme paket girer, kurye havuzdan alır, navigasyonla teslim eder, yönetici canlı haritada izler.

- [ ] Kimlik doğrulama ve tenant altyapısı (RLS dahil)
- [ ] Kiracı hizmet bölgesi yapılandırması (`service_area`: poligon / bbox, şehir, varsayılan harita merkezi; Places kısıtı) ([ADR-0006](kararlar/0006-hizmet-bolgesi-eskisehir.md))
- [ ] İşletme yönetimi (ekleme, düzenleme, çıkarma)
- [ ] Paket ekleme, Google Places adres otomatik tamamlama (session token)
- [ ] Sipariş havuzu
- [ ] Kurye uygulaması: vardiya aç / kapat, konum servisi
- [ ] Kurye uygulaması: havuzdan paket alma
- [ ] Kurye uygulaması: harita ve tek dokunuş navigasyon (`google.navigation:`)
- [ ] Kurye uygulaması: teslim etme
- [ ] Panel: canlı harita (WebSocket)
- [ ] Demo seed verisi (kurgusal firma, işletme, kurye, paket)

**Portföy çıktısı:** Paket oluşturma, havuzdan alma ve navigasyon başlatmayı gösteren GIF; panelde canlı harita ekran görüntüsü.

---

## İterasyon 2 — Duraklar ve rota optimizasyonu ⏳

**Hedef:** Birden fazla paketi akıllıca gruplayıp verimli rota üretmek; operasyonu yönetilebilir kılmak.

- [ ] Paketleri duraklara gruplama
- [ ] Rota optimizasyonu: OSRM + VROOM kurulumu ve entegrasyonu
- [ ] Eskişehir OSM çıkarımı (osmium) ile OSRM + VROOM; ~30 örnek adreste Google geocode ve OSRM snapping karşılaştırması (adres kalitesi testi) ([ADR-0006](kararlar/0006-hizmet-bolgesi-eskisehir.md))
- [ ] Kurye başına yarıçap filtresi (PostGIS `ST_DWithin`)
- [ ] Elle atama ve yeniden atama
- [ ] Konum kesilme uyarısı (heartbeat)
- [ ] Uygulama içi self-update (PackageInstaller)
- [ ] OEM pil optimizasyonu yönlendirme ekranları

**Portföy çıktısı:** Optimize edilmiş rota GIF'i, yarıçap filtresi ve konum kesilme uyarısı ekran görüntüleri.

---

## İterasyon 3 — Hakediş, kasa ve atama ⏳

**Hedef:** Kurye güvenini artıran şeffaf hakediş ve finansal takip.

- [ ] Paket bazlı hakediş hesabı ve kuryenin hakediş ekranı
- [ ] Yönetici raporları (kurye, işletme, dönem bazlı)
- [ ] Kasa / nakit mutabakatı (kapıda ödeme takibi)
- [ ] ETA hesaplama ve gösterimi
- [ ] Adil atama ve otomatik / akıllı atama

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
