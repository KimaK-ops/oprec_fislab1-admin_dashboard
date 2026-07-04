# AGENTS.md

## Tentang project
Website registrasi "Open Recruitment Calon Asisten Fisika Laboratorium I" — form pendaftaran multi-step berbasis SPA client-side (bukan multi-page), vanilla HTML/CSS/JS, tanpa build step.

Bahasa UI: Bahasa Indonesia, gaya santai/informal (contoh: "Semangat bre🔥🔥", "Dicentang dulu bre, baru bisa submit😌"). JANGAN ubah nada ini jadi formal kecuali diminta eksplisit.

## Struktur file
- `index.html` — semua halaman dalam satu file, ditampilkan/disembunyikan lewat class `.page` / `.page.active`, dikontrol `showPage(id)` di script.js
- `script.js` — semua logic: navigasi page, validasi per step, autosave localStorage, upload Storage, insert ke Supabase, render tabel pengumuman
- `style.css` — pakai CSS variables di `:root`, jangan hardcode warna baru. Variable yang tersedia: `--navy`, `--blue`, `--mid`, `--accent`, `--accent2`, `--white`, `--bg`, `--bg2`, `--surface`, `--gray50`, `--gray200`, `--gray400`, `--gray700`, `--red`, `--green`, `--text`, `--subtext`. Font: Work Sans + Friz Quadrata dari CDN.
- `image/meme.png` — gambar di halaman sukses submit, jangan diubah/dihapus
- `admin.html` + `admin.js` — BELUM ADA, lihat section Fase 2 di bawah

## Halaman (SPA, satu file)
1. `#page-lobby` — landing, tombol "Isi Formulir"
2. `#page-form` — form 2 step: `#step-1` (data diri + pilihan judul), `#step-2` (upload dokumen)
3. `#page-success` — setelah submit sukses
4. `#page-lolos-berkas` — pengumuman lolos berkas, **masih hardcode** dari array `pesertaBerkas` di script.js (belum fetch dari DB)
5. `#page-lolos-final` — pengumuman final aslab, **masih hardcode** dari array `aslabFinal` di script.js

Catatan: tombol back di `#step-2` HTML memanggil `nextStep(1)`, bukan `goBack()` — `goBack()`/`goHome()` ada di script.js tapi dipakai di konteks lain. Jangan asumsikan tombol back = `goBack()`.

## Supabase config (satu tempat di script.js)
- `SUPABASE_URL` dan `SUPABASE_ANON_KEY` dideklarasikan sekali di atas script.js (`script.js:17-18`). **Gunakan nama `SUPABASE_ANON_KEY`** konsisten di script.js dan admin.js nanti — jangan ganti jadi `SUPABASE_PUBLISHABLE_KEY` walau value-nya berawal `sb_publishable_`. Taruh config di satu tempat, jangan hardcode berulang.
- `BUCKET = 'pendaftar_dokumen'` (`script.js:19`, underscore — harus persis sama dengan nama bucket di Supabase Dashboard). Bucket Storage harus **PRIVATE** — akses file lewat signed URL, bukan URL langsung.
- Library `supabase-js@2` di-load via CDN `<script>` di index.html (`index.html:427`). JANGAN install via npm/bundler.
- **Secret key TIDAK dipakai** di project ini. Jangan taruh `SUPABASE_SECRET_KEY` di kode manapun (client atau serverless).

### Path pattern Storage
Setiap file diupload ke path: `<nrp>/<folder>_<timestamp>.<ext>`
- `folder` ∈ `cv` | `transkrip` | `bukti_follow`
- `ext` diturunkan dari MIME (`pdf` / `jpg` / `png` / `webp`)
- Lihat `uploadDokumen()` di `script.js:214-223`. Path disimpan di kolom `file_*_path` tabel.

## Skema tabel `pendaftar_aslab`
Inferensi dari `kumpulkanFormData()` (`script.js:226-250`) + insert (`script.js:291-296`). Verifikasi ke Supabase Dashboard jika ragu.

| kolom | tipe | sumber |
|---|---|---|
| `nama_lengkap` | text | input |
| `nrp` | text | input |
| `email` | text | input |
| `no_wa` | text | input |
| `motivasi` | text | textarea |
| `skala_prioritas` | int (1-10) | radio |
| `bisa_tinkercad_proteus` | boolean | radio ya/tidak |
| `ipk` | numeric (0-4) | number input |
| `judul_a1` / `judul_a2` | text | select (sebelum ETS) |
| `judul_b1` / `judul_b2` | text | select (sesudah ETS) |
| `penjelasan_sebelum_ets` | text | textarea |
| `penjelasan_sesudah_ets` | text | textarea |
| `kesediaan_hadir` | text (`bersedia` / `tidak_bersedia`) | radio |
| `file_cv_path` | text | Storage path |
| `file_transkrip_path` | text | Storage path |
| `file_bukti_follow_path` | text | Storage path |
| `status` | text/enum | **ditambah saat Fase 2, belum ada di kode/script.js sekarang** |

Nama kolom snake_case, konsisten dengan id input di HTML.

## Aturan upload (live constraint, jangan dilanggar)
- `file_cv` — **PDF saja**, maks 10 MB
- `file_transkrip` — PDF **atau gambar** (JPG/PNG/WEBP), maks 5 MB
- `file_bukti_follow` — PDF **atau gambar** (JPG/PNG/WEBP), maks 5 MB

Hanya CV yang wajib PDF murni. Validasi ada di `validateFile()` (`script.js:185-203`) dan `accept` attribute di HTML (`index.html:194,205,216`). Aturan ini juga berlaku saat setup Storage bucket dan logic preview/download di admin panel nanti.

## Autosave localStorage — JANGAN dihapus
`saveFormData()` / `restoreFormData()` / `initAutoSave()` (`script.js:378-455`) menyimpan draft form ke `localStorage` key `aslab_form`. Ini berjalan paralel dengan pengiriman backend — jangan diganti/dihapus, pengiriman ke Supabase adalah tambahan di atasnya, bukan pengganti.

## Submit flow (sudah implement)
`submitForm()` (`script.js:253-318`): cek `#agree` → `validateStep(1)` + `validateStep(2)` → validasi 3 file → disable tombol submit + state loading → upload 3 file ke Storage → insert 1 baris ke `pendaftar_aslab` → `clearFormData()` + pindah `#page-success`. Kalau insert gagal, file yang sudah terupload di-rollback via `storage.remove()`. Jangan ubah flow ini tanpa menjaga rollback-nya.

## Halaman pengumuman (scope terpisah, setelah Fase 2)
`#page-lolos-berkas` dan `#page-lolos-final` render dari array hardcode `pesertaBerkas` / `aslabFinal` (sekarang placeholder `'—'`). Setelah admin panel jadi dan kolom `status` dipakai, ubah kedua halaman ini jadi fetch read-only dari Supabase (filter by `status`, pakai `SUPABASE_ANON_KEY` seperti biasa). Jangan sentuh dulu kecuali diminta eksplisit.

## Deployment — cara komunikasi
Saya belum pernah deploy website. Kalau membahas/membantu deployment (Vercel, env variable, DNS, dll):
- Jelaskan **SATU langkah per balasan**, jangan sekaligus semua
- Pakai bahasa sederhana, hindari istilah teknis tanpa penjelasan singkat
- Tunggu konfirmasi langkah sebelumnya berhasil sebelum kasih langkah berikutnya

## Fase 2 — Admin panel (roadmap, belum dikerjakan)
`admin.html` + `admin.js` **belum dibuat**. Spec-nya:

- File terpisah dari `index.html`/`script.js`, JANGAN digabung ke file publik. Tidak di-link dari navigasi publik manapun.
- Fitur: login admin, lihat daftar pendaftar dari `pendaftar_aslab`, lihat/download 3 dokumen per pendaftar lewat signed URL, update status pendaftar (lolos berkas / lolos final).
- Autentikasi: **Supabase Auth** (email + password), BUKAN custom password check manual. Akun admin dibuat manual lewat Supabase Dashboard (Authentication > Users > Add user). **JANGAN bikin halaman signup admin yang terbuka ke publik.**
- Pakai `SUPABASE_ANON_KEY` yang sama (bukan secret key). Akses tambahan didapat dari sesi login via RLS, bukan dari key berbeda.

### RLS tambahan yang diperlukan (selain anon)
- SELECT di `pendaftar_aslab`: izinkan `authenticated`, larang `anon`
- UPDATE di `pendaftar_aslab` (ubah status lolos berkas/final): izinkan `authenticated`, larang `anon`
- SELECT di `storage.objects` bucket `pendaftar_dokumen`: izinkan `authenticated` generate signed URL, larang `anon`

### Keterkaitan ke halaman pengumuman
Setelah admin update kolom `status`, `index.html` (`#page-lolos-berkas`, `#page-lolos-final`) diubah dari array hardcode jadi fetch data asli dari Supabase (read-only, filter by `status`, pakai `SUPABASE_ANON_KEY`).
