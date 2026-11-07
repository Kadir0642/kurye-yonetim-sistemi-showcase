# ADR-0033: Yük testi darboğazlarının giderilmesi (panel veri çekme, kuyruk yeniden deneme aralığı, konum yazımı)

- **Durum:** Kabul edildi
- **Tarih:** 2026-11-07
- **İlgili:** Yük testi raporu (§5.1, §5.2, §5.4), [ADR-0009](0009-rota-uyumlu-atama.md) / [ADR-0023](0023-dagitim-algoritmasi-v2.md) (dağıtım pencereleri), [ADR-0029](0029-konum-kademeleri-ve-rota-geri-bildirimi.md) (konum kademeleri)

## Bağlam

25 kurye + 10 panel yük testi üç darboğaz gösterdi: (1) panelin liste yenilemeleri isteklerin %67-71'i; (2) otomatik atamanın eşiğe takılıp bekleyenlere düşen paketleri koda gömülü 60 sn'lik yeniden deneme işini bekliyor (30 dk koşuda 92 paketin 17'sini yönetici elle atadı); (3) `POST /courier/locations` p99 sakin makinede bile 0,7 sn.

## Karar

**1. Panel: olay güdümlü veri çekme.** `package.updated` artık "beş listeyi yeniden çek" değildir:

| Olay `change` | Yerinde güncelleme | Yeniden çekilen sorgular |
|---|---|---|
| `CREATED` | - | paket listeleri, harita paketleri, özet |
| `STATUS_CHANGED` | önbellekteki listeler ve kuyruk | + canlı kurye (`activePackageIds`) |
| `EDITED`, `READY_CHANGED`, `ACCEPT_CHANGED` | önbellekteki listeler ve kuyruk | hiçbiri (üyelik ve sayılar değişmez) |
| bilinmeyen | - | hepsi (eski davranış) |

Bekleyenler kuyruğu zaten `dispatch.queue` olaylarıyla (ADDED / UPDATED / REMOVED / OVERDUE) yerinde güncelleniyordu; paket olayında ayrıca yeniden çekilmez. Geçersiz kılma 1 sn debounce ile (en geç 4 sn sonra) ve kapsamların birleşimiyle çalışır; sekme gizliyken bekletilir, görünür olunca uygulanır. Yoklama yalnızca güvencedir: liste / kuyruk / canlı harita / rota / devir **120 sn** (eski 30-60 sn), rota motoru durumu ve menü rozeti **300 sn**, seçili kuryenin izi 30 sn; WebSocket kopukken en çok 30 sn'ye iner; sekme gizliyken durur; 30 sn'den uzun gizli kalan sekme görünür olunca anlık görüntüler bir turda tazelenir. `courier.position` hiçbir liste yenilemesini tetiklemez (zaten tetiklemiyordu).

*Reddedilen:* olay yükünden bütün listeleri yerinde kurmak (filtreli / sayfalı liste üyeliği istemcide doğru hesaplanamaz); sunucu önbelleği / ETag (sonraki adım, gerekirse).

**2. Bekleyenler kuyruğu yeniden deneme aralığı kiracı ayarıdır.** `settings.dispatch.retryIntervalS` (tam sayı 15-120 sn, varsayılan **60**: bugünkü davranış korunur; pilot önerisi **30**). Sütun `tenants.dispatch_retry_interval_s` (CHECK 15-120), migrasyon 0010, sözleşme 0.6.3 (yalnızca ekleme). Alan dışı ya da tam sayı olmayan değer 400 `VALIDATION_FAILED`; yalnızca OWNER. İş artık 5 sn'de bir tikler; kiracı başına son deneme zamanı Redis'te tutulur (`t:<kiracı>:dispatch:retry-at`), böylece yeniden başlatmaya ve çok örnekli çalışmaya dayanıklıdır.

**3. Kapasite açılınca erken yeniden deneme (aynı ayarın arkasında).** Kuyrukta paket varken, son denemeden beri bir paket DELIVERED / FAILED / CANCELLED olduysa ya da bir kurye vardiyaya girdiyse, yeniden deneme aralığı dolmasa da en erken **10 sn** sonra çalışır (`RETRY_EARLY_MIN_GAP_MS`). Sorgu yalnızca kuyruk doluyken ve 10 sn eşiği geçince çalışır (5 sn'lik tikte iki küçük `EXISTS`). Olay kancası yerine "son denemeden beri" sorgusu seçildi: paket servisine (başka sahip) kanca eklemek gerekmez, kaçan olay olamaz, çok örnekte çalışır. Etkisiz durum: aralık 15 sn iken de aynı kurallar geçerlidir.

*Atanmayan paketlerin asıl nedeni ayar değil geometridir:* kuyruğa düşenlerin nedeni `OVER_THRESHOLD` (en iyi kuryenin sapması 5 dk eşiğini aşıyor). Yeniden deneme yalnızca durum değiştiğinde fark yaratır (kurye boşalır, konum değişir); 20 sn'lik toplu pencere (`batchWindowS`) ve eşik (`detourThresholdMin`) pilot yoğunluğuna göre ayrı ayarlanır (rapor §5.2).

**4. Konum yazımı: `synchronous_commit = off`.** `POST /courier/locations` aşama aşama zamanlandı (50 kurye, navigasyon karışımı, 955 istek): `findOpenShift`, Redis (`ZADD`, Lua `SET_IF_NEWER`), olay yayını, `replanner.onLocation` ve varış / devir dinleyicileri 1-5 ms; 100 ms'yi aşan 127 istekte gecikmenin neredeyse tamamı `INSERT INTO location_pings` idi (en uzun 2,5 sn). Neden işlem onayındaki WAL fsync beklemesidir: aynı pencerede sırayla ölçülen karşılaştırmada (`infra/loadtest/bench-location-insert.mjs`, 90 sn, ~20 ekleme/sn) `on`: p50 5,0 / p95 44,6 / p99 93,0 / max 254 ms, `off`: p50 3,6 / p95 5,2 / p99 13,6 / max 37 ms. Konum satırı eklemesi `SET LOCAL synchronous_commit = off` ile yapılır. **Bedeli:** sunucu veya veritabanı çökerse son ~0,6 sn'lik (3 x `wal_writer_delay`) onaylı nokta kaybolabilir; veri bozulmaz ve geri kalan işlemler (paket, atama, vardiya) etkilenmez. İz çizgisinde küçük boşluk dışında etkisi yoktur (güncel konum Redis'te ayrıca tutulur, kurye zaten 15 sn'de bir gönderir).

## Sonuçlar

- Panel istek yükü ve sunucu iş yükü azalır; operatörün gördüğü canlılık değişmez (yerinde güncelleme, kuyruk ve harita olayları). Olay kaçarsa en geç 120 sn (WS kopukken 30 sn) içinde düzelir.
- Pilot kuyruk bekleme süresi ayarla kısalır; ürün sahibinin pilot tablosundaki değer 30 sn'dir. Panel "Pilot ayarlarını uygula" düğmesi 30 sn'yi de ayarlar.
- Konum p99 kuyruk ucu veritabanı disk eşzamanlılığından bağımsızlaşır; çökme penceresi belgelenmiştir.
- Yük testi artık navigasyon / duran konum profillerini (`--profile nav`), eski ve yeni panel modelini (`--panel-model old|new`) ve `POST /courier/locations` p99 uyarısını (varsayılan eşik 500 ms) içerir.

## Alternatifler

- Yeniden deneme aralığını yalnızca 60 → 30 sn sabit değiştirmek: kiracıya göre ayarlanamaz, ürün kararı kodda kalırdı.
- Konum ekleme için toplu (batch) yazıcı / kuyruk: çökmede kayıp penceresi daha uzun, karmaşık; `synchronous_commit` aynı kazancı tek satırla verir.
- `synchronous_commit = off`'u tüm bağlantılara uygulamak: paket / atama yazımlarında kayıp kabul edilemez.
