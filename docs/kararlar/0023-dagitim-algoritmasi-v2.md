# ADR-0023: Dağıtım algoritması v2: kayan ufuklu toplu atama + sınırlı kuryeler arası ALNS (bayrak arkasında); CBBA yalnızca araştırma taban çizgisi

- **Durum:** Kabul edildi (Faz 2: bayrak arkasında). Faz 1 (araştırma + benchmark) tamamlandı; Faz 2 kayan ufku `DISPATCH_V2` arkasında üretime aldı, ALNS yalnızca gölge kipte (`DISPATCH_V2_ALNS_SHADOW`). Gerçek ALNS taşımaları (`DISPATCH_V2_ALNS`) pilot gölge verisine göre ayrı kararla açılır.
- **Tarih:** 2026-11-06
- **İlgili:** [ADR-0009](0009-rota-uyumlu-atama.md), [ADR-0013](0013-rota-motoru-siniri-ve-yaklasik-mod.md), [ADR-0014](0014-toplu-atama-regret-ekleme.md), [ADR-0017](0017-saha-verisinden-ogrenen-harita.md); tasarım: DAGITIM-ALGORITMASI-V2.md; sonuçlar: benchmark v2

## Bağlam

- Bugünkü motor (ADR-0009, ADR-0014) en yakın kuryeye göre km'yi %30, geç teslimi %19,9'dan %3,2'ye indirdi; ama atanmış, alınmamış paket hiçbir zaman yer değiştirmez (ADR-0014 madde 2) ve 20 sn pencere aynı işletme kümelerini zayıf topluyor (%59,7).
- Ürün sahibi "Dağıtım algoritması v2" işini onayladı ve çok-robotlu görev dağıtımı (MRTA) literatüründen CBBA ile karşılaştırma istedi (araştırma ve portföy değeri).
- Faz 1'de üç yaklaşım aynı simülatör, tohum ve OSRM matrisiyle, üretimdeki sert kısıtlarla (kapasite, tazelik, alış ETA, 8 dk gecikme sınırı) uygulandı ve ölçüldü: kayan ufuklu toplu atama (RH), kuryeler arası ALNS, CBBA. Parametreler ayrı ayar tohumlarında (11-20) seçildi; sonuçlar tohum 1-10, varsayılan ve tepe ağırlıklı iki senaryo.

## Karar

**1. Üretime alınacak (Faz 2, onaydan sonra): kayan ufuklu toplu atama**, `DISPATCH_V2` bayrağı arkasında, OTOMATİK modda bugünkü 20 sn toplu pencerenin yerine.
- Pencere W = 60 sn (kiracı ayarı, 30-60), regret-3 ekleme, aynı işletme birimleri (ADR-0014 madde 3 aynen).
- Gecikmeli atama: en iyi seçeneğin paket başı sapması 7 dk'yı aşıyor ve sipariş 3 dk'dan gençse sipariş bir sonraki pencereye kalır. 15 dk kuralı ve elle atama değişmez.
- Gerekçe (CUR'a göre, eşleştirilmiş): teslimat süresi −1,2 / −1,6 dk, paket sapması −1,0 / −1,1 dk, aynı işletme oranı +12 puan (her ikisi de anlamlı), tepe gününde geç teslim −0,7 puan; km değişmez. Hesaplama p95 ~2 ms.

**2. Üretime alınacak, ayrı bayrakla ve kademeli: sınırlı kuryeler arası ALNS** (`DISPATCH_V2_ALNS`).
- Yalnızca atanmış ve alınmamış paketler; alınmış paket, kilitli durak ve kuryesi alışa 5 dk'dan yakın paket asla taşınmaz; paket başına en çok 1 taşıma; taşıma en az 150 sn amaç kazancı getirmeli.
- Yıkım: rastgele, en kötü, Shaw, rota; onarım: açgözlü, regret-2/3; uyarlanır ağırlık + benzetimli tavlama; 250 yineleme ve 300 ms süre sınırı; her pencere kapanışında ve en çok 60 sn'de bir.
- Önce **gölge kip** (hesapla ve kaydet, uygulama), sonra yalnızca yoğunlukta açık. Gerekçe: tepe gününde km −%3,9, geç teslim −0,9 puan, teslimat −1,5 dk (anlamlı); varsayılan günde km kazancı anlamlı değil; bedeli 100 pakette 3-5 yeniden atama bildirimi.
- ADR-0014 madde 2'nin "mevcut paketler kuryeler arasında yeniden dağıtılmaz" kuralı bu bayrak açıkken yukarıdaki sınırlarla gevşer; bayrak kapalıyken aynen geçerlidir.

**3. CBBA üretime alınmaz.** Benchmark'ta taban çizgisi ve araştırma çıktısı olarak kalır. Tam bağlı CBBA km'de eşit, gruplamada −17/−19 puan geride; teklif bükme kapatılınca gruplama kısmen düzelir ama kararların %4-8'i yakınsamaz ve ileti 5-8 kat artar; sınırlı menzilde günde ~230-250 çift atama ancak merkezi kayıtla çözülür. Bilgi ve iletişim zaten merkezde olduğundan merkeziyetsiz uzlaşmanın bu alanda getirisi yoktur.

**4. Mimari:** v2 algoritmaları `services/api/src/routing/v2/` altında saf modüllerdir (ADR-0013 madde 1 ilkeleri). Ortak `Fleet` modeli uygulanabilirliği üretimdeki `cheapestInsertion` / `scan` ile denetler; tek global amaç: rota süresi + 0,25 · teslim varışları + taşıma cezası. Faz 2'de motor, `planBatch`'teki gibi bir `TravelContext` kurup `planWindow` ve `runAlns`'i çağırır; sonuç (kurye başına dizi + taşıma listesi) mevcut atama ve rota yazım yolundan yazılır.

## Faz 2 uygulaması (bayrak arkasında)

- **Bayraklar:** `DISPATCH_V2` (ortam, varsayılan kapalı) **ve** kiracı ayarı `settings.dispatch.v2Enabled` (varsayılan kapalı) birlikte açıkken etkin; ayrıca `DISPATCH_V2_ALNS_SHADOW` (ortam, varsayılan kapalı). Kiracı ayarları: pencere 30-60 sn (60), sapma eşiği 7 dk, en uzun bekletme 3 dk ([MIMARI §6.1.1](../MIMARI.md#611-dağıtım-v2-ayarları-adr-0023-faz-2)).
- **Motor:** `RoutingEngine.planDispatch` (sözleşmede isteğe bağlı yöntem) gerçek motorda ince matris blokları (gölge ALNS için tek kare `table`) kurar ve saf `planDispatchWindow`'u çağırır (`services/api/src/routing/v2/dispatch.ts`: Fleet + planWindow + runAlns). Sahte motor aynı saf işlevi Haversine matrisle çağırır. `npm run benchmark:v2` içindeki `RH60-PROD` varyantı kararları bu motor yöntemiyle verir; tüm tohum ve ölçütlerde Faz 1 `RH60` ile birebir aynı sonucu üretti (kopya değil, aynı kod yolu).
- **Uygulama (A2):** OTOMATİK modda yeni paket ve bekleyenlerin yeniden denemesi Redis pencere listesine girer; pencere kapanınca (zamanlayıcı + 5 sn süpürücü, çok örnekli kilit) önceki pencerelerde bekletilenlerle birlikte planlanır. Atamalar bugünkü yoldan (`packages.assign`, kilit altında kapasite / vardiya yeniden denetimi, `ASSIGNED_AUTO`, `batch`, `groupId`) yazılır; bekletilenler `WINDOW_HOLD` ("Pencere bekliyor"), uygulanabilir kurye yoksa `NO_FEASIBLE`. Bekletme süresi dolunca en iyi uygun kurye seçilir (bugünkü 5 dk sapma eşiği v2 penceresinde uygulanmaz; yerini gecikmeli atama alır, benchmark'taki gibi). Ret ve iade, ONERI / AKILLI_HAVUZ önerileri ve yaklaşık mod bugünkü gibi; toplu öneri v2 planlayıcısıyla (bekletmesiz).
- **Gölge ALNS:** her pencere kapanışında (pencerede yeni atananlar cezasız) ve dönemsel işte en çok 60 sn'de bir, vardiyadaki tüm konumlu kuryeler üzerinde; öneriler ve tahmini kazanç `dispatch_shadow_runs` / `dispatch_shadow_moves`'a yazılır, hiçbir atama değişmez. ADR-0014 madde 2 bu fazda aynen geçerlidir.
- **Gözlem:** `dispatch_windows` (pencere başına aday, atanan, bekletilen, kuyruğa düşen, süre; paket başına sonuç), yapılandırılmış günlük (`dispatch.v2.window`, `dispatch.v2.shadow`), `GET /metrics/dispatch-v2` (sayaçlar, sipariş → atama gecikmesi, karar süresi p50 / p95, gölge öneri ve kazanç).
- **Ölçülen süreler (yerel OSRM, 25 kurye):** öneri motoru p95 ~20-25 ms, uçtan uca öneri p95 125 ms (bütçe 300 ms); v2 pencere kararı (20 paket) motor p95 ~58-65 ms, API içinde (aday süzme ve anlık görüntü dahil) p95 ~256 ms; gölge ALNS (75 atanmış paket, kare matris dahil) p95 ~225-268 ms.

## Sonuçlar

- (+) Ölçülmüş, istatistiksel olarak anlamlı iyileşme: daha kısa teslimat, daha az sapma, daha iyi aynı işletme gruplaması; yoğunlukta daha az km ve geç teslim.
- (+) Aynı kısıt kodu ve açıklanabilir karar ailesi (regret); yeni dış bağımlılık, ücretli servis veya altyapı yok.
- (+) Kademeli açılış: iki ayrı bayrak, ALNS için gölge kip; geri dönüş bayrağı kapatmak kadar kolay.
- (+) Portföy: MRTA (SSI / CBBA) ile VRP (ALNS) arasında aynı simülasyonda ölçülmüş karşılaştırma.
- (−) Gecikmeli atama ve 60 sn pencere, kuryenin yeni paketi görmesini ortalama yarım dakika ile birkaç dakika geciktirir (hazırlık süresi içinde kalması beklenir; pilotta ölçülecek).
- (−) ALNS yeniden atamaları kurye güvenini etkileyebilir; bildirim metni, yönetici görünürlüğü ve günlük taşıma sınırı Faz 2'de tasarlanmalı.
- (−) v2 amacında adalet terimi yok; kurye başına paket dağılımı ayrıca izlenmeli.
- (−) Shaw yıkımı gruplamayı artırmadı (ALNS gruplamayı biraz düşürdü); gruplama hedefi için amaçta açık bir terim gerekir (İterasyon 3b-3).
- (−) Sonuçlar sentetik senaryoya dayanır (10 tohum, hazırlık anı bilinmiyor, sürücü davranışı yok); pilotta gölge kip ve karşılaştırma gerekir.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Bugünkü motorda kalmak (20 sn pencere, regret-2, eşik kuyruğu) | Ölçülen v2 kazançlarını bırakır; kayan ufuk aynı karmaşıklıkta |
| Pencerede yinelemeli Macar eşleştirmesi | Regret-3'ten anlamlı farkı yok; kurye başına çoklu atama ve aynı işletme birimlerini doğal taşımaz |
| ALNS'yi her zaman ve daha gevşek sınırlarla açmak | Yineleme 4 katına çıkınca sonuç değişmedi; taşıma eşiği düşünce yeniden atama iki katına çıktı, kazanç artmadı |
| CBBA (merkeziyetsiz açık artırma) | Bilgi ve iletişim zaten merkezde; gruplama ve geç teslimde geride; yakınsama için teklif bükme sinerjiyi siler; sınırlı menzilde çift atama |
| CBBA ile görevleri her pencerede serbest bırakmak | Yoğunlukta km kazancı ALNS'ye yakın ama 3,5 kat yeniden atama, daha kötü geç teslim ve gruplama |
| Değer fonksiyonu / pekiştirmeli öğrenme (Didi, Meituan tarzı) | Veri gerektirir; pilot verisi birikince gelecek iş |

## Not (2026-11-06): gruplama

İterasyon 3b-3 gruplama sonucu (κ, τ, sonradan bağlama, işletme duyarlı kapanış; v2 yolunda bayraklı): bkz. [ADR-0014 sonuç notu](0014-toplu-atama-regret-ekleme.md) ve benchmark.
