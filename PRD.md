# PRD — DBN Baking Center

**Versi:** 1.0
**Dibuat untuk:** Eksekusi oleh AI Coding (Claude Code / Sonnet 5) atau developer lain
**Status:** Siap eksekusi (MVP scope terkunci)

---

## 1. Overview

**Nama Project:** DBN Baking Center

**Deskripsi singkat:** Website marketplace kelas baking online single-brand (bukan multi-instructor/multi-seller). Pengguna bisa mendaftar, browsing katalog kelas, memasukkan beberapa kelas ke keranjang, checkout via Midtrans Snap, dan setelah bayar sukses langsung mendapat akses link Google Drive (video + materi PDF) untuk tiap kelas yang dibeli. Admin punya dashboard terpisah untuk mengelola kelas, kuota, transaksi, dan user.

**Tujuan:**
- Menjual kelas baking online secara digital dengan sistem pembayaran otomatis (tanpa transfer manual/konfirmasi manual).
- Delivery konten (video + resep) otomatis setelah pembayaran terverifikasi.
- Sistem harus **fungsional dan aman dulu**, tampilan visual dikerjakan belakangan (default: skema putih/merah/gold, teks hitam, mobile-first — bisa diimplementasi basic dulu).

**Target Pengguna:**
- **End-user:** individu yang ingin belajar baking, membeli akses 1 atau lebih kelas video sekaligus.
- **Admin (pemilik DBN):** mengelola kelas, memantau transaksi, mengelola user, tidak ada role instructor terpisah (single-brand).

**Skala Target Awal:** 1–10 user aktif dalam 3–6 bulan pertama. Arsitektur **wajib sederhana**, tidak boleh over-engineered (tidak perlu microservices, queue system berat, atau caching layer kompleks).

**Constraint Infrastruktur Kritis:**
- VPS: 1 vCPU, 1GB RAM, 20GB SSD storage.
- PostgreSQL local di VPS yang sama dengan aplikasi.
- Semua file (video thumbnail, asset, dsb — KECUALI video kelas & PDF resep yang di Google Drive) disimpan di storage VPS langsung, bukan cloud storage pihak ketiga.
- Domain baru dibeli di Domainesia, DNS akan diarahkan ke Cloudflare, SSL wajib aktif dari awal (lihat Section 6 untuk guide).
- Midtrans mode **Sandbox** selama development, baru ganti ke Production saat akan live.

---

## 2. Scope

### MVP (Wajib ada, harus selesai dan no-error sebelum lanjut fase berikutnya)
- Registrasi & login user dengan verifikasi email
- Login admin (role terpisah, akses dashboard admin)
- Katalog kelas (list + detail kelas: nama, deskripsi, harga, kuota tersisa, thumbnail)
- Keranjang (cart) — bisa tambah beberapa kelas sebelum checkout
- Reservasi kuota otomatis saat checkout dimulai (anti race condition/overselling)
- Checkout multi-kelas dalam 1 transaksi via Midtrans Snap
- Webhook Midtrans untuk update status transaksi otomatis (tanpa perlu cek manual)
- Setelah pembayaran sukses: sistem otomatis menampilkan link Google Drive per kelas yang dibeli di dashboard user
- Dashboard user: riwayat transaksi + daftar kelas yang sudah dimiliki + link drive masing-masing
- Review/rating kelas (hanya bisa oleh user yang sudah membeli kelas tsb)
- Dashboard admin: CRUD kelas (termasuk atur kuota & link Drive), lihat semua transaksi, lihat semua user
- Sistem logging transaksi & error (minimal log ke file, karena ini berkaitan uang)

### Fase Lanjutan (Nice-to-have, TIDAK dikerjakan di MVP)
- Sertifikat otomatis
- Reminder otomatis (email/notifikasi)
- Refund otomatis
- Link Google Drive unik per user (expiring link/permission per-email) — MVP tetap pakai **shared link statis per kelas**
- Multi-instructor / multi-brand
- Desain visual premium (MVP cukup layout bersih, styling detail belakangan)

### Eksplisit di luar scope (jangan dikerjakan kecuali diminta ulang)
- Migrasi storage video dari Google Drive ke storage lain
- Payment method selain Midtrans Snap
- Aplikasi mobile native

---

## 3. User Flow

### Flow Utama — End User
1. User membuka website → lihat katalog kelas (tanpa perlu login untuk browsing).
2. User daftar akun (email, password, nama) → menerima email verifikasi → klik link verifikasi → akun aktif.
3. User login.
4. User membuka detail kelas → klik "Tambah ke Keranjang" (bisa lebih dari satu kelas).
5. User buka halaman keranjang → review kelas yang dipilih beserta total harga.
6. User klik "Checkout" → sistem cek kuota real-time untuk semua kelas di keranjang.
   - Jika ada kelas yang kuotanya sudah habis → sistem tampilkan pesan, minta user hapus kelas tsb dari keranjang sebelum lanjut.
   - Jika semua kelas masih tersedia → sistem **reserve kuota** untuk semua kelas tsb (timeout 15 menit) dan membuat transaksi dengan status `pending`.
7. Sistem redirect ke Midtrans Snap popup/page.
8. User menyelesaikan pembayaran (atau membatalkan/timeout).
9. Midtrans mengirim notifikasi webhook ke server:
   - **Sukses** → status transaksi jadi `paid`, kuota kelas dikurangi permanen, link Drive tiap kelas jadi bisa diakses di dashboard user.
   - **Gagal/expired** → status jadi `failed`/`expired`, kuota yang direservasi dikembalikan ke pool.
10. User diarahkan ke halaman "Riwayat Transaksi" untuk melihat status.
11. Jika sukses, user buka "Kelas Saya" → lihat daftar kelas yang dimiliki beserta tombol/link ke Google Drive masing-masing.
12. User bisa memberi review/rating untuk kelas yang sudah dibeli.

### Flow Utama — Admin
1. Admin login (route/role terpisah dari user biasa).
2. Admin membuka dashboard → lihat ringkasan (jumlah kelas, transaksi terbaru, user terbaru).
3. Admin bisa: tambah kelas baru (nama, deskripsi, harga, kuota, thumbnail, link Google Drive), edit kelas, nonaktifkan kelas.
4. Admin bisa melihat semua transaksi (status, user, kelas yang dibeli, jumlah bayar) — read-only, tidak ada tombol "hapus data" tanpa konfirmasi eksplisit.
5. Admin bisa melihat daftar user (read-only untuk MVP; suspend/ban jadi opsional jika diminta).

---

## 4. Fitur Detail

### 4.1 Registrasi & Verifikasi Email
- **Input:** nama, email, password (min 8 karakter, kombinasi huruf+angka disarankan)
- **Proses:** simpan user dengan status `unverified`, generate token verifikasi (expiry 24 jam), kirim email berisi link verifikasi.
- **Output:** email terkirim, user tidak bisa login sebelum verifikasi.
- **Edge case:**
  - Email sudah terdaftar → tolak dengan pesan jelas.
  - Token expired → sediakan tombol "kirim ulang email verifikasi".
  - User klik link token yang sudah dipakai → tampilkan pesan "sudah terverifikasi, silakan login".
- **Catatan implementasi:** ikuti pola dari reference project login+email verification yang sudah ada.

### 4.2 Login
- **Input:** email, password
- **Proses:** validasi kredensial, cek status `verified`, buat session/JWT.
- **Output:** redirect ke dashboard sesuai role (user/admin).
- **Edge case:** password salah 5x berturut-turut → beri jeda/captcha sederhana (opsional tapi disarankan karena ini menyangkut uang & akun).

### 4.3 Katalog & Detail Kelas
- **Input:** tidak perlu login untuk lihat katalog.
- **Output:** list kelas (nama, harga, thumbnail, sisa kuota), detail kelas (deskripsi lengkap, review dari pembeli sebelumnya).
- **Edge case:** kuota 0 → tampilkan badge "Kelas Penuh", tombol beli dinonaktifkan.

### 4.4 Keranjang (Cart)
- **Input:** kelas yang dipilih user.
- **Proses:** simpan cart per user (di database, bukan hanya session, supaya persist jika user reload/logout-login).
- **Output:** list kelas di cart + subtotal.
- **Edge case:** kelas yang sama tidak bisa dimasukkan dua kali; jika user sudah pernah beli kelas tsb sebelumnya, tombol "tambah ke keranjang" diganti "Sudah dimiliki".

### 4.5 Checkout & Reservasi Kuota
- **Input:** daftar kelas di cart.
- **Proses:**
  1. Validasi ulang kuota semua kelas di cart (real-time, dalam satu database transaction).
  2. Jika semua tersedia → buat record transaksi status `pending`, kurangi kuota "tersedia" sementara (reserved), simpan `reserved_until` (now + 15 menit).
  3. Panggil Midtrans Snap API → dapatkan `snap_token` → kirim ke frontend.
- **Output:** Snap popup/redirect untuk pembayaran.
- **Edge case:**
  - Kuota berubah di antara load cart dan klik checkout → tolak, minta refresh cart.
  - Ada job/cron sederhana yang jalan tiap beberapa menit untuk membatalkan transaksi `pending` yang sudah lewat `reserved_until`, dan mengembalikan kuota.

### 4.6 Webhook Midtrans (Notification Handler)
- **Input:** payload notifikasi dari Midtrans (transaction_status, order_id, signature key).
- **Proses:** **wajib verifikasi signature key** sebelum memproses (keamanan kritis — cegah notifikasi palsu). Update status transaksi sesuai `transaction_status` Midtrans (`capture`/`settlement` → `paid`, `expire`/`cancel`/`deny` → `failed`).
- **Output:** jika `paid` → kuota dikurangi permanen (dari reserved jadi terjual), akses ke link Drive dibuka di akun user.
- **Edge case:** notifikasi duplikat dari Midtrans (bisa terjadi) → proses harus idempotent, jangan proses ulang jika status sudah final.
- **Catatan implementasi:** ikuti pola dari reference project Midtrans Sandbox yang sudah terbukti jalan 100%.

### 4.7 Akses Konten (Google Drive Delivery)
- **Proses:** setiap kelas punya 1 link Google Drive statis (folder berisi video + PDF resep) yang diisi admin saat membuat/edit kelas.
- **Output:** setelah transaksi `paid`, link tsb muncul di halaman "Kelas Saya" milik user, khusus untuk kelas-kelas yang ada di transaksi tsb.
- **Edge case:** jika transaksi berisi 3 kelas, tampilkan 3 link terpisah, jangan digabung jadi satu.
- **Catatan keamanan:** karena link shared, dokumentasikan risiko (link bisa disebar user) sebagai known limitation MVP, bukan bug.

### 4.8 Review/Rating
- **Input:** rating (1-5) + komentar teks, hanya untuk kelas yang dimiliki user (status transaksi `paid`).
- **Output:** rating rata-rata tampil di halaman detail kelas.
- **Edge case:** user hanya bisa review 1x per kelas (bisa edit review sendiri, tidak bisa buat baru).

### 4.9 Dashboard Admin
- **Fitur:** CRUD kelas, lihat semua transaksi (dengan filter status), lihat semua user.
- **Edge case:** hapus kelas yang sudah pernah dibeli user → **tidak boleh hard-delete**, gunakan soft-delete (flag `is_active = false`) supaya histori transaksi & akses user yang sudah beli tidak rusak.

---

## 5. Data Model

Entitas utama (PostgreSQL):

**users**
- id, name, email (unique), password_hash, role (`user`/`admin`), is_verified, verification_token, verification_token_expires, created_at

**classes**
- id, title, description, price, quota_total, quota_available, thumbnail_path, gdrive_link, is_active, created_at, updated_at

**carts**
- id, user_id (FK), created_at

**cart_items**
- id, cart_id (FK), class_id (FK), added_at

**transactions**
- id, user_id (FK), midtrans_order_id (unique), total_amount, status (`pending`/`paid`/`failed`/`expired`), reserved_until, created_at, updated_at

**transaction_items**
- id, transaction_id (FK), class_id (FK), price_at_purchase

**reviews**
- id, user_id (FK), class_id (FK), rating (1-5), comment, created_at

**Relasi kunci:**
- 1 user → banyak transactions
- 1 transaction → banyak transaction_items (multi-kelas per transaksi)
- 1 class → banyak transaction_items & reviews
- quota_available di tabel `classes` di-update via database transaction (row lock) saat reservasi & saat webhook diproses, untuk mencegah race condition (dua user checkout bersamaan berebut slot terakhir).

---

## 6. Desain & UX Guidelines

- **Prioritas:** fungsi dan keamanan dulu, visual belakangan. Boleh pakai layout HTML/EJS polos tanpa styling detail di tahap awal, asal semua flow berjalan tanpa error.
- **Skema warna (untuk dipakai saat tahap styling nanti):** putih/merah/gold sebagai warna utama, teks hitam. Mobile-first (breakpoint mobile jadi prioritas layout).
- Tidak perlu referensi kompetitor spesifik — cukup ikuti pola umum marketplace kelas online (katalog grid, halaman detail, cart, checkout).

### Panduan Domain & SSL (Domainesia → Cloudflare)
Karena domain baru dibeli di Domainesia dan rencana pakai Cloudflare:
1. Login ke akun Cloudflare → "Add a Site" → masukkan domain dari Domainesia.
2. Cloudflare akan memberi 2 nameserver (misal `xxx.ns.cloudflare.com`).
3. Login ke panel Domainesia → menu domain → ubah nameserver domain ke 2 nameserver dari Cloudflare tsb (bukan nameserver default Domainesia).
4. Tunggu propagasi DNS (bisa 1-24 jam, biasanya lebih cepat).
5. Setelah aktif di Cloudflare, tambahkan DNS record type `A` mengarah ke IP publik VPS.
6. Di Cloudflare, aktifkan SSL/TLS mode **"Full"** (bukan "Flexible") jika nanti Nginx di VPS juga pasang sertifikat sendiri, atau **"Flexible"** dulu untuk testing awal jika belum sempat pasang Let's Encrypt di server (lebih cepat tapi koneksi Cloudflare→VPS tidak terenkripsi).
7. Untuk production sebelum live dengan uang asli: upgrade ke SSL "Full (Strict)" + pasang Let's Encrypt via Certbot + Nginx reverse proxy di VPS.
8. Midtrans Snap membutuhkan HTTPS aktif di redirect URL & notification URL — pastikan langkah 6-7 selesai sebelum test transaksi Sandbox end-to-end dari domain publik.

---

## 7. Tech Stack

| Komponen | Pilihan | Alasan |
|---|---|---|
| Backend | Node.js + Express | Sudah jadi preferensi awal, ekosistem luas, ringan untuk VPS kecil |
| Templating | EJS | Server-side render sederhana, cocok untuk VPS 1GB RAM (hindari build step SPA berat) |
| Database | PostgreSQL (local di VPS) | Sudah diputuskan, cocok untuk relasi transaksi & kuota yang butuh integritas data (ACID) |
| Payment | Midtrans Snap (Sandbox → Production) | Sudah ada reference project yang terbukti jalan |
| Auth | Session-based (express-session) — direkomendasikan dibanding JWT | Lebih simpel untuk di-debug solo, tidak perlu refresh token logic |
| Email verifikasi | Nodemailer (SMTP) | Sesuai reference project login yang sudah ada |
| Reverse proxy + SSL | Nginx + Certbot (Let's Encrypt) | Standar, ringan, gratis |
| Storage asset lokal | Filesystem VPS langsung (folder `/public/uploads` atau sejenis) | Sesuai keputusan, hemat biaya, cukup untuk skala kecil |
| Process manager | PM2 | Auto-restart aplikasi jika crash, penting karena solo maintainer tanpa monitoring tim |

**Catatan constraint teknis wajib (karena VPS 1 vCPU/1GB RAM):**
- Batasi PostgreSQL `max_connections` dan connection pool aplikasi (misal pool size 5-10, jangan default besar).
- Sediakan swap file minimal 1-2GB di VPS untuk mencegah OOM kill saat traffic naik sedikit.
- Hindari dependency Node.js yang berat/tidak perlu (audit `package.json` secara berkala).
- Jangan jalankan proses build-heavy (seperti webpack dev server) di production VPS yang sama.

---

## 8. Rules & Constraints (Wajib diikuti AI Coding)

**Struktur folder (rekomendasi):**
```
/src
  /controllers
  /routes
  /models
  /middlewares
  /views (EJS)
  /public (CSS, JS, uploads)
  /utils
  /config
/tests
.env.example
```

**Aturan wajib:**
1. **Jangan pernah hardcode API key, secret Midtrans, kredensial database, atau SMTP credentials** — semua wajib lewat `.env`, dan `.env` wajib masuk `.gitignore`. Sediakan `.env.example` sebagai template.
2. **Verifikasi signature key Midtrans di webhook wajib ada** — tanpa ini, transaksi bisa dipalsukan.
3. **Semua operasi kuota (reservasi & pengurangan) wajib pakai database transaction dengan row lock** (`SELECT ... FOR UPDATE` di PostgreSQL) untuk mencegah race condition saat 2+ user checkout bersamaan.
4. **Dilarang hard-delete** data kelas, user, atau transaksi yang sudah punya relasi — gunakan soft-delete (flag `is_active`/`deleted_at`).
5. **Dilarang menghapus/mengubah data transaksi tanpa konfirmasi eksplisit** dari admin (tidak ada tombol "hapus transaksi" otomatis di MVP).
6. Setiap fitur yang menyentuh uang (checkout, webhook, update status transaksi) **wajib dites manual dengan checklist** sebelum dianggap selesai — lihat Section 10 Definition of Done.
7. Konvensi penamaan: `camelCase` untuk variabel/fungsi JS, `snake_case` untuk nama tabel & kolom database, commit message format `feat: ...` / `fix: ...` / `chore: ...`.
8. Batasan dependency: hindari menambah package baru kecuali benar-benar diperlukan (VPS kecil, hindari bloat). Setiap penambahan dependency baru harus disebutkan alasannya di commit message atau README.
9. Semua error di proses pembayaran & webhook wajib di-log (minimal ke file log lokal, format timestamp + detail error) — jangan biarkan error transaksi silent-fail.
10. Dokumentasi wajib di-update setiap milestone selesai (lihat Section 9) — README.md harus selalu mencerminkan cara install, cara jalankan, dan struktur env variable terbaru, karena project ini akan dilanjutkan oleh AI/dev lain.

---

## 9. Milestone / Roadmap

Checklist bertahap, tiap tahap harus **selesai & tanpa error** sebelum lanjut ke tahap berikutnya.

- [ ] **M0 — Setup Awal**
  - Setup repo, struktur folder, `.env.example`, koneksi PostgreSQL lokal di VPS
  - Setup Nginx + domain (Domainesia → Cloudflare) + SSL dasar (boleh mode Flexible dulu)

- [ ] **M1 — Auth**
  - Registrasi + verifikasi email (ikuti reference project login)
  - Login/logout, session management
  - Role-based access (user vs admin)

- [ ] **M2 — Katalog Kelas**
  - CRUD kelas di sisi admin
  - Tampilan katalog & detail kelas di sisi user

- [ ] **M3 — Cart & Reservasi Kuota**
  - Tambah/hapus kelas dari cart
  - Logic reservasi kuota (15 menit) dengan row lock
  - Cron job pembatalan reservasi expired

- [ ] **M4 — Integrasi Midtrans**
  - Buat transaksi + panggil Snap API (ikuti reference project Midtrans)
  - Handle webhook notifikasi + verifikasi signature
  - Update status transaksi & kuota otomatis

- [ ] **M5 — Delivery Konten**
  - Halaman "Kelas Saya" menampilkan link Drive per kelas yang dibeli
  - Riwayat transaksi user

- [ ] **M6 — Review & Admin Dashboard Lengkap**
  - Fitur review/rating
  - Dashboard admin: lihat transaksi & user

- [ ] **M7 — Hardening & Testing End-to-End**
  - Full test alur: registrasi → beli multi-kelas → bayar sandbox → cek akses drive
  - Setup SSL Full (Strict) + Let's Encrypt untuk production-ready
  - Review checklist keamanan (Section 8)

- [ ] **M8 — Dokumentasi Final**
  - README lengkap (cara install, environment variable, cara deploy ulang)
  - Dokumentasi arsitektur singkat untuk dev/AI berikutnya

---

## 10. Definition of Done

**Per fitur dianggap selesai jika:**
- Fitur berjalan sesuai flow di Section 3 tanpa error di console/log.
- Semua edge case yang disebutkan di Section 4 sudah ditangani (bukan cuma happy path).
- Untuk fitur yang menyentuh uang (checkout, webhook, kuota): sudah dites minimal dengan skenario — (a) bayar sukses, (b) bayar gagal/expired, (c) dua transaksi bersamaan rebutan kuota terakhir — dan hasilnya sesuai ekspektasi tanpa data korup/dobel.
- Tidak ada API key/secret yang ter-hardcode atau ter-commit ke git.
- Perubahan struktur database (migration) sudah didokumentasikan.

**Per milestone (M0–M8) dianggap selesai jika:**
- Semua checklist di dalam milestone tsb tercentang.
- README/dokumentasi sudah diperbarui mencerminkan progres milestone tsb.
- Tidak ada known bug kritis yang mempengaruhi alur pembayaran atau keamanan akun.

**Project keseluruhan dianggap "siap production" jika:**
- M0–M8 selesai semua.
- SSL sudah Full (Strict), bukan Flexible.
- Midtrans sudah siap di-switch ke Production key (tapi tetap Sandbox sampai benar-benar akan live).
- Dokumentasi cukup lengkap sehingga dev/AI lain bisa lanjutkan tanpa perlu tanya ulang ke pemilik project.
