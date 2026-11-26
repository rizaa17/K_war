# Tech Stack dan Arsitektur Sistem - K_War
Versi 2.4 - 26 November 2026
Dokumen ini adalah sumber kebenaran. Aturan perubahan: lihat PRD bagian 11.

## 1. Diagram Arsitektur
```
+-------------------------------------------------------------+
|                 TABLET / HP KASIR (PWA terpasang)           |
|  React 18 + Vite + TypeScript | Tailwind (Senior-Friendly)  |
|  Web Speech API: kembalian bersuara (offline)               |
|  IndexedDB (Dexie.js) - penyimpanan PERSISTEN               |
|   - keranjang, transaksi belum terkirim, katalog (tanpa HPP)|
|   - sesi laci, counter nota, antrean (outbox)               |
+------------------------------|------------------------------+
                               | push_bundle / pull_ledger (JWT perangkat)
                               v
+-------------------------------------------------------------+
|                     SUPABASE (PostgreSQL)                   |
|  RLS | RPC: push_bundle, pull_ledger, Persetujuan QR        |
|  Edge Functions: provision_device, r2_presign, ai_proxy,    |
|                  send_push, daily_digest                    |
|  Database Webhook -> send_push                              |
+------|------------------------|------------------|----------+
       |                        |                  |
       v                        v                  v
+--------------+     +----------------------+  +-----------------+
| CLOUDFLARE R2|     | HP PENGELOLA (PWA)   |  | GITHUB ACTIONS  |
| foto produk  |     | Supabase Auth, Push, |  | backup harian + |
| QRIS, faktur |     | pindai QR, dasbor    |  | keep-alive +    |
| backup (priv)|     +----------------------+  | daily_digest    |
+--------------+                               +-----------------+
```

## 2. Rincian Teknologi
- **Frontend**: React 18, Vite, TypeScript, Tailwind CSS (skala font besar, kontras AAA).
- **PWA & Offline**: Workbox (Service Worker), Dexie.js.
- **BaaS**: Supabase (PostgreSQL 15+, Auth, PostgREST, Edge Functions, Database Webhooks).
- **Hosting frontend**: Cloudflare Pages.
- **Media**: Cloudflare R2 (unggah lewat URL bertanda tangan). URL publik foto selama MVP memakai `r2.dev` bawaan (tanpa membeli domain); foto di-cache Service Worker. Verifikasi saat setup bahwa R2 dapat dipakai dalam kuota gratis (setahu kami Cloudflare meminta metode pembayaran terdaftar); bila tidak diinginkan, cadangan: Supabase Storage (kuota gratis kecil).
- **Otomasi**: GitHub Actions.
- **AI (gratis)**: Google Gemini API free tier, dipanggil hanya lewat Edge Function `ai_proxy` (kunci tidak pernah di klien). Pencocokan dikte dan Smart Insight utama memakai kode lokal tanpa AI. Batas kuota gratis berubah; cek dokumentasi resmi dan atur batas harian di `ai_usage`.
- **Keputusan Pemilik**: foto faktur dan kemasan boleh diproses layanan gratis Google (data masukan dapat dipakai Google untuk peningkatan produk).
- **Notifikasi**: Web Push (VAPID) lewat Edge Function `send_push`. Gratis; berjalan di Chrome Android dan iOS 16.4+ bila PWA dipasang.
- **WhatsApp**: hanya tautan `wa.me`.
- **Pindai QR**: `BarcodeDetector` bila tersedia, cadangan pustaka JavaScript kecil.
- **Lovable AI Gateway tidak dipakai** (berbasis kredit berbayar).

## 3. Identitas, Keamanan & Isolasi Data
- Tablet Kasir memegang **JWT perangkat** (klaim `device_id`, `device_role`; berlaku 30 hari, diperpanjang tiap sinkronisasi) dari `provision_device`, lewat token pairing sekali pakai 10 menit. RLS memeriksa klaim dan `devices.is_active`; klaim selalu menang atas isi permintaan.
- **RLS default-deny** di semua tabel dan Pengelola menulis hanya lewat RPC (Model Data bagian 1b). Supabase Security Advisor dijalankan tiap rilis.
- HP Pengelola login Supabase Auth; hak Pemilik/Kerabat diturunkan dari `members.auth_user_id`. HP itu juga terdaftar sebagai perangkat `hp_manager` terikat ke akunnya, dan setiap RPC memeriksa perangkat aktif lewat header `x-install-id`. Access token dibuat singkat (<= 15 menit).
- **Tidak ada PIN** di mana pun. Tindakan sensitif memakai Persetujuan QR; konfirmasi faktur/harga/opname dilakukan langsung di HP Pengelola.
- HPP (`cost_price_hpp`) tidak pernah dikirim ke Tablet Kasir; tablet tidak punya akses `SELECT` ke tabel `products`, hanya lewat RPC.
- Identitas penyetuju diambil dari token, bukan dari isi permintaan; token acak 128-bit, sekali pakai, terikat `payload_hash`.
- `audit_logs` hanya dapat di-INSERT (tidak UPDATE/DELETE).
- Pencabutan HP yang hilang: Pemilik dari HP lain (`revoke_device`), atau Administrator Teknis lewat dasbor Supabase. Keduanya menonaktifkan perangkat **dan menghapus sesi login** HP itu (trigger `devices`), sehingga hak menyetujui langsung hilang.
- Data pelanggan dibatasi: tablet hanya menerima nama, limit, dan saldo; telepon/alamat hanya di HP Pengelola.

## 4. Topologi Multi-Device & Sinkronisasi
- Tiap perangkat adalah node independen; transaksi disimpan di Dexie dengan UUID v4 dari klien.
- Worker latar memanggil `push_bundle` per item; item ditandai tersinkron hanya setelah ada id di `accepted_ids`. Item ditolak masuk "Antrean Bermasalah".
- Katalog, pelanggan, anggota, pengaturan toko ditarik lewat `pull_ledger` (delta, upsert idempoten, tanda hapus lembut).
- Waktu acuan sinkronisasi adalah `server_timestamp`, bukan jam perangkat. Jam perangkat yang meleset hanya memengaruhi `created_at`; tiap sinkronisasi membandingkan jam perangkat dengan jam server: selisih > 10 menit membuat peringatan ke Pengelola, dan nota yang `created_at`-nya berselisih > 24 jam ditandai `CLOCK_SKEW`.
- Stok dan saldo dihitung dari catatan pergerakan; stok negatif diperbolehkan dan ditandai.
- **Push dulu, baru pull**; pull tidak menimpa outbox. Stok/hutang yang tampil di tablet = nilai server +/- isi outbox. Bundle maksimal 50 item.

## 5. Manajemen Penyimpanan Lokal
1. **Persisten**: meminta `navigator.storage.persist()` pada peluncuran pertama dan memandu pemasangan PWA agar IndexedDB tidak dihapus peramban saat memori penuh.
2. **Retensi 3 hari** (`local_retention_days`): purge `is_synced = 1` dan `synced_at` > 72 jam (boolean disimpan sebagai angka 0/1 karena IndexedDB tidak dapat mengindeks boolean).
3. **Counter nota** disimpan di `appSettings`, terpisah dari transaksi.
4. **Migrasi skema**: setiap perubahan = `version(n+1)` dengan `.upgrade()`; versi lama tidak diubah.
5. **Pembaruan aplikasi** (Service Worker) tidak otomatis memotong sesi; ditawarkan lewat tombol besar saat keranjang kosong.
6. **Mode Latihan** memakai database `KWarPracticeDB` terpisah.
7. Supabase Postgres adalah sumber kebenaran permanen.

## 6. Cadangan & Perawatan
- GitHub Actions harian (`cron: 0 20 * * *` UTC = 03:00 WIB; sesuaikan bila warung di WITA/WIT). **Repositori privat**: workflow terjadwal di repo publik dinonaktifkan otomatis setelah lama tanpa aktivitas repo (setahu kami 60 hari) sehingga backup berhenti diam-diam.
- Koneksi `pg_dump` memakai **connection pooler Supabase** (runner GitHub umumnya IPv4, koneksi langsung Supabase IPv6).
- Dump **dienkripsi** dengan `age` (kunci publik di GitHub Secrets) sebelum diunggah ke bucket privat R2; retensi 30 hari lewat **aturan lifecycle R2**, bukan skrip. **Kunci privat** disimpan di dua tempat terpisah: Administrator Teknis dan Pemilik (dicetak/tersegel). Tanpa kunci itu backup tidak dapat dipulihkan.
- Dump mencakup skema `public` dan `auth` (agar `members.auth_user_id` valid saat restore); dipastikan di uji restore pertama.
- Langkah yang sama menjalankan query kesehatan agar proyek Supabase gratis tidak ter-pause.
- **Heartbeat**: tiap workflow memanggil `report_job` setelah selesai. `pg_cron` di Supabase memeriksa `ops_heartbeats` tiap 6 jam dan memberi tahu Pengelola bila backup lebih tua dari 36 jam, sehingga kegagalan diam-diam (workflow dinonaktifkan) tetap terdeteksi.
- `daily_digest` dijalankan workflow terpisah di malam hari.
- Uji pemulihan (restore) minimal sekali per kuartal; hasilnya dicatat lewat `report_job('restore_test')`.

## 7. Kompatibilitas Perangkat (Wajib Diuji Sebelum Membeli)
- Peramban target: **Chrome Android** (Web Bluetooth, WebUSB, Web Speech API, BarcodeDetector). iOS Safari tidak mendukung Web Bluetooth/WebUSB; HP Pengelola berbasis iOS tetap dapat memakai dasbor, push, dan pindai.
- **Printer thermal 58mm**: Web Bluetooth hanya mendukung Bluetooth LE. Banyak printer murah memakai Bluetooth klasik dan tidak akan terdeteksi. Pilih printer yang terbukti BLE/USB, atau gunakan aplikasi perantara (mis. RawBT) bila perlu.
- **Suara**: kembalian bersuara (TTS) bergantung paket suara bahasa Indonesia di Android; unduh paket suara agar tetap offline. Dikte (SpeechRecognition) umumnya butuh internet.

## 8. Perkiraan Kapasitas & Batas Gratis
Perkiraan kasar, perlu dicocokkan dengan data UAT 1 (PRD bagian 10).
- Satu nota rata-rata 3 butir menghasilkan sekitar 10 baris (nota, butir, pergerakan kas/stok/kasbon) atau kira-kira 2 KB termasuk indeks. Pada 100-500 nota/hari: 0,2-1 MB/hari, jadi 70-350 MB/tahun. Batas database gratis Supabase (500 MB, dari ingatan; cek yang berlaku) tercapai dalam 1,5 sampai 7 tahun.
- Rencana: pantau ukuran database di `daily_digest` dan beri peringatan di 70% kuota. Bila mendekati batas, arsipkan transaksi lama (ekspor lalu hapus) setelah backup terverifikasi.
- Foto produk di R2 (batas gratis puluhan GB) dan kuota Gemini diperiksa di dokumentasi resmi sebelum rilis R3.
- Ukuran bundle <= 50 item menjaga tiap panggilan `push_bundle` jauh di bawah batas waktu statement server.

## 9. Setup Awal & Ganti HP Pemilik (Runbook Administrator Teknis)
**Setup awal (F-46)**
1. Buat proyek Supabase, repositori GitHub **privat**, bucket R2 (publik untuk foto, privat untuk faktur dan backup), dan proyek Cloudflare Pages.
2. Jalankan migrasi `schema.sql` beserta RLS (Model Data bagian 1b), lalu **jalankan skrip seed** (service role): pengaturan toko (zona waktu, `help_phone`), Laci Bersama dan Kas Simpanan, butir sistem "Barang Lain" (id tetap), dan anggota Pemilik (`simple_mode = true`).
3. Undang Pemilik (tautan masuk), daftarkan HP Pemilik, lalu daftarkan tablet pertama lewat token pairing yang dibuat Pemilik.
4. Impor produk (CSV), unggah foto, dan pilih 6-8 favorit bersama Pemilik.
5. Pasang backup: kunci `age`, secret GitHub, workflow, `ops_heartbeats`, dan pengawas `pg_cron`.
6. Jalankan TC-56 (RLS) dan TC-67 (heartbeat) sebelum data nyata dimasukkan; jalankan Supabase Security Advisor.

**Ganti HP Pemilik / HP hilang (F-45)**
1. Cabut HP lama (dasbor Supabase atau `revoke_device` dari HP Kerabat bila Kerabat berhak; sesi login ikut terhapus).
2. Kirim tautan masuk baru ke Pemilik (`send_login_link`).
3. Buat token pairing lewat skrip admin; daftarkan HP baru. Data dan hak Pemilik tidak berubah karena terikat ke akun, bukan ke HP.
4. Periksa Audit Log bahwa HP lama tidak lagi memanggil RPC (TC-54).
