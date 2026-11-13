# Uygulama içi navigasyonda sapma dayanıklılığı: kestirme, ters yön, yol dışı, GPS sıçraması (Kasım 2026)

> Soru: kurye rotadan kendi bildiği kestirmeyle ayrıldığında (yan sokak, tek yönlü sokakta ters yön, otopark / site içi / yaya yolu) ya da GPS konumu sıçradığında navigasyon **ne kadar sürede, kaç istekle ve ne kadar sakin** biçimde kuryenin gerçek konumundan makul bir rotaya döner? Önce ölçüldü, sonra yalnızca ölçümün gösterdiği yerler düzeltildi, sonra aynı ölçüm tekrarlandı.

Dal: `feature/nav-sapma-dayanikliligi`. Kod: `session.ts`, `offRoute.ts`, yeni `fixGate.ts`. Ölçüm düzeneği: `navScenario.ts`, `deviation.test.ts`, senaryo üretici `nav-sapma-generate.mts`. Önceki akıcılık çalışması (NAVIGASYON-AKICILIK.md, [ADR-0024](../kararlar/0024-navigasyon-render-performansi.md)) korunur; sunucu ve sözleşme değişmedi (API `bearings` davranışı aynı).

## 1. Özet

| Soru | Önce | Sonra |
|---|---|---|
| Paralel sokak kestirmesi (S1): ayrılmadan doğru rotaya | 8 sn (cihaz yönü yoksa 19 sn) | **4 sn** (cihaz yönü yoksa 8 sn) |
| Erken dönüş, ileri kavşak gereksiz (S4): doğru rotaya / yanlış rota süresi | 9 sn / 19 sn (cihaz yönü yoksa 20 / 30 sn) | **5 sn / 11 sn** (yön yoksa da) |
| Ters yön (S2 bölünmüş yol, S2b tek şerit): ilk yeni rota | 10 / 9 sn | **6 / 5 sn** |
| Ters yön ve yol dışında: istek sayısı / okunan "U dönüşü" | 2 + 2 + 1 / 0 | **aynı / 0**; ayrıca "Yola çıkınca rota güncellenecek" sakin durumu (S2b 17 sn, S3 21 sn) |
| Tek GPS sıçraması 110-150 m (S5a): işaretin sıçraması | **116 m** | **9 m** (2 okuma reddedildi) |
| GPS kayması 8-10 sn, 60-70 m (S5b, S5c), kentsel kanyon + durakta bekleme (S6), 15 sn tünel (S7) | 0 gereksiz istek | **0 gereksiz istek** (değişmedi) |
| Konum başına JS işi (Node, jest / emülatör Hermes) | ~0,05 ms / 3,1 ms | ~0,05 ms / 3,1 ms (+ en çok 3 ikili arama; O(1)/O(log n)); JS CPU %5,6 -> %5,6 |
| Emülatörde (cihaz yönü yok) S1 / S4 / S2b | - | 3 / 2 / 2 istek, U dönüşü okunmadı, S2b'de sakin durum (§6). Harita fps'i bu oturumda emülatör GL yavaşlığı yüzünden karşılaştırılamadı |

Ölçülen en büyük sorun **tek GPS sıçramasında işaretin 116 m zıplaması** idi (kapıyla giderildi); ikinci sorun açık sapmada bile **5 sn'lik sabit bekleme** (yön farkıyla 1,5 sn'ye indi). Mevcut motor gürültü, kayma, tünel ve durakta beklemede **zaten sakin** idi (0 gereksiz istek); bu davranış korundu ve testle kilitlendi.

## 2. Yöntem

**Senaryolar gerçek yol ağından, iz sentetik.** `scripts/nav-sapma-generate.mts` kendi OSRM'imize (Eskişehir OSM çıkarımı, ODbL) sorarak senaryo başına (a) temel rotayı, (b) kuryenin **gerçek** yolunu (yol ağından ya da yol dışı düz çizgi), (c) 1 Hz **sentetik GPS izini** üretir. Hiçbir iz gerçek bir kişiden kaydedilmedi. Gürültü tohumludur (deterministik): AR(1) konum hatası σ 3,5 m (kanyon / durakta 9 m), doğruluk 4-8 m (kanyon 14-32 m), cihaz yönü hareketteyken gerçek yön ± 4°, dururken yok ya da rastgele.

**Yeniden rota "kâhini".** İzin her saniyesindeki **gerçek** konumdan hedefe, API'nin yaptığı gibi rota hesaplanıp (`navigation.service.ts`: hız >= 2 m/sn ve yön varsa `bearings=<yön>,60`, yol bulunamazsa yönsüz) API'nin kendi dönüştürücüsüyle (`mapOsrmRoute`, Türkçe talimat) `NavigationRoute`'a çevrilir ve fikstüre (`src/__tests__/fixtures/nav-sapma/*.json.gz`, toplam ~330 KB) yazılır. Testte uygulama bir yeniden rota istediğinde, istek anındaki saniyenin rotası 600 ms gecikmeyle döner; gönderilen yön gerçek yönden > 60° farklıysa yönsüz rota kullanılır (yaklaşım). Böylece jest OSRM'siz, deterministik çalışır.

**Oynatma.** `NavigationSession` gerçek koduyla, sahte saatle, 100 ms adımla sürülür (yanıt gecikmesi, 1 sn'lik `tick`, kamera hızı dahil). Ölçüler (her senaryo için):

| Ölçü | Tanım |
|---|---|
| İlk yeni rota | Olaydan (gerçek yolun rotadan > 15 m ayrıldığı saniye) ekrana ilk yeni rotanın gelmesine kadar |
| Doğru rotaya | Olaydan (yasal ağ dışı bölüm varsa kuryenin yasal yola döndüğü andan) gösterilen rotanın **5 sn kesintisiz makul** olmasına kadar. Makul: gerçek konum çizgiye <= 25 m, çizgi o noktada kuryenin gidiş yönünde (<= 90°), kalan yol <= en iyi yol (gerçek konumdan) x 1,25 + 150 m |
| Yanlış rota süresi | Aynı pencerede makul olmayan saniyeler |
| İstek (ağ dışı) | Yeniden rota istek sayısı (parantezde: kurye yol dışında / ters yöndeyken yapılanlar). "Olaydan önce" yapılan gereksiz istek ayrıca sayıldı: **tüm senaryolarda önce ve sonra 0** |
| Sakin durum | "Yola çıkınca rota güncellenecek" şeridinin göründüğü süre |
| Salınım | Bant talimatının 15 sn içinde A -> B -> A dönmesi |
| U dönüşü anonsu | Olaydan sonra okunan "U dönüşü" cümlesi |
| İşaret sıçraması | İki kamera güncellemesi arasında işaretin yer değiştirmesi eksi gerçek hareket (en büyük) |
| Bitiş oranı | Ölçüm sonunda gösterilen rotanın kalan yolu / en iyi yol (varışta 1) |
| `feed()` ms | Okuma başına JS süresi, Node (jest); telefonda Hermes ~3-5 kat yavaştır |

Yeniden üretme: `cd apps/courier-mobile; NAV_SAPMA_OUT=sonuc.json npx jest src/__tests__/nav/deviation.test.ts` (tablo JSON'u); `NAV_SAPMA_DEBUG=S2b` saniye saniye döker. Fikstürleri yeniden üretmek (OSRM açıkken): `npx tsx scripts/nav-sapma-generate.mts`. "Önce" sütunu aynı test, motor dosyaları `ac96116` (bu dalın ilk ölçüm commit'i, `main` motoruyla aynı) sürümüne alınarak koşuldu.

## 3. Senaryolar

| # | Senaryo | Kurgu (OSRM'den, Eskişehir) |
|---|---|---|
| S1 | Paralel sokak kestirmesi | 3,7 km'lik rotanın 1,85. km'sinde yan sokağa dönüp ~70 m ötedeki paralel sokaktan gider, 2,25. km'de rotaya katılır; 9 m/sn |
| S2 | Bölünmüş yolun tek yönlü şeridinde ters yön | Yasal yaklaşım, sonra bir bulvar şeridinde 217 m ters; yasal geri dönüş 699 m. Gerçek yol 768 m, yasal rota 782 m; 7 m/sn |
| S2b | Tek şeritli tek yönlü caddede ters yön | 216 m ters (yasal geri dönüş 636 m); gerçek yol 615 m, yasal rota 688 m |
| S3 | Yol dışı alanı kesme (otopark / site içi / yaya yolu) | Rotada 540 m dolanan bir köşeyi 146 m'lik düz çizgiyle, ortası en yakın yola >= 30 m uzak bir alandan 4 m/sn keser, rotaya geri katılır |
| S4 | Erken dönüş: kalan rotanın U dönüşü / ileri kavşağı gereksiz | 4,0 km'lik temel rota 3,9. km'de U dönüşü yapıyor; kurye 1,33. km'de (U dönüşüne 2,5 km kala) başka yola döner, U dönüşsüz 3,6 km'de varır |
| S5a | Rotadayken tek GPS sıçraması | 60. ve 140. saniyede tek okuma 110-150 m yana, iyi doğruluk bildirerek (en kötü durum); 11 m/sn |
| S5b | GPS kayması | 8 sn boyunca 60 m'ye kadar yana kayma, doğruluk 8 -> 30 m (kısmen dürüst) |
| S5c | GPS kayması, yanıltıcı doğruluk | 10 sn boyunca 70 m'ye kadar, doğruluk 4-8 m'de kalır |
| S6 | Kentsel kanyon, düşük hız, durakta bekleme | Tüm iz σ 9 m, 3 m/sn; ortada 90 sn durma, yön rastgele / yok |
| S7 | Tünel / alt geçit | 15 sn okuma yok, sonra ilk okuma doğruluğu 35 m |

## 4. Önce / sonra (birim: tablo başlığında; "önce → **sonra**")

| Senaryo | İlk yeni rota (sn) | Doğru rotaya (sn) | Yanlış rota süresi (sn) | İstek (ağ dışı) | Sakin durum (sn) | Salınım | U dönüşü anonsu | İşaret sıçraması (m) | Reddedilen okuma | Bitiş oranı |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| S1 | 8 → **4** | 8 → **4** | 7 → **3** | 1 (0) → **1 (0)** | 0 → **0** | 0 → **0** | 0 → **0** | 28 → **7** | 0 → **0** | 1 → **1** |
| S2 | 10 → **6** | 0 → **0** | 0 → **0** | 3 (2) → **3 (2)** | 0 → **4** | 1 → **0** | 0 → **0** | 25 → **22** | 0 → **0** | 1 → **1** |
| S2b | 9 → **5** | 0 → **0** | 0 → **0** | 2 (2) → **2 (2)** | 0 → **17** | 1 → **1** | 0 → **0** | 24 → **26** | 0 → **0** | 1 → **1** |
| S3 | 14 → **10** | 0 → **0** | 0 → **0** | 1 (1) → **1 (1)** | 0 → **21** | 0 → **0** | 0 → **0** | 29 → **22** | 0 → **0** | 1 → **1** |
| S4 | 9 → **5** | 9 → **5** | 19 → **11** | 2 (0) → **2 (0)** | 0 → **0** | 0 → **0** | 0 → **0** | 28 → **28** | 0 → **0** | 1 → **1** |
| S5a | - | 0 → **0** | 0 → **0** | 0 → **0** | 0 → **0** | 0 → **0** | 0 → **0** | **116 → 9** | 0 → **2** | 1 → **1** |
| S5b | - | 0 → **0** | 0 → **0** | 0 → **0** | 0 → **0** | 0 → **0** | 0 → **0** | 25 → **25** | 0 → **0** | 1 → **1** |
| S5c | - | 0 → **0** | 0 → **0** | 0 → **0** | 0 → **0** | 0 → **0** | 0 → **0** | 30 → **30** | 0 → **0** | 1 → **1** |
| S6 | - | 0 → **0** | 3 → **3** | 0 → **0** | 0 → **0** | 0 → **0** | 0 → **0** | 24 → **26** | 0 → **0** | 1 (varış yok) |
| S7 | - | 0 → **0** | 0 → **0** | 0 → **0** | 0 → **0** | 0 → **0** | 0 → **0** | 6 → **6** | 0 → **0** | 1 → **1** |

Okuma başına JS süresi (`feed()`, Node, ortalama / p95): önce 0,04-0,08 / 0,05-0,21 ms, sonra 0,04-0,09 / 0,05-0,17 ms (aynı makinede, derleme yükü altında; fark gürültü içinde).

**Cihaz yön / hız vermediğinde** (bazı ucuz telefonlar, düşük hız; motor ikisini ardışık okumalardan türetir; ölçüm komutu `NAV_SAPMA_NO_HEADING=1`, yön yerine hep 0° bildiren cihaz için `NAV_SAPMA_ZERO_HEADING=1`; bu iki ortam değişkeni yalnızca tabloyu üretmek içindir, ilk `describe` bloğunun cihaz yönlü eşikleri o koşulda bilerek kalır; koşula ait eşikli testler ayrı bloktadır). Bu koşul ilk ölçümde yoktu; emülatör sürüşü (§6) gösterince eklendi:

| Senaryo | İlk yeni rota (sn) | Doğru rotaya (sn) | Yanlış rota süresi (sn) | İstek (ağ dışı) | Sakin durum (sn) | Salınım | U dönüşü anonsu | İşaret sıçraması (m) | Reddedilen okuma | Bitiş oranı |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| S1 | 8 → **8** | 19 → **8** | 18 → **7** | 2 (0) → **1 (0)** | 0 → **0** | 1 → **0** | 0 → **0** | 28 → **9** | 0 → **0** | 1 → **1** |
| S2 | 10 → **6** | 6 → **0** | 6 → **0** | 4 (2) → **3 (2)** | 0 → **4** | 1 → **0** | 0 → **0** | 29 → **22** | 0 → **0** | 1 → **1** |
| S2b | 9 → **5** | 0 → **0** | 0 → **0** | 2 (2) → **2 (2)** | 0 → **23** | 2 → **1** | 0 → **0** | 28 → **26** | 0 → **0** | 1 → **1** |
| S3 | 14 → **14** | 0 → **0** | 0 → **0** | 1 (1) → **1 (1)** | 0 → **17** | 0 → **0** | 0 → **0** | 30 → **22** | 0 → **0** | 1 → **1** |
| S4 | 9 → **5** | 20 → **5** | 30 → **11** | 3 (0) → **2 (0)** | 0 → **0** | 0 → **0** | 0 → **0** | 27 → **27** | 0 → **0** | 1 → **1** |
| S5a | - → **-** | 0 → **0** | 0 → **0** | 0 (0) → **0 (0)** | 0 → **0** | 0 → **0** | 0 → **0** | 127 → **13** | 0 → **2** | 1 → **1** |
| S5b | - → **-** | 0 → **0** | 0 → **0** | 0 (0) → **0 (0)** | 0 → **0** | 0 → **0** | 0 → **0** | 27 → **27** | 0 → **0** | 1 → **1** |
| S5c | - → **-** | 0 → **0** | 0 → **0** | 0 (0) → **0 (0)** | 0 → **0** | 0 → **0** | 0 → **0** | 33 → **33** | 0 → **0** | 1 → **1** |
| S6 | - → **-** | 0 → **0** | 3 → **3** | 0 (0) → **0 (0)** | 0 → **0** | 0 → **0** | 0 → **0** | 22 → **26** | 0 → **0** | 1 → **1** |
| S7 | - → **-** | 0 → **0** | 0 → **0** | 0 (0) → **0 (0)** | 0 → **0** | 0 → **0** | 0 → **0** | 10 → **10** | 0 → **0** | 1 → **1** |

Burada **önceki motorun asıl sorunu** görülüyor: yeniden rota isteği ham cihaz yönünü taşıdığı için yön yoktu, sunucu yön kısıtı uygulamadı ve yeni rota kuryeyi geri döndürmek istedi (S1 19 sn, S4 20 sn doğru rotaya, ek istek ve salınım). Süzülmüş seyir yönü gönderilince ilk rota doğru (S1 8 sn, S4 5 sn). Yön konumdan türetildiğinde GPS'in yana kayması da "yön değişti" gibi görünür; bu yüzden açık sapma kararı burada 5 okuma ister (cihaz yönüyle 2): S1'de hızlı karar devreye girmez (8 sn, eskisiyle aynı bekleme), ama kaymada (S5b, S5c) yine **0** gereksiz istek.

**İnceleme düzeltmesi (PR #74 sonrası).** İlk emülatör koşularında (aşağıdaki §6 tablosu) motor emülatörün 0°'sini gerçek cihaz yönü sayıyordu: istek yanlış yönle gidiyor, açık sapma 2 okumayla tetiklenebiliyordu (S1'deki 3 isteğin bir nedeni). Artık tam 0° ve konumla çelişen yön güvenilmez; yön konumlardan türetilir. Hep 0° bildiren cihazda jest sonuçları "cihaz yönü yok" tablosuyla aynıdır (S1 8 sn / 1 istek, S5b-S5c 0 istek). Düzeltmeden sonraki emülatör S1 tekrarı bu çalışmada yapılamadı (emülatör ve mock portu başka bir işle paylaşılıyordu); telefon sürüşünde (K1) ya da sonraki emülatör kabulünde ölçülmeli. Aynı düzeltmede geçici rotanın yeniden denemeleri sunucu hatası / 429 sonrası 15 / 30 sn beklemeye de uyar.

Yanıt gecikmesine duyarlılık: aynı test 2,5 sn sunucu gecikmesiyle koşulduğunda süreler ~2 sn uzar (S1 6 sn, S4 7 sn), istek sayısı, salınım ve U dönüşü anonsu değişmez.

**Okuma.** S1 / S4'te kazanç açık sapmanın hızlı tanınmasından gelir (5 sn bekleme yerine yön farkı 2 okuma + 1,5 sn). S2 / S2b / S3'te istek sayısı zaten düşüktü (motorun yeniden rota soğuması ve "kımıldama" kapısı); değişen, ilk rotanın daha erken gelmesi, ters yöndeyken işaretin geride takılmaması ve kuryenin önünde yasal rota yokken **sakin durum**dur (U dönüşü okunmaz, şerit "Yola çıkınca rota güncellenecek" der, kurye dönünce / ilerleyince hemen yeniden istenir). S2'de OSRM, yön kısıtıyla bulvarın **karşı şeridinden** kuryenin yönünde rota buldu (yasal, makul); S2b'de karşı şerit olmadığından ilk rota kuryeyi geri döndürmek ister: burada sakin durum devreye girer. S2b'deki tek "salınım" farklı rotaların aynı metinli talimatıdır ("Sola dönün").

## 5. Yapılan değişiklikler (en küçük etkili)

| # | Değişiklik | Ölçümdeki gerekçe | Konum başına maliyet |
|---|---|---|---|
| 1 | **GPS sıçrama kapısı** (`FixGate`): son kabul edilen okumadan `15 m + 33 m/sn x dt + min(doğruluk, 50)` ötesindeki okuma reddedilir; bir sonraki okuma adaya tutarlıysa ya da üst üste 3 red olursa yeni konum kabul edilir (gerçek yer değişimi, tünel çıkışı takılmaz). Gecikme eklemez | S5a: işaret 116 m zıplıyordu | 1 mesafe hesabı |
| 2 | **Açık sapmada hızlı karar**: rotaya > 12 m uzakken, rotadan uzaklaşırken, hız >= 4 m/sn ve gidiş yönü rota yönünden >= 40° farklı okuma art arda 2 kez (yön cihazdan) ya da 5 kez (yön konumlardan türetildiyse) ise (mandallı, rotaya dönünce sıfırlanır) yoldan çıkma beklemesi 5 sn -> 1,5 sn. Rota yönü en yakın segmentten değil, oturtulan ilerlemenin -10 / +20 m'sinden (yanal GPS kayması yanlış "sapma" üretmesin; ilk denemede S5b/S5c'de 1 gereksiz istek görüldü, bu düzeltmeyle 0) | S1 8 sn, S4 9 sn | 2 ikili arama (yalnızca rotadan > 12 m uzakken) |
| 3 | **Yeniden rota isteğinde süzülmüş seyir yönü** (hareketteyken; cihaz yönü yoksa okumalardan): sunucu ±60° yön kısıtıyla rotayı kuryenin önünden başlatır | Cihaz yönü yokken S1 19 sn -> 8 sn, S4 20 sn -> 5 sn (yukarıdaki ikinci tablo) | 0 |
| 4 | **Geçici rota (sakin durum)**: gelen rota kuryenin arkasından başlıyorsa (ilk ~40 m yönü gidişe > 120°) ya da başlangıcı yola > 25 m uzaksa (yol dışı): U dönüşü anonsu okunmaz, şerit "Yola çıkınca rota güncellenecek", olağan yoldan çıkma istekleri durur; kurye >= 45° döner ya da >= 120 m ilerlerse (en az 4 sn arayla) yeniden istenir; kurye rotaya kendi yönünde uyarsa durum biter | S2b, S3: ters yönde / yol dışında kuryeye geri dön diyen rota | 1 mesafe hesabı (yalnızca geçici durumda) |
| 5 | **Bayat yanıt**: istekten sonra kurye rotaya döner ve >= 2 sn rotada kalırsa (kısa kestirme, kayma) gelen yeni rota uygulanmaz; bu arada başka rota geldiyse (yeni atama) yine uygulanmaz | Ölçümde tetiklenmedi; ağ gecikmesi uzun olan telefonda "rota değişti" sıçramasını önler (birim testli) | 0 |
| 6 | **İşaret kuryeyi izlemiyorsa oturtulmaz**: rotadan > 8 m uzakken gidiş yönü rotaya > 70° (keserek / ters) ya da geçici rotada ham konum gösterilir; en yakın nokta kayıtlı ilerlemeden > 15 m uzaksa kamera ileri tahmini yapılmaz | S1 28 m, S2 47 m'lik ara sıçrama (ilerleme tek yönlü olduğundan işaret 30-50 m ileride / geride takılıyordu) | 1 ikili arama |

Değişmeyenler: 40 m yoldan çıkma eşiği, 5 sn olağan bekleme, 10 / 20 / 40 sn soğuma, kımıldama kapısı (50 m), yaklaşık mod payları, 3 başarısızlıkta dış harita önerisi, durak durak ilerleme ve yeni atamada rota değişimi (`replaceRoute`), course-up kamera, öngörülü kamera, React'in konum başına çizilmemesi (ADR-0024). **Uygulama hiçbir zaman ters yön ya da yaya yolu önermez**: rota yalnızca sunucudan (OSRM motosiklet profili) gelir; bu çalışma yalnızca kurye bunu kendisi yaptıktan sonra uygulamanın sakin kalmasını sağlar.

Denenip geri alınan: GPS kaymasında işareti 60 m'ye kadar rotada tutmak (S5c'de işaret sıçraması 30 -> 74 m'ye çıktı: tutma bırakıldığında ham konuma tek seferde atlıyordu).

## 6. Emülatör doğrulaması

Ortam: AVD `Medium_Phone_API_36.1` (`-gpu host`), sürüm APK (x86_64, `EXPO_PUBLIC_NAV_PERF=1`), mock sunucu (`/__mock/nav-scenario?id=<senaryo>`: başlangıçta senaryonun temel rotası, yeniden rotada istek konumuna en yakın izin saniyesindeki kâhin rotası), iz `scripts/nav-sapma-playback.mjs` ile 1 Hz `adb emu geo fix`. **"Önce" APK'sı** aynı ağaçta yalnızca motor dosyaları (`src/navigation/engine`, `src/components/nav`) `ac96116` sürümüne alınarak derlendi; diğer her şey aynı. `geo fix` hız vermez; **yön yerine sabit 0°** bildirir (mock günlüğü: "yön 0 / gerçek 153"). Bu sürümde motor 0°'yi (ve konumdan türetilen yönle ±60° tutmayan cihaz yönünü) güvenilmez sayar; emülatör §4'teki "cihaz yönü yok / hep 0°" koşuludur (jest'te ikisi aynı sonucu verir).

**Senaryo sürüşleri (sonra APK'sı).** Görüntüler: [S1](nav-sapma/s1-sonra.gif), [S4](nav-sapma/s4-sonra.gif), [S2b](nav-sapma/s2b-sonra.gif) (olaydan 4 sn önce başlayan ~1 sn aralıklı ekran görüntüleri). Kayıtlar sesli yönlendirme kapatılmadan önce alındı; alt çubuktaki "Ses" düğmesi artık yoktur ([ADR-0015 eki](../kararlar/0015-uygulama-ici-navigasyon.md#ek-kasım-2026-sesli-yönlendirme-kapalı)).

| Senaryo | Yeniden rota isteği (mock günlüğü) | Gözlem |
|---|---:|---|
| S1 paralel sokak | 3 (jest: 1) | İlk istek kurye yan sokağa yeni girmişken gitti; o konumdan OSRM'in en iyi yolu bir sonraki kesişen sokaktan ana caddeye dönmekti (ileri yönlü, U dönüşü yok). Kurye paralel sokakta devam edince ikinci, sonra üçüncü rota geldi; üçüncüsü geçici rota oldu (şerit görünür), ardından doğru paralel rota. "U dönüşü" okunmadı. Ayrılmadan doğru rotaya ~25 sn. Fark, emülatörde konum güncellemesinin ve isteğin jest'tekinden ~1-2 sn farklı anlarda olmasından: kesişen sokakların önünden geçerken her yeni konum "en iyi yol ana caddeye dön" olabilir; bu, gerçek telefonda da beklenmeli (K1) |
| S4 erken dönüş | 2 (jest: 2) | İlk rota (dönüşten ~9 sn sonra) kuryeyi bulvara geri çağırdı (yön kısıtı yokken ilk istek); ikinci rota (+10 sn) U dönüşsüz doğru rota, kalan 2,8 -> 2,0 km. Eski U dönüşü bantta ve seste kayboldu |
| S2b tek şeritli caddede ters yön | 2 (jest: 2) | İlk rota kuryenin önünden başlamadığı için **"Yola çıkınca rota güncellenecek"** şeridi ~20 sn göründü, U dönüşü okunmadı, araya istek girmedi; kurye caddeden çıkınca ikinci rota doğru |

**Akıcılık (aynı yöntem: NAVIGASYON-AKICILIK §1.3, standart 1 467 m rota, 40 km/sa, 110 sn, iç içe koşular).** Bu oturumda emülatörün GL yolu belirgin biçimde yavaştı (harita **2-7 fps**; aynı yöntemle Kasım başında 27-34 fps ölçülmüştü): ilk 6 koşuda emülatör penceresi simge durumundaydı, pencere açılınca 4 koşu daha yapıldı. Harita fps'i bu yüzden **karşılaştırma için geçersizdir**; değişiklik GL'ye iş eklemez (yalnızca JS motoru değişti). JS iş parçacığı ölçüleri her iki turda da anlamlıdır:

| Ölçü (medyan; koşu sayısı) | Önce | Sonra |
|---|---:|---|
| `session.feed` ortalama ms (Hermes, okuma başına; dinleyiciler dahil), 5 koşu | 3,1 | 3,1 |
| `session.feed` en büyük ms, 5 koşu | 30,8 | 21,2 |
| JS iş parçacığı CPU %, 5 koşu | 5,6 | 5,6 |
| JS olay döngüsü gecikmesi p95 ms, 5 koşu | 43,9 | 44,7 |
| `NavMap` React çizimi / sn | 0,1 | 0,1 |
| Kamera komutu / sn | 0,9 | 0,9 |
| PSS sonra (MB) | 237 | 228 |
| Harita fps (pencere açık; 2 koşu) | 6,0 / 7,2 | 3,5 / 6,2 (ilk koşu pencere açıldıktan hemen sonra) |
| Harita fps (pencere simge durumunda; 3 koşu) | 2,0 / 1,7 / 1,9 | 1,7 / 1,9 / 1,7 |

Sonuç: JS tarafında ek yük görülmedi (okuma başına iş ve JS CPU aynı aralıkta, React çizimi ve kamera komut sıklığı değişmedi). Harita fps'inin gerileme olmadığını kesin göstermek için ölçüm sağlıklı bir emülatör oturumunda ya da telefonda tekrarlanmalı (NAVIGASYON-AKICILIK §7).

Emülatörde denenmeyen: S3, S5a-c, S6, S7 (jest'te ölçüldü; emülatörde `geo fix` doğruluk bildiremediği için kayma ve sıçrama senaryoları gerçekçi değildir).

## 7. Sınırlar (dürüstçe)

- **İz sentetik, kâhin yaklaşık.** Yeniden rota, kuryenin gönderdiği gürültülü konumdan değil, o saniyenin gerçek konumundan hesaplanmış rotadır; gönderilen yön gerçek yönden > 60° farklıysa yönsüz rota varsayılır. Gerçek API ±60° kısıtını gönderilen yönle uygular.
- **Kayma ve sıçrama modeli basit.** Gerçek telefonda çok yollu yansıma daha uzun (10-30 sn) ve ardışık sıçramalar üretebilir; kapı üst üste ikinci tutarlı sıçramayı gerçek sayar. Cihaz yönünün (Doppler) kaymadan etkilenmediği varsayıldı; yönü konumdan türeten cihazlarda kayma yan sapma gibi görünebilir.
- **Senaryo sayısı küçük** (10). Senaryolar OSRM'den otomatik seçildi; S2 bir bölünmüş yol şeridi, S2b tek şeritli cadde. "3 km ilerideki kavşak" yerine S4'te ayrılma noktasından 2,5 km ilerideki U dönüşü vardır.
- Kentsel kanyonda (S6) düşük hızda yanal sapma 3 sn "yanlış rota" sayıldı (gürültü 25 m'yi aştı); istek üretmediği için değiştirilmedi.
- Kayma sırasında işaret 25-30 m yana gidebilir (önce de öyleydi); bunu yumuşatmak gecikme eklemeden bu çalışmada çözülmedi.
- Yalnızca uygulama mantığı ölçüldü: mobil ağ gecikmesi (600 ms varsayıldı), API'nin kendi süresi, pil ve gerçek GPS çipinin davranışı telefonda ölçülmelidir.

## 8. Yalnızca gerçek telefonun gösterebileceği

- Gerçek GPS'in kestirme / otopark / bina arası davranışı (kayma süresi ve büyüklüğü, cihaz yönünün güvenilirliği, düşük hızda yön).
- Mobil ağda yeniden rota süresi (istek + çizim) ve "Yola çıkınca rota güncellenecek" şeridinin sürücü için anlaşılırlığı.
- Gerçek tek yönlü sokakta ve bölünmüş yolda OSRM'in yönlü rota seçimi (karşı şerit mi, geri dönüş mü).
- Pil ve ısı (ek iş ölçülemeyecek kadar küçük, ama ekran / GPS baskındır).

Telefon sürüşü kontrol listesi: TELEFON-TESTI-PROTOKOLU.md §8.

## 9. Ürün sahibi kararı bekleyen

Güvenli varsayılan seçildi; her biri tek sabittir (`session.ts`, `offRoute.ts`, `fixGate.ts`).

| Konu | Seçilen varsayılan | Alternatif |
|---|---|---|
| Ters yön / yaya yolu politikası | **Uygulama asla önermez**; kurye yaptıktan sonra sakin durum + kurye dönünce / 120 m'de yeniden rota | Bilinen kestirmelere izin (profil değişikliği, ADR gerekir; önerilmez) |
| Yoldan çıkma eşiği | 40 m (değişmedi) | 30 m (daha hızlı, kanyonda daha çok yanlış alarm) |
| Olağan bekleme / açık sapmada bekleme | 5 sn / **1,5 sn** (yön >= 40°, 2 okuma, >= 4 m/sn) | 3 sn / 1 sn |
| Sıçrama kapısı | 33 m/sn (~120 km/sa) + 15 m + doğruluk; 2. tutarlı okuma ya da 3 red sonrası kabul | Daha sıkı (25 m/sn) ya da kapalı |
| Geçici rota tetiklerinde yeniden istek | Dönüş >= 45° ya da >= 120 m, en az 4 sn arayla | 80 m / 3 sn (daha çok istek) |
| Bayat yanıt | Kurye istekten sonra >= 2 sn rotadaysa yeni rota uygulanmaz | Her yanıtı uygula (eski davranış) |
