# ADR-0014: Toplu (batch) atama: mevcut rotalara regret-2 ekleme, gecikme sınırı ve aynı işletme gruplaması

- **Durum:** Önerildi (İterasyon 2 spesifikasyonuyla birlikte onaya sunuldu)
- **Tarih:** 2026-11-02
- **İlgili:** [ADR-0009](0009-rota-uyumlu-atama.md), [ADR-0013](0013-rota-motoru-siniri-ve-yaklasik-mod.md), İterasyon 2 SPEC §7.12

## Bağlam

Ürün sahibi pilot müşterinin en yüksek önceliğini şöyle tarif etti: elinde birden çok paket olan kurye tek bir akıllı rota görür; **yeni paket veya paketler** geldiğinde sistem onları **mevcut rotasına en iyi uyan** kuryeye, mevcut paketleri belirgin biçimde bozmadan (her mevcut paket için bir gecikme sınırı ve tazelik sınırı) yerleştirir ve rota durak olarak yeniden kurulur.

"Paketler" vurgusu önemlidir: aynı restorandan art arda 3 sipariş veya yoğun saatte bir patlama sık görülür. Paketleri tek tek, geliş sırasıyla atamak (ADR-0009'un temel akışı) iki sorun doğurur:
- İlk paket en iyi kuryeyi "kapar", ikinci paket için daha kötü bir seçim kalır (sıraya bağımlılık).
- Aynı restorandan üç paket üç farklı kuryeye dağılabilir; oysa tek kurye tek alışla üçünü taşıyabilir.

Seçenekler: (1) VROOM'a tüm kuryelerin mevcut rotalarını sabit veya kısmen sabit adımlar olarak, yeni paketleri de shipment olarak vermek; (2) mevcut yerleştirme kodunun üzerine sıralı ekleme + regret sezgiseli.

## Karar

**1. Toplu atama, regret-2 ekleme sezgiseliyle yapılır** (rota motoru `planBatch`, en çok 20 paket):

```text
birimler = grupla(paketler)                 # madde 3
her birim u ve her kurye c için: maliyet[u][c] = u'nun c'nin (geçici) rotasına en ucuz uygulanabilir eklenmesi
döngü (birim kaldıkça):
    her u için: b1 = en iyi maliyet, b2 = ikinci en iyi (yoksa +∞) ; regret[u] = b2 - b1 (b2 = ∞ ise büyük sabit M)
    u* = argmax regret  (eşitlikte küçük b1, sonra paket kimliği)
    u* uygulanabilir kurye bulamadıysa: grup ise tekil paketlere böl ve devam; tekilse "atanamadı" (neden ile)
    u*'ı en iyi kuryesine yerleştir; yalnızca o kuryenin maliyet satırını yeniden hesapla
```

Maliyet, tekil atamadaki skorun aynısıdır (ADR-0009: sapma + α·gecikme + β·bekleme + γ·alış gecikmesi + φ·adalet). Yüksek regret = "bu paketi şimdi en iyi kuryesine vermezsek çok kaybederiz"; bu, sıraya bağımlılığı kırar.

**2. Mevcut paketler korunur.**
- Toplu atama mevcut (atanmış) paketlerin kuryesini **değiştirmez**; yalnızca yeni paketleri yerleştirir.
- **Gecikme sınırı (yeni sert kısıt):** her mevcut paketin teslim ETA'sı, kuryenin toplu atama **öncesi** planına göre en çok `maxAddedDelayPerPackageMin` (varsayılan 8 dk) ileri kayabilir. Sınır birden çok eklemenin toplamına uygulanır (plan içinde birikerek aşılamaz). Aynı kısıt tekil yerleştirmede ve VROOM yeniden sıralamasında da geçerlidir (zorunlu değişiklik hariç).
- Tazelik sınırı (alıştan teslime 35 dk) her pakete uygulanır (alınmış paketlerde orijinal alıştan).
- Toplu yerleştirmeden sonra değişen her kurye için normal nihai sıralama (VROOM / EXACT + kararlılık kuralları) çalışır.

**3. Aynı işletme gruplaması.** Aynı `groupKey`'li (işletme), alış noktası aynı (≤ 25 m) ve teslim noktaları birbirine `groupSplitM` (varsayılan 3 km) içinde olan paketler, kurye kapasitesine sığdığı ölçüde tek birim olarak tek kuryeye eklenmeye çalışılır (birimin içi ardışık en ucuz eklemeyle). Hiçbir kuryeye birlikte sığmıyorsa birim tekil paketlere bölünür. Aynı noktadaki ardışık alışların servis süresinin yarıya inmesi (SPEC §7.4) grubu maliyet olarak da doğal kılar.

**4. Ne zaman çalışır.** OTOMATIK modda `batchWindowS` (varsayılan 20 sn) içinde gelen paketler birlikte planlanır; ONERI modunda yönetici bekleyenler kuyruğunda "Toplu öneri" ile planı görür ve tek tıkla (veya düzenleyerek) uygular. Tek paket için `planBatch` tekil öneriyle aynı sonucu verir.

## Sonuçlar

- (+) Sıraya bağımlılık azalır; aynı restoran siparişleri tek kuryede toplanır; mevcut paketlerin gecikmesi sınırlı ve açıklanabilir.
- (+) Tekil öneriyle **aynı kod** (yerleştirme, uygulanabilirlik, skor, etiket); ayrı bir çözücünün davranışını öğrenmek ve doğrulamak gerekmez. Her atama için "neden bu kurye, kaç dakika" yanıtı vardır.
- (+) Yalnızca OSRM matrisine dayanır; VROOM düşse de çalışır, yaklaşık modda bile anlamlı sonuç verir. Deterministiktir (test ve benchmark).
- (+) Hızlı: 20 birim x 25 kurye ilk tur ~256 bin dizi değerlendirmesi, sonraki turlarda tek kurye satırı; CPU < 150 ms.
- (-) Sezgiseldir; küresel optimumu garanti etmez. Benchmark C stratejisi bunu ölçer; gerekirse ileride VROOM ile karşılaştırma stratejisi eklenir.
- (-) Mevcut paketler kuryeler arasında yeniden dağıtılmaz (bilinçli kararlılık tercihi); bunu yalnızca yönetici veya devir yapar.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| VROOM'a tüm araçların mevcut rotaları (sabit / kısmen sabit adımlar) + yeni shipment'lar | Mevcut paketleri kuryeye bağlamak `skills` ile mümkün ve gecikme sınırı zaman penceresi olarak ifade edilebilir; ancak (a) yeni paketlerin alıştan teslime azami süresi yerel olarak ifade edilemez, sonuç sonradan atılabilir; (b) VROOM'un amacı toplam süredir, bizim skorumuz (alış gecikmesi, adalet) değildir; atama başına sapma ve gerekçe üretmez; (c) mevcut paketlerin sırası serbest kalır, kararlılık kuralları ayrıca uygulanmalıdır; (d) 25 araç ve ~120 shipment'ta keşif düzeyine göre yüzlerce ms ile saniyeler arası sürer ve VROOM düştüğünde toplu atama da düşer. VROOM, değişen her kurye için nihai sıralamada kullanılmaya devam eder |
| Paketleri geliş sırasıyla tek tek atamak | Sıraya bağımlı; aynı restoran siparişlerini dağıtabilir (pilotun tarif ettiği sorun) |
| Tam yeniden optimizasyon (mevcut paketler dahil, kuryeler arası taşıma) | Kuryenin elindeki işin habersiz değişmesi güveni bozar (ADR-0009 kararlılık ilkesi); yönetici ve devir zaten bu işi denetimli yapar |
| Regret-k (k > 2) veya yerel arama (LNS) | Küçük kazanç beklentisi, daha fazla karmaşıklık; benchmark gösterirse sonra |

## Güncelleme (2026-11-06): İterasyon 3b-3 kapsamı

Madde 3'ün (aynı işletme gruplaması) iyileştirilmesi SPEC-3b §8'de tanımlandı: aynı alış ödülü κ (90 sn), ortak / ayrık değerlendirme τ (0,10), sonradan bağlama (120 sn) ve işletme duyarlı pencere kapanışı (en çok +40 sn). **Kapsam kararı:** değişiklikler dağıtım v2 kayan ufkuna ([ADR-0023](0023-dagitim-algoritmasi-v2.md)) ve `suggest` sıralamasına uygulanır; bu ADR'deki v1 `planBatch` (20 sn pencere, regret-2) **değişmez** ve v2 kapalı kiracılar için donmuş yedek olarak kalır. Gerekçe: v2 OTOMATİK'in hedef yoludur ve 60 sn pencereyi zaten kullanır; iki yolu birlikte değiştirmek doğrulama yükünü ikiye katlar; varsayılan ONERI modunda karar `suggest` ile verildiği için κ ve bağlama v2 kapalıyken de yöneticiye ulaşır. Tümü `settings.dispatch.grouping.enabled` (varsayılan kapalı) arkasındadır; benchmark sonucu (30 tohum, eşleştirilmiş GA, ablation) uygulama görevince buraya eklenir.

## Sonuç (2026-11-06): İterasyon 3b-3 benchmark

Motor ve ölçüm: [benchmark v2 gruplama raporu](../benchmark/v2/gruplama.md) (30 tohum, iki senaryo, eşleştirilmiş %95 GA, ablation). **Gruplamanın gerçek etkisi v2 taban çizgisine (RH60-PROD, %74,4 / %60,4) göre okunmalıdır, NC'ye göre değil:** NC'ye göre +33 puanın neredeyse tamamı dağıtım v2'nin kendisinden (60 sn pencere, regret-3) gelir. Spec varsayılanlarıyla (κ 90, bağlama 120, τ 0,10; RH60-G) PROD'a göre fark **istatistiksel olarak sıfırdan ayırt edilemez**: varsayılan gün +1,4 puan [-1,1; 4,0] (%75,8, mutlak hedef %75 geçildi ama taban çizgisi zaten %74,4), tepe gün -1,2 puan [-3,3; 0,9] (%59,3; mutlak hedef karşılanmadı). Ablation (PROD'a göre, varsayılan / tepe): κ +4,8 / +5,1 puan ✱; sonradan bağlama +5,2 / +3,9 ✱; **τ = 0,10 gruplamayı azaltıyor** (-4,0 / -5,7 ✱) ve kazancı siliyor; kapanış uzatması `ALL` kipinde +1,8 / +3,0 puan ve +10 sn atama gecikmesi p95; `HISTORY` sentetik veride etkisiz (π_b < 0,3). **Önerilen değerler** (tohum 101-110'da seçildi, 1-30'da doğrulandı): `kappaS` 150, `attachSlackS` 240, `tau` 0,5 (RH60-G-T) PROD'a göre **+7,8 / +9,5 puan ✱** (%82,2 / %69,9); bedel: toplam km +%1,09 / +%1,17 ✱, geç teslim +0,28 / +0,45 puan. τ olmadan ölçüm (RH60-KA: κ 90 + bağlama 120, +5,3 / +3,5 puan ✱) farkın tamamının τ = 0,10'dan geldiğini gösterir; önerilen bileşimde τ = 0,5 bu veride hiç tetiklenmez (RH60-KA-T ≡ RH60-G-T), kazanç κ 150 ve bağlama 240'tandır. Bu değerler **gruplama açıldığında** uygulanır (SPEC §5.5'ten sapma, SPEC-3b'de belgelendi); ana anahtar kapalı başlar, ürün sahibi spec değerlerine dönebilir. Tepe gündeki mutlak %75 hedefi karşılanmadı (en iyi %71,4) ve pilot verisine bırakıldı; `groupSplitM` 5000 ayrı bir ADR adayıdır (bilgi amaçlı: +5,9 / +10,3 puan, km değişmedi). `suggest` yolunda (B-G − B) varsayılan günde +10,3 puan ✱, tepe günde km +%1,4 ve geç teslim +0,9 puan ✱ bedeli var. Hazır zamanı bilinirse/tahmin edilirse gruplama düşer ve km +%25-40 artar ([hazır zamanı üç koşul](../benchmark/README.md)); hazır zamanı dağıtımda açılmadan önce bu ayrıca incelenmelidir. v1 `planBatch` değişmedi (varsayılan `results.csv` bayt düzeyinde aynı).
