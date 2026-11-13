# ADR-0039: "Ters atama" ölçütü (tanım, eşikler ve nasıl ölçüldüğü)

- **Durum:** Önerildi (eşikler sentetik veriden kalibre edildi; pilot verisiyle yeniden kalibre edilecek)
- **Tarih:** 2026-11-13
- **İlgili:** [ADR-0009](0009-rota-uyumlu-atama.md) (rota uyumlu atama, sapma eşiği, "Ters yön" etiketi), [ADR-0014](0014-toplu-atama-regret-ekleme.md) (toplu atama), [ADR-0023](0023-dagitim-algoritmasi-v2.md) (v2 kayan ufuk, gecikmeli atama), [ADR-0028](0028-havuz-gorunurlugu-ve-atama-kabulu.md) (pilot kipi, kabul süresi), [ADR-0033](0033-yuk-testi-darbogazlari.md) (yeniden deneme aralığı); rapor: [TERS-ATAMA-2026-11](../benchmark/TERS-ATAMA-2026-11.md)

## Bağlam

Ürünün vaadi, yöneticinin paneli izleyip sistemi düzeltmek zorunda kalmamasıdır: sistem bir paketi atıyorsa yönetici bunun bir gerekçesi olduğuna güvenebilmeli. Sahada "yanlış atama" denince kastedilen, paketin kuryenin **gidiş yönünün tersine** verilmesidir (kurye yolunun üstündeki bir paket yerine geri dönmesini gerektiren bir paket). Bugüne kadar elimizde bu kavramın ölçülebilir bir tanımı yoktu:

- Motorun `detourS` ölçütü (ADR-0009) yalnızca **süredir**; 4 dakikalık bir geri dönüşle 4 dakikalık bir yan sapmayı ayırt etmez.
- Motorun `OPPOSITE_DIRECTION` etiketi (`reverseAngleDeg` 120°) yalnızca sapma eşiğinin (5 dk) **üstündeki** adaylara verilir ve yalnızca alışın açısına bakar; bu yüzden OTOMATİK kipin eşik altı atamalarında hiç görünmez.
- Benchmark'taki "ters dönüş" sayacı ardışık bacaklar arası açıdır (rota kalitesi); bir atama kararının iyi olup olmadığını söylemez.

"Bizim otomatik atamamız şu koşullarda vakaların yüzde X'inde terstir" diyebilmek ve X'i düşürmek için kararın kendisini, verildiği anda sınıflandıran tek bir tanım gerekiyor.

## Karar

**1. Ölçüm anı ve girdiler.** Her atama, **atandığı anda**, kuryenin o an taahhüt edilmiş rotasına göre sınıflandırılır. Girdiler:
- kuryenin konumu P ve atamadan önceki kalan durakları s₁ … sₙ (s₁ kilitli hedef);
- yeni paketin alış (R) ve teslim (D) noktası;
- **Δt** = yeni paketin rotaya eklediği yolculuk süresi (yerleştirme marjinal maliyeti; yeni paketin kendi servis süresi hariç, motorun `detourS` tanımıyla aynı). Aynı kararda bir kuryeye birden çok paket verildiyse paket başına "çıkarma marjinali" T(sonra) − T(sonra ∖ paket) kullanılır.

**2. Geometri.** Rota koridoru = P, s₁, …, sₙ kırık çizgisi. Gidiş yönü ĥ = P'den en az 100 m uzaktaki ilk durağa birim vektör. Bir X noktası (R veya D) **geride** sayılır, eğer:
- X'in koridora uzaklığı > **w** (koridor dışında; rota daha sonra o tarafa dönüyorsa nokta koridordadır ve geride sayılmaz), **ve**
- X'in ĥ yönündeki geri izdüşümü > **w** (kuryenin arkasında).

**3. Sınıflar** (ilk eşleşen kural; dereceli skor 0-3):

| Sınıf | Kural | Derece |
|---|---|---:|
| `IDLE` (boşta) | Taahhüt edilmiş durak yok: yön tanımsız, ters olamaz | 0 |
| `SEVERE` (ağır ters) | R veya D geride, Δt ≥ τ ve (Δt ≥ τₐ veya geri mesafe ≥ 2 km) | 3 |
| `AGAINST` (ters) | R veya D geride ve Δt ≥ τ | 2 |
| `DETOUR` (sapma) | Δt ≥ τ ama geride nokta yok (yan sapma) | 1 |
| `ALONG` (yol üstü) | Diğer her şey | 0 |

**"Ters atama" = AGAINST ∪ SEVERE.** Ayrı bir boyut olarak, atamanın önceden taahhüt edilmiş bir durağı 5 dakikadan fazla geciktirip geciktirmediği de raporlanır (ters olmasa da kötü atama).

**4. Eşikler veriden** (kalibrasyon tohumları 101-105; rapor tohumları 1-10'dan ayrık; ~260 bin kahin yerleştirme örneği):

| Eşik | Değer | Dayanak |
|---|---:|---|
| Koridor w | 600 m | Δt ≤ 60 sn olan (tartışmasız yol üstü) eklemelerde noktanın koridora uzaklığının P90'ı 626 m |
| Ters için en az Δt (τ) | 150 sn | Koridor içi eklemelerin Δt medyanı 159 sn: "olağan bir yol üstü ekleme kadar bile maliyeti olmayan" geri nokta zarar sayılmaz |
| Ağır ters (τₐ) | 360 sn | Koridor içi eklemelerin Δt P90'ı 379 sn |
| Ağır ters geri mesafe | 2000 m | Ucuz eklemelerde ham geri mesafenin P90'ı ~2 km |

τ'ya duyarlılık (60 / 150 / 240 / 360 sn) raporda tablo olarak verilir: OTOMATİK v1 her τ'da en düşük orana sahiptir, diğer algoritmaların kendi aralarındaki sırası τ'ya bağlıdır. Bu yüzden oran her zaman τ ile birlikte raporlanır.

**5. Kenar durumlar.**
- *Boştaki kurye:* yön yoktur, `IDLE`; ters oranının paydasında kalır (sistemin verdiği bir karardır) ama ters olamaz. Rotası olan kuryelere atamalardaki oran ayrıca raporlanır.
- *İşletmede bekleyen kurye:* P = işletme; yön bir sonraki **farklı** duraktan alınır. Tüm duraklar kuryenin bulunduğu noktadaysa yön tanımsızdır ve ters olamaz.
- *Aynı işletme gruplaması:* R zaten rotadaki bir alışla aynı noktadaysa R koridordadır; teslimi ters yöndeyse atama **ters sayılır** (kurye için gerçekten geri dönüştür) ama ayrıca işaretlenir ("aynı işletme alışı" payı raporda).
- *Kapasite 5 / bilinçli toplama:* kural kapasiteye bakmaz; kahin karşılaştırması "en iyi kurye kapasite yüzünden atlandı" ve "makul alternatif yalnızca dolu kuryelerdeydi" oranlarını ayrıca verir.
- *Yönetici ve zorunlu atama:* başlık oranı yalnızca **sistem** atamalarını sayar (otomatik ve algoritma kararları); yönetici modeli ve 15 dk zorunlu atama ayrı raporlanır.

**6. Neden kümesi.** Her ters atama tek nedene bağlanır (öncelik sırasıyla): kabul süresi aşımı sonrası yeniden atama; v2 gecikmeli atama sınırı; makul alternatif yalnızca dolu kuryelerdeydi (kapasite); makul alternatif yoktu (tek seçenek); alternatif vardı ama toplu / pencere kararı seçti; alternatif vardı ama tekil skor (yalnızca süre) seçti. "Makul alternatif": ters olmayan ve Δt'si seçilenden en çok 120 sn kötü, vardiyadaki başka bir kurye.

**7. MRTA karşılığı.** Kuryeler çok görevli robotlar (kapasite 5, *multi-task robots*), paketler tek robot gerektiren görevlerdir (*single-robot tasks*); OTOMATİK v1 anlık (*instantaneous*) değil zaman uzanımlı atamadır (*time-extended*: görev, robotun mevcut görev paketine/demetine — CBBA'daki *bundle* — eklenir). Δt, yerleştirme sezgiselinin (*insertion heuristic*) marjinal maliyetidir; CBBA'da bir robotun teklifi tam bu marjinal kazançtır. Ters atama, marjinal maliyetin geometrik bir alt kümesidir: marjinal maliyet tek başına "ne kadar" sorusunu yanıtlar, ters ölçütü "hangi yönde" sorusunu ekler. Bu yüzden yalnızca marjinal maliyeti en aza indiren bir açgözlü atama (tekil skor) veya bir toplu atama (regret, pencere) ters kararlar verebilir: kalan tek seçenek terse, ya da ters seçenek marjinal olarak ucuzsa.

**8. Kod ve kullanım.** Sınıflandırıcı saf bir modüldür (`services/api/src/routing/ters.ts`: `classifyAssignment`, `cheapestInsertion`, `maxDelayCausedS`; G/Ç yok, birim testli). Benchmark (`npm run benchmark:ters`) her atamayı bu modülle sınıflandırır. Üretim dağıtım davranışı ve varsayılanları **değişmez**; bu ADR yalnızca ölçütü tanımlar. Canlı panelde gösterim ayrı bir iştir (rapordaki "Canlı ölçüm" bölümü).

## Sonuçlar

- (=) İlk ölçüm (sentetik, 8 senaryo × 10 tohum, kapasite 5): sistem atamalarında ters oranı en yakın kurye %17,3, OTOMATİK v1 (pilot ayarları) %10,3 [9,8; 10,7], dağıtım v2 %14,8. Benchmark'ta denenen yön koruması v1'de %3,1'e, v2'de %7,1'e indiriyor ve teslim süresine anlamlı bir maliyeti yok; uygulanması ürün sahibi kararıdır ([rapor](../benchmark/TERS-ATAMA-2026-11.md) bölüm 8).
- (+) "Ters atama" artık tek kurallı, tekrar üretilebilir bir sayıdır; algoritma ve ayar değişiklikleri bu sayıyla karşılaştırılır.
- (+) Eşikler tat değil dağılım yüzdelikleriyle seçildi ve duyarlılık tablosuyla raporlanır.
- (+) Sınıflandırıcı yalnızca konum, durak sırası ve Δt ister; canlı sistemde atama anında hesaplanabilir.
- (−) Geometri kuş uçuşu doğru parçalarıyla yapılır; nehir, demiryolu veya çevre yolu boyunca ters görünmeyen ama yol ağında geri dönüş gerektiren durumlar kaçabilir (Δt bunu kısmen yakalar).
- (−) Eşikler sentetik talepten kalibre edildi; pilot verisi geldiğinde aynı yöntemle yeniden kalibre edilmeli (rapor: istenecek veri listesi).
- (−) Konum gecikmesi (bayat konum) benchmark'ta modellenmedi; canlıda sınıflandırma kuryenin son bilinen konumuyla yapılacağı için oran biraz farklı çıkabilir.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Yalnızca sapma süresi eşiği (Δt > 5 dk = kötü) | Yönü görmez: kısa bir geri dönüşü "iyi", uzun bir yol üstü toplamayı "kötü" sayar; sahadaki şikâyetle örtüşmez |
| Motorun `OPPOSITE_DIRECTION` etiketi (alış açısı > 120° ve sapma > eşik) | Eşik altında hiç tetiklenmez; teslim noktasını ve rotanın sonraki bacaklarını görmez (rota geri dönüyorsa yanlış alarm) |
| Ardışık bacak açısı (benchmark "ters dönüş" sayacı) | Rota kalitesi ölçütüdür, karar ölçütü değildir; iyi bir sıralama da açı üretir |
| Yol ağı üzerinde ilerleme (OSRM rota geometrisine izdüşüm) | Daha doğru ama her atamada rota geometrisi ister (OSRM `route` çağrısı); önce basit ve saf tanım, gerekirse sonra |
| Elle etiketlenmiş "yanlış atama" verisi | Henüz pilot verisi yok; geldiğinde bu tanımın doğrulama kümesi olacak |
