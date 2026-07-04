# AGENTS.md

## Tentang Project
Website registrasi "Open Recruitment Calon Asisten Fisika Laboratorium I" — form pendaftaran multi-step berbasis SPA client-side (bukan multi-page), untuk mahasiswa yang mendaftar jadi asisten lab.

Bahasa UI: Bahasa Indonesia, gaya santai/informal (contoh: "Semangat bre🔥🔥", "Dicentang dulu bre, baru bisa submit😌"). JANGAN ubah nada bahasa ini jadi formal kecuali diminta eksplisit.

## Struktur File
- `index.html` — semua halaman ada dalam satu file, ditampilkan/disembunyikan lewat class `.page` / `.page.active`, dikontrol fungsi `showPage(id)` di script.js
- `script.js` — semua logic: navigasi antar page, validasi per step, autosave ke localStorage, render tabel pengumuman
- `style.css` — pakai CSS variables di `:root` (--navy, --blue, --mid, --accent, --gray50/200/400/700, --red, --green, --text, --subtext), font Work Sans + Friz Quadrata dari CDN. Pakai variable yang sudah ada, jangan hardcode warna baru.
- `image/meme.png` — gambar di halaman sukses submit, jangan diubah/dihapus

## Halaman yang ada
1. `#page-lobby` — landing page, tombol "Isi Formulir"
2. `#page-form` — form 2 step: `#step-1` (data diri + pilihan judul) dan `#step-2` (upload dokumen)
3. `#page-success` — ditampilkan setelah submit
4. `#page-lolos-berkas` — pengumuman lolos seleksi berkas (data HARDCODE di array `pesertaBerkas` di script.js, belum terhubung backend)
5. `#page-lolos-final` — pengumuman final jadi aslab (data HARDCODE di array `aslabFinal`, belum terhubung backend)

Dua halaman pengumuman di atas sengaja belum dikerjakan — jangan disentuh dulu kecuali diminta eksplisit.

## ⚠️ Temuan penting — WAJIB diperbaiki saat integrasi backend
Sebagian besar input di Step 1 tidak punya `id`/`name`, jadi belum bisa dibaca untuk dikirim ke backend. Tambahkan `id` berikut (snake_case) ke elemen yang sesuai:

### Step 1 — data primitif
| Label di form | Kondisi HTML saat ini | id yang harus ditambahkan |
|---|---|---|
| Nama Lengkap | `<input type="text">` tanpa id | `nama_lengkap` |
| NRP | `<input type="text">` tanpa id | `nrp` |
| Email Aktif | `<input type="email">` tanpa id | `email` |
| Nomor WhatsApp | `<input type="tel">` tanpa id | `no_wa` |
| Motivasi Mendaftar | `<textarea>` tanpa id | `motivasi` |
| Skala Prioritas | radio `name="range"`, **tanpa `value`** | ganti name → `skala_prioritas`, tambahkan `value="1"` s.d. `"10"` sesuai urutan label |
| Bisa Tinkercad & Proteus | radio `name="opsi"`, **tanpa `value`** | ganti name → `bisa_tinkercad_proteus`, value=`"ya"` / `"tidak"` |
| IPK Terbaru | `<input type="number">` tanpa id | `ipk` |
| Judul Sebelum ETS - 1 | `<select id="judul-a1">` (sudah ada) | tetap `judul-a1` |
| Judul Sebelum ETS - 2 | `<select id="judul-a2">` (sudah ada) | tetap `judul-a2` |
| Judul Sesudah ETS - 1 | `<select id="judul-b1">` (sudah ada) | tetap `judul-b1` |
| Judul Sesudah ETS - 2 | `<select id="judul-b2">` (sudah ada) | tetap `judul-b2` |
| Penjelasan judul sebelum ETS | `<textarea>` tanpa id | `penjelasan_sebelum_ets` |
| Penjelasan judul sesudah ETS | `<textarea>` tanpa id | `penjelasan_sesudah_ets` |
| Kesediaan hadir 27-28 Agustus | radio `name="opsi1"`, **tanpa `value`** | ganti name → `kesediaan_hadir`, value=`"bersedia"` / `"tidak_bersedia"` |

### Step 2 — upload dokumen
| Label | `accept` saat ini | Maks ukuran | id yang harus ditambahkan |
|---|---|---|---|
| Curriculum Vitae | `.pdf` saja | 10 MB | `file_cv` |
| Transkrip Nilai | `.pdf,image/*` (bukan PDF-only!) | 5 MB | `file_transkrip` |
| Bukti Follow IG Madya | `.pdf,image/*` (bukan PDF-only!) | 5 MB | `file_bukti_follow` |

Catatan: hanya CV yang wajib PDF murni. Transkrip dan bukti follow bisa PDF **atau gambar** (JPG/PNG) — jangan dipaksa PDF-only saat validasi backend maupun saat setup Storage bucket.

Checkbox persetujuan sudah punya id: `#agree`.

## Bug yang harus dirapikan saat menyentuh fungsi terkait
- `nextStep`, `goBack`, `goHome` masing-masing dideklarasikan DUA KALI di script.js (deklarasi kedua menimpa yang pertama, jadi tidak error, tapi berantakan). Gabungkan jadi satu definisi kalau menyentuh fungsi ini.
- `setStep()` melakukan loop pill dari 1 sampai 4 (`for (let i = 1; i <= 4; i++)`), tapi HTML yang ada cuma punya 2 elemen `.form-step` (`step-1`, `step-2`). Konfirmasi ke user dulu sebelum mengubah/menghapus logic ini — jangan diasumsikan salah.

## Stack Backend
- Backend: Supabase (Postgres untuk data primitif, Storage untuk 3 file upload)
- Hosting: Vercel — project ini tetap vanilla HTML/JS, JANGAN dikonversi ke framework (Next.js/React/dll)
- Akses Supabase dari client pakai `supabase-js` lewat CDN `<script>` di index.html — jangan install via npm/bundler, project ini sengaja tanpa build step
- Config: `SUPABASE_URL` dan `SUPABASE_ANON_KEY`. anon key aman ditaruh di client, tapi taruh di satu tempat saja (bukan hardcode berulang) biar mudah diganti

## Konvensi
- Nama tabel & kolom Supabase pakai snake_case, konsisten dengan id di tabel atas
- Bucket Storage harus PRIVATE (bukan public) — admin mengakses file lewat signed URL, bukan URL langsung
- Jangan hapus logic `saveFormData` / `restoreFormData` / `initAutoSave` (autosave ke localStorage) yang sudah berjalan — pengiriman ke backend itu tambahan di atasnya, bukan pengganti
- `submitForm()` saat ini HANYA mengecek checkbox `#agree`, lalu langsung `clearFormData()` + pindah ke `#page-success` — belum ada pengiriman data sama sekali. Ini yang perlu diubah menjadi: validasi → upload 3 file ke Storage → insert satu baris ke tabel → baru clear & pindah ke halaman sukses. Tambahkan state loading (disable tombol submit) selama proses upload agar user tidak klik dua kali.

## Catatan komunikasi soal deployment
Saya belum pernah deploy website sama sekali. Kalau membahas atau membantu proses deployment (Vercel, environment variable, DNS, dll):
- Jelaskan SATU langkah per balasan, jangan sekaligus semua langkah
- Pakai bahasa sederhana, hindari istilah teknis tanpa penjelasan singkat
- Tunggu saya konfirmasi langkah sebelumnya berhasil sebelum kasih langkah berikutnya