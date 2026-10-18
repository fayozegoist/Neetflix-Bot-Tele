# 📋 ADMIN.md — Format Input Convert & Generate Token

Panduan format input yang **didukung** untuk `/conv`, file `.txt` (Convert Netscape), dan `/gen`. Semua fitur convert **admin only**; semua input otomatis diverifikasi live ke server Netflix sebelum masuk Vault.

---

## ✅ Format yang didukung

### 1. JSON Array (export Cookie-Editor / EditThisCookie)
Export cookie dari ekstensi browser, lalu tempel atau kirim sebagai `.txt`:

```json
[
  { "name": "NetflixId", "value": "v%3D2%26gt%3D..." },
  { "name": "SecureNetflixId", "value": "..." }
]
```
- Hanya dipakai cookie bernama `NetflixId`; cookie lain diabaikan.
- `value` boleh URL-encoded (otomatis di-decode).

### 2. Netscape HTTP Cookie File (format cookies.txt)
```
# Netscape HTTP Cookie File
# https://curl.se/docs/http-cookies.html

.netflix.com	TRUE	/	TRUE	1767225600	NetflixId	v%3D2%26gt%3D...
```
- Pemisah **tab**, sesuai standar curl/wget `cookies.txt`.
- Baris diawali `#` dianggap komentar.
- Bisa juga tanpa domain, langsung:
```
NetflixId	v%3D2%26gt%3D...
```

### 3. Raw header Cookie (copy dari DevTools)
```
Cookie: NetflixId=v%3D2%26gt%3D...; Secure; HttpOnly
```
atau tanpa prefix:
```
NetflixId=v%3D2%26gt%3D...; Secure; HttpOnly
```

### 4. Nilai `NetflixId` saja
```
v%3D2%26gt%3D17NjE1...
```
Scan otomatis pola `NetflixId=...` di mana pun dalam teks.

### 5. Blok multi-akun dalam satu file `.txt` (bulk)
Pisahkan tiap akun dengan baris metadata yang diawali `Email:`:
```
# Email: akun1@gmail.com
# Country: Indonesia (ID)
# Plan: Premium
# Member Since: March 12, 2021
# Next Billing: April 12, 2026
NetflixId=v%3D2%26gt%3DAAAA...

# Email: akun2@outlook.com
# Country: United States (US)
# Plan: Standard
# Member Since: July 3, 2022
# Next Billing: May 1, 2026
NetflixId=v%3D2%26gt%3DBBBB...
```
- Field metadata yang dikenali: `Email`, `Country` / `Region`, `Plan`, `Member Since`, `Next Billing` (dengan/tanpa awalan `#`).
- Jika hanya 1 blok tanpa `Email:`, fallback dipisah per baris kosong.
- Akun yang **gagal verifikasi langsung dibuang** (dilaporkan sebagai "Dibuang (Mati)").
- Akun valid masuk Vault dengan ID berurutan (tampil di output convert).

---

## 🧾 `/gen` — input token

| Input | Contoh | Perilaku |
|---|---|---|
| ID Vault | `/gen 12` | Ambil cookie tersimpan dengan ID tersebut di Vault |
| JSON / Netscape / Cookie header | `/gen NetflixId=...` | Langsung verifikasi & generate token |
| Kosong | `/gen` | Error format |

Output link login mengikuti nilai token Netflix:
- 💻 Browser: `https://netflix.com/?nftoken=<token>`
- 📱 HP: `https://netflix.com/unsupported?nftoken=<token>`
- 📺 TV: `https://netflix.com/tv2?nftoken=<token>`

🎥 Random per region tersedia via tombol 🎲 Generate Random → pilih region (SEA / Asia / Europe / America / Global).

---

## ⚠️ Aturan & batasan

| Aturan | Nilai |
|---|---|
| Hanya file `.txt` (mime `text/plain` atau ekstensi `.txt`) | ✔ |
| Maks. akun per run convert | 40 (sisa dikirim ulang di batch berikutnya) |
| Verifikasi sebelum masuk Vault | Wajib (fetch token live per akun) |
| Auto-validasi cron | Tiap 30 menit, batch 40 — akun mati dihapus otomatis |
| Kuota user gratis | 10x/hari (reset 00:00 WIB) + jeda 1 menit |
| Convert & admin | Bebas limit/cooldown |

---

## 🔐 Checklist rahasia (jangan pernah commit)

- `BOT_TOKEN`, `SECRET_TOKEN`, `SETUP_KEY` → `npx wrangler secret put ...`
- Vars non-rahasia → GitHub repo **Settings → Secrets and variables → Actions** (dipakai workflow deploy) atau `.dev.vars` lokal.
