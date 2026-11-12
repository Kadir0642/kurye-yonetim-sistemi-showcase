# ADR-0035: Kurye mola durumu

- **Durum:** Kabul edildi (MVP+ 5, ürün sahibi isteği: "kurye vardiyadayken mola verebilsin; molada yeni paket atanmasın, mevcut rotası sürsün")
- **Tarih:** 2026-11-07
- **İlgili:** [ADR-0009](0009-rota-uyumlu-atama.md) (atama), [ADR-0023](0023-dagitim-algoritmasi-v2.md) (dağıtım v2), [ADR-0028](0028-havuz-gorunurlugu-ve-atama-kabulu.md) (kabul ve süre aşımı), [ADR-0029](0029-konum-kademeleri-ve-rota-geri-bildirimi.md) (konum kademeleri); sözleşme 0.6.5

## Bağlam

Kurye vardiya açıkken yemek, dua, araç bakımı gibi kısa ara verir. Bugün tek seçenek vardiyayı kapatmaktır; bu da aktif paket varken 409 verir, paketsizken konum izini ve vardiya istatistiğini böler. Menajerin ihtiyacı: kimin molada olduğunu görmek ve otomatik atamanın molada olana paket vermemesi. Kuryenin ihtiyacı: mevcut paketlerini teslim etmeye devam etmek, yeni paket gelmemesi, mola bitince sorunsuz dönmek.

## Karar

**1. Veri modeli (migration 0012, sözleşme 0.6.5).** `shifts.break_started_at` ve `shifts.break_ends_at` (ikisi de null = molada değil; CHECK birlikte dolu / boş). Her mola ayrıca `shift_breaks` satırıdır (`started_at`, `ends_at` = başlangıçtaki azami bitiş, `ended_at`, `end_reason` `COURIER | TIMEOUT | SHIFT_ENDED`); açık mola vardiya başına tek (kısmi benzersiz dizin). Kiracı ayarı `tenants.shift_break_max_min` (10-120 dk, **varsayılan 45**; `settings.shifts.breakMaxMin`, yalnızca OWNER `PATCH /tenant`, aralık dışı 400). `Shift` yanıtı isteğe bağlı `onBreak`, `breakStartedAt`, `breakEndsAt` alır (yeni sunucu her zaman gönderir; eski istemci yokluğu "molada değil" sayar). Vardiya kapanınca açık mola `SHIFT_ENDED` ile kapanır ve yeni vardiya molada başlamaz.

**2. Uçlar.** `POST /courier/shifts/break/start` ve `POST /courier/shifts/break/end` (yalnızca COURIER, kiracı + kullanıcıya bağlı). Mevcut vardiya uçlarıyla tutarlı olsun diye yol `shifts/` altındadır (`shift/` değil). **Doğal olarak idempotent:** moladayken başlat ve molada değilken bitir değişiklik yapmadan güncel vardiyayı 200 döner; olay yayınlanmaz. Açık vardiya yoksa 409 `SHIFT_NOT_OPEN`. `Idempotency-Key` gerekmez (çevrimdışı kuyruğa alınmaz: mola çevrimiçi eylemdir, bağlantı yokken düğme kapalıdır). Panel için `GET /couriers/{courierId}/breaks?date=` (OWNER, DISPATCHER; bir Europe/Istanbul gününde başlayan molalar, başka kiracı 404).

**3. Azami süre.** `ShiftBreakJobsService` her 10 sn `breakEndsAt` geçmiş molaları bitirir (Redis `SET NX` iş kilidi; çok örnekte tek çalıştırır; yeniden başlatmaya dayanıklı çünkü son tarih veritabanındadır). Bitiş `LEAST(ends_at, now())` ile kaydedilir, neden `TIMEOUT`, WS `shift.updated` `by: "SYSTEM"`.

**4. Dağıtım.** Molada kurye **aday değildir**; tek nokta `SuggestionsService.prefilter` (öneri, toplu plan, OTOMATİK atama ve kabul süresi aşımı sonrası yeniden dağıtım hepsi buradan geçer). Molada kurye `InfeasibleReason` `ON_BREAK` ("Molada") ile elenenlere yazılır. Dağıtım v2 gölge ALNS'nin filo listesi (`shiftCourierIds`) etkilenmez (rota planı sürer). Havuzdan `claim` molada 422 `COURIER_NOT_AVAILABLE`. **Yönetici elle atayabilir** (panel "Molada" uyarısı gösterir), **bekleyen kabul kartı molada da kabul / ret edilebilir**: mola hiçbir mevcut işi bloklamaz, yalnızca yeni aday olmayı keser. Mevcut rota yeniden planlama ve devir akışları değişmez.

**5. Olaylar.** Panel: `courier.shift` `action` değerlerine `BREAK_STARTED` ve `BREAK_ENDED` (`reason: COURIER | TIMEOUT`) eklendi; `data.shift` güncel mola alanlarını taşır. Kurye: `shift.updated` (mola başladı / bitti). Konum olaylarına dokunulmadı.

**6. Kurye uygulaması (maket A2).** Vardiya ekranında eldiven boyu (64 dp) "Mola ver" düğmesi; molada kalan süre sayacı (sunucu saatine göre) ve "Moladan dön"; rota ekranında "Moladasın — yeni paket atanmaz" şeridi; süre dolunca "Mola süreniz doldu" bildirimi. **Konum kademeleri değişmez** (PR #49): mola konum görevini durdurmaz, kademe seçimi (duran kurye 30 sn GPS / 60 sn kalp atışı) olduğu gibi çalışır; kayıtlı konum yalnızca sunucuda aday sayılmaz.

**7. Panel (maket P2).** Kurye listesinde ve canlı harita satırında "Molada · 12 dk" çipi (duraklat ikonu + etiket + geçen dakika); canlı harita işaretçisi kesikli çerçeve ve duraklat rozetiyle "duraklatılmış" stilinde (bayat uyarısından ayrı); seçili kuryenin bugünkü mola geçmişi (olaylardan tazelenir); atama penceresinde molada kurye için uyarı; Ayarlar > Genel'de "Azami mola süresi (dk)" (10-120). Sahte API'de `molada` senaryosu.

## Sonuçlar

- Menajer molayı görür; OTOMATİK atama molada olana paket vermez. Mola yeni bir vardiya durumu gerektirmez: vardiya açık kalır, istatistik ve konum izi bölünmez.
- Son tarih veritabanında olduğundan yeniden başlatma molayı kaybettirmez; en kötü gecikme iş aralığı (10 sn).
- Devir (kuryeler arası) **aday listesi** moladaki kuryeyi dışlar (takip paketi). Bilinen sınır: yöneticinin doğrudan hedef seçmesi (`evaluateTarget`, yol B) molayı engel saymaz. Dağıtım v2 pencere kapanışı uzatma kararı (`closeExtension`) da molada kuryeyi "işletmenin alınmamış alışı olan kurye" saymaz; gölge ALNS filosu molada kuryeyi içermeye devam eder (rotası sürer).
- `shift_breaks` mola raporlaması (kurye başına süre, sıklık) için temel olur; ayrı rapor bu kararın dışındadır.

## Alternatifler

- **Vardiyayı kapat / aç:** aktif paket varken mümkün değil ve konum izini böler; reddedildi.
- **Molayı yalnızca istemci durumu yapmak:** sunucu dağıtımı molayı bilmezdi; reddedildi.
- **Ayrı `courier_status` tablosu / numaralandırması:** tek bir "mola" durumu için gereğinden ağır; `shifts` üzerinde iki sütun yeterli.
- **`/courier/shift/break/*` yolu:** mevcut uçlar `shifts/` altında; tutarlılık için `shifts/break/*`.
