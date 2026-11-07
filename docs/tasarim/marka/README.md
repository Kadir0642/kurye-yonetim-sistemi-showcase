# Marka dili v1

Kurye Yönetim Sistemi'nin kurumsal görsel dili. Navigasyon tasarım varyantı **B · Koyu / premium**'dan türetildi ([seçim kaydı](../navigasyon/secim.md), [B v1 maketi](../navigasyon/b-v1.html)). Ürün başka kurye ekiplerine çok kiracılı olarak satılacağı için tek ve tanınır bir dil hedeflenir: **siyaha yakın yüzeyler, beyaz metin, tek neon yeşil vurgu.**

Tüm değerler: [`tokens.json`](tokens.json). Henüz hiçbir uygulama koduna uygulanmadı.

## Renk

### Neon (birincil)

| Belirteç | Hex | Kullanım |
|---|---|---|
| `neon.500` | `#C8FF3A` | **Birincil.** Koyu zeminde vurgu, ok, etkin durum; dolgu olarak her zeminde (üstünde siyah metin `#0B0D10`, 16,5:1) |
| `neon.400` | `#D4FF66` | Koyu zeminde üzerine gelme / basılı |
| `neon.600` | `#B8F03A` | Gece modu (bir ton kısık, daha az parlama) |
| `neon.700` | `#4D7C0F` | **Açık zeminde** ikon, kenarlık, büyük metin (beyazda 5,0:1) |
| `neon.800` | `#3F6212` | **Açık zeminde** normal metin ve bağlantı (beyazda 7,1:1) |

Kural: neon 500/600 açık zeminde asla metin ya da ince çizgi olmaz (beyazda 1,0:1). Açık zeminde neon yalnızca dolgudur ve kenarı `neon.700` ile çizilir.

### Yüzeyler ve griler

| Belirteç | Hex | Kullanım |
|---|---|---|
| `ink.950` | `#0A0C0F` | Gece kromu, en koyu zemin |
| `ink.900` | `#111318` | Gündüz kromu, koyu panel |
| `ink.800` / `ink.700` | `#1A1E25` / `#22262E` | Yükseltilmiş yüzey, düğme |
| `ink.600` | `#2A2E36` | Kenarlık |
| `grey.200` | `#B4BAC4` | Koyu zeminde ikincil metin (9,5:1) |
| `grey.300` / `grey.400` | `#A3AAB5` / `#8E96A3` | Gece ikincil metin (8,4:1) / ipucu |
| `grey.100` / `grey.50` | `#ECEEF1` / `#F6F7F9` | Açık zemin, açık harita |
| `white` | `#FFFFFF` | Koyu zeminde birincil metin (18,6:1) |

### Anlamsal renkler

| Anlam | Koyu zeminde | Açık zeminde | Not |
|---|---|---|---|
| Başarı | `#3DDC97` (10,5:1) | `#047857` | Neon değil nane: "tamamlandı" marka rengiyle karışmasın |
| Uyarı | `#FBBF24` (11,1:1) | `#B45309` | Uyarı şeridi dolgusu `#FBBF24`, metni `#1F1300` (10,9:1) |
| Tehlike | `#FF7A7A` (7,4:1) | `#C5221F` | Dolu düğmede `#FF4D4F` + **siyah** metin (beyaz yalnızca 3,9:1) |
| Bilgi | `#22D3EE` (10,3:1) | `#0E7490` | Gece rotası, varış saati |

Her anlamsal rengin bir de yumuşak zemin tonu var (`softDark` / `softLight`).

## Biçim

- **Köşe yarıçapı:** çip 8 · bildirim 12 · rozet 16 · düğme 18 · kart 20 · alt sayfa 26 · hap tam.
- **Aralık:** 2, 4, 8, 12, 16, 20, 24, 32.
- **Gölge:** üç seviye (`elevation` 3 / 8 / 12). Bulanıklık (blur) yok; "cam" etkisi opak koyu yüzey + 1 px `rgba(255,255,255,0.10)` kenarla.
- **Dokunma:** kurye uygulamasında ≥ 56 dp (eldiven), ana düğmeler 64 dp.

## Tipografi

- **Inter** (arayüz) + **Barlow Condensed** (büyük rakamlar: mesafe, süre, sayaçlar, KPI). İkisi de Google Fonts, OFL, ücretsiz; uygulamaya gömülür, panelde self-host edilir.
- Ölçek: display 84/800 (dar) · displaySm 40/700 (dar) · numeral 38/800 (dar) · h1 28/700 · h2 22/700 · title 21/600 · body 16/500 · label 15/700 · caption 13/600 · overline 12/800 BÜYÜK HARF.
- Rakamlarda `tabular-nums`.

## İkonlar

- **Arayüz:** Lucide tarzı çizgi ikon, 24 ızgara, çizgi 2,25, yuvarlak uç. Lucide ISC lisanslıdır (ücretsiz).
- **Manevra:** geometrik kalın çizgi (14/100), köşeli uç, dolu sivri ok başı; kavşağın diğer kolları %22 opak "hayalet". Tam set: [B v1](../navigasyon/b-v1.html).

## Yönetici paneline uygulanması (şimdi değil)

Panel (Mantine) aynı belirteçleri bir tema olarak alır: `primaryColor` yerine 10 tonlu özel bir `neon` paleti tanımlanır (`primaryShade` açık temada 7, koyu temada 5), birincil düğmeler neon dolgu + siyah metindir, bağlantı ve odak halkası açık zeminde `neon.800` / `neon.700` olur. Kenar çubuğu ve üst çubuk `ink.900` koyu kromdur; içerik alanı varsayılan olarak açık (`grey.50` zemin, beyaz kart) kalır, çünkü dispeçer uzun süre tablo ve form okur; koyu tema kullanıcı seçeneği olarak `theme.dark` rollerini kullanır. Durum rozetleri anlamsal renklerin açık/koyu karşılıklarını, sayısal KPI'lar Barlow Condensed'i kullanır. Panel haritası kurye gündüz "gümüş" paletiyle aynı aileye geçer. Bu dönüşüm ayrı bir iterasyonda, kendi ADR'siyle yapılacak; kiracıya özel logo / renk (white-label) bu sürümde kapsam dışı.
