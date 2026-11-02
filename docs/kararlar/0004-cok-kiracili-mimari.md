# ADR-0004: Çok kiracılı (multi-tenant) mimari

- **Durum:** Kabul edildi (RLS İterasyon 2'ye ertelendi; aşağıdaki uygulama notuna bakın)
- **Tarih:** 2026-10-30

## Bağlam

Ürün ilk müşteriden sonra başka kurye firmalarına da satılabilir. Sonradan çok kiracılığa geçmek, veri modelini ve yetkilendirmeyi baştan değiştirmeyi gerektirir. Pazar araştırmasına göre rakipler çok sayıda firma ve işletmeyi tek platformda barındırıyor.

## Karar

Baştan çok kiracılı tasarım:

- **Kiracı (tenant) = kurye firması.** İşletmeler (restoranlar), kuryeler, paketler, duraklar, vardiyalar, hakedişler ve kasa kayıtları kiracıya aittir.
- Tek veritabanı, ortak şema; tüm kiracıya ait tablolarda `tenant_id` kolonu.
- **PostgreSQL satır düzeyi güvenlik (RLS):** her oturumda kiracı bağlamı (ör. `SET LOCAL app.tenant_id`) ayarlanır; politikalar `tenant_id` eşleşmesini zorlar. Uygulama katmanı hatalarına karşı ikinci savunma hattıdır.
- Kullanıcı rolleri kiracı içinde tanımlanır (firma yöneticisi, kurye; ileride işletme kullanıcısı).
- Pilot aşamada tek kiracı vardır; ancak kod ve şema çok kiracılıdır.

## Uygulama notu (İterasyon 1, 2026-10-30)

- İterasyon 1'de izolasyon **uygulama katmanında** zorlanır: JWT'deki `tid` her isteğin kiracı bağlamıdır; tüm sorgular depo (repository) katmanında `tenant_id` ile filtrelenir; yolda veya gövdede kiracı kimliği kabul edilmez.
- **Veritabanı düzeyinde ikinci hat:** kiracıya ait tablolar arası yabancı anahtarlar **bileşik**tir (`(tenant_id, id)`); böylece bir kiracının paketi başka kiracının işletmesine veya kuryesine bağlanamaz.
- **PostgreSQL RLS İterasyon 2'de** açılır. Gerekçe: pilotta tek kiracı var. RLS, her isteğin bir işlem içinde `SET LOCAL app.tenant_id` ile çalışmasını gerektirir; bu da bağlantı havuzu, arka plan işleri (konum kesildi kontrolü, saklama süresi silme) ve seed ile birlikte ilk iterasyonda gereksiz risk getirir. Şema RLS'ye hazırdır: `tenant_id` her tabloda bulunur ve uygulama ayrı bir DB rolüyle bağlanacak şekilde tasarlanır. e2e testleri kiracılar arası erişimin 404 döndüğünü doğrular.

## Sonuçlar

- (+) Yeni firma eklemek veri taşıma gerektirmez; satış modeline hazır.
- (+) RLS ile kiracılar arası veri sızıntısı riski azalır.
- (-) Her sorgu ve migration kiracı bağlamını hesaba katmalı; test altyapısı bunu doğrulamalı.
- (-) Bağlantı havuzu ile RLS bağlamını yönetmek ek dikkat ister (işlem bazlı `SET LOCAL`).
- (-) Büyük kiracıların ağır sorguları diğerlerini etkileyebilir; ölçek büyürse kiracı bazlı partition veya ayrı veritabanı değerlendirilir.

## Alternatifler

| Alternatif | Değerlendirme |
|---|---|
| Kiracı başına veritabanı | En güçlü izolasyon; operasyon ve migration yükü yüksek, bu ölçek için gereksiz |
| Kiracı başına şema | Ara çözüm; migration karmaşıklığı artar |
| Tek kiracılı başlayıp sonra dönüştürmek | Pilot için hızlı, ancak sonradan dönüşüm maliyetli ve riskli |
