# AGENTS.md

## Tentang project
Website registrasi "Open Recruitment Calon Asisten Fisika Laboratorium I" — SPA client-side (satu `index.html`), vanilla HTML/CSS/JS, **tanpa build step, tanpa package.json, tanpa test/lint config**. Edit langsung di file, buka di browser.

Bahasa UI: Bahasa Indonesia, gaya santai/informal (contoh: "Semangat bre🔥🔥", "Dicentang dulu bre, baru bisa submit😌"). JANGAN ubah nada ini jadi formal kecuali diminta eksplisit.

## Struktur file
- `index.html` — semua halaman publik dalam satu file, ditampilkan/disembunyikan lewat class `.page` / `.page.active`, dikontrol `showPage(id)` di script.js
- `script.js` — semua logic publik: navigasi page, validasi per step, autosave localStorage, upload Storage, insert ke Supabase, render tabel pengumuman
- `style.css` — pakai CSS variables di `:root`, jangan hardcode warna baru. Variable yang tersedia: `--navy`, `--blue`, `--mid`, `--accent`, `--accent2`, `--white`, `--bg`, `--bg2`, `--surface`, `--gray50`, `--gray200`, `--gray400`, `--gray700`, `--red`, `--green`, `--text`, `--subtext`. Font: Work Sans + Friz Quadrata dari CDN.
- `image/meme.png` — gambar di `#page-success`, jangan diubah/dihapus
- `admin.html` + `admin.js` + `admin.css` — admin panel, file terpisah, tidak di-link dari navigasi publik manapun. Lihat section "Admin Panel".

## Halaman (SPA, satu file)
1. `#page-lobby` — landing, tombol "Isi Formulir"
2. `#page-form` — form 2 step: `#step-1` (data diri + pilihan judul), `#step-2` (upload dokumen + checkbox `#agree`)
3. `#page-success` — setelah submit sukses
4. `#page-lolos-berkas` — pengumuman lolos berkas, **masih hardcode** dari array `pesertaBerkas` di script.js
5. `#page-lolos-final` — pengumuman final aslab, **masih hardcode** dari array `aslabFinal` di script.js

Gotcha navigasi:
- Tombol back di `#step-2` HTML memanggil `nextStep(1)`, **bukan** `goBack()`. `goBack()`/`goHome()` (di script.js) dipakai di konteks lain dan keduanya memanggil `clearFormData()` dulu. Jangan asumsikan tombol back = `goBack()`.
- Tombol ke `#page-lolos-berkas` / `#page-lolos-final` **dikomentari** di `#page-lobby` (blok `<!-- ... -->`). Kedua halaman ada di DOM tapi tidak reachable dari UI publik saat ini — hanya via `showPage('...')` langsung.

## Supabase config (satu tempat di script.js)
- `SUPABASE_URL` dan `SUPABASE_ANON_KEY` dideklarasikan sekali di atas script.js (blok `/* ── Supabase config ── */`). **Gunakan nama `SUPABASE_ANON_KEY`** konsisten di script.js dan admin.js — jangan ganti jadi `SUPABASE_PUBLISHABLE_KEY` walau value-nya berawal `sb_publishable_`. Taruh config di satu tempat, jangan hardcode berulang.
- `BUCKET = 'pendaftar_dokumen'` (underscore — harus persis sama dengan nama bucket di Supabase Dashboard). Bucket Storage harus **PRIVATE** — akses file lewat signed URL, bukan URL langsung.
- Library `supabase-js@2` di-load via CDN `<script>` di **index.html dan admin.html**. JANGAN install via npm/bundler.
- **Secret key TIDAK dipakai** di project ini. Jangan taruh `SUPABASE_SECRET_KEY` di kode manapun.

### Path pattern Storage
Setiap file diupload ke path: `<nrp>/<folder>_<timestamp>.<ext>`
- `folder` ∈ `cv` | `transkrip` | `bukti_follow`
- `ext` diturunkan dari MIME (`pdf` / `jpg` / `png` / `webp`)
- Lihat fungsi `uploadDokumen(nrp, file, folder, jenis)` di script.js. Path disimpan di kolom `file_*_path` tabel.

## Skema tabel `pendaftar_aslab`
Inferensi dari `kumpulkanFormData()` + insert di `submitForm()` + admin.js (`fetchPendaftar`/`updateStatus`/`bukaModalDokumen`). Verifikasi ke Supabase Dashboard jika ragu.

| kolom | tipe | sumber |
|---|---|---|
| `id` | uuid/serial (PK) | auto — dipakai admin.js untuk `.eq('id', id)` update & key baris tabel |
| `created_at` | timestamptz | auto — dipakai admin.js untuk `.order('created_at', { ascending: false })` |
| `nama_lengkap` | text | input |
| `nrp` | text | input |
| `email` | text | input |
| `no_wa` | text | input |
| `motivasi` | text | textarea |
| `skala_prioritas` | int (1-10) | radio, `parseInt` di `kumpulkanFormData` |
| `bisa_tinkercad_proteus` | boolean | radio ya/tidak, disimpan sebagai bool |
| `ipk` | numeric (0-4) | number input, `parseFloat` |
| `judul_a1` / `judul_a2` | text | select (sebelum ETS) |
| `judul_b1` / `judul_b2` | text | select (sesudah ETS) |
| `penjelasan_sebelum_ets` | text | textarea |
| `penjelasan_sesudah_ets` | text | textarea |
| `kesediaan_hadir` | text (`bersedia` / `tidak_bersedia`) | radio |
| `file_cv_path` | text | Storage path |
| `file_transkrip_path` | text | Storage path |
| `file_bukti_follow_path` | text | Storage path |
| `status` | text/enum | admin.js only (read di `renderTabel` + update di `updateStatus`). **Tidak ada di script.js** — form publik tidak pernah menulis kolom ini. Default `pending`. |

Nama kolom snake_case. Catatan: id input HTML untuk judul pakai kebab (`judul-a1`) → dipetakan manual ke kolom snake `judul_a1` di `kumpulkanFormData`.

## Aturan upload (live constraint, jangan dilanggar)
- `file_cv` — **PDF saja**, maks 10 MB
- `file_transkrip` — PDF **atau gambar** (JPG/PNG/WEBP), maks 5 MB
- `file_bukti_follow` — PDF **atau gambar** (JPG/PNG/WEBP), maks 5 MB

Hanya CV yang wajib PDF murni. Validasi ada di `validateFile(file, jenis)` di script.js dan `accept` attribute di HTML. Aturan ini juga berlaku saat setup Storage bucket dan logic preview/download di admin panel.

### Gotcha upload (sedang tidak konsisten — perbaiki sebelum test end-to-end)
- **Input file di index.html belum punya `id`**, tapi `submitForm()` membacanya via `getElementById('file_cv' | 'file_transkrip' | 'file_bukti_follow')`. Tanpa id, `getElementById` null → `.files[0]` throw TypeError. Kalau upload error "Cannot read properties of null", ini penyebabnya — tambahkan `id` ke 3 `<input type="file">` di `#step-2`.
- **Hint text di HTML salah**: ketiga upload-area nulis "Maks. 10 MB", padahal `validateFile` membatasi transkrip & bukti_follow ke 5 MB. Update hint HTML biar konsisten dengan constraint.

## Autosave localStorage — JANGAN dihapus
`saveFormData()` / `restoreFormData()` / `initAutoSave()` menyimpan draft form ke `localStorage` key `aslab_form`. Berjalan paralel dengan pengiriman backend — jangan diganti/dihapus, pengiriman ke Supabase adalah tambahan di atasnya, bukan pengganti. `clearFormData()` dipanggil saat submit sukses dan di `goBack()`/`goHome()`.

## Submit flow (sudah implement)
`submitForm()`: cek `#agree` → `validateStep(1)` + `validateStep(2)` → validasi 3 file via `validateFile` → disable tombol submit + state loading → upload 3 file ke Storage → insert 1 baris ke `pendaftar_aslab` → `clearFormData()` + pindah `#page-success`. Kalau insert gagal, file yang sudah terupload di-rollback via `storage.remove(uploadedPaths)`. Jangan ubah flow ini tanpa menjaga rollback-nya.

## Halaman pengumuman (scope terpisah, setelah Fase 2)
`#page-lolos-berkas` dan `#page-lolos-final` render dari array hardcode `pesertaBerkas` / `aslabFinal` (sekarang placeholder `'—'`). Setelah admin panel dipakai dan kolom `status` terisi, ubah kedua halaman ini jadi fetch read-only dari Supabase (filter by `status`, pakai `SUPABASE_ANON_KEY`). Butuh RLS anon SELECT untuk `pendaftar_aslab` agar halaman publik bisa baca — pasang saat halaman siap di-wire. Jangan sentuh dulu kecuali diminta eksplisit.

## Git remotes (repo asli; snapshot kerja ini tidak berisi `.git`)
- `origin` → `KimaK-ops/oprec_fislab1-admin_dashboard` (fork yang ada admin panel; push ke sini)
- `upstream` → `KimaK-ops/OpenRecruitment_Fislab1` (repo publik asli tanpa admin)
- `.gitattributes` LF-normalize auto.
- Verifikasi remote via `git remote -v` di repo sebenarnya.

## Deployment — cara komunikasi
Saya belum pernah deploy website. Kalau membahas/membantu deployment (Vercel, env variable, DNS, dll):
- Jelaskan **SATU langkah per balasan**, jangan sekaligus semua
- Pakai bahasa sederhana, hindari istilah teknis tanpa penjelasan singkat
- Tunggu konfirmasi langkah sebelumnya berhasil sebelum kasih langkah berikutnya

## Admin Panel (Fase 2, sudah dibuat)
`admin.html` + `admin.js` + `admin.css` — sudah dibuat dan berjalan.

- File terpisah dari `index.html`/`script.js`, tidak di-link dari navigasi publik manapun. `admin.html` me-load `style.css` lalu `admin.css` (admin.css meng-override).
- Fitur: login admin (Supabase Auth), lihat daftar pendaftar dari `pendaftar_aslab`, lihat/download 3 dokumen per pendaftar lewat signed URL (expiry 300 detik), lihat detail pendaftar, update status pendaftar lewat dropdown.
- Autentikasi: **Supabase Auth** (email + password) via `signInWithPassword()`. Akun admin dibuat manual lewat Supabase Dashboard (Authentication > Users > Add user). **TIDAK ada halaman signup.**
- Cek sesi saat `DOMContentLoaded` via `checkSession()` (`getSession()`) — kalau ada langsung ke `page-dashboard`, kalau null tampilkan `page-login`.
- Config (`SUPABASE_URL`/`SUPABASE_ANON_KEY`/`BUCKET`) dideklarasi ulang di `admin.js` — **harus sama persis dengan script.js**. Kalau ubah di satu file, ubah juga yang lain.
- Status enum (5 state, lihat `STATUS_OPTS` di admin.js): `pending` / `lolos_berkas` / `gagal` / `lolos_final` / `ditolak_final`.
- Cache pendaftar di `pendaftarCache` (untuk filter search tanpa re-fetch). Catatan penamaan: `filterTabel` (satu 'l') di admin.js vs `filterTable` (dua 'l') di script.js untuk halaman publik — jangan tertukar saat grep.

### RLS yang sudah dipasang
- SELECT di `pendaftar_aslab`: `authenticated` boleh, `anon` larang (kecuali untuk halaman pengumuman nanti)
- UPDATE di `pendaftar_aslab`: `authenticated` boleh, `anon` larang
- SELECT di `storage.objects` bucket `pendaftar_dokumen`: `authenticated` boleh generate signed URL, `anon` larang

### Keterkaitan ke halaman pengumuman (scope terpisah, belum dikerjakan)
Setelah admin update status, ubah `#page-lolos-berkas` / `#page-lolos-final` jadi fetch read-only dari Supabase (filter by `status`: `lolos_berkas` untuk `#page-lolos-berkas`, `lolos_final` untuk `#page-lolos-final`, pakai `SUPABASE_ANON_KEY`). Butuh RLS anon SELECT untuk `pendaftar_aslab` — pasang setelah halaman pengumuman siap di-wire.
