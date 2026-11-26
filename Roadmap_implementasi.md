# Roadmap Implementasi - K_War
Versi 2.4 - 26 November 2026
Dokumen ini adalah sumber kebenaran. Aturan perubahan: lihat PRD bagian 11.

Prinsip: **rilis kecil yang benar-benar dipakai di warung** lebih baik daripada semua fitur sekaligus. Setiap rilis diakhiri UAT bersama penjaga lansia. Perkiraan waktu untuk satu pengembang; geser sesuai kenyataan. Mulai dari irisan tipis yang membuktikan risiko terbesar (offline + sinkronisasi + shift), baru menambah.

## Rilis 1a - Jual Tunai Offline yang Aman dan Siap Dipakai (Minggu 1-5)
**Tujuan**: satu tablet berjualan tunai harian dengan data aman, tanpa PIN, dan Pemilik lansia dapat memantau dari HP-nya. Rilis ini harus bisa dipakai nyata, karena itu katalog produk, setup awal, dan bantuan ikut di sini.
- **Dasar**: PWA React 18 + Vite + TypeScript, Tailwind dengan token Senior-Friendly (palet AAA), pemasangan PWA dan `storage.persist()` (F-38); Dexie v1 (is_synced 0/1, `cashDrawers`, indeks `favorite_order`), counter nota, purge 3 hari (F-36).
- **Server**: `schema.sql` **dengan RLS default-deny dan matriks izin (Model Data 1b)**, trigger, RPC `push_bundle` (per item, idempoten, nota bertanda, `voids`) dan `pull_ledger` (tanpa HPP, memuat laci), Edge Function `provision_device` dan `r2_presign`.
- **Perangkat dan anggota**: Device Provisioning + token pairing + JWT + Remote Revoke (F-01), anggota dan hak akses (F-02), "Siapa yang jaga?" (F-03).
- **Katalog (baru di R1a)**: impor CSV, tambah/ubah produk manual, scan barcode + foto, memilih 6-8 favorit (F-29 a-b) di HP Pengelola; foto di R2.
- **Kasir**: grid favorit + geser per kategori, scan barcode/kamera, pencarian, Barang Lain, multi-keranjang, Batalkan Barang Terakhir, anti tekan ganda (F-04 s.d. F-07).
- **Giliran jaga**: Mulai Jaga/Selesai Jaga dengan Hitung Uang, sesi yatim, selisih final (F-08, F-18); indikator sinkronisasi + baterai (F-09).
- **Bayar**: tunai, kembalian bersuara, Tukar Receh, nomor nota (F-10, F-11, F-16); batalkan nota shift sendiri (F-17); **konfirmasi angka tidak wajar (F-42)**.
- **Lansia**: **tombol Bantuan (F-43)**, Mode Latihan (F-37), pengaturan aksesibilitas (F-39).
- **HP Pemilik**: login tautan masuk, **Beranda Sederhana (F-44)**, laporan giliran jaga, Kelola Produk, tutup paksa sesi yatim, **Pemulihan Akses (F-45)**.
- **Setup awal (F-46)**: skrip seed + runbook (Tech Stack 9).
- **Backup harian terenkripsi + keep-alive + heartbeat dan pengawas `pg_cron`** (repositori privat, kunci backup disimpan dua orang, uji restore pertama).
- **UAT 1a**: uji lapangan 1-2 minggu di warung memakai satu tablet, hanya tunai; ukur "transaksi 3 barang <= 60 detik" dan jumlah permintaan bantuan.
- Bila minggu 5 terlampaui: tunda pengaturan aksesibilitas lanjutan dan Mode Latihan penuh, jangan katalog atau setup awal.

## Rilis 1b - Kasbon & Persetujuan (Minggu 6-9)
**Tujuan**: kasbon dan tindakan sensitif aman, dengan jalur notifikasi untuk Pemilik lansia.
- Buku kasbon, cicilan, pelanggan baru oleh kasir (F-19), split tunai + kasbon (F-12).
- **Persetujuan QR online** dengan polling (F-30), jenis `DEBT_OVERLIMIT` dan `VOID_LATE`; HP Pemilik: tombol SETUJUI PERMINTAAN di Beranda Sederhana, pindai QR, daftar Permintaan Menunggu.
- **Edge Function `send_push` minimal**: hanya notifikasi permintaan persetujuan (F-33 bagian R1b).
- Batalkan/kembalikan nota lama dan retur sebagian (F-17 bagian R1b).
- Tinjauan Perlu Dicek dan Nota Bermasalah (F-41), `resolve_rejection`, `review_transaction`.
- **UAT 1b**: uji lapangan 1-2 minggu dengan kasbon dan persetujuan; fokus apakah lansia memakai jalur notifikasi.

## Rilis 2 - Stok, Kulakan & Operasional Warung (Minggu 10-14)
- Penerimaan kulakan oleh kasir dengan Penerimaan Sementara (F-21), konfirmasi nota kulakan dan HPP Moving Average di HP Pengelola (F-22), titik merah harga naik, status stok HABIS/KRITIS/MENIPIS/AMAN.
- Penyesuaian stok rusak (F-23), hitung stok/opname (F-24), Pakai Sendiri (F-14).
- Gas/isi ulang (F-25), konsinyasi (F-26), pelacakan kadaluwarsa (F-27).
- Daftar Belanja Pasar + kirim WhatsApp (F-28).
- QRIS statis dengan konfirmasi "Uang Sudah Masuk" (F-13) dan rekap QRIS per shift.
- Cetak struk 58mm (setelah uji printer nyata) dan kirim nota WhatsApp (F-15), pengingat kasbon (F-20).
- Setoran & Transfer Kas (F-40).
- Beranda Lengkap + Bacakan Laporan (F-32), notifikasi push jenis lain (F-33), Smart Insight berbasis aturan lokal (F-35).
- `daily_digest` (ringkasan malam, nomor nota terlewat, ukuran database).
- **UAT 2**.

## Rilis 3 - AI Gratis & Persetujuan Offline (Minggu 15-18)
- Edge Function `ai_proxy` (Gemini free tier), sakelar AI, batas harian `ai_usage`.
- Foto Faktur (F-22), Voice Quick Order dengan pencocok lokal (F-34), isi nama dari foto kemasan (F-29c), poles Smart Insight (F-35).
- Persetujuan QR offline dua arah (F-31).
- Penyempurnaan berdasarkan UAT: ukuran huruf, suara, susunan favorit.
- **UAT 3** dan uji pemulihan backup.

## Ditunda / Di Luar Rencana
- PPOB (pulsa, token, dll.): setelah pengguna lancar dengan K_War; butuh agregator dan deposit.
- QRIS dinamis / perangkat EDC: setelah warung memiliki QRIS merchant.
- API WhatsApp, katalog web pelanggan, fitur AI berbayar: tidak dikerjakan.

## Risiko yang Dipantau
- Printer thermal BLE tidak cocok: uji perangkat sebelum membeli (Tech Stack bagian 7).
- Kuota gratis Gemini dan batas layanan gratis lain berubah: aplikasi inti tidak boleh bergantung pada AI; periksa dokumentasi resmi sebelum tiap rilis.
- Jam perangkat meleset: dicatat server dan dilaporkan (`CLOCK_SKEW`).
- Lansia sulit memindai QR: jalur notifikasi/daftar "Permintaan Menunggu" sudah tersedia sejak R1b dan diuji di UAT 1b.
- Estimasi waktu satu pengembang: R1a (5 minggu) adalah yang terpadat; lihat aturan penundaan di bagian R1a.
