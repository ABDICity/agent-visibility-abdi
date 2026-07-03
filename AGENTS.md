# AGENTS.md — Panduan Pengembangan Ekosistem ABDI

Dokumen operasional taktis untuk AI Agents dan Pengembang internal yang berkolaborasi dalam pengembangan **ABDI (Aplikasi Berjamaah Dakwah Islamicity 4B Kaffah: Berdakwah, Bersyariah, Berjamaah, Bermuamalah)**.

---

## 🏗️ Filosofi & Arsitektur Utama

Proyek ini dibangun di atas konsep **Single Content Store, Multi-Surface Projections** yang ditenagai oleh Cloudflare Workers AI dan KV Storage. 

Konten dakwah dan kemitraan ekonomi diekstraksi serta diperkaya (*enriched*) **hanya sekali** oleh kecerdasan buatan, lalu diproyeksikan ke berbagai gerbang data (`/llms.txt`, `/index.json`, `/<slug>.md`, JSON-LD) demi keterbacaan maksimal oleh AI dunia.

### 📁 Struktur Folder Utama
*   `src/worker/index.ts` : Aplikasi Hono (Framework API). Berisi rute untuk semua gerbang informasi (*surfaces*) jemaah + JSON API.
*   `src/enrichment/index.ts` : Pusat pengayaan Workers AI (Mengubah halaman dakwah mentah/HTML menjadi data *Resource* terstruktur).
*   `src/enrichment/surfaces.ts` : Fungsi perentalan murni (*pure render functions*), satu fungsi spesifik untuk setiap format gerbang data.
*   `src/lib/store.ts` : Sistem basis data lokal berbasis Cloudflare KV (Fungsi: *get / upsert / clear*).
*   `src/lib/content.ts` : Data sampel awal untuk demo aplikasi (Data kuliner UKM & artikel dakwah dasar).
*   `src/lib/types.ts` : Definisi tipe data bersama (*Shared types* seperti `Resource`, `RawResource`, `Env`).
*   `src/lib/web-bot-auth.ts` : *Pilihan* Modul verifikasi identitas AI (Mati secara bawaan, aktifkan jika memerlukan otentikasi ketat).
*   `src/react-app/` : Antarmuka eksplorasi gerbang data (*Surface-explorer UI*) untuk admin.
*   `test/index.test.ts` : Pengujian otomatis Worker via `SELF.fetch` (Menggunakan *vitest-pool-workers*).

---

## ⚖️ Aturan & Konvensi Pengembangan (Prinsip Kaffah)

1.  **Fungsi `surfaces.ts` Harus Murni (*Pure*):** Fungsi perentalan hanya menerima `RenderCtx` dan mengembalikan tipe data teks/objek. Tidak boleh ada proses I/O (Input/Output) atau pemanggilan basis data langsung di dalam file ini agar kode mudah diuji secara modular.
2.  **Sistem Pengayaan Tidak Boleh Gagal Total (*Zero Hard-Fail*):** Fungsi `enrichResource` wajib memiliki sistem cadangan (*fallbackEnrichment*) jika terjadi eror pada model AI global atau kegagalan *parsing* data. Informasi dakwah harus tetap tampil kepada jemaah meskipun dalam kondisi performa AI menurun (*degraded*).
3.  **Pemisahan Keterbacaan dan Identitas Keamanan:** Modul *Web Bot Auth* bertugas mengonfirmasi *siapa* bot yang datang, bukan mengatur *apa* yang boleh mereka baca. Modul ini wajib berdiri sendiri dan diisolasi dengan variabel `ENABLE_WEB_BOT_AUTH` agar tidak mengganggu performa jalur data utama.

---

## 🛠️ Langkah Cerdas Menambahkan Fitur Gerbang Data Baru (Surfaces)

Jika Anda ingin menambahkan format data baru (misal: `/jadwal-solat.json` atau `/kas-muamalah.txt`), ikuti 4 langkah terstruktur ini:

1.  **Tulis Render Murni:** Tambahkan fungsi perentalan baru di dalam file `src/enrichment/surfaces.ts`.
2.  **Daftarkan Jalur Rute:** Buat rute pemanggilan baru di `src/worker/index.ts`. Jangan lupa kirimkan header `Content-Signal` yang sesuai serta aktifkan middleware `cors()` jika data tersebut akan diambil secara lintas-domain oleh aplikasi frontend eksternal.
3.  **Visualisasikan di Dashboard Admin:** Masukkan rute baru Anda ke dalam daftar *surfaces* di endpoint `/api/site` agar muncul di antarmuka web pemantau.
4.  **Tulis Kode Pengujian:** Tambahkan unit pengujian baru di dalam berkas `test/index.test.ts`.

---

## 🎯 Validasi & Sinkronisasi Kode

Sebelum melakukan rilis kode (*deployment*) ke produksi, pastikan seluruh standarisasi teknis telah terpenuhi:

```bash
# 1. Jalankan proses kompilasi kode (TypeScript & Vite)
npm run build

# 2. Jalankan pengujian otomatis (Membutuhkan kredensial akses live Workers AI)
npm test

# 3. Sinkronisasikan ulang tipe data apabila Anda melakukan perubahan pada file wrangler.jsonc
npx wrangler types

=================================================================================================


# AGENTS.md — Agent Visibility Template

Notes for AI agents working on this template.

## What this template is

One enriched content store (`Resource[]`) projected onto many agent-discovery
surfaces. The data is enriched once by Workers AI and cached in KV; every
surface (`/llms.txt`, `/index.json`, `/<slug>.md`, `/robots.txt`, JSON-LD) is
just a different rendering of that same store.

## Architecture

```
src/
  worker/index.ts        Hono app: routes for every surface + JSON API
  enrichment/index.ts    Workers AI enrichment (raw page -> structured Resource)
  enrichment/surfaces.ts Pure render functions, one per surface
  lib/store.ts           KV-backed enriched store (get / upsert / clear)
  lib/content.ts         Sample content (zero-config demo data)
  lib/types.ts           Shared types (Resource, RawResource, Env, SiteConfig)
  lib/web-bot-auth.ts    OPTIONAL agent-identity module (off by default)
  react-app/             Surface-explorer UI
test/index.test.ts       Worker tests (vitest-pool-workers, via SELF.fetch)
```

## Conventions

- **`surfaces.ts` is pure.** Render functions take `RenderCtx` and return
  strings/objects. No I/O. This keeps surfaces easy to test and add to.
- **Enrichment must never hard-fail.** `enrichResource` falls back to
  `fallbackEnrichment` on any model/parse error so surfaces always render.
- **Keep readability and identity separate.** Web Bot Auth is about _who_ an
  agent is, not _what_ it can read. It lives in its own module and is gated by
  `ENABLE_WEB_BOT_AUTH`. Don't wire it into the core surfaces.

## Adding a surface

1. Add a pure renderer to `src/enrichment/surfaces.ts`.
2. Add a route in `src/worker/index.ts` (send the `Content-Signal` header for
   text/JSON surfaces; add `cors()` if agents fetch it cross-origin).
3. Add it to the `/api/site` `surfaces` list so the UI shows it.
4. Add a test in `test/index.test.ts`.

## Validating changes

```bash
npm run build   # tsc -b && vite build
npm test        # vitest (hits live Workers AI — needs credentials)
```

After editing `wrangler.jsonc`, rerun `npx wrangler types`.
