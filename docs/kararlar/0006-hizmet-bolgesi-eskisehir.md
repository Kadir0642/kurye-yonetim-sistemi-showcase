# ADR-0006: Hizmet bölgesi (Eskişehir pilotu) ve çok şehre hazır tasarım

- **Durum:** Kabul edildi
- **Tarih:** 2026-10-30

- **Güncelleme (2026-11-03):** d) adres defteri, telefon yerine adres anahtarıyla ve saha verisinden öğrenilen teslim noktalarıyla [ADR-0017](0017-saha-verisinden-ogrenen-harita.md) içinde yeniden tanımlandı; Google ToS riski bu yoldan kalkar. Harita, geocoder ve rota çıkarımı [ADR-0016](0016-eskisehire-ozel-kendi-barindirdigimiz-harita-platformu.md) ile tek OSM çıkarımından beslenir. 30 adreslik kalite testi İterasyon 3'tedir (PLAN 3a-8), Google'sız.

## Bağlam

Ürün sahibinden gelen bilgiye göre pilot müşteri yalnızca **Eskişehir** içinde faaliyet gösteriyor. Çekirdek bölge **Odunpazarı ve Tepebaşı** ilçeleridir; çevre ilçeler (ör. Çifteler, Mihalgazi, Sivrihisar) isteğe bağlı olarak kapsama eklenebilir. Ürün başka illerde satılırsa yeni şehirler sonradan eklenecektir.

Bu bilgi harita mimarisini ([ADR-0001](0001-hibrit-harita-mimarisi.md)) doğrudan etkiler: tüm Türkiye yerine tek il için OSRM / VROOM çalıştırmak çok daha küçük bir sunucu ve daha ucuz Google kullanımı demektir. Eskişehir bir öğrenci şehri olduğundan yurt, kampüs ve ofis adreslerinde tekrarlayan müşteri oranı yüksek olabilir.

## Karar

**Hizmet bölgesi veridir, kod değildir.** Her kiracının (tenant) veritabanında bir `service_area` kaydı bulunur: poligon veya sınırlayıcı kutu (bbox), şehir adı, varsayılan harita merkezi ve yakınlaştırma düzeyi. Kodda Eskişehir'e özgü sabit değer (koordinat, il adı, bbox) bulunmaz; pilot yalnızca ilk kiracının `service_area` verisidir.

Etkiler:

**a) OSRM / VROOM için yalnızca Eskişehir OSM çıkarımı.** Geofabrik Türkiye PBF dosyasından, il sınırı poligonu veya bbox ile `osmium` kullanılarak Eskişehir çıkarımı üretilir. Böylece bellek ihtiyacı küçük kalır (yaklaşık 2 GB RAM'li bir VPS yeterli beklenir; ölçülecek) ve ön işleme dakikalar sürer. Şehir eklemek için birleştirilmiş poligonlarla yeniden çıkarım yapılır; şehir sayısı artarsa tüm Türkiye çıkarımına geçilir (tahmini 4-8 GB RAM).

**b) Google Places Autocomplete (New) kısıtlaması.** İstekler `includedRegionCodes: ["tr"]` ve kiracının `service_area` dikdörtgeninden türetilen `locationRestriction` ile gönderilir. Öneriler daha alakalı olur, daha az tuş vuruşu gerekir ve maliyet düşer.

**c) Hizmet bölgesi doğrulaması.** Adres girişi ve doğrulama, `service_area` dışında kalan koordinatları reddeder veya yöneticiyi uyarır (politika kiracı ayarı olabilir).

**d) İşletme bazlı adres defteri.** Her işletme için kayıtlı müşteri adresleri tutulur (telefon → `place_id` + adres metni). Google şartları gereği `place_id` süresiz saklanabilir; enlem / boylam önbelleği sınırlıdır, gerektiğinde Place Details ile yenilenir. Tekrar eden müşteriler (yurt, kampüs, ofis) yeni Google çağrısı gerektirmez. Telefon numarası kişisel veridir; saklama ve KVKK yönü ADR-0005 ile birlikte ele alınır. **Açık madde:** Google ToS'un tam uygulama biçimi (hangi alanın ne kadar süre saklanacağı) netleştirilecek.

**e) ETA.** Eskişehir'de trafik ılımlı olduğundan başlangıçta OSRM tahmini (canlı trafik yok) kabul edilir. Ölçülen hata yüksek çıkarsa yalnızca bu amaçla Google trafik duyarlı Routes kullanılır.

**f) İsteğe bağlı, sonraya:** işletme bazında bölge (mahalle) tabanlı fiyatlandırma ve mesafe kademeleri.

## Sonuçlar

- (+) Daha küçük ve ucuz sunucu; daha hızlı OSM güncelleme ve ön işleme.
- (+) Daha isabetli adres önerileri ve daha düşük Google maliyeti.
- (+) Başka bir şehre satış, kod değişikliği olmadan yeni `service_area` + yeniden çıkarım ile mümkün.
- (-) Hizmet bölgesi dışı adresler için özel bir akış (red / uyarı / yönetici onayı) tasarlanmalı.
- (-) Çok şehirli kiracıda tek OSRM çıkarımı birleştirilmiş bölgeyi kapsamalı; şehir sayısı arttıkça tüm Türkiye çıkarımına geçiş gerekebilir.
- (-) Adres defteri ek veri ve saklama sorumluluğu getirir.

## Açık riskler

- **Açık risk:** Eskişehir'de OSM yol ve adres kalitesi henüz doğrulanmadı ([ADR-0001](0001-hibrit-harita-mimarisi.md) açık riskiyle ilişkili). Görev: İterasyon 1/2 içinde yaklaşık **30 gerçekçi örnek adres** (gerçek müşteri verisi kullanmadan, kurgusal ya da kamuya açık yer adlarıyla) üzerinde Google geocode sonucu ile OSRM yola oturtma (snapping) sonucu karşılaştırılacaktır.
- **Açık risk:** Adres defterinde Google ToS uyumu (yukarıdaki d maddesi).

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Şehri kodda sabitlemek (hardcode) | Başka şehirlere satışı engeller; her genişleme kod değişikliği ve yeniden dağıtım gerektirir |
| İlk günden tüm Türkiye çıkarımı | Şimdilik gereksiz maliyet: daha büyük RAM (4-8 GB) ve daha uzun ön işleme |
