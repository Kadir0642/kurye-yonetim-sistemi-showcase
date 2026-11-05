# Benchmark: "en yakın kurye" ile rota uyumlu atama

> Bu rapor `npm run benchmark` ile üretilir (tek komut, tekrar üretilebilir). Sayılar ölçülmeden belgeye girmez; hipotezle çelişen sonuçlar olduğu gibi yazılır.

![Benchmark: A, D, B, C stratejileri](benchmark.png)

## Özet

Sentetik bir Eskişehir günü (20 kurye, ~400 sipariş, 10 tohum). Her ölçüt 10 tohumun ortalaması ± standart sapmadır; A'ya göre farklar ortalamalar üzerindendir.

| Ölçüt | A | D | B | C |
|---|---:|---:|---:|---:|
| Toplam km | 1886.15 ± 112.55 | 1644.68 ± 114.08 | 1317.71 ± 72.94 | 1317.79 ± 72.06 |
| Ort. teslimat süresi (sipariş → teslim, dk) | 38.61 ± 1.10 | 35.62 ± 1.10 | 27.64 ± 0.71 | 27.52 ± 0.52 |
| Ort. alış → teslim (dk) | 23.91 ± 0.94 | 19.35 ± 0.77 | 12.42 ± 0.58 | 12.14 ± 0.40 |
| Geç teslim (alış → teslim > 35 dk, %) | 19.91 ± 2.09 | 12.53 ± 1.70 | 3.18 ± 0.98 | 2.66 ± 0.70 |
| Geri dönüş oranı (km / doğrudan km) | 1.80 ± 0.04 | 1.57 ± 0.05 | 1.26 ± 0.02 | 1.26 ± 0.02 |
| Ters dönüş (100 paket başına) | 102.81 ± 3.09 | 81.18 ± 4.02 | 63.71 ± 2.61 | 64.44 ± 3.09 |
| Yüklü oran (%) | 35.53 ± 2.08 | 31.74 ± 1.98 | 25.19 ± 1.55 | 24.51 ± 0.94 |
| Aynı işletme kümesi aynı kuryede (%) | 46.41 ± 7.12 | 45.52 ± 10.36 | 50.52 ± 9.89 | 59.69 ± 13.61 |

**A'ya göre yüzde fark** (negatif = daha iyi; yüklü oran ve aynı kuryeye düşme oranında yüksek = farklı anlam taşır, bkz. ölçüt tanımları):

| Ölçüt | D − A | B − A | C − A |
|---|---:|---:|---:|
| Toplam km | -12.8% | -30.1% | -30.1% |
| Ort. teslimat süresi (sipariş → teslim, dk) | -7.7% | -28.4% | -28.7% |
| Ort. alış → teslim (dk) | -19.1% | -48.0% | -49.2% |
| Geç teslim (alış → teslim > 35 dk, %) | -37.1% | -84.0% | -86.7% |
| Geri dönüş oranı (km / doğrudan km) | -12.8% | -30.1% | -30.1% |
| Ters dönüş (100 paket başına) | -21.0% | -38.0% | -37.3% |
| Yüklü oran (%) | -10.7% | -29.1% | -31.0% |
| Aynı işletme kümesi aynı kuryede (%) | -1.9% | +8.9% | +28.6% |

**Duyarlılık uyarısı:** yukarıdaki fark, stratejilerin hazır olma anını bilmediği varsayılan (pilotun bugünkü) durum içindir. İşletmenin hazır olma anı biliniyorsa (aşağıdaki duyarlılık koşusu) toplam km farkı A'ya göre B için -8.1%, C için -13.9%, geç teslim farkı B için -96.5%, C için -98.4% olur: kilometre kazancı belirgin biçimde küçülür, geç teslim kazancı büyür. Başlık sayıları bu yüzden tek başına değil, iki tabloyla birlikte okunmalıdır.

## Hipotezler ve sonuç

Hipotezler (ROTA-UYUMLU-ATAMA §16) hedeftir, sonuç değildir; aşağıda ölçülen değerlerle karşılaştırılır.

- **H1 (B, A'ya göre toplam km'de ve geç teslimde belirgin azalma sağlar):** doğrulandı (toplam km -30.1%, geç teslim -84.0%).
- **H2 (C, düşük yönetici yüküyle B'ye yakın kalır):** doğrulandı (C − B: toplam km +0.0%, geç teslim -0.5 puan). Yönetici yükü bu simülasyonda ölçülmedi.
- **H3 (D, yalnızca sıra optimizasyonunun yetmediğini gösterir):** doğrulandı (toplam km: D -12.8%, B -30.1%; atama kararı sıralamadan daha büyük pay taşıyor).
- **H4 (toplu pencere aynı işletme siparişlerini aynı kuryede toplar):** kısmen: fark var ama tohumlar arası standart sapma düzeyinde (aynı kuryeye düşme oranı: A: %46.4, D: %45.5, B: %50.5, C: %59.7; C − A = +13.3 puan).

## Strateji tanımları

| Kod | Strateji | Atama | Sıra |
|---|---|---|---|
| A | En yakın kurye (pilotun bugünkü yazılımı) | Kapasitesi uygun, vardiyadaki en yakın kurye (kuş uçuşu, anlık konum) | Eklenme sırası (FIFO: önce alışlar, sonra teslimler alış sırasıyla) |
| D | A + sıra optimizasyonu | A ile aynı | Motorun `planRoute` + EXACT sıralayıcısı (kararlılık kuralları açık) |
| B | Rota uyumlu, ONERI üst sınırı | Her zaman `suggest()` 1. aday | `planRoute` + EXACT |
| C | Rota uyumlu, OTOMATIK + toplu pencere | 20 sn pencerede gelenler `planBatch` ile; sapma ≤ 5 dk ise atanır, değilse kuyruk; 60 sn'de yeniden deneme; 3 dk sonra "yönetici" 1. adayı atar | `planRoute` + EXACT |

Uygun kurye yoksa sipariş bekler ve kurye olaylarında yeniden denenir (tüm stratejilerde aynı). Motor stratejileri (B, C, D) 15 dk bekleyen siparişi "yönetici elle atar" varsayımıyla kapasitesi olan en yakın kuryeye en az kötü yerleştirmeyle verir; gün sonunda (23:00 sonrası) bekleyen sipariş kalırsa tüm kuryeler mesai dışı da sipariş alabilir. Kimse sipariş reddetmez.

## Senaryo

- **Düğüm evreni** (tüm tohumlarda aynı, tamamen kurgusal): 40 işletme 4 kümede (Merkez / Adalar, Odunpazarı tarihi bölge, Kampüs / Tepebaşı, Vişnelik / Osmangazi; merkezler İterasyon 1 SPEC §14 semtleri), 600 teslim noktası (10 mahalle merkezi çevresinde, σ = 600 m Gauss, hizmet bölgesine kırpılmış, kümeye göre ağırlıklı), 20 kurye başlangıç noktası. Toplam 660 düğüm; gerçek OSRM (motosiklet profili) `table` ile süre ve mesafe matrisi.
- **Talep:** 11:00-23:00, ortalama ~400 sipariş, homojen olmayan Poisson; öğle (12:00-14:00) ve akşam (18:30-21:30) tepeleri. Tepe saatlerde siparişlerin ~%15'i aynı işletmeden 2-3'lü kümeler (0-60 sn arayla).
- **Hazırlık süresi:** 5-20 dk kesik lognormal. Varsayılan senaryoda stratejiler hazır olma anını **bilmez** (pilotun bugünkü durumu): kurye işletmede gerçek hazır olma anına kadar bekler. Duyarlılık koşusu aşağıdadır.
- **Kuryeler:** 20 (12 tam gün 11:00-23:00, 8 yoğun saat 11:30-14:30 ve 18:00-22:30); kapasite 4; servis 120 sn (alış; aynı noktada ardışık ikinci alış 60 sn) ve 90 sn (teslim).
- **Hareket gürültüsü:** gerçekleşen bacak süresi = matris süresi × lognormal(0, 0,15); gürültü (tohum, kurye, başlangıç, bitiş) üzerinden türetilir, böylece tüm stratejiler aynı rastgele sayıları görür.
- **Konum:** en yakın kurye seçimi bacak ortasındaki kuryenin enterpolasyonlu konumunu kullanır. Rota uyumlu stratejilerde sıradaki durak kilitlidir ve hesap o durağın tahmini bitişinden başlar (kalan süre kilitli durağın servis süresine yazılır).

Tohumlar: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10.  
Matris özeti (sha256): `748bf740090acd3ee2cb481acf1c79db0aefc6861c7f2899e08bf6952776bfcd`.

| Tohum | Sipariş | Aynı işletme kümesi | Tepe saat payı |
|---:|---:|---:|---:|
| 1 | 417 | 20 | %75.1 |
| 2 | 365 | 16 | %75.6 |
| 3 | 395 | 22 | %71.1 |
| 4 | 420 | 22 | %75.0 |
| 5 | 383 | 14 | %71.8 |
| 6 | 410 | 27 | %76.6 |
| 7 | 371 | 18 | %74.1 |
| 8 | 397 | 21 | %77.1 |
| 9 | 415 | 16 | %72.3 |
| 10 | 391 | 25 | %75.2 |

## Ölçüt tanımları

| Ölçüt | Tanım |
|---|---|
| Toplam km | Tüm kuryelerin kat ettiği yol mesafesi (matris mesafeleri) |
| Ortalama teslimat süresi | Sipariş → teslim ve ayrıca alış → teslim |
| Geç teslim % | Alış → teslim > 35 dk olan paketler |
| Geri dönüş oranı | Kat edilen km / Σ doğrudan alış → teslim km |
| Ters dönüş | Ardışık bacaklar arası yön değişimi > 120° olan bacak sayısı (100 paket başına) |
| Yüklü oran | Kurye vardiya süresinin paket taşıyarak geçen payı |
| Aynı kuryeye düşme | Aynı işletmeden art arda gelen (2-3'lü) siparişlerin hepsinin aynı kuryeye düşme oranı |

## Duyarlılık: işletmenin hazır olma anı biliniyor

Aynı tohumlar; stratejiler (motor) `readyAt` bilgisini kullanır (bekleme maliyeti ve sıralama). A bilgiyi kullanmaz (en yakın kurye seçimi ve FIFO sıra hazır olma anından etkilenmez); D, B ve C kullanır.

| Ölçüt | A | D | B | C |
|---|---:|---:|---:|---:|
| Toplam km | 1886.15 ± 112.55 | 1720.36 ± 103.52 | 1734.05 ± 87.16 | 1624.49 ± 85.28 |
| Ort. teslimat süresi (sipariş → teslim, dk) | 38.61 ± 1.10 | 31.57 ± 0.76 | 26.31 ± 0.55 | 25.73 ± 0.34 |
| Ort. alış → teslim (dk) | 23.91 ± 0.94 | 15.40 ± 0.59 | 11.16 ± 0.46 | 10.36 ± 0.23 |
| Geç teslim (alış → teslim > 35 dk, %) | 19.91 ± 2.09 | 3.02 ± 1.12 | 0.70 ± 0.42 | 0.33 ± 0.27 |
| Geri dönüş oranı (km / doğrudan km) | 1.80 ± 0.04 | 1.64 ± 0.04 | 1.65 ± 0.05 | 1.55 ± 0.03 |
| Ters dönüş (100 paket başına) | 102.81 ± 3.09 | 88.14 ± 2.86 | 75.55 ± 3.48 | 74.10 ± 3.49 |
| Yüklü oran (%) | 35.53 ± 2.08 | 27.47 ± 1.61 | 22.80 ± 1.24 | 21.42 ± 1.21 |
| Aynı işletme kümesi aynı kuryede (%) | 46.41 ± 7.12 | 48.39 ± 14.92 | 32.90 ± 9.30 | 50.60 ± 10.36 |

| Ölçüt | D − A | B − A | C − A |
|---|---:|---:|---:|
| Toplam km | -8.8% | -8.1% | -13.9% |
| Ort. teslimat süresi (sipariş → teslim, dk) | -18.2% | -31.8% | -33.4% |
| Ort. alış → teslim (dk) | -35.6% | -53.3% | -56.7% |
| Geç teslim (alış → teslim > 35 dk, %) | -84.8% | -96.5% | -98.4% |
| Geri dönüş oranı (km / doğrudan km) | -8.8% | -8.0% | -13.8% |
| Ters dönüş (100 paket başına) | -14.3% | -26.5% | -27.9% |
| Yüklü oran (%) | -22.7% | -35.8% | -39.7% |
| Aynı işletme kümesi aynı kuryede (%) | +4.3% | -29.1% | +9.0% |

## Motor öneri süresi (bu makinede)

Matrix sağlayıcıyla saf CPU: `suggest()` çağrısı p50 0.40 ms, p95 1.36 ms (n = 5524); `planBatch` p50 0.63 ms, p95 2.08 ms (n = 8835). Süreler makineye bağlıdır ve CSV'ye girmez; gerçek OSRM ile uçtan uca ölçüm için `npm run bench:suggest -w @kurye/api`.

## Sınırlamalar

- Taban çizgisi A, pilotun mevcut yazılımının basitleştirilmiş modelidir (kuş uçuşu en yakın kurye + eklenme sırası); gerçek yazılım farklı davranabilir. Farklar bu taban çizgisine göredir.
- Canlı trafik yok; OSRM sabit hızlarla süre üretir (bacak başına ±%15 gürültü yalnızca değişkenliği taklit eder).
- Talep sentetiktir; gerçek siparişlerin zaman ve mekân dağılımı farklı olabilir. Sonuçlar bu ölçekte (20 kurye, ~400 sipariş, küçük bir şehir) geçerlidir.
- Sürücü davranışı yok (rota dışına çıkma, kural dışı sürüş, mola, kaza); kurye kararları kusursuz uygulanır, kimse sipariş reddetmez.
- Yönetici kararları idealize: ONERI üst sınırında yönetici her zaman 1. adayı seçer ve gecikmesizdir; yönetici yükü ölçülmez.
- Servis süreleri sabittir; teslimde müşteri bekleme / ulaşılamama yok.
- Kurye yeniden konumlanması yok: boşta kurye son teslim noktasında bekler.
- Motor sapma tanımı: `detourS`, yeni paketin kendi sabit servis süresini içermez (bkz. PR açıklaması / `EngineDeps.detourExcludesOwnService`); eşikler (5 dk) bu tanıma göredir.
- Toplu pencere 20 sn iken aynı işletme siparişleri arasındaki 0-60 sn aralık nedeniyle bir kısım küme farklı pencerelere düşer; tekil yerleştirme marjinal süreyi en aza indirir, "aynı kurye" tercihini ayrıca ödüllendirmez.

## Tekrar üretme

```powershell
npm install
npm run benchmark                  # docs/benchmark/ altındaki dosyaları yeniden üretir (önbellekli matris)
npm run benchmark -- --refresh-matrix   # matrisi çalışan OSRM'den yeniden hesaplar (npm run routing:up)
npm run benchmark -- --quick --travel haversine   # OSRM'siz duman testi (yayımlanmaz)
```

İki ardışık koşu bayt düzeyinde aynı `results.csv` dosyasını üretir (zaman damgası yok, tohumlu PRNG, duvar saati kullanılmaz).

## Veri atfı

Süre ve mesafe matrisi © OpenStreetMap katkıcıları verisinden, [ODbL 1.0](https://www.openstreetmap.org/copyright) lisansıyla üretilmiştir (Geofabrik Türkiye çıkarımı, Eskişehir kesimi). Önbellek dosyası `services/api/bench/cache/eskisehir-matrix.v1.json.gz` aynı atıfla depodadır. Düğümler (işletme, teslim noktası, kurye başlangıcı) tamamen kurgusaldır.
