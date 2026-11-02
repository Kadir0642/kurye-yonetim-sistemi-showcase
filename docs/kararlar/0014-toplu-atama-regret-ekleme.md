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
