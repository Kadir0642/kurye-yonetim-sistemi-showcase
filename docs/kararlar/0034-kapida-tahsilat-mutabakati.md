# ADR-0034: Kapıda tahsilat kaydı ve kasa mutabakatı

- **Durum:** Kabul edildi (ürün sahibi, MVP+ madde 6: "kurye nakit / kart tahsil ettiğini işaretler, yönetici kurye başına günlük nakdi görür ve kasa teslimini onaylar")
- **Tarih:** 2026-11-07
- **İlgili:** [ADR-0020](0020-cevrimdisi-eylem-kuyrugu-ve-idempotency.md) (çevrimdışı kuyruk, Idempotency-Key), [ADR-0026](0026-teslim-geofence-politikasi.md) (teslim geofence), [ADR-0032](0032-musteri-verisi-saklama.md) (anonimleştirme tutarlara dokunmaz); SPEC-4c; sözleşme 0.6.4; migration `0011_tahsilat`

## Bağlam

Paketler `paymentType` (CASH / CARD / PAID_ONLINE) ve `amountKurus` taşıyordu; ancak paranın gerçekten alınıp alınmadığı ve yönetime teslim edilip edilmediği hiçbir yerde kayıtlı değildi. Kurye firması kapıda nakit toplayan kuryelerle her gün "kasa" hesaplaşmasını kâğıt ve hafızayla yapıyordu. Kısıtlar: teslim eylemi çevrimdışı kuyruğa girer (aynı `Idempotency-Key` ile tekrar gönderilir), eldivenli kullanım (büyük hedef), ücretli hizmet / ödeme sağlayıcı yok (SoftPOS kapsam dışı), tutar kişisel veri değildir ama serbest not olabilir.

## Karar

**1. Tahsilat teslim eyleminin parçasıdır.** `POST /courier/packages/{id}/deliver` gövdesine isteğe bağlı `collection: { method: CASH | CARD | NONE, amountKurus, note? }` eklenir. Ayrı bir "tahsil ettim" çağrısı yoktur; böylece çevrimdışı kuyrukta teslimle **aynı gövde ve aynı Idempotency-Key** ile gider, kısmen uygulanma ve sıralama sorunu olmaz. Geofence gerekçesiyle yeniden gönderimde (yeni anahtar) gövde aynen korunur. Alan gönderilmezse (eski istemci, panelden `mark`) kayıt NULL kalır = "kayıt yok".

**2. Veri modeli (migration 0011, yalnızca ekleme).** `packages` += `collected_method` (CASH / CARD / NONE), `collected_amount_kurus`, `collected_at` (teslim olay zamanı), `collected_note` (en çok 200 karakter) + tutarlılık CHECK'i (hepsi NULL ya da yöntem + tutar + zaman dolu; NONE ise tutar 0). Yeni tablo `cash_handovers` (kiracı, kurye, `business_date` Europe/Istanbul günü, `amount_kurus`, not, kaydeden kullanıcı, zaman) **yalnızca eklenir**: veritabanı tetikleyicisi UPDATE / DELETE'i reddeder. Düzeltme, negatif tutarlı ve **notlu** yeni satırdır (CHECK). Tutar 0 olamaz. Gün dizini `packages_delivered_day_idx (tenant_id, delivered_at) WHERE status = 'DELIVERED'`.

**3. Teslim tahsilat yüzünden asla reddedilmez.** Sunucu kaydı paketin ödeme bilgisine göre düzeltir (`normalizeCollection`, birim testli): önceden ödenmiş / tutarı 0 pakette `NONE / 0`; `NONE` tutarı 0; `CASH` / `CARD` tutarı `[0, paket tutarı]` aralığına kırpılır, 0 kalırsa `NONE`. Düzeltme olayın `data.collection.adjusted` alanına yazılır. Gerekçe: çevrimdışı kuyrukta 400 / 422 "kalıcı hata" sayılıp teslimi çöpe atar; paranın kaydı teslim kaydından daha az önemlidir. Yalnızca biçim hatası (bilinmeyen yöntem, negatif / tamsayı olmayan tutar) 400 verir. İkinci (farklı anahtarlı) teslim isteği tahsilatı değiştirmez: ilk yazım kazanır.

**4. Kurye uygulaması.** Kapıda ödemeli (CASH / CARD) ve tutarı > 0 paketlerde teslim akışına tek ek adım: büyük tutar, paketin ödeme tipine uyan birinci düğme ("Nakit aldım" / "Kartla aldım"), ikinci yöntem ve "Alamadım". "Tutarı değiştir" kısmi tahsilat içindir; eksik tutarda ve "Alamadım"da hazır sebep seçmek (ya da yazmak) zorunludur. Önceden ödenmiş pakette adım yoktur. Adım rota kartı, paket ayrıntısı ve navigasyon varış kartından aynı `runDeliverFlow` içinde sorulur (geofence adımından önce); vazgeçilirse hiçbir şey kuyruğa yazılmaz. Profilde "Bugün üzerimde" kartı (`GET /courier/payments/today`; çevrimdışıyken son değer).

**5. API (hepsi kiracıya bağlı, `x-roles`).** `GET /payments/daily?date=&courierId=` (OWNER / DISPATCHER): kurye başına `packageCount`, `expectedCashKurus`, `collectedCashKurus`, `collectedCardKurus`, `uncollectedCount` / `uncollectedKurus` (kaydı olup eksik), `unrecordedCount` (kaydı olmayan teslim), `handedOverKurus`, `outstandingKurus = tahsil edilen nakit - teslim edilen kasa`, kasa kayıtları. `POST /payments/handover` (OWNER / DISPATCHER) 201; aynı kullanıcı + kurye + gün + tutar 10 sn içinde tekrarlanırsa yeni satır açmaz (çift tıklama; kurye başına advisory kilit). `GET /courier/payments/today` (COURIER, yalnızca kendi verisi). Başka kiracının kuryesi 404. Gün, paketin `delivered_at` anına göre Europe/Istanbul takvim günüdür; gelecek güne kasa teslimi 400.

**6. Panel.** Menüde **Tahsilat** (OWNER / DISPATCHER): gün seçici (varsayılan bugün, URL `?gun=`), özet kartları, kurye tablosu (beklenen nakit, tahsil edilen, kart, alınamayan [+ kayıtsız], teslim edilen kasa, **kuryede kalan** rozeti: Tamam / Kuryede nakit var / Fazla teslim), satırda **Kasa teslim al** (tutar kuryede kalanla ön dolu, not, eksi tutar notlu düzeltme), kasa kayıtları listesi ve istemci tarafı CSV (UTF-8 BOM, `;`, formül kaçışlı). Kuryeler listesinde "Bugün nakit" sütunu; paket ayrıntısında tahsilat satırı. Metrikler sayfasına dokunulmaz.

**7. Anonimleştirme ([ADR-0032](0032-musteri-verisi-saklama.md)).** Yöntem, tutar ve zaman kişisel veri değildir ve **kalır**; günlük toplamlar anonimleştirmeden önce ve sonra aynıdır (e2e). Serbest `collected_note` müşteri bilgisi içerebileceğinden diğer serbest metinler gibi `NULL` yapılır. Not olay verisine yazılmaz (olayda yalnızca yöntem + tutar + `adjusted`).

## Sonuçlar

- (+) Nakit akışı uçtan uca izlenebilir; çevrimdışı teslimde para kaydı kaybolmaz ve çift sayılmaz (ADR-0020 anahtarı).
- (+) Yeni ücretli servis yok; CSV tarayıcıda üretilir.
- (−) Gün sınırı teslim anına göredir: gece yarısından sonra senkronlanan çevrimdışı teslim, cihaz olay zamanına (`occurredAt`, sunucu sınırlı) göre doğru güne yazılır; ama teslim edildiği gün kapatılmış bir kasa teslimi sonradan değişir (kasa kayıtları yalnızca eklenir, düzeltme yeni satırdır).
- (−) Tahsilat teslimden sonra düzeltilemez (kurye "Nakit aldım" yerine "Alamadım" demişse): fark `kuryede kalan` ve eksi tutarlı kasa düzeltmesiyle kapanır. Paket başına tahsilat düzeltme ekranı ileri iş.
- (−) Önceki günlerin açık bakiyesi (devreden) ayrı gösterilmez; kasa günü kuryenin o gün aldığı nakitle eşleşir. Yönetici geçmiş güne kasa teslimi yazabilir.
- (−) Kart "alındı" beyanı kuryeye dayanır; bankayla otomatik doğrulama yoktur (SoftPOS ayrı iş).
- Devir: nakit sorumluluğu devirle yeni kuryeye geçer ([ADR-0010](0010-kuryeler-arasi-devir.md)); tahsilat teslimi yapan kuryeye yazılır, o kuryenin kasasında görünür.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Ayrı `POST .../collect` çağrısı | Çevrimdışı kuyrukta iki ayrı eylem: sıra ve kısmi uygulanma riski; tahsilat ile teslim ayrışabilir |
| Geçersiz tahsilatı 400 ile reddetmek | Kuyruktaki teslim kalıcı hata sayılıp kaybolur; kırpma + `adjusted` işareti daha güvenli |
| `cash_handovers` kaydını düzenlenebilir / silinebilir yapmak | Mutabakat kaydı denetim izidir; düzeltme yeni (eksi) satır |
| Gün toplamını kapatıp kilitlemek ("kasa kapandı") | Pilotta gereksiz süreç; yalnızca-ekle kayıt aynı güvenceyi daha az sürtünmeyle verir |
| SoftPOS / ödeme sağlayıcı entegrasyonu | Ücretli servis kuralı; kapsam dışı |

## Uygulama notları

- **Kod:** `services/api/src/payments/` (`collection.ts` + `collection.spec.ts`, `payments.service.ts`, `payments.controller.ts`); `packages.service.ts` `courierAction` (DELIVER); şema `payments.ts`, migration `0011_tahsilat.sql`; mobil `lib/collection.ts`, `components/delivery/CollectionSheet.tsx`, `state/deliverFlowCore.ts`, `components/TodayCashCard.tsx`; panel `pages/PaymentsPage.tsx`, `lib/payments.ts`.
- **Test:** `services/api/test/tahsilat.e2e-spec.ts` (kayıt + Idempotency tekrarı, düzeltmeler, iki kurye toplamı, kasa teslimi + çift tıklama + yalnızca-ekle, roller, kiracılar arası 404, anonimleştirme), mobil `collection.test.tsx`, panel `payments.test.tsx` / `payments.test.ts`.
- **Migration numarası:** 0010 `LOADFIX` PR'ına aittir; bu iş 0011 kullanır. İki dal birleşince `drizzle/meta/_journal.json` ve `0011_snapshot.json` (`prevId`) birleştirme sırasına göre yeniden üretilmelidir.
