# Vidgen Docs (Mintlify)

Proyek dokumentasi berdiri sendiri untuk **docs.vidgen.id**. Folder ini sengaja terpisah
dari `apps/web`: satu-satunya dependency-nya adalah Mintlify, jadi bisa di-push ke repo
terpisah dan di-deploy tanpa menyentuh aplikasi utama.

Subjek dokumennya adalah lima endpoint publik di `apps/web/src/app/api/v1/` — itu-satu-satunya
permukaan API yang boleh dijanjikan ke pelanggan.

## Struktur

```text
docs-site/
├── docs.json                 konfigurasi Mintlify (tema, nav bilingual, playground)
├── openapi.json              spec OpenAPI 3.1 (artefak bantu, tidak dipakai build)
├── index.mdx                 beranda / landing
├── quickstart.mdx introduction.mdx register.mdx authentication.mdx api-base-url.mdx
├── concepts/                 models, credits, reference-media, concurrency-limits, idempotency
├── guides/                   nodejs, python, handling-results, motion-control
├── api-reference/            overview, generate, get-job, upload, models, credits
├── troubleshooting/          errors, faq
├── en/                       cermin Bahasa Inggris (struktur sama persis)
├── logo/{light,dark}.svg     wordmark Vidgen
└── favicon.svg
```

Bahasa default adalah Indonesia (entri pertama di `navigation.languages`), dengan switcher
ke `en/`. Tiap halaman Inggris adalah terjemahan 1:1 dari halaman Indonesia-nya — kalau kamu
mengubah yang satu, ubah yang lain.

## Pratinjau lokal

```bash
cd docs-site
npx mintlify dev
```

Buka `http://localhost:3000`. Kalau portnya bertabrakan dengan dev server aplikasi,
`npx mintlify dev --port 3300`.

Validasi cepat tanpa Mintlify:

```bash
node -e "JSON.parse(require('fs').readFileSync('docs.json','utf8')); console.log('docs.json OK')"
```

## Deploy ke Mintlify

1. **Push folder ini ke repo sendiri.** Isi `docs-site/` adalah akar proyek Mintlify, jadi
   `docs.json` harus berada di root repo tujuan.
   ```bash
   cd docs-site
   git init && git add -A && git commit -m "docs: Vidgen API documentation"
   git remote add origin https://github.com/<akun>/<repo-docs>.git
   git push -u origin main
   ```
   Alternatif: tetap di repo Vidgen dan tunjuk Mintlify ke folder `docs-site` saat membuat
   proyek (dashboard menanyakan path root). Pisah repo lebih disarankan supaya riwayat
   dokumentasi tidak bercampur dengan riwayat rilis aplikasi.
2. **Import di [mintlify.com](https://www.mintlify.com)** → *Create site* → pilih repo →
   konfirmasi path root. Mintlify membaca `docs.json` dan membangun situs.
3. **Sambungkan domain** di *Settings → Custom domain*, tambahkan `docs.vidgen.id`.
4. **Record DNS** di penyedia domain `.id` kamu:

   | Type | Name | Value |
   | --- | --- | --- |
   | CNAME | `docs` | `cname.mintlify.builders` |

   Sertifikat TLS dibuat otomatis setelah rekor terpropagasi. Jangan buat rekor `A`/`ALIAS`
   untuk subdomain ini; CNAME saja.
5. Halaman `/docs` internal di aplikasi `apps/web` sudah **dihapus** dan dibalut redirect 308
   ke sini (`next.config.ts`). Domain docs ditulis di satu tempat, `DOCS_URL` di
   `apps/web/src/lib/seo.ts`, dan dipakai ulang oleh sidebar, kartu API Key, dan 404.
   Kalau domain docs berubah, samakan `DOCS_URL`, `next.config.ts`, dan
   `docs.json` → `seo.metatags.canonical` sekaligus.

## Mengganti base URL

Origin produksi ditulis di beberapa tempat. Kalau domain akhir berbeda dari `vidgen.id`,
ubah semuanya sekaligus:

```bash
grep -rn "vidgen.id" docs-site --include=*.mdx --include=*.json
```

Yang wajib disamakan:

| Lokasi | Nilai | Fungsi |
| --- | --- | --- |
| `docs.json` → `api.mdx.server` | `https://vidgen.id` | origin yang dipakai playground/API reference |
| `docs.json` → `logo.href`, `navbar`, `footer` | `https://vidgen.id/...` | tautan ke produk |
| `openapi.json` → `servers[0].url` | `https://vidgen.id` | klien yang dihasilkan spec |
| `*.mdx` | `https://vidgen.id/api/v1/...` | contoh perintah di semua halaman |

## Menjaga dokumen tetap benar

Dokumen ini dibuat dengan membaca kode, bukan asumsi. Saat salah satu hal berikut berubah,
halaman terkait wajib ikut berubah:

| Perubahan di kode | Halaman yang harus ikut |
| --- | --- |
| `MAX_USER_CONCURRENT`, `DAILY_VIDEO_LIMIT`, `MAX_QUEUE_SIZE` | `concepts/concurrency-limits`, `api-reference/credits`, `introduction`, `index`, `troubleshooting/errors` |
| Batas upload (100 MB, 60 berkas/jam, rentang durasi, ambang resolusi) di `api/v1/upload/route.ts` + `lib/media/probe.ts` | `concepts/reference-media`, `api-reference/upload`, `guides/motion-control` |
| Field body / `parameters` yang dikenali di `api/v1/generate/route.ts` + `validateParameters` | `api-reference/generate`, `concepts/models` |
| Bentuk respons di `api/v1/*` | seluruh `api-reference/*` + `openapi.json` |
| Format/panjang prefix API key di `lib/auth/api-keys.ts` | `authentication`, `register` |
| Aturan tagih & pengembalian kredit (`costOf`, refund) | `concepts/credits` |
| Status `JobStatus` | `introduction`, `api-reference/get-job`, `openapi.json` |

Pemeriksaan cepat bahwa path yang didokumentasikan masih ada:

```bash
ls apps/web/src/app/api/v1
```

## Catatan implementasi

- **Playground interaktif** memakai `api.mdx` (bukan impor `openapi.json`), sehingga setiap
  halaman referensi bisa punya penjelasan panjang di samping form request. `openapi.json`
  disediakan sebagai artefak untuk menghasilkan klien; tidak wired ke navigation.
- **`authMethod: 'none'`** di `api-reference/models.mdx` sesuai kode: endpoint katalog memang
  tidak memanggil `authenticateApiKey`. Kalau kelak diberi autentikasi, hapus baris itu.
- **CORS tidak dikirim** oleh API, dan dokumen sengaja mengatakannya di muka supaya
  calon integrator tidak menghabiskan setengah jam untuk kesalahan frontend.
- **`navigation.languages` dengan kode `id`**: kalau validator Mintlify menolak kombinasi
  bahasa ini, hapus pembungkus `languages` dan sisakan hanya `tabs` berbahasa Indonesia, lalu
  pindahkan folder `en/` ke luar proyek. Situs tetap jalan penuh.

## Yang sengaja tidak didokumentasikan

Endpoint berautentikasi sesi cookie (`/api/generate`, `/api/transcribe`, `/api/storyboard/*`,
`/api/watermark/*`, `/api/face-mask/*`, `/api/admin/*`, `/api/payments/*`) bukan bagian dari
produk API publik. Permintaan ke halaman-halaman itu dari integrasi eksternal tidak didukung,
dan dokumentasinya ada di internal saja. Nama sistem di belakang Vidgen juga tidak pernah
muncul di dokumen mana pun.
