# MEDIAFAIRY Netflix Token Bot — Cloudflare Worker

Bot Telegram generator Token Auto-Login Netflix. Dikembangkan dengan TypeScript multi-module dan dijalankan di Cloudflare Workers (free plan).

> Repo ini memuat **hasil build ter-obfuscate** (`dist/`). Source code asli tidak dipublikasikan; seluruh modul di-obfuscate pada tahap build lokal, dengan pengecualian `donation.js` yang sengaja dibiarkan terbaca.

## Struktur Build

```
dist/
├── index.js          # entry: fetch + scheduled, /setwebhook, routing webhook
├── netflix.js        # fetchNfToken, extractCookieDict, extractMetadata
├── generate.js       # processGeneration + filter region
├── convert.js        # konversi cookie ke vault
├── validate.js       # validasi otomatis (cron, cursor)
├── broadcast.js      # broadcast bertahap (cursor)
├── vault.js          # paginasi vault
├── gate.js           # verifikasi member channel + reward join
├── kv.js             # abstraksi Cloudflare KV (vault, users, stats, cursor)
├── bot.js            # konteks Bot (config + KV + helper Telegram)
├── config.js         # readConfig
├── telegram.js       # tg(), sendMessage(), escapeHtml()
├── stats.js          # kuota, tanggal WIB, sleep
├── menu.js           # keyboard menu & region
├── donation.js       # menu donasi QRIS (tanpa obfuscate)
├── types.js          # tipe (kosong setelah kompilasi)
└── handlers/
    ├── message.js    # handler pesan (perintah admin, menu, /gen, /conv, file .txt)
    └── callback.js   # handler callback (vault_, random_, verify_join)
```

Teknik obfuscation: string-array base64 + rotasi, control-flow flattening, unicode escape.

## Fitur

- 🔑 Generate Token manual (`/gen [teks/id]`) & 🎲 Generate Random per region
- 📁 Vault global dengan paginasi
- 🔒 Gate wajib join channel — akses bot hanya untuk member channel (diverifikasi pada setiap interaksi)
  - 🎁 Reward +20 kuota ekstra (sekali per user, dilindungi dari keluar-masuk channel)
- 📊 Limit gratis 10x/hari (reset 00:00 WIB) + jeda 1 menit antar generate (admin bebas)
- 💳 Menu Donasi QRIS — dukungan sukarela
- 🧹 Validasi cookie otomatis via Cron Trigger — per-batch (default 40 akun/run) agar tetap di bawah batas 50 subrequest; cookie mati dihapus otomatis dan dilaporkan ke admin
- 📢 Broadcast bertahap (cursor) ke seluruh user + 📊 statistik server
- Penyimpanan Cloudflare KV

## Deploy

### A. Otomatis via GitHub Actions (disarankan)

1. Buat API Token Cloudflare (template *Edit Cloudflare Workers*) dan catat Account ID.
2. Tambahkan repository secrets (**Settings → Secrets and variables → Actions**):
   - Wajib: `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`
   - Opsional (ada default): `ADMIN_ID`, `ADMIN_USERNAME`, `GROUP_ID`, `CHANNEL_ID`, `CHANNEL_URL`, `FREE_DAILY_LIMIT`, `COOLDOWN_MS`, `DONATION_QRIS_URL`, `BATCH_SIZE`
3. Set rahasia worker sekali saja:
   ```
   npx wrangler secret put BOT_TOKEN
   npx wrangler secret put SECRET_TOKEN    # wajib: proteksi webhook & /setwebhook
   npx wrangler secret put SETUP_KEY       # opsional
   ```
4. Push ke `main` → workflow `.github/workflows/deploy.yml` menjalankan `npm ci` lalu `wrangler deploy` dengan seluruh variabel dari secrets. Nilai asli tidak pernah di-commit.

### B. Manual dari mesin lokal

1. Install & login:
   ```
   npm i
   npx wrangler login
   ```
2. Buat namespace KV dan salin ID-nya ke `wrangler.toml`:
   ```
   npx wrangler kv namespace create Neetflix_KV
   ```
3. Jadikan bot sebagai **Admin di channel** agar `getChatMember` dapat memverifikasi member.
4. Set rahasia (tidak di-commit):
   ```
   npx wrangler secret put BOT_TOKEN
   npx wrangler secret put SECRET_TOKEN
   npx wrangler secret put SETUP_KEY       # opsional
   ```
5. Deploymen — `dist/` sudah tersedia di repo:
   ```
   npx wrangler deploy --var ADMIN_ID:... --var GROUP_ID:... ...
   ```
6. Ikat webhook Telegram (sekali, wajib menggunakan key):
   ```
   https://<nama-worker>.workers.dev/setwebhook?key=<SECRET_TOKEN>
   ```

> Jika sebelumnya bot live di Cloudflare Pages, hapus project Pages lama — hanya boleh ada 1 webhook aktif untuk token yang sama.

## Cron (validasi otomatis)

`wrangler.toml` → `[triggers] crons = ["*/30 * * * *"]` — tiap 30 menit bot memvalidasi `BATCH_SIZE` akun. Seluruh vault tersapu setiap `ceil(N / BATCH_SIZE) × 30` menit. Atur `BATCH_SIZE` (maksimal ~45) sesuai ukuran vault agar tidak melewati batas 50 subrequest.

## Perintah Admin

| Perintah | Fungsi |
|---|---|
| `/clearvault` | Kosongkan seluruh Vault |
| `/del [ID]` | Hapus 1 akun dari Vault |
| `/addquota [ID_USER] [JUMLAH]` | Tambah kuota ekstra |
| `/resetlimit [ID_USER]` | Reset limit harian user |
| `/broadcast [PESAN]` | Kirim pesan ke semua user (bertahap) |
| `/stats` | Statistik server |
| `/conv [teks]` / kirim file `.txt` | Konversi cookie ke Vault |

Detail format input yang didukung: lihat [ADMIN.md](./ADMIN.md).

## Batasan Free Plan yang Diakomodasi

- **50 subrequest/request** — generate/acak/konversi dibatasi per-batch; validasi & broadcast memakai cursor KV; validasi dan broadcast tidak dijalankan pada run cron yang sama.
- **KV** — 100.000 read/hari, 1.000 write/hari, 1.000 delete/hari, 1.000 list/hari.
- **100.000 request/hari** termasuk cron — cron tiap 30 menit hanya ±48/hari.

## Pengembangan

Source `src/` dan pipeline build/obfuscate (`scripts/obfuscate.mjs`, `tsconfig.build.json`) tidak dimuat di repo publik. Rebuild dilakukan dari salinan source pribadi:

```
npm i
npm run typecheck                    # pemeriksaan tipe TypeScript
npm run build && npm run obfuscate   # hasil ke dist/
cp .dev.vars.example .dev.vars       # env lokal untuk wrangler dev
npm run dev
```

## Lisensi

[MIT](./LICENSE)
