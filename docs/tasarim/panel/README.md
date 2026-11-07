# Yönetim paneli: marka geçişi maketleri

> **Durum:** ürün sahibi **P2 "Koyu kenar çubuğu + açık içerik"** seçti (2026-11-07, [`secim.md`](secim.md)); uygulama `feature/panel-marka-maket` dalında, ayrıntı: `../panel-uygulama-notu.md`.

Kurye uygulamasında onaylanan dil ([ADR-0027](../../kararlar/0027-marka-kimligi-ve-navigasyon-gorsel-dili.md) ve ekindeki A2 kararı, `../kurye-app/index-acik.html`) yönetim paneline (React + Mantine 9) nasıl taşınacak? Bu klasör üç varyantı gösterir. Ortak kural: uygulama içi ekranlar **açık ve sakin** (nötr gri zemin `#F3F4F1`, beyaz kart, birincil eylem marka yeşili `#3F6212` + beyaz yazı), navigasyon ve harita **koyu**; neon `#C8FF3A` açık yüzeyde dolgu ya da metin olmaz.

| Dosya | İçerik |
|---|---|
| [`index.html`](index.html) | 8 masaüstü ekran (1440×900) + tablet canlı harita (1024×768), her biri P1 / P2 / P3: giriş, canlı harita, paketler + paket çekmecesi, adres defteri, işletmeler + QR bağlantı modalı, Ayarlar › Teslim ve hazırlık (Ölçüm / Zorunlu = OBSERVE / ENFORCE), Ayarlar › Dağıtım (v2), metrikler. Her varyantın açıklaması, belirteç şeridi, bileşen örnekleri ve Mantine eşlemesi; karşılaştırma tablosu ve öneri. "Harita karosu: Koyu" düğmesi koyu karonun kromla birlikte nasıl durduğunu gösterir. |

**Açmak için:** `index.html` dosyasını tarayıcıda açın (JavaScript satır içi, sunucu gerekmez; `python -m http.server` ile `docs/` altından da açılır). Bağlantılar: `index.html#paketler` tek ekran, `index.html#paketler/2` tek ekran + tek varyant, `index.html#harita/3/koyu` koyu karoyla. Çerçeveye tıklayınca tam genişlikte açılır. İnternet yoksa yazı tipleri sistem yazı tiplerine düşer. Tüm ad, adres, telefon ve bağlantılar kurgusaldır; harita çizimleri temsilîdir.

## Neyi seçiyorsunuz

Üçü de **aynı paleti** ve aynı bileşen kurallarını kullanır (durum çipi %10 ton + koyu yazı + ikon, odak halkası 2 px `#3F6212`, hedef ≥ 40 px). Fark kenar çubuğunun rengi ve bilgi yoğunluğu.

| | P1 · Açık A2 birebir | P2 · Koyu kenar çubuğu + açık içerik | P3 · Yoğun operasyon |
|---|---|---|---|
| Kimin için | İşletme sahibi, ara sıra giren yönetici | Gün boyu paneli açık tutan menajer / dispeçer, satış demosu | Yoğun saatte 30+ paketi izleyen dispeçer |
| Kenar çubuğu | Beyaz, etkin öğe yeşil ton + kenarlı neon nokta | `#111318`, etkin öğe 4 px neon gösterge + neon ikon (paneldeki tek neon) | Beyaz 76 px ikon rayı |
| Göz yorgunluğu (uzun süre) | Orta (ekranın tamamı açık) | **En düşük** (çevredeki sabit alan koyu, okunan alan açık) | Orta (küçük yazı, ama daha az kaydırma) |
| Bilgi yoğunluğu (paket listesi) | ~12 satır (52 px) | ~12 satır | **~17 satır (40 px)**, yan panel, toplu seçim, kısayollar |
| Marka tutarlılığı | Kurye uygulamasıyla birebir; marka zayıf | **Kurye uygulaması + navigasyon kromu**; marka güçlü | P1 kadar |
| Koyu harita karosuyla | Açık çerçevede "delik"; kontroller koyu karoda koyu kroma geçer | **Kesintisiz koyu**; kontroller hep koyu krom | P1 gibi |
| Mantine eforu | **S–M** | **M** | **L** |

## Öneri: P2 · Koyu kenar çubuğu + açık içerik

1. **Marka:** neon açık yüzeyde yaşayamıyor (beyazda 1,0:1); panelde neonun meşru tek yeri koyu bir yüzey. P2 onu kalıcı bir yere koyar, okunan içeriği A2 açık bırakır; ürün sahibinin "uygulama içi açık, navigasyon / harita koyu" kararıyla birebir.
2. **Uzun süre ekrana bakan menajer:** geniş, sabit, koyu şerit toplam parlaklığı düşürür; parlaklık sınırı okuma alanının dışında. Tablolar ve formlar P1 ile aynı (≥ 7:1).
3. **Harita köprüsü:** koyu karo seçildiğinde kenar çubuğu + harita kesintisiz koyu; harita kontrolleri kurye navigasyonundaki koyu krom.
4. **Efor makul (M);** P1'e dönüş tek tema satırı.
5. **P3'ün kazanımı ayrı varyant değil:** sonraki aşamada **"Yoğunluk: Rahat / Sıkı"** kullanıcı seçeneği olarak (yalnızca Paketler ve Canlı harita listeleri); ikon rayı, yan panel ve klavye kısayolları bağımsız küçük işler olarak eklenebilir.

## Uygulama planı (seçimden sonra)

| Aşama | İş | Efor |
|---|---|---|
| 0 · Yazı tipleri ve belirteçler | `@fontsource-variable/inter` + `@fontsource/barlow-condensed` (OFL, npm, ücretsiz; self-host, üçüncü taraf istek yok). `src/theme/brand.ts`: `tokens.json → courierApp` rolleriyle tek kaynak; `tokens.json` eşitliği ve kontrast çiftleri için birim test | S |
| 1 · Tema override | `theme.ts`: 10 tonlu `brand` paleti (7 = `#3F6212`), `primaryShade { light: 7 }`, `defaultRadius 'md'` (10 px), `cssVariablesResolver` (gövde `#F3F4F1`, metin `#111318`, dimmed `#4A5260`, kenar `#E4E6E1`), Button / Input yüksekliği 40 px, Card / Modal / Drawer varsayılanları, overlay bulanıklığı kaldırılır. Sayfaların çoğu bu adımla kendiliğinden geçer | S |
| 2 · Kabuk | `AppLayout`: logo ("K + ok" koyu karo) kenar çubuğuna, üst çubukta arama + canlı + uyarı + kullanıcı; P2 için `ink` paleti, koyu `AppShell.Navbar`, `NavLink` CSS modülü (neon gösterge, koyu zeminde neon odak halkası); tablet altında ray | S–M |
| 3 · Ortak bileşenler | `StatusChip` (paket + konum + adres durumu, ikon + etiket; `PACKAGE_STATUS` ton + Tabler ikonu alır), `KpiCard` (Barlow Condensed), `MeterRow`, `SaveBar`, `MapControl` (koyu krom), `EmptyState`; ham `color="indigo"` / gradyan ve `lib/status.ts` Mantine renk adları temizlenir | M |
| 4 · Sayfa sayfa | Giriş → Canlı harita (sağ panel, sayaçlar) → Paketler + çekmece → İşletmeler + QR modalı → Adres defteri → Ayarlar (Teslim ve hazırlık seçilebilir kartlar, Dağıtım v2) → Metrikler → Kuryeler / Devirler / paket oluştur. Her sayfa için ekran görüntüsü güncellenir (`docs/assets/screenshots`) | M |
| 5 · (İsteğe bağlı) Yoğunluk | P3: `data-density="compact"`, Table `stickyHeader` + `striped`, 40 px satır, Paketler yan paneli, `useHotkeys` (J/K/Enter/A/X/N, `/`) | M |

İkonlar: panel mevcut **Tabler** ikonlarını (MIT) kullanmaya devam eder, çizgi 1,6 → 2 (Lucide ile aynı geometri ailesi); yeni bağımlılık gerekmez.

## Değişmeyenler

- **Harita stilleri ve karolar** (`mapStyles.admin`, PMTiles, glyph seti; ADR-0018), kurye rota renk paleti (`ROUTE_PALETTE`), MapLibre katmanları. Değişen yalnızca haritanın çevresindeki krom (kontroller, lejant, seçili halka) ve paket işaretlerinin durum renkleri.
- **İş mantığı:** API sözleşmesi, WebSocket olayları, roller, yönlendirme, form doğrulama, MSW sahte katmanı ve testlerin davranışı. Metinler (`i18n/tr*.ts`) aynı kalır; yalnızca birkaç kısa etiket eklenir (ör. "Kaydedilmemiş değişiklik").
- İşletme hazır sayfası (`/isletme`, girişsiz) bu kapsamın dışında; ayrı küçük iş (A2 açık, büyük "HAZIR" düğmesi).
- Koyu tema (kullanıcı seçeneği) şimdi yok; belirteçler hazır (`tokens.json → theme.dark`), sonra ayrı iş.

## Erişilebilirlik kuralları

- **Metin kontrastı ≥ 4,5:1 (WCAG AA), hedef ≥ 7:1** (ADR-0027 açık zemin kuralı): gövde `#111318` beyazda 18,6:1, gri zeminde 16,8:1; ikincil `#4A5260` 7,9:1 / 7,1:1; birincil düğme beyaz / `#3F6212` 7,1:1; P2 kenar çubuğunda `#F5F7FA` 17,3:1, `#B4BAC4` 9,5:1. Durum çiplerinde metin koyu tondadır (≥ 7:1), renk ikonda ve %10 zeminde.
- **Arayüz bileşenleri ≥ 3:1:** denetim kenarı `#808896` (beyazda 3,6:1), kapalı anahtar izi `#808896`.
- **Odak her zaman görünür:** `:focus-visible` 2 px `#3F6212` halka, 2 px boşluk; koyu kenar çubuğunda neon `#C8FF3A`. Tablo satırları klavyeyle odaklanır (P3'te zorunlu).
- **Masaüstünde en küçük hedef 40 px** (düğme, giriş alanı, ikon düğmesi, menü öğesi); satır içi küçük düğmelerde görsel 32 px + 40 px dokunma alanı.
- **Renk tek başına anlam taşımaz:** durumlar ikon + etiket + renk; harita işaretlerinde biçim (paket kare, kurye daire) + renk.
- En küçük yazı 12 px (büyük harfli küçük başlıklar 11,5 px, kalın, geniş aralıklı); hareket yalnızca durum değişiminde ve `prefers-reduced-motion` ile kapanır.

## Açık sorular

1. Varyant: P1, **P2 (öneri)** veya P3?
2. P3'ün yoğun görünümü sonraki aşamada kullanıcı seçeneği olarak eklensin mi?
3. Giriş ekranında P2'nin koyu marka paneli mi, P1'in sade ortalanmış kartı mı?
