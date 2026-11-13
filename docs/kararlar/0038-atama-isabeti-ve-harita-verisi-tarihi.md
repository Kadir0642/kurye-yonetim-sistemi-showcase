# ADR-0038: Atama isabeti göstergesi, harita verisi tarihi ve pilot kapasitesi 5

- **Durum:** Kabul edildi (eşikler ve varsayılan dönem ürün sahibi onayına açık, aşağıdaki tabloda)
- **Tarih:** 2026-11-13
- **İlgili:** [ADR-0009](0009-rota-uyumlu-atama.md) (atama ve ret), [ADR-0021](0021-haftalik-osm-guncelleme-hatti.md) (haftalık veri hattı), [ADR-0023](0023-dagitim-algoritmasi-v2.md) (v2), [ADR-0028](0028-havuz-gorunurlugu-ve-atama-kabulu.md) (kabul / süre aşımı), [ADR-0036](0036-gecikme-uyarisi.md) (teklif / ret tanımları)

## Bağlam

Pilot görüşmesi öncesi üç küçük, birbirinden bağımsız ihtiyaç: (A) yöneticiye "sistem atamaları ne kadar doğru çıkıyor?" sorusunu tek ve dürüst bir sayıyla yanıtlamak, (B) panelde kullanılan harita verisinin kaç günlük olduğunu göstermek ve haftalık güncellemeyi zamanlamaya hazır bırakmak, (C) pilot ön ayarında kurye başına aktif paket sınırını 5 yapmak.

## Karar

### A. Atama isabeti

**Tanım (tek cümle).** Dönemde *sistemin yaptığı* atamalardan, paketi atanan kuryenin **atama geri alınmadan teslim aldığı** atamaların oranı; sonuçlanmamış (sürüyor) atamalar paydaya girmez.

**Birim paket değil atamadır:** ölçü her atamayı ayrı sayar; bir paket reddedilip yeniden atanırsa bir isabetsiz (ret) ve ikinci atama alınırsa bir isabetli atama olur. Yüzde "paketlerin yüzde kaçı ilk atamada doğru gitti" değil, "sistemin verdiği kararların yüzde kaçı tuttu" demektir.

- **Nüfus:** `package_events` içinde `ASSIGNED` / `REASSIGNED` / `ASSIGNED_AUTO` olayı ve `data.assignment.source` ∈ {`AUTO` (anlık otomatik atama ve v2 pencere), `BATCH` (toplu plan)}. **Not:** toplu plan atamaları sistemin önerdiği, ancak yönetici onayıyla uygulanan atamalardır ("sistem önerir, yönetici onaylar"); sistemin planının kalitesini ölçtüğü için nüfustadır. AUTO ve BATCH şimdilik ayrı gösterilmez (tek oran); ayrım gerekirse kaynağa göre kırılım eklenir. Yönetici elle / öneri / sürükle-bırak atamaları (`MANUAL`, `SUGGESTION`, `DRAG_DROP`, `CREATE_ASSIGN`), kuryenin havuzdan alışı (`CLAIM`, `SMART_POOL_CLAIM`) ve devir (`HANDOVER`) nüfusa girmez: ya insan kararıdır ya da sistemin atamasının sonucudur. (Öneriden atama kasıtlı dışarıda: karar yöneticinindir; ADR-0036 ekindeki "teklif" tanımıyla farkı budur, orada yönetici ataması da teklif sayılır.)
- **Sonuç = o atamadan sonra, aynı pakette, olay sırasına (`id`) göre gerçekleşen ilk karar verici olay.** Diğer olaylar (düzenleme, kabul, rota yeniden sıralama, devir istekleri) sonucu değiştirmez.

| Sonuç | Karar verici olay |
|---|---|
| **İsabet** | `PICKED_UP` (paket atanan kuryeden alındı; sonradan devir / başarısızlık sayılmaz) |
| Iska: yönetici geri aldı / değiştirdi (`MANAGER_CHANGED`) | `UNASSIGNED`, ya da aynı pakete yeni atama (`REASSIGNED`, `ASSIGNED`, `ASSIGNED_AUTO`) |
| Iska: kurye reddetti (`COURIER_REJECTED`) | `REJECTED`; eski uçla bırakma `RELEASED` de burada (kurye atamayı geri verdi) |
| Iska: kabul süresi doldu (`TIMED_OUT`) | `ASSIGNMENT_EXPIRED` |
| Iska: kurye devretti (`COURIER_HANDED_OVER`) | `HANDOVER_COMPLETED` (alıştan önce) |
| Sürüyor (`open`) | karar verici olay yok; paydadan çıkar, ayrıca gösterilir |
| Dışarıda (`excluded`) | `CANCELLED` (alınmadan iptal); paketin kendisi ortadan kalktığı için atamanın doğruluğuyla ilgisi yoktur, ne isabet ne ıska sayılır |

**Oran** = isabet / (isabet + ıska). **Asgari örneklem:** çözülmüş (isabet + ıska) atama sayısı 10'un altındaysa yüzde verilmez (`hitRate = null`, `sufficient = false`), panel "Yeterli veri yok" yazar; sayılar yine görünür.

**Dönem** atamanın yapıldığı zamana göredir (Europe/Istanbul gün sınırları). Sonuç sonradan belli olabileceğinden bugünün oranı gün içinde değişir ve "sürüyor" bugün daha kalabalıktır; geçmiş günlerin sayıları da sonradan kayabilir (örn. akşam alınan bir paket sabah atamasını isabete çevirir). Bu bilinçli: alternatif (sonucun zamanına göre saymak) atamanın kalitesini yanlış güne yazardı.

**Uç:** `GET /metrics/assignment-hit-rate` (OWNER, DISPATCHER; kiracıya bağlı; `period` = `today` | `7d` | `30d` ya da `from` / `to`, en çok 31 gün; hiçbiri yoksa son 7 gün). Yanıt: toplamlar, oran, ıska dağılımı, gün serisi. **Kurye kırılımı yoktur** (yalnızca toplamlar; kişi sıralaması istenirse ayrı karar).

**Sorgu.** Her sistem ataması için `LEFT JOIN LATERAL` ile paket başına ilk karar verici olay alınır (`package_events_tenant_package_idx`: tenant, paket, `id`); dış tarama var olan kısmi dizini (`package_events_courier_offers_idx`, tenant + `created_at`) kullanır, **yeni migrasyon gerekmedi**. **Bilinen sınır:** dış tarama kiracının tüm atama geçmişini dizinle tarayıp tarihe göre süzer (mevcut dizin `(tenant_id, courier_id, created_at)` olduğundan tarih aralığı doğrudan daraltmaz). Pilot hacminde ölçülen süre yeterlidir; veri büyüyünce `package_events (tenant_id, created_at)` üzerinde atama olayları için kısmi dizin eklenecektir (takip işi, şimdilik migrasyon yok). EXPLAIN ANALYZE (900 bin olaylı, 20 kiracılı deneme verisi, son 7 gün, kiracıda ~2 bin sistem ataması): 22 ms, tüm erişimler dizinli. Sınıflama saf bir fonksiyondur (`metrics/assignment-hit-rate.ts`) ve SQL ile aynı tür listesini kullanır.

**Panel:** Metrikler sayfasının Atama bölümünün başında kompakt "Atama isabeti" kartı: yüzde, dönem anahtarı (Bugün / 7 gün / 30 gün; varsayılan 7 gün), Türkçe etiketli ıska dağılımı, günlük sütun eğilimi, tanımı tek cümleyle veren bilgi ipucu, boş ve yetersiz-örneklem durumları. Mevcut bileşen ve marka renkleri.

### B. Harita verisi tarihi

- `GET /map/data-status` (OWNER, DISPATCHER): yol (rota motoru), vektör karo ve adres arama verisinin kaynak OSM tarihi, yaşı (gün) ve `stale` bayrağı (varsayılan eşik 30 gün, `MAP_DATA_STALE_DAYS`). İstek anında PBF ayrıştırılmaz; tarih, derleme sırasında yazılan künyelerden okunur ve 60 sn önbelleklenir.
- **Kaynaklar (sırayla):** haftalık hattın etkin sürümü (`releases/<sürüm>/release.json`; hat artık kaynak PBF başlığındaki `osmDataTimestamp` değerini yazar), geliştirme kurulumunda `infra/routing/data/data-info.json` (rota derlemesi yazar; yoksa `.build-stamp` zamanı, `basis = BUILD_TIME`), son olarak aynı haftalık koşunun yayınladığı karo sürümünden çıkarım (`INFERRED`). Karo: yayınlanmış sürümün `osmDataTimestamp` değeri. Bilinmiyorsa `UNKNOWN` (uç yine 200).
- `osmium extract` PBF başlığındaki zaman damgasını atar; bu yüzden tarih önce Türkiye PBF'inin başlığından (yalnızca ilk blok okunur) alınır.
- **Panel:** Ayarlar > Hizmet bölgesi altında "Harita verisi: gg.aa.yyyy" kartı; 30 günden eskiyse nazik sarı uyarı.
- **Haftalık güncelleme zamanlanmadı.** `infra/pipeline/weekly-update.ps1` ve README'deki "Haftalık güncellemeyi zamanlama" bölümü, ürün sahibi isterse çalıştıracağı Görev Zamanlayıcı komutunu, beklenen indirme boyutunu (~0,65 GB) ve süreyi (~10-15 dk) verir; sunucuda aynı iş cron / systemd zamanlayıcısıdır. Betiğin `-Check` kipi hiçbir şey indirmeden önkoşulları doğrular.

### C. Pilot kapasitesi 5

Pilot ön ayarı (panelde "Pilot ayarlarını uygula", CALISMA-KITABI §12) `maxActivePackagesPerCourier = 5` yazar; demo tohum kiracısı 5 ile açılır. **Şema varsayılanı 4 kalır** ve mevcut kiracıların verisine migrasyonla dokunulmaz. Sınırı v1 öneri ve toplu plan motoru, v2 pencere ve havuzdan alış kiracı ayarından okur (testler: motor birim testi 4 / 5 aktif paket sınırı, e2e'de 5. paket girer 6. girmez). Yönetici elle atamada sınır uygulanmaz (mevcut, bilinçli davranış). Değiştirme yeri: Panel > Ayarlar > Atama.

## Sonuçlar

- (+) Tek, açıklanabilir bir sayı; ıska nedenleri doğrudan eylem gösterir (süre aşımı çoksa süre / bildirim, ret çoksa eşik / kapasite).
- (+) Şema ve migrasyon değişmedi; mevcut olay verisinden geriye dönük hesaplanır.
- (−) Sonuç gecikmeli belli olur; bugünün oranı kararsızdır, yeni kurulumda ilk günlerde "yeterli veri yok" görünür.
- (−) Yöneticinin iptal ettiği paket (müşteri vazgeçti) paydadan çıkar; yanlış atanmış paketin iptal edilmesi ıska olarak yansımaz (iptal nedeni olaydan ayrıştırılamaz).
- (−) Üretim API konteyneri hat dizinini görmüyorsa yol / adres tarihi karo sürümünden çıkarılır (`INFERRED`); hat dizinini salt okunur bağlamak tam doğruluğu verir.

## Ürün sahibi kararı bekleyen (tek satırda değiştirilebilir varsayılanlar)

| Karar | Uygulanan | Nerede değişir |
|---|---|---|
| Asgari örneklem | 10 çözülmüş atama | `MIN_SAMPLE` (`metrics/assignment-hit-rate.ts`) |
| Varsayılan dönem | Son 7 gün | `HitRateQuery` / panel `useState('7d')` |
| Öneriden (SUGGESTION) atama nüfusa girsin mi | Hayır (insan kararı) | `SYSTEM_ASSIGNMENT_SOURCES` |
| Alınmadan iptal ıska sayılsın mı | Hayır (dışarıda) | `outcomeOf('CANCELLED')` |
| Harita verisi eskime eşiği | 30 gün | `MAP_DATA_STALE_DAYS` |
| Haftalık güncelleme zamanlansın mı | Hayır (komut hazır, sahibi karar verir) | `infra/pipeline/README.md` |

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Mevcut `badAssignmentRate` ((ret + elle yeniden atama + devir isteği) / atama) | Atamanın sonucuna değil sayılara dayanır, sürmekte olanı ve kabul süresi aşımını ayırmaz, payda ile pay aynı kümeden değil |
| Sonucu olay zamanına göre döneme yazmak | Atamanın kalitesi yanlış güne düşer |
| Veritabanına sonuç sütunu / materialize tablo | Migrasyon + yazım yolunda değişiklik; mevcut dizinle 22 ms yeterli |
| Kurye bazlı isabet sıralaması | Küçük örneklemde haksız kişi değerlendirmesi riski; bu karar yalnızca toplamlar içindir |
| Harita tarihini istekte PBF'ten okumak | 600 MB'lık dosya, konteynerde yok, gereksiz G/Ç |
