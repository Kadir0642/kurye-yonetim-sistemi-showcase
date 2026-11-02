# ADR-0007: Backend ve web yığını — uçtan uca TypeScript monorepo

- **Durum:** Kabul edildi
- **Tarih:** 2026-10-30

## Bağlam

- Ürün sahibi kod yazmıyor; tüm bileşenler yapay zekâ ajanlarıyla geliştiriliyor. Öncelikler sağlamlık, performans ve bakım kolaylığı; bunun için olgun açık kaynak bileşenler yeniden kullanılır.
- Bütçe: önce pilot kurye firmasına (Eskişehir) demo yapılacak, maliyet sıfıra yakın olmalı. Ürün yalnızca müşteri beğenirse canlıya alınacak.
- Ölçek hedefi: kiracı başına 20-25 eşzamanlı kurye, günde yaklaşık 1500 paket. Konum trafiği saniyede birkaç mesajdır (teknik araştırma).
- İterasyon 1'de üç bileşen (API, panel, mobil uygulama) **paralel** olarak ayrı ajanlarca geliştirilecek. Bu yüzden ortak sözleşme önceden ve tek yerde tanımlanmalıdır (contract-first).
- Kurye uygulaması React Native + Expo (TypeScript) ile yazılır ([ADR-0002](0002-kurye-uygulamasi-react-native-expo.md)).

## Karar

**Uçtan uca TypeScript, npm workspaces monoreposu.** pnpm kurulu olmadığı için npm kullanılır.

```text
packages/contracts     OpenAPI 3.1 (openapi.yaml) + üretilen TS tipleri + WebSocket protokolü
services/api           NestJS API
apps/admin-web         React yönetim paneli
apps/courier-mobile    React Native + Expo kurye uygulaması
infra/                 docker-compose (PostgreSQL 16 + PostGIS, Redis)
```

**API (`services/api`)**
- NestJS, Fastify adaptörüyle.
- Drizzle ORM + `postgres` (postgres-js) sürücüsü. PostgreSQL 16 + PostGIS (`geography(Point, 4326)`, GIST indeksleri, `ST_DWithin`).
- Redis (`ioredis`): son konum, "konum kesildi" takibi, pub/sub fan-out, hız sınırlama.
- Gerçek zamanlı kanal: **düz WebSocket** (NestJS `WsAdapter`, `ws` kütüphanesi) ve küçük bir JSON olay protokolü (`packages/contracts/websocket-events.md`). Socket.IO **kullanılmaz**: tarayıcı `WebSocket`'i ve React Native'in yerleşik `WebSocket`'i ek istemci kütüphanesi olmadan bağlanabilir. Redis pub/sub fan-out sayesinde API yatay ölçeklenebilir.
- Kimlik doğrulama: kısa ömürlü JWT access token (15 dk) ve opak, tek kullanımlık, rotasyonlu refresh token. Refresh token'lar veritabanında hash'lenmiş saklanır; tekrar kullanım tespit edilirse token ailesi iptal edilir.
- **Doğrulama: zod (v4)**, küçük bir özel `ZodValidationPipe` ile. Gerekçe:
  - Tek şema hem çalışma zamanı doğrulaması hem TS tipi üretir.
  - Şemalar `@kurye/contracts` tiplerine karşı derleme zamanında kontrol edilebilir (`z.ZodType<Schemas['PackageCreate']>`).
  - Aynı şemalar WebSocket mesajları ve ortam değişkenleri için de kullanılır.
  - Dekoratör ve `class-transformer` yükü yoktur.
  - `nestjs-zod` paketi henüz NestJS 12'yi desteklemediği için kullanılmaz; pipe birkaç satırdır.
- Parola hash'i: Argon2id (`@node-rs/argon2`; hazır ikili dağıtır, Windows'ta derleme araçları gerektirmez).
- Loglama: `nestjs-pino`. Token, parola ve müşteri telefonu loglarda maskelenir.

**Sözleşme (`packages/contracts`)**
- `openapi.yaml` (OpenAPI 3.1) gerçeğin kaynağıdır. Tipler `openapi-typescript` ile `src/generated/openapi.d.ts` dosyasına üretilir ve commit edilir.
- Paket yalnızca tip içerir; tüketiciler `import type` kullanır.
- API, e2e testlerinde yanıtları bu şemaya karşı (AJV) doğrular.

**Panel (`apps/admin-web`)**
- React + Vite + TypeScript, TanStack Query, React Router, Mantine UI.
- `openapi-fetch` ile sözleşmeye bağlı istemci. Harita için MapLibre GL JS ([ADR-0008](0008-acik-kaynak-harita.md)).
- Arayüz metinleri Türkçe.

**Sürüm ilkesi**
- Ajanlar paketleri kurulum anındaki güncel kararlı sürümlerle kurar ve `package-lock.json` commit edilir. Brief'lerde yazım tarihinde doğrulanan ana sürümler verilir.
- React sürümü tüm workspace'lerde tek tutulur: Expo SDK'nın beklediği sürüm, kök `overrides` ile sabitlenir.
- TypeScript 6.0.x kullanılır. TS 7 (yeni native derleyici) NestJS'in dekoratör metadata ihtiyacı ve araç uyumluluğu doğrulanana kadar kullanılmaz.

## Sonuçlar

- (+) Tek dil ve tek tip sistemi: sözleşme değişikliği üç bileşende derleme hatası olarak görünür.
- (+) Olgun, yaygın bileşenler: ajanların doğru kod üretme olasılığı yüksek, topluluk desteği geniş.
- (+) Sıfır lisans maliyeti; yerelde `docker compose` ile tüm altyapı çalışır.
- (+) Düz WebSocket ve Redis fan-out ile yatay ölçeklenebilir; Android/iOS istemcisi ek bağımlılık gerektirmez.
- (-) npm workspaces, pnpm'e göre daha yavaş kurulur ve paket yükseltme (hoisting) farklılıkları olabilir. Paralel geliştirmede `package-lock.json` birleştirme çakışmaları beklenir; birleştirme sırasında yeniden üretilir.
- (-) Düz WebSocket'te yeniden bağlanma, kalp atışı ve olay sonrası anlık görüntü yenileme istemcide elle yazılır (protokolde tanımlıdır).
- (-) zod şemaları OpenAPI'den otomatik üretilmez; uyum, derleme zamanı tip kontrolü ve e2e şema doğrulamasıyla sağlanır.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Express adaptörü | Fastify daha hızlı, şema dostu; NestJS ikisini de destekler |
| Prisma | Ağır motor ikilisi, PostGIS desteği zayıf (ham SQL gerekir); Drizzle SQL'e daha yakın ve hafif |
| TypeORM | Bakım temposu ve tip güvenliği Drizzle'dan zayıf |
| Socket.IO | Kendi protokolü; Android/iOS tarafında ek istemci kütüphanesi gerektirir; düz WebSocket bu ölçek için yeterli |
| class-validator | Dekoratör tabanlı, tipleri tekrar ettirir; zod tek şemadan tip ve doğrulama üretir |
| Kod-önce (code-first) OpenAPI (`@nestjs/swagger`) | Paralel geliştirmede sözleşmenin önce sabitlenmesi gerekiyor; contract-first seçildi |
| pnpm / Turborepo | Makinede pnpm kurulu değil; npm workspaces bu boyut için yeterli |
| Go / Kotlin backend | Uçtan uca tip paylaşımı kaybolur |
