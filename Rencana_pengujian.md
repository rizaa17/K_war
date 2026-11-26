# Rencana Pengujian (Test Plan) - K_War
Versi 2.4 - 26 November 2026
Dokumen ini adalah sumber kebenaran. Aturan perubahan: lihat PRD bagian 11.

## 1. Strategi
Pengujian berfokus pada keandalan di warung nyata:
1. Offline total dan ketahanan data.
2. Multi-perangkat, sinkronisasi, idempotensi, item bermasalah.
3. Laci kas, shift, pergerakan kas.
4. Kasbon dan Persetujuan QR (tanpa PIN).
5. Keamanan peran (HPP tidak bocor ke kasir).
6. Perangkat keras nyata: printer, suara, kamera.
7. Backup, notifikasi, AI gratis.
8. Uji lapangan dengan lansia (UAT) pada akhir tiap rilis.

## 2. Matriks Kasus Uji

| ID | Rilis | Modul | Skenario | Prosedur | Hasil yang Diharapkan |
| :--- | :--- | :--- | :--- | :--- | :--- |
| TC-01 | R1a | Offline | Transaksi saat internet mati | Matikan Wi-Fi/data tablet; lakukan 10 transaksi | Semua tersimpan di Dexie.js, stok lokal berkurang, kembalian bersuara tetap aktif |
| TC-02 | R1a | Multi-keranjang | Reload / daya mati | 3 barang di Keranjang 1, buka Keranjang 2, tutup paksa/restart | Kedua keranjang pulih persis |
| TC-03 | R1a | Sync push | Rekoneksi | Online setelah TC-01 | `push_bundle` mengirim; semua `accepted_ids`; tanpa duplikasi saat dikirim ulang |
| TC-04 | R1a | Sync idempoten | Sinyal putus-nyambung | Putus koneksi tepat setelah request, kirim ulang bundle yang sama | Server mengembalikan hasil yang sama; tidak ada nota ganda |
| TC-05 | R1a | Sync parsial | Satu item buruk | Sisipkan 1 nota dengan `product_id` yang tidak pernah ada di bundle 5 nota | 4 diterima, 1 ditolak (`UNKNOWN_PRODUCT`) masuk Antrean Bermasalah; indikator di kasir menampilkan jumlahnya. (Produk yang hanya soft-delete tidak ditolak: lihat TC-57) |
| TC-06 | R1a | Validasi pembayaran | Total tidak cocok | Kirim `qris_static` dengan `debt_amount > 0` | Ditolak `TOTAL_MISMATCH` |
| TC-07 | R1a | Perangkat dicabut | Revoke saat ada nota offline | Cabut tablet; tablet membuat 3 nota lalu online | Layar terkunci ke aktivasi, data lokal tidak dihapus; server menyimpan 3 nota di karantina (`DEVICE_REVOKED`) |
| TC-08 | R2 | Laci kas | Bayar kulakan dari laci | Kasir menerima barang, bayar Rp200.000 dari laci | `cash_movements` -200.000 seketika; saldo laci berkurang; **stok jual bertambah seketika** (`purchase_receive`); kekurangan dari Kas Simpanan dicatat saat Pengelola mengonfirmasi harga |
| TC-09 | R2 | HPP | Kenaikan modal | Beli 10 @10.000; kulakan 10 @12.000 dikonfirmasi Pengelola | HPP Rp11.000; titik merah di HP Pengelola, tidak ada popup, tidak tampil di kasir |
| TC-10 | R1a | Keamanan peran | HPP tidak bocor | Periksa respons `pull_ledger` dan IndexedDB tablet | Tidak ada `cost_price_hpp`; `SELECT products` langsung ditolak untuk tablet |
| TC-11 | R1b | Kasbon over-limit | Persetujuan QR online | Limit 100.000, hutang 80.000, kasbon 50.000; minta persetujuan; Pengelola menyetujui | Transaksi tertahan sampai disetujui; lalu lanjut; `approval_request_id` tersimpan |
| TC-12 | R1b | QR keamanan | Pemalsuan | (a) Setujui token kedaluwarsa; (b) pakai token dua kali; (c) ubah nominal setelah disetujui; (d) kirim `owner_member_id` palsu | Semua ditolak; identitas penyetuju selalu dari login |
| TC-13 | R1b; R3 | QR offline | Tablet tanpa internet | Minta persetujuan saat offline | Pesan jelas "belum bisa"; kasbon over-limit ditolak (sebelum Rilis 3); setelah Rilis 3, QR dua arah berhasil |
| TC-14 | R1b | Persetujuan jarak jauh | Notifikasi | Pengelola di HP menerima push dan menekan Setujui | Tablet terbuka otomatis dalam ≤ 5 detik |
| TC-15 | R1a | Voice Return | Kembalian offline | HP Android tanpa SIM/Wi-Fi | Suara lancar tanpa kuota AI; tombol uji suara memastikan paket bahasa terpasang |
| TC-16 | R1a | Backup | Cron | Jalankan workflow | Dump terenkripsi (`age`) di R2 privat, lifecycle 30 hari, Supabase tidak paused, `ops_heartbeats` terisi, notifikasi bila gagal; uji restore (termasuk skema `auth`) berhasil |
| TC-17 | R2 | Cetak 58mm | Printer nyata | Cetak nota di Chrome Android dengan printer BLE/USB yang dipakai | Struk keluar berformat `kw/...`; bila printer tidak terdeteksi, pesan jelas dan transaksi tetap tersimpan |
| TC-18 | R2 | QRIS statis | Kondisional + konfirmasi | Bandingkan QRIS kosong vs terunggah | Tombol hanya muncul jika ada gambar; nota baru selesai setelah "UANG SUDAH MASUK", waktu tercatat |
| TC-19 | R1a | Nomor nota | Multi-device & purge | DEV1 dan DEV2 bertransaksi hari yang sama; purge berjalan; transaksi lanjut | `kw/0001/DEV1/...`, `kw/0001/DEV2/...` tidak bentrok; urutan DEV1 tetap naik setelah purge |
| TC-20 | R1a | Jam perangkat | Nota lewat tengah malam | Buat nota pada 23:59 lalu 00:01 menurut jam perangkat | Tanggal nota dan struk mengikuti jam perangkat; urutan kembali ke 0001 pada hari baru |
| TC-21 | R1a | Blind count | Tutup shift | Kasir isi pecahan lembar + koin | Kasir tidak melihat selisih; HP Pengelola menerima rincian + selisih |
| TC-22 | R1b | Cicilan kasbon | Kasir terima cicilan | Pelanggan cicil Rp50.000 | Buku kasbon -50.000; kas laci +50.000; saldo = jumlah catatan buku |
| TC-23 | R2 | Barang rusak | Persetujuan | Kasir catat 2 sirup pecah | Tertahan QR; stok terpotong hanya setelah disetujui; bukan alasan void |
| TC-24 | R1a | Barang Lain | Harga bebas | Tambahkan "Bumbu pecel" Rp5.000 dan bayar tunai | Stok tak berubah; omzet tercatat; laporan menampilkan total "Barang Lain" |
| TC-25 | R1a | Undo & tekan ganda | Ketahanan UI | Tekan Bayar dua kali cepat; batalkan barang terakhir | Hanya satu transaksi; undo menghapus barang terakhir |
| TC-26 | R2 | Pakai Sendiri | Pemakaian pribadi | Pilih Pakai Sendiri untuk 2 rokok | Tertahan QR; setelah disetujui stok -2, tanpa uang masuk, tercatat sebagai pemakaian pribadi |
| TC-27 | R2 | Gas/isi ulang | Tukar tabung | Jual 1 gas isi (tukar kosong); kulakan 5 isi, kembalikan 5 kosong | Penjualan: isi -1, kosong +1; kulakan: isi +5, kosong -5 |
| TC-28 | R2 | Konsinyasi | Titipan | Terima 10 titipan, terjual 4 (harga setor 3.000); bayar penitip | Utang penitip 12.000 → 0 setelah dibayar; kas berkurang |
| TC-29 | R2 | Kadaluwarsa | Pemberitahuan | Produk ED 20 hari lagi (batas 30) | Muncul di daftar "Segera Habis Masa Berlaku" dan push |
| TC-30 | R1a | Stok negatif | Dua perangkat jual barang terakhir | Dua tablet offline menjual stok terakhir | Kedua nota tersimpan; stok -1 ditandai di ringkasan |
| TC-31 | R1a; R1b | Void aturan | Shift sendiri vs lain | (a) Void nota shift sendiri; (b) void nota kemarin; (c) void nota shift sendiri yang sudah tersinkron | (a) tanpa QR, alasan wajib; (b) butuh QR + internet; (c) server menerima void dan menandai nota `voided` (tidak diabaikan); semuanya di Audit Log |
| TC-32 | R1a | Retensi | Auto-purge | 10 tersinkron (5 > 3 hari), 2 belum terkirim | 5 lama terhapus; 5 baru + 2 belum terkirim tetap; server utuh; `is_synced` bertipe 0/1 dan tersaring benar |
| TC-33 | R1a | Penyimpanan persisten | Anti-eviction | Buka pertama kali; periksa `navigator.storage.persisted()` | `true`; PWA terpasang |
| TC-34 | R3 | Foto Faktur | AI gratis | Foto faktur → isi kolom | Kolom terisi; Pengelola wajib memeriksa; kuota harian terpotong; saat kuota habis muncul pesan ramah dan input manual tetap bisa |
| TC-35 | R3 | Voice Order | Dikte | Online: "Gula pasir satu kilo dan telur dua kilo"; lalu offline | Online: daftar usulan dikonfirmasi kasir; offline: tombol redup |
| TC-36 | R2 | WhatsApp | Tautan wa.me | Kirim nota, pengingat kasbon, daftar belanja | WhatsApp terbuka dengan teks sesuai; nomor 62...; tidak ada pengiriman otomatis |
| TC-37 | R2 | Bacakan laporan | Suara dasbor | Tekan "BACAKAN LAPORAN" tanpa internet | Ringkasan terbaca jelas |
| TC-38 | R1b; R2 | Notifikasi | Web Push | Minta persetujuan, tutup shift, stok habis | Push tiba di HP Pengelola (Android, dan iOS bila PWA terpasang) |
| TC-39 | R1a | Mode Latihan | Isolasi | Buat transaksi di Mode Latihan | Tidak ada perubahan stok/kas/server; pita kuning terlihat |
| TC-40 | R1a | Pembaruan PWA | Aman saat transaksi | Rilis versi baru saat keranjang berisi | Tidak memuat ulang paksa; tawaran "PERBARUI SEKARANG" muncul saat keranjang kosong |
| TC-41 | R1a | Impor produk | CSV & kamera | Impor 100 produk CSV; daftarkan 1 produk lewat scan + foto | Semua tampil di katalog; foto WebP ≤ 200 KB di R2 |
| TC-42 | R1a | Aksesibilitas | Kontras & target sentuh | Ukur dengan alat kontras; uji sentuh dengan pengguna lansia | Teks penting ≥ 7:1; tombol utama ≥ 56 dp; lansia menyelesaikan transaksi tunai tanpa bantuan |
| TC-43 | R1a | Buka shift | Modal awal | Pilih penjaga; sistem mengunci layar transaksi dan menampilkan Buka Shift; input Rp100.000; tekan "BUKA LACI & MULAI JUALAN" | Sesi `open`, `opening_balance = 100000`, ada `cash_movements` jenis `opening`; layar penjualan aktif; tersimpan di IndexedDB dan Supabase saat online |
| TC-44 | R1a | Aksesibilitas | Pengaturan F-39 | Ganti ukuran huruf ke 3 tingkat; matikan bunyi/getar; aktifkan "Ucapkan nama barang" | Ukuran huruf berubah di semua layar dan tersimpan per perangkat; bunyi/getar mengikuti pengaturan; nama barang terucap offline |
| TC-45 | R1a | Grid favorit | Susunan tetap dan geser | Pemilik memilih 8 favorit; jual produk non-favorit jauh lebih sering selama beberapa hari; geser (dan tekan panah) ke halaman berikutnya | Halaman 1 selalu 8 favorit dengan urutan tetap (tidak berubah sendiri); halaman berikut per kategori urut abjad dengan judul besar; kartu hanya menampilkan status stok bila MENIPIS/KRITIS/HABIS |
| TC-46 | R2 | Smart Insight | Aturan lokal | Tutup shift setelah void berulang dan selisih kas berulang | Pengelola melihat 2-3 kalimat insight dan tanda kejanggalan; bekerja offline; poles AI hanya bila toggle aktif |
| TC-47 | R2 | Ringkasan malam | daily_digest | Jalankan workflow digest | Pengelola menerima ringkasan harian, stok kritis, kadaluwarsa terdekat, dan jumlah item bermasalah |
| TC-48 | R1a | Admin darurat | Cabut lewat dasbor | Administrator Teknis menonaktifkan perangkat di dasbor Supabase | Perangkat tersebut diperlakukan sebagai dicabut (sama dengan TC-07) |
| TC-49 | R3 | Foto kemasan | AI gratis | Foto kemasan produk saat mendaftarkan produk | Nama terisi sebagai saran; Pengelola wajib memeriksa; kuota harian terpotong; saat kuota habis input manual tetap bisa |
| TC-50 | R1a | Provisioning & hak akses | Perangkat dan anggota | Daftarkan tablet dan HP Pengelola; atur Kerabat Tepercaya; atur "Hanya Laci Pribadi" untuk kasir | Perangkat terdaftar dengan `install_id` dan JWT; Kerabat dapat menyetujui; kasir tidak dapat membuka Laci Bersama dan tidak dapat menyetujui apa pun |
| TC-51 | R1a | Empat mode input | Grid, scanner, kamera, ketik | Masukkan barang lewat keempat mode, lalu barang curah | Scanner < 100 ms; modal takaran curah muncul (1/4, 1/2, 1, 2 kg + input manual); tanpa kode PLU |
| TC-52 | R1a; R1b | Tukar receh & split | Pembayaran | Total Rp45.000 dibayar Rp50.000, pilih Tukar Receh; lalu split tunai Rp40.000 + kasbon Rp60.000 pada total Rp100.000 | Kembalian jadi Rp0 dengan barang receh; laci +40.000 dan kasbon +60.000 |
| TC-53 | R2 | Stok opname | Pengelola | Hitung fisik berbeda dari sistem di HP Pengelola | `stock_movements` jenis `opname` untuk selisih; tercatat di Audit Log; kasir tidak punya akses |
| TC-54 | R1a | Keamanan peran | Sesi HP dicabut | Pemilik mencabut HP Kerabat (juga lewat dasbor Supabase); HP itu mencoba memanggil `confirm_qr_approval` dengan sesi lamanya | Ditolak (`DEVICE_NOT_BOUND`/401) segera; sesi Auth terhapus; Audit Log `DEVICE_REVOKED` (aksi dasbor tercatat `actor_kind = admin`) |
| TC-55 | R1a | Keamanan peran | Aksi terlarang Kerabat | Kerabat mencoba mengubah anggota, mendaftarkan/mencabut perangkat, `update_settings`, serta `UPDATE devices/members/products` langsung lewat PostgREST | Semua ditolak; hanya `SELECT` PostgREST dan RPC yang diizinkan peran Kerabat yang berhasil |
| TC-56 | R1a | Keamanan data | RLS default-deny | Dengan anon key dan JWT tablet, coba `SELECT/INSERT/UPDATE` pada semua tabel dan `v_consignment_payable`; jalankan Supabase Security Advisor | Semua ditolak; tidak ada tabel tanpa RLS; tidak ada peringatan view `security definer`; `audit_logs` tak dapat di-UPDATE/DELETE bahkan oleh Pemilik |
| TC-57 | R1a | Sync nota bertanda | Kegagalan lunak diterima | (a) Tablet A dan B offline, keduanya mengkasbonkan pelanggan yang sama melewati limit gabungan; (b) jual produk yang di-soft-delete Pengelola saat tablet offline; (c) hapus data lokal tablet sehingga counter nota kembali 0001 | Semua nota **diterima**; kas, stok, dan kasbon tercatat; bertanda `OVER_LIMIT_NO_APPROVAL`, `PRODUCT_DELETED`, `INVOICE_NO_CONFLICT` (nomor berakhiran `~xxxx`); tampil di F-41; **tidak** tampil di kasir |
| TC-58 | R1b | Persetujuan QR | Satu persetujuan satu penggunaan | Setelah `DEBT_OVERLIMIT` disetujui untuk satu nota, coba tempelkan `approval_request_id` yang sama pada nota kedua (nominal dan pelanggan sama) | Nota kedua diterima dengan tanda `APPROVAL_MISSING` (persetujuan sudah terpakai); indeks unik mencegah dua penggunaan; hash dengan `tx_id` berbeda tidak cocok |
| TC-59 | R1a; R1b | Void & retur | Jalur API | (a) Void sebelum nota tersinkron; (b) void setelah tersinkron; (c) retur 1 dari 3 butir nota campuran tunai+kasbon; (d) retur nota QRIS; (e) retur melebihi yang terjual | (a)(b) status `voided`, stok kembali, kas dan kasbon terkoreksi; (c) hanya butir itu dikoreksi, kasbon dikoreksi lebih dulu; (d) tanpa `cash_movements`, ada Audit Log; (e) ditolak `RETURN_EXCEEDS_SOLD` |
| TC-60 | R1a | Kontrol kas | Modal awal, tumpang tindih, sesi yatim | (a) Tutup shift dengan fisik Rp1.000.000, buka shift berikutnya dengan modal Rp800.000; (b) dua tablet offline membuka Laci Bersama; (c) biarkan sesi terbuka > 16 jam lalu buka shift baru; (d) Pengelola Tutup Paksa sesi anggota lain | (a) `OPENING_MISMATCH` dengan selisih terlihat hanya di HP Pengelola; (b) `SESSION_OVERLAP`; (c) layar mewajibkan Tutup Shift lebih dulu; (d) sesi tertutup, Audit Log `SESSION_FORCE_CLOSED` |
| TC-61 | R1a | Blind count | Selisih menunggu data lengkap | Tutup shift saat 3 dari 10 nota belum tersinkron; sinkronkan sisanya | Laporan awal berlabel "selisih sementara" (`is_complete = false`); setelah semua tiba, selisih final dihitung dan push dikirim sekali |
| TC-62 | R2 | HPP & stok | Penerimaan Sementara dan stok negatif | (a) Terima 24 barang saat stok 0 lalu cek tablet; (b) stok -3 lalu kulakan 3 dikonfirmasi; (c) tolak sebuah faktur | (a) stok jual langsung 24 dan tidak tampil HABIS; (b) `Stok Lama = 0`, HPP = harga baru, tanpa pembagian nol, stok tidak berubah saat konfirmasi; (c) `purchase_reverse` dan uang laci dikembalikan |
| TC-63 | R1a | Konsistensi lokal | Pull tidak menimpa outbox | Tablet offline menjual 3 butir stok 10 (outbox belum terkirim); online; jalankan pull sebelum push selesai | Siklus selalu push lalu pull; stok tampil 7, bukan 10; saldo kasbon lokal = server + outbox |
| TC-64 | R1a | Perhitungan | Pembulatan, paket, curah | Nota Rp4.375 + Rp10.050 tunai; qty 4 pada paket 3=Rp10.000 (satuan Rp3.500); 1/4 kg x Rp17.500; QRIS untuk total yang sama | Total dibulatkan ke Rp100 (`rounding_adj` tercatat) untuk tunai; paket 4 = Rp13.500; curah Rp4.375; QRIS tanpa pembulatan; validasi `total_final = subtotal + rounding_adj` lolos |
| TC-65 | R1a | Tukar receh | Kombinasi tidak pas | Kembalian Rp300 dengan produk receh Rp500, Rp200, Rp100; lalu Rp100 dengan hanya Rp500; lalu tanpa produk receh | Rp300 = Rp200 + Rp100 (kembalian jadi Rp0); Rp100 dengan hanya Rp500: tombol tidak menambah barang dan kembalian tunai tetap; tanpa produk receh tombol tersembunyi |
| TC-66 | R2 | Stok opname | Penjualan terlambat | Hitung fisik produk A di HP Pengelola saat tablet offline menyimpan penjualan A yang terjadi sebelum hitung; sinkronkan setelah opname | Stok akhir = hitungan fisik; penjualan terlambat tercatat tetapi `applied_to_cache = false`; tidak ada koreksi ganda |
| TC-67 | R1a | Backup | Heartbeat dan pengawas | Nonaktifkan workflow backup selama > 36 jam | `pg_cron` memicu notifikasi ke Pengelola; peringatan juga dari kegagalan eksplisit; uji ulang setelah workflow dinyalakan |
| TC-68 | R1a | Performa NFR | Angka performa | Android RAM 3-4 GB: cold boot sampai siap memindai; Bayar sampai tersimpan; IndexedDB setelah simulasi 500 nota/hari selama 3 hari | Cold boot < 2 dtk; simpan < 300 ms; kalkulasi < 16 ms; IndexedDB < 50 MB |
| TC-69 | R2 | Setoran kas | Konfirmasi dan opname kas | Kasir menyetor Rp500.000; Pengelola (a) menekan UANG DITERIMA, (b) pada setoran lain menekan TIDAK DITERIMA; hitung ulang Kas Simpanan | (a) laci -500.000 seketika, Kas Simpanan +500.000 setelah konfirmasi; (b) laci kasir dikembalikan, Audit Log; opname kas mencatat `adjustment` |
| TC-70 | R1b | Kasbon | Pelanggan baru dari kasir | Kasir menambah "Bpk. Joko" dan mengkasbonkan Rp50.000 | Limit Rp0 sehingga `DEBT_OVERLIMIT` diminta; setelah disetujui nota lanjut; tablet tidak menyimpan telepon/alamat; Pengelola dapat menetapkan limit |
| TC-71 | R1a | Sync | Bundle besar | Tablet offline membuat 200 nota, lalu online | Klien memecah menjadi bundle <= 50 item; tidak ada timeout; semua id tersinkron; bundle > 50 yang dikirim manual ditolak `BUNDLE_TOO_LARGE` |
| TC-72 | R1a | Provisioning | Token pairing dan JWT | Pakai token pairing dua kali; pakai token > 10 menit; biarkan JWT perangkat kedaluwarsa lalu online | Token terpakai/kedaluwarsa ditolak `PAIRING_INVALID`; JWT kedaluwarsa diperpanjang lewat `refresh_device_token`; perangkat dicabut tidak dapat memperpanjang |
| TC-73 | R1a | Pencegahan salah | Konfirmasi angka tidak wajar | (a) Total Rp50.000 dibayar Rp500.000; (b) jumlah 100 untuk barang non-curah; (c) Barang Lain Rp600.000; (d) total nota Rp1.200.000; (e) transaksi biasa di bawah semua ambang | (a)-(d) muncul pertanyaan "... BENAR?" dengan BENAR, LANJUT / UBAH LAGI; transaksi belum tersimpan sebelum BENAR; UBAH LAGI mempertahankan keranjang; (e) tanpa pertanyaan tambahan |
| TC-74 | R1a | Bantuan & baterai | Tombol bantuan dan indikator baterai | Tekan BANTUAN lalu HUBUNGI PEMILIK; buka ketiga panduan tanpa internet; turunkan baterai ke 15%; jalankan di peramban tanpa Battery API; kosongkan `help_phone` | WhatsApp terbuka ke `help_phone` dengan teks siap kirim; panduan terbuka offline; pita "COLOK PENGISI DAYA" tampil; tanpa Battery API indikator tersembunyi tanpa galat; tanpa `help_phone` tombol WhatsApp tersembunyi |
| TC-75 | R1a | HP Pemilik | Beranda Sederhana dan pemulihan akses | (a) Login Pemilik (`simple_mode`); (b) matikan mode untuk satu anggota; (c) Pemilik ganti HP: Administrator mencabut HP lama, mengirim tautan masuk, membuat token pairing, daftarkan HP baru | (a) hanya tombol besar yang tersedia di rilis itu + MENU LAINNYA, tanpa tombol mati; (b) Beranda Lengkap tampil; (c) HP baru berfungsi dengan hak yang sama; HP lama langsung ditolak |
| TC-76 | R1a | Setup awal | Runbook dan katalog | Jalankan runbook Tech Stack bagian 9 pada proyek kosong: seed, undang Pemilik, daftarkan HP dan tablet, impor CSV 100 produk, unggah foto, pilih 8 favorit | Laci, Kas Simpanan, Barang Lain, pengaturan terbuat; kasir dapat menjual dari katalog; favorit tampil di halaman 1; TC-56 dan TC-67 lulus |

### Kriteria Lulus Tiap Rilis
- Semua TC yang kolom **Rilis**-nya memuat rilis tersebut lulus di perangkat nyata warung (TC bertanda dua rilis, mis. `R1a; R1b`, diuji sebagian di tiap rilis sesuai fiturnya).
- Tiap rilis juga menjalankan ulang TC rilis sebelumnya yang relevan (regresi).
- R1a: TC-56 (RLS) dan TC-67 (heartbeat backup) wajib lulus sebelum data nyata dimasukkan.
- UAT satu hari penuh dengan penjaga lansia tanpa kehilangan data; catatan kebingungan dibukukan sebagai perbaikan UI.
