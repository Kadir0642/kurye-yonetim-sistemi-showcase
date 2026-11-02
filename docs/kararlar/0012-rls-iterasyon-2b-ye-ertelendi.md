# ADR-0012: PostgreSQL RLS İterasyon 2b'ye ertelendi; İterasyon 2 tabloları RLS'ye hazır yazılır

- **Durum:** Önerildi (İterasyon 2 spesifikasyonuyla birlikte onaya sunuldu)
- **Tarih:** 2026-11-02
- **Değiştirdiği karar:** [ADR-0004](0004-cok-kiracili-mimari.md) uygulama notundaki "RLS İterasyon 2'de açılır" zamanlaması

## Bağlam

[ADR-0004](0004-cok-kiracili-mimari.md) RLS'yi ikinci savunma hattı olarak kararlaştırdı ve İterasyon 1'de pilotta tek kiracı olduğu için İterasyon 2'ye erteledi. İterasyon 2'nin planı bu kez şunlarla çakışıyor:

- RLS, her isteğin bir işlem içinde `set_config('app.tenant_id', $1, true)` ile çalışmasını ister. İterasyon 1 kodunda servisler sorgularının çoğunu işlem dışında, bağlantı havuzundan (`this.db`) yapar; ayrıca arka plan işleri (konum kesildi, saklama) kiracılar arası çalışır. RLS'yi açmak `services/api` içindeki **neredeyse her servisi** değiştirir.
- İterasyon 2, `services/api` üzerinde iki paralel görev (A2: rota ve atama alanı, H2: devir alanı) ve rota motoru görevi (R) çalıştırıyor. Her servise dokunan bir RLS değişikliği bu görevlerle sürekli birleştirme çakışması üretir.
- Pilotta hâlâ tek kiracı var. Uygulama katmanı süzgeci, bileşik yabancı anahtarlar ve "başka kiracının kaynağı 404" e2e testi İterasyon 1'de çalışıyor.
- İterasyon 2'nin ana hedefi müşteriye görünen farktır (rota uyumlu atama); RLS görünmez bir sağlamlaştırmadır.

## Karar

1. **RLS İterasyon 2b'de, ayrı ve tek başına bir görev olarak** açılır: İterasyon 2 görevleri (R, A2, H2, B2, C2) birleştikten sonra, başka görev `services/api` üzerinde çalışmıyorken.
2. **İterasyon 2'de RLS'ye hazırlık zorunludur** (A2 ve H2 kabul kriteri):
   - Yeni tabloların hepsinde `tenant_id NOT NULL` ve kiracılar arası bileşik yabancı anahtarlar (`(tenant_id, x_id)`) bulunur.
   - Yeni veri erişim fonksiyonları `tenantId`'yi ilk parametre alır ve her sorguya `tenant_id = $tenantId` ekler (İterasyon 1 SPEC §4 kuralı).
   - Kiracılar arası sorgu yalnızca arka plan işlerinde bulunur ve kod içinde `// RLS: cross-tenant job` yorumuyla işaretlenir (İterasyon 2b'de ayrı DB rolüne taşınacak yerlerin listesi).
   - Kiracı izolasyonu e2e testi (`test/isolation.e2e-spec.ts`) tüm yeni `/{id}` uçlarını kapsayacak biçimde genişletilir.
3. **İterasyon 2b tasarımı (bu ADR ile sabitlenir, uygulama 2b'de):**
   - İki veritabanı rolü: `kurye_app` (`NOBYPASSRLS`, API istekleri) ve `kurye_jobs` (`BYPASSRLS`, yalnızca işler ve migration).
   - `withTenant(tenantId, fn)` yardımcısı: bir işlem açar, `set_config('app.tenant_id', tenantId, true)` çalıştırır ve `fn(tx)`'i çağırır. Tüm istek yolu buna taşınır.
   - Politika: `USING (tenant_id = current_setting('app.tenant_id', true)::uuid)` ve aynı `WITH CHECK`; ayar yoksa hiçbir satır görünmez (güvenli varsayılan).
   - Kabul: tüm e2e testleri `kurye_app` rolüyle geçer; bilinçli olarak `tenant_id` süzgeci kaldırılmış bir sorgu testinde 0 satır döner; p95 gecikme artışı < %10.

## Sonuçlar

- (+) İterasyon 2 paralel görevleri birbirine ve RLS'ye takılmaz; ana hedef (rota uyumlu atama) önce teslim edilir.
- (+) Yeni tablolar baştan RLS'ye hazır yazıldığı için 2b işi mekanik kalır (politika + `withTenant` + rol).
- (-) Bir iterasyon daha uygulama katmanı süzgeci tek hattır; bir depo fonksiyonunda unutulan `tenant_id` süzgeci veriyi sızdırabilir. Risk, bileşik FK'lar, kod inceleme kuralı ve genişletilen izolasyon testiyle sınırlanır. Pilot tek kiracılı olduğu sürece gerçek sızıntı yüzeyi yoktur.
- (-) İkinci kiracı canlıya alınmadan önce 2b tamamlanmalıdır (YOL-HARITASI'nda ön koşul olarak yazılır).

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| RLS'yi İterasyon 2'de A2 göreviyle birlikte açmak | A2 zaten en büyük görev; tüm servislerin işleme taşınması H2 ve R ile çakışır, gecikme ve hata riski ana hedefi tehlikeye atar |
| RLS'yi İterasyon 2'nin ilk görevi olarak, diğerlerinden önce açmak | Tüm İterasyon 2 görevlerini bir görevin arkasına dizer; paralellik kaybolur |
| RLS'den tamamen vazgeçmek | ADR-0004'ün gerekçesi (satış sonrası çok kiracı, ikinci savunma hattı) geçerli |
| Yalnızca yeni İterasyon 2 tablolarına RLS | Karma model: aynı istekte bazı tablolar `withTenant` ister, bazıları istemez; hata ayıklaması zor, tutarsız güvenlik |
