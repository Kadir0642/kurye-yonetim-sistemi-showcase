# ADR-0017: Saha verisinden öğrenen harita (öğrenilmiş teslim noktaları ve öğrenilmiş hızlar)

- **Durum:** Kabul edildi (ürün sahibi onayı: 2026-11-03; KVKK hukuki onayı ve kurye aydınlatma metni güncellemesi pilottan önce zorunlu, bkz. Sonuçlar)
- **Tarih:** 2026-11-03
- **İlgili:** [ADR-0005](0005-konum-takibi-ve-kvkk.md), [ADR-0006](0006-hizmet-bolgesi-eskisehir.md) (d: adres defteri), [ADR-0009](0009-rota-uyumlu-atama.md), [ADR-0011](0011-osrm-motosiklet-profili-ve-rota-altyapisi.md) (kalibrasyon çarpanı), [ADR-0013](0013-rota-motoru-siniri-ve-yaklasik-mod.md), [ADR-0016](0016-eskisehire-ozel-kendi-barindirdigimiz-harita-platformu.md), İterasyon 3 planı

## Bağlam

Genel amaçlı haritaların iki zayıflığı bizim kullanım alanımızda sahadan düzeltilebilir:

1. **Kapı düzeyi adres.** OSM'de Türkiye için kapı numarası kapsamı zayıftır ([ADR-0008](0008-acik-kaynak-harita.md), [ADR-0016](0016-eskisehire-ozel-kendi-barindirdigimiz-harita-platformu.md)). Oysa kurye her teslimde "Teslim ettim"e basar ve bu olay zaten konumla gelir (`CourierActionRequest.position`, `package_events`). Eskişehir bir öğrenci şehridir (yurt, kampüs, site); aynı adres tekrar tekrar teslim alır ([ADR-0006](0006-hizmet-bolgesi-eskisehir.md)). Adres defteri ([ADR-0006](0006-hizmet-bolgesi-eskisehir.md) d) planlandı ama bugüne kadar uygulanmadı (şemada yok).
2. **Süreler.** OSRM canlı trafik bilmez; motosiklet profili ve tek bir `ROUTING_DURATION_FACTOR` çarpanı ([ADR-0011](0011-osrm-motosiklet-profili-ve-rota-altyapisi.md)) ortalamayı düzeltir, ama hangi sokağın gerçekte yavaş / hızlı olduğunu ve günün saatini bilmez. Kuryelerin GPS izi (vardiya sırasında zaten toplanıyor, `location_pings`) bunun gerçek ölçümüdür.

Kısıtlar: konum verisi kurye kişisel verisidir ([ADR-0005](0005-konum-takibi-ve-kvkk.md): yalnızca vardiyada, ham veri 30 gün saklanır); müşteri adresi / telefon kişisel veridir; depoda kişisel veri olmaz.

## Karar

### A. Öğrenilmiş teslim noktaları

**Ne:** Bir adres için kuryelerin **fiilen teslim ettiği yerin** (bina girişi, kapı önü, site girişi) öğrenilmiş konumu. Geocoder'ın verdiği nokta ("ev noktası") ile teslimin gerçekleştiği nokta farklı olabilir.

**Anahtar (kişi değil, adres):** `address_key` = normalize edilmiş adres metni (küçük harf, Türkçe karakter katlama, kısaltma açma, daire / kat ayrıştırılıp atılır) + geocoder noktasının ~25 m'lik ızgara hücresi. **Müşteri adı, telefonu ve daire numarası anahtara veya kayda girmez.** (ADR-0006 d "telefon -> place_id" taslağı bu ADR ile yerini alır: telefon kişisel veri olduğu için kullanılmaz.)

**Gözlem:** Başarılı `deliver` olayında (yalnızca `DELIVERED`, `FAILED` değil) kuryenin olay anındaki konumu. Süzme: doğruluk <= 30 m, `isMock` değil, geocoder noktasına uzaklık <= 300 m (daha uzaksa adres büyük olasılıkla yanlış, gözlem sayılmaz, ayrıca "adres şüpheli" işareti olur), olay zamanı cihaz zamanıdır (çevrimdışı kuyruk, PLAN 3a-6).

**Veri modeli (taslak, ayrıntı İterasyon 3b görev brief'inde):**

| Tablo | Alanlar | Not |
|---|---|---|
| `delivery_point_observations` | `tenant_id`, `address_key`, `position`, `accuracy_m`, `observed_at` | **Kurye kimliği yok.** 90 gün sonra silinir |
| `address_book_entries` | `tenant_id`, `address_key`, `geocoded_position`, `learned_position`, `sample_count`, `effective_n`, `spread_m`, `confidence`, `first_seen_at`, `last_observed_at`, `status` (`ACTIVE`, `OVERRIDDEN`, `STALE`) | Kalıcı özet |

**Hesap:** Ağırlıklı geometrik medyan (aykırılara dayanıklı). Gözlem ağırlığı = `0,5^(yaş_gün / 90)` (yarı ömür 90 gün: bina girişi değişebilir, eski gözlem azalır) x doğruluk ağırlığı. Yeterli sayıda gözlemle iki ayrı küme çıkarsa (ör. yurdun iki girişi; küme yarıçapı ~30 m, her biri >= 3 gözlem) birincil küme kullanılır, ikincil kaydı ileride "alternatif giriş" olarak gösterilir (İterasyon 3b'de yalnızca birincil).

**Güven:** `confidence = (1 - exp(-effective_n / 3)) x 1 / (1 + spread_m / 40)` (başlangıç sabitleri benchmark / adres testiyle ayarlanır, PLAN 3a-8). Eşik: `confidence >= 0,6` ve `effective_n >= 2` iken öğrenilmiş nokta kullanılır. Son gözlem 12 aydan eskiyse `STALE`, kullanılmaz.

**Kullanım (yalnızca eşik aşılınca):**
- **Rota:** `DROPOFF` durağının konumu olarak (OSRM matrisi ve VROOM girdileri) öğrenilmiş nokta kullanılır; böylece ETA ve sapma hesabı gerçek kapıya göre yapılır.
- **Navigasyon hedefi:** kuryenin navigasyon intent'ine öğrenilmiş nokta verilir; kuryeye "Öğrenilmiş teslim noktası (N teslim)" etiketi gösterilir ve geocoder noktası ikinci seçenek olarak durur.
- **Panel:** adres defteri ekranı: öğrenilmiş / geocoder noktası, güven, gözlem sayısı; yönetici noktayı elle sabitleyebilir (`OVERRIDDEN`, öğrenmeyi durdurur) veya sıfırlayabilir.
- **İşletme alış noktası:** aynı yöntemle işletmenin gerçek kapısı (`Aldım` konumları) öğrenilir; `businesses.location` düzeltme önerisi olarak sunulur, otomatik değiştirilmez.

**KVKK ve saklama:**
- Bunlar **işletme teslim noktalarıdır**, kurye izi değildir: gözlemde kurye kimliği ve saat dilimi tutulmaz, yalnızca teslimin gerçekleştiği konum ve tarih saklanır.
- Gözlemler 90 gün, adres özetleri son gözlemden 12 ay sonra silinir (`STALE` -> silme). Ham `location_pings` saklaması değişmez (30 gün, [ADR-0005](0005-konum-takibi-ve-kvkk.md)).
- Adres metni müşteri kişisel verisi sayılabilir: normalize anahtar tutulur, **tam adres metni adres defterine kopyalanmaz** (paketin kendi saklama süresine tabidir); anahtar tek yönlü özet (hash) olarak da saklanabilir (hukuk görüşüne göre, PLAN P-9).
- Kuryenin teslim ettiği konumun başkası (yönetici) tarafından izlenmesi için kullanılmaz; kurye bazlı rapor üretilmez.

**Sınırlar (dürüst):** Kurye "Teslim ettim"e kapıdan uzakta basarsa (yürürken, uzaktan) gözlem gürültülüdür; medyan ve doğruluk süzgeci bunu azaltır, teslim geofence'i (PLAN 3b-4) kötü gözlemleri baştan engeller. Kapı içi (kat, daire) çözülmez.

### B. Öğrenilmiş hızlar

**Ne:** Yol parçası (OSM `from_node -> to_node`) x haftanın saati başına, kuryelerin GPS izinden **gerçekleşen motosiklet hızı**; OSRM'ye hız dosyası olarak verilir.

**1. Toplama (ilk günden, uygulamadan bağımsız; pilotun ilk gününde başlar).** Günlük iş, o güne ait vardiya izlerini işler **ve yalnızca özetleri saklar**: ham ping saklaması (30 gün) uzatılmaz. Toplama, öğrenme etkin olmasa da (`LEARNED_SPEEDS_ENABLED=false`) çalışır; böylece öğrenmeyi açacağımız gün elimizde haftalarca veri vardır.
- **Girdi:** yalnızca açık vardiya sırasındaki `location_pings` (ham veri zaten yalnızca vardiyada vardır).
- **Süzme:** doğruluk <= 25 m, `isMock` değil; durağan bölümler (hız < 1 m/sn ve > 30 sn) ve alış / teslim noktalarının 40 m çevresi atılır (park, bekleme); vardiyanın ilk ve son 200 m'si atılır (kurye evi / başlangıç yeri sızmasın).
- **Harita eşleme:** OSRM `match` servisi (motosiklet profili, `gaps=split`, `tidy=true`, `annotations=nodes`, nokta başına `radiuses`; `--max-matching-size` yeterince yükseltilir, izler <= 100-200 noktalık parçalara bölünür). Eşleşme güveni düşük bölümler (`confidence` < 0,5) atılır. Bacaklar arası (ardışık iki GPS noktası) gözlenen süre ve eşlenen yolun mesafesi ile hız `mesafe / süre` bulunur ve bacağın içindeki yol parçalarına uzunluk ağırlıklı dağıtılır (seyrek örneklemede bu, parça başına hızı yumuşatır: bilinen sapma, [sınırlamalar](#sonuçlar)).
- **Aykırı değer:** parça hızı > 1,3 x o parçanın profil serbest hızı veya < 3 km/sa (ve durak çevresi dışı) atılır.
- **Birleştirilebilir özet:** `segment_speed_stats(from_node, to_node, how_bucket, n, sum_kmh, sum_sq_kmh, hist[12] (5 km/sa'lik kutular), courier_days, last_updated)`. Özet **toplanabilir** (artımlı birleşir), kurye kimliği tutulmaz, zaman damgası yalnızca haftanın saati kovası (168) olarak kalır. `courier_days` = parçaya katkı veren kurye-gün sayısı (kimlik değil, sayaç; k-anonimliğe yakın bir eşik için kullanılır).

**2. Uygulama (yalnızca yeterli veri birikince, 3c).**
- **Yayın eşiği:** parça için `n >= 20` ve `courier_days >= 5` (küçük bir şehirde 20-25 kurye ile tek kuryeyi ele vermemek için); altındakiler profil varsayılanında kalır.
- **Karıştırma (shrinkage):** `v = (n x v_gözlem + k x v_profil) / (n + k)`, `k = 20`; `v`, profil hızının [0,5; 1,5] katına kıstırılır. v_gözlem histogramdan kırpılmış ortalama (P10-P90).
- **Zaman boyutu:** OSRM hız dosyası (`from_osm_id,to_osm_id,speed_kmh`) durağandır. Hafta-saat boyutu iki katmanla çözülür: (i) **mekânsal** düzeltme tüm gün ortalamasından hız dosyasıyla OSRM'ye girer (`osrm-customize --segment-speed-file`, MLD'de yalnızca customize yeniden çalışır, ~2 sn), (ii) **zamansal** düzeltme `f(hafta_saati)` çarpanı olarak API'de uygulanır (bugünkü tek `ROUTING_DURATION_FACTOR`'ın saat bazlı hâli; 168 değerli tablo, hesaplama: gözlenen / öngörülen bacak süreleri). Ölçüm, bu ayrışımın yetmediğini gösterirse (MAE kazancı), gün dilimi başına ayrı OSRM veri kümesi (2-3 küme) değerlendirilir; bu, çok `osrm-routed` süreci ve motorda zaman duyarlı sağlayıcı demektir (karar o zaman).
- **Güvenceler:** (a) her yayın öncesi **tutulan veri (held-out) doğrulaması**: son haftanın bacak süreleri üzerinde eski ve yeni modelin MAE'si; yeni model daha kötüyse yayınlanmaz; (b) yayın sürümlüdür, son 2 sürüm saklanır, tek komutla geri dönülür; (c) haftalık OSM güncellemesi ([ADR-0016](0016-eskisehire-ozel-kendi-barindirdigimiz-harita-platformu.md)) hız dosyasını yeni OSM düğüm kimlikleriyle yeniden eşler (kimliği bulunamayan parçalar sessizce atılır); (d) `ROUTING_DURATION_FACTOR` öğrenilmiş modelin üzerinde artık kalibrasyon çarpanı olarak kalır.
- **Ölçüm:** ETA hatası (MAE, P90) öğrenmeden önce / sonra, ayrıca benchmark güncellemesi (PLAN 3c-1). Taban çizgisi adil olmalı: profil + **kalibre edilmiş** global çarpan (ADR-0011), çiğ profil değil.

**3. Gizlilik.**
- Yalnızca vardiya içi izler; yalnızca toplu ve anonim özetler kalıcıdır; özette kurye kimliği, ping zaman damgası, yol boyunca sıralı iz **yoktur** (parça başına ayrı sayaçlar, sıralı konum dizisi yeniden kurulamaz).
- Ham ping saklaması aynen sürer (30 gün, [ADR-0005](0005-konum-takibi-ve-kvkk.md)); özet üretimi bu sürenin dolmasından önce çalışır.
- **Amaç sınırlılığı:** özetler yalnızca rota süresi tahmini içindir; kurye performansı, hız ihlali veya denetim için kullanılmaz ve bu amaçla sorgulanmaz.
- Kurye aydınlatma metni ve açık rıza metni yeni amacı ("anonim hız istatistiği üretimi") yazar; yeni `consentVersion`; özet toplama da bu metin yürürlüğe girmeden başlamaz. Yani KVKK hukuki onayı (pilot öncesi, PLAN P-9) hem toplamanın hem öğrenmenin ön koşuludur; onay gecikirse toplama da gecikir, ham saklama süresi uzatılarak telafi edilmez.

## Sonuçlar

- (+) Google'ın yapamayacağı bir avantaj: bizim saha ve kendi kuryelerimizin gerçek teslim noktaları ve gerçek hızları. Veri biriktikçe kalite artar; rakipte yoktur.
- (+) ADR-0006 d'deki Google ToS (koordinat önbellekleme) riski bu yoldan tamamen kalkar: öğrenilmiş noktalar kendi verimizdir.
- (+) Toplama ilk günden başlar, ham saklama süresi uzamaz (özetler toplanabilir ve anonim).
- (+) Tutulan veri doğrulaması ve geri dönüş, kötü öğrenmenin canlıya çıkmasını engeller.
- (-) **Pilot süresi bağımlılığı:** öğrenilmiş hızlar için birkaç haftalık pilot verisi gerekir (3c); öğrenilmiş teslim noktaları tekrar eden adreslerde bile ancak 2-3 teslimden sonra devreye girer. Pilot ölçeği küçükse (20-25 kurye) veri seyrek kalabilir; eşikler bilerek yüksektir ve çoğu parça profil varsayılanında kalır. Kazancın büyüklüğü bilinmiyor; benchmark'ta tahmin edilemez, pilotta ölçülür.
- (-) Seyrek GPS örneklemesi (15 m / 60 sn seyreltme, [ADR-0005](0005-konum-takibi-ve-kvkk.md)) parça başına hız tahminini kabalaştırır; eşleme hataları (özellikle paralel yollar) gürültü ekler. Eşikler ve karıştırma bunu sınırlar, ortadan kaldırmaz.
- (-) Kurye aydınlatma metni değişir; hukuki sign-off olmadan canlıya alınamaz (pilot öncesi iş, PLAN P-9).
- (-) Yeni tablolar, iki zamanlanmış iş (özet toplama, hız dosyası yayını) ve yeni bir işletim yüzeyi.
- **Açık sorular:** gözlem saklama (90 gün) ve özet saklama (12 ay) süreleri hukuk görüşüne göre; `address_key`'in düz metin mi hash mi saklanacağı; k-anonimlik eşiği (`courier_days >= 5`) pilot ölçeğinde yeterli mi.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Yalnızca tek global `ROUTING_DURATION_FACTOR` (bugün) | Hangi sokağın / saatin yavaş olduğunu bilmez; ölçülebilir iyileştirme potansiyeli bırakır (3c'de ölçülür) |
| Trafik duyarlı Google Routes ile ETA | Üçüncü taraf bağımlılığı, paket başı maliyet, konum verisi dışarı çıkar; ürün sahibinin yönüne aykırı ([ADR-0016](0016-eskisehire-ozel-kendi-barindirdigimiz-harita-platformu.md)) |
| Ham pingleri uzun süre saklayıp sonra öğrenmek | KVKK ölçülülüğüne aykırı; ADR-0005 saklama süresini uzatmayı gerektirir. Toplanabilir özet aynı değeri ham veri tutmadan verir |
| Teslim noktasını müşteri telefonuna bağlamak | Telefon kişisel veridir; adres anahtarı yeterlidir ve müşteriyi tanımlamaz |
| Makine öğrenmesi (ör. grafik sinir ağıyla ETA) | Veri azken açıklanamaz ve aşırı uyum riski; parça bazlı karıştırılmış istatistik şeffaf ve hata ayıklanabilir. Veri birikince yeniden değerlendirilir |
| Teslim noktası olarak her zaman son gözlemi kullanmak | Gürültülüdür; ağırlıklı medyan + güven eşiği daha dayanıklıdır |

## Güncelleme (2026-11-06): İterasyon 3b spesifikasyonu

SPEC-3b §6 A bölümünü uygulanabilir ayrıntıya indirdi; karar özü değişmedi, şu noktalar netleşti:

- **Güven sabitleri:** biçim aynı, başlangıç sabitleri **N0 = 2, S0 = 75 m** (önce 3 ve 40). Gerekçe: σ = 15 m gürültüde 3 / 40 ile eşik (0,6) ancak ~7 gözlemde aşılıyordu; 3a-8 adres testi "k ≥ 3'ten sonra öğrenilmiş noktayı varsayılan yap" önerdi. Yeni sabitlerle σ = 15 m'de 3 gözlem, σ = 30 m'de 5 gözlem eşiği aşar. Uygulama görevi değerleri adres kalitesi simülasyonuyla doğrular ve sonucu buraya "Uygulama notları" olarak ekler.
- **Adres anahtarı:** açık soru kapandı: **HMAC-SHA256** (sunucu sırrı `ADDRESS_KEY_SECRET`) normalize metin + ~25 m hücre; adres metni adres defterine girmez. Sır döndürülürse öğrenme sıfırlanır (yalnızca sızıntıda).
- **Saklama:** 90 günü geçen gözlemler silinmeden önce özetin önsel noktasına katlanır (aksi hâlde 12 aylık özet 90. günde boşalırdı); gözlemde tarih saklanır, saat saklanmaz.
- **Kullanım bayrağı:** gözlem toplama ve kullanım `DELIVERY_POINT_LEARNING_ENABLED` (varsayılan kapalı) ile KVKK onayından sonra açılır; firma bazında `settings.addressBook.useLearnedPoints`.
- **Geofence ile ilişki:** geçersiz kılınan, düşük doğruluklu ve sahte konumlu teslimler gözlem sayılmaz ([ADR-0026](0026-teslim-geofence-politikasi.md)).

## Uygulama notları (İterasyon 3b-1, Görev Q3b)

- **Kod:** `services/api/src/address-book/` (saf: `address-key.ts` normalleştirme + HMAC + 3 x 3 komşu hücre, `estimate.ts` ağırlıklar + Weiszfeld + aykırı ayıklama + Kish + güven + önsel katlama, `observation-filter.ts`, `target.ts`; G/Ç: `address-book.service.ts`, gece işi `address-book-jobs.service.ts`). Şema `drizzle/0005_iterasyon3b.sql`. `DELIVERY_POINT_LEARNING_ENABLED=false` iken hiçbir satır yazılmaz; açılışta bayrak kapalıysa tamamlanmamış paketlerdeki saklı hedefler temizlenir (kapalıyken 3a davranışı).
- **Sabitler doğrulandı, değiştirilmedi (N0 = 2, S0 = 75 m):** gerçek fonksiyonla ağsız simülasyon (`npm run geo:learned -w @kurye/api`, sonuç `docs/benchmark/adres-kalitesi/ogrenilmis-nokta.md`): sigma = 15 m ve %10 aykırıda k = 3 gözlemde medyan hata 14 m (hedef <= 25 m), <= 30 m payı %91,6. **Sapma:** SPEC §6.4 örneği (σ = 15 m, n_eff = 3 için güven 0,62) eşit ağırlık varsayar; gerçek GPS doğruluğu 5-25 m olduğundan doğruluk ağırlığı (min(1, 10/doğruluk)) n_eff'i düşürür ve k = 3'te güven eşiği (0,6) denemelerin yalnızca %61'inde aşılır (k = 4: %89, k = 5: %96). Eşik aşılmayan adreste geocoder noktası kalır (davranış bozulmaz), yani yalnızca öğrenme biraz gecikir. Sessizce değiştirilmedi; ürün sahibi isterse N0 = 1,5 veya doğruluk referansının 15 m'ye gevşetilmesi ayrı karar olarak ölçülebilir.
- **Sapma (süzgeç sırası):** OBSERVE kipinde tolerans dışı teslim (`OUTSIDE_OBSERVED`) gözlem olarak alınmaz (ADR-0026 madde 4); bu yüzden "300 m uzak gözlem" sayacı (`rejected_far_count`) yalnızca geofence KAPALI iken artar.
- Önsele katlama yalnızca aykırı olmayan eski gözlemleri toplar (aykırılar önseli kirletmesin); tek yönlü bozunma 90 gün yarı ömürlüdür. Etkin nokta yalnızca CREATED / IN_POOL paketlerde yeniden yazılır.
- `geo:quality` (`run.ts`) artık gerçek fonksiyonu kullanır. Kendi Photon kapalı olduğundan tam koşu (README tablosu) bu görevde yenilenemedi; sonraki Photon'lu koşuda güncellenir.
