# Dokumen Persyaratan Non-Fungsional (NFR) - K_War
Versi 2.4 - 26 November 2026
Dokumen ini adalah sumber kebenaran. Aturan perubahan: lihat PRD bagian 11.

## 1. Performa & Latensi
- Penambahan barang via scanner/sentuhan ke keranjang maksimal 100 ms.
- Cold boot PWA < 2 detik pada Android low-end (RAM 3-4 GB), siap memindai.
- Kalkulasi total/kembalian < 16 ms.
- Penyelesaian transaksi (tombol Bayar sampai tersimpan di Dexie) < 300 ms.

## 2. Keandalan & Ketahanan Data
- **Offline 100%**: alur kasir, input barang, tahan keranjang, buka/tutup shift, terima kulakan, cicilan kasbon berjalan tanpa internet. Pengecualian yang diumumkan jelas di layar: Persetujuan QR online, void nota lama, dikte suara, Foto Faktur, notifikasi.
- **Zero Data Loss**: IndexedDB dikunci dengan `navigator.storage.persist()`; transaksi `is_synced = 0` tidak pernah dihapus. Tetap aman saat daya mati, aplikasi ditutup paksa, atau reload.
- **Sinkronisasi idempoten**: `client_bundle_id` dan `id` per item; pengiriman ulang tidak menggandakan data. Pemrosesan per item; satu item buruk tidak menahan lainnya.
- **Perangkat dicabut**: data tertunda dikarantina, bukan dibuang.
- **Konsistensi lokal**: stok dan saldo kasbon yang tampil = nilai server +/- isi outbox; pull tidak menimpa outbox (push dulu, baru pull).
- **Skala**: dirancang untuk <= 10 perangkat dan <= 500 nota/hari; satu bundle <= 50 item.
- **Waktu**: tanggal dan jam pada nota/struk mengikuti jam perangkat. Jadwal backup memakai zona waktu yang dikonfigurasi (contoh dokumen: WIB). `store_settings.timezone` hanya dipakai server untuk batas hari pada laporan; tidak memengaruhi nota/struk. Sesuaikan zona waktu dan jadwal backup bila warung di WITA/WIT.

## 3. Aksesibilitas & Antarmuka Lansia
- **Tipografi**: nama barang ≥ 18 px, harga ≥ 22 px, total belanja ≥ 32 px, label tombol ≥ 18 px tebal. Pengguna dapat memperbesar ukuran huruf (3 tingkat).
- **Target sentuh**: ≥ 56 x 56 dp untuk tombol utama.
- **Kontras**: WCAG AAA (≥ 7:1) untuk teks dan nominal. Warna aksi (hijau, merah, biru, kuning tua) hanya dipakai sebagai teks di atas **putih murni** `#FFFFFF`, atau sebagai latar tombol dengan teks putih. Di atas Abu Terang `#F8FAFC` hijau dan kuning tua turun di bawah 7:1 (6,81 dan 6,78), jadi di latar abu hanya teks hitam `#0F172A`. Palet resmi: teks `#0F172A`, hijau `#166534` (7,1:1), merah `#991B1B` (8,3:1), biru `#1E40AF` (8,7:1), kuning tua `#92400E` (7,1:1) di atas putih. Angka rasio diverifikasi ulang dengan alat pengecek kontras saat implementasi.
- **Status tidak hanya warna**: stok dan peringatan selalu disertai teks ("HABIS", "KRITIS", "MENIPIS", "AMAN") dan ikon.
- **Navigasi satu arah**: tanpa akordeon tersembunyi; satu tombol BAYAR di layar utama. Dialog hanya berurutan linear (Bayar -> Cara Bayar -> Tunai), maksimal 3 modal berturut-turut, tiap modal punya tombol KEMBALI besar dan tidak ada modal di dalam modal.
- **Umpan balik**: bunyi dan getar singkat saat barang masuk; nama barang dapat diucapkan (toggle).
- **Pencegahan salah tekan**: tombol Bayar dinonaktifkan sesaat setelah ditekan; "Batalkan Barang Terakhir" selalu tersedia.
- **Mode Latihan** tersedia untuk pembelajaran tanpa risiko.
- **Bahasa layar sehari-hari**: semua teks tombol, pesan, dan judul memakai kata yang dipakai di warung ("Mulai Jaga", "Selesai Jaga", "Batalkan Nota", "Perlu Dicek", "Hitung Stok", "Nota Kulakan", "Beranda"). Istilah teknis (void, shift, sinkronisasi, opname, faktur, blind count) hanya untuk dokumen dan kode.
- **Tata letak tetap**: posisi tombol dan kartu tidak berubah sendiri (grid favorit bersusunan tetap).
- **Tanpa batas waktu di layar**: tidak ada tombol atau pesan penting yang hilang sendiri; pesan informasi minimal 10 detik dan dapat ditutup dengan sentuhan; "Batalkan Barang Terakhir" tetap tampil sampai barang berikutnya ditambahkan atau BAYAR ditekan.
- **Gestur punya cadangan**: setiap gerakan geser punya tombol panah besar; tidak ada fungsi yang hanya dapat dilakukan dengan menyeret atau menekan lama.
- **Konfirmasi angka tidak wajar** (F-42) mencegah salah ketik nol tanpa memperlambat transaksi biasa.

## 4. Keamanan & Hak Akses
- Tablet memakai JWT perangkat; RLS memeriksa `device_id` dan `is_active` di server. Revoke berlaku seketika pada koneksi berikutnya.
- **Tanpa PIN**. Persetujuan: token acak 128-bit, sekali pakai, kedaluwarsa 5 menit, terikat `payload_hash`; identitas penyetuju dari login server.
- HPP dan laba tidak pernah berada di Tablet Kasir.
- `audit_logs` bersifat tambah-saja; kasir tidak dapat mengubah atau menghapus.
- Kunci AI hanya di Edge Function; panggilan AI dibatasi harian dan dapat dimatikan.
- Data backup dienkripsi (`age`) sebelum diunggah; kunci privat disimpan di dua orang berbeda.
- RLS default-deny di semua tabel; Pengelola menulis hanya lewat RPC; `audit_logs` tidak dapat diubah/dihapus (Model Data bagian 1b).
- Pencabutan HP Pengelola juga menghapus sesi login; access token <= 15 menit.
- Tablet hanya menyimpan nama pelanggan (tanpa telepon/alamat); data lokal tablet tidak dienkripsi (risiko diterima, ditutup oleh Remote Revoke).
- Sesi HP Pengelola panjang; keamanannya mengandalkan kunci layar HP.

## 5. Efisiensi Biaya & Proteksi Kuota
- **Kembalian bersuara** 100% Web Speech API, tanpa biaya.
- Seluruh layanan memakai paket gratis (Supabase, Cloudflare Pages/R2, GitHub Actions, Gemini free tier, Web Push).
- Fitur AI memiliki sakelar utama (`ai_features_enabled`), sakelar dikte, batas harian di `ai_usage`, dan pesan ramah saat kuota habis; aplikasi inti tidak bergantung pada AI.
- Tidak ada API WhatsApp berbayar; hanya `wa.me`.

## 6. Efisiensi Memori Lokal & Retensi
- Transaksi `is_synced = 1` disimpan 3 hari (72 jam) lalu dibersihkan otomatis (saat aplikasi dibuka dan setelah sinkronisasi sukses).
- Transaksi `is_synced = 0` dilindungi mutlak.
- Penggunaan IndexedDB dijaga < 50 MB pada operasional sibuk; foto produk di-cache dengan batas ukuran.
- Void/retur nota di luar shift aktif sendiri membutuhkan internet (diambil dari server).

## 7. Cadangan & Perawatan
- Backup harian 03:00 WIB via GitHub Actions (`cron: 0 20 * * *` UTC), memakai connection pooler.
- Dump dienkripsi, disimpan di bucket privat R2, retensi 30 hari lewat lifecycle rule R2.
- Query kesehatan harian mencegah Supabase gratis ter-pause. Repositori GitHub privat.
- Uji pemulihan minimal sekali per kuartal; hasilnya dicatat.
- Peringatan ke Pengelola bila backup gagal **atau tidak berjalan** (heartbeat `ops_heartbeats` > 36 jam, diperiksa `pg_cron` di Supabase, bukan GitHub).

## 8. Kompatibilitas
- Target: Chrome Android versi terkini. Fitur bergantung peramban (Web Bluetooth/WebUSB, SpeechRecognition, BarcodeDetector) wajib punya cadangan yang jelas atau pesan "tidak tersedia di perangkat ini". Indikator baterai memakai Battery Status API; bila tidak tersedia, indikator disembunyikan.
