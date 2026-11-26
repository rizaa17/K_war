# Spesifikasi Fungsional & Acceptance Criteria - K_War
Versi 2.4 - 26 November 2026
Dokumen ini adalah sumber kebenaran. Aturan perubahan: lihat PRD bagian 11.
Penomoran berurutan per modul. Tanda **[R1a/R1b/R2/R3]** menunjukkan rilis (lihat Roadmap).

## Modul 1: Perangkat, Anggota & Penjaga
### F-01: Pendaftaran Perangkat (Device Provisioning) [R1a]
- **Given** Pemilik membuka "Kelola Perangkat" di HP-nya. **When** menekan "Daftarkan Perangkat Baru" dan memindai QR/memasukkan token pairing di perangkat baru. **Then** perangkat terdaftar dengan `install_id`, kode unik (DEV1/TAB1), peran (`hp_manager`/`tablet_cashier`), menerima JWT perangkat, status aktif. Token pairing: acak 128-bit, sekali pakai, berlaku 10 menit, dibuat Pemilik. JWT perangkat berlaku 30 hari dan diperpanjang otomatis tiap sinkronisasi sukses (Spesifikasi API bagian 1).
- **When** Pemilik menekan "Cabut Izin". **Then** perangkat terkunci ke layar aktivasi pada koneksi berikutnya; data offline yang belum terkirim tetap ada dan dikirim ke karantina server untuk ditinjau (tidak dihapus). Untuk HP Pengelola, pencabutan juga menghapus sesi login HP itu sehingga ia tidak dapat lagi menyetujui apa pun.

### F-02: Pengaturan Hak Akses Anggota [R1a]
- Pemilik mengatur peran, akses laci, foto kartu, dan "Kerabat Tepercaya".
- **When** "Kerabat Tepercaya = ON", Pemilik mengirim **tautan undangan** sekali pakai ke Kerabat; Kerabat membuka tautan di HP-nya dan membuat login (Supabase Auth). Pemilik lalu mendaftarkan HP itu lewat F-01 (Kerabat sendiri tidak dapat mendaftarkan atau mencabut perangkat). Setelah itu Kerabat dapat memberi Persetujuan QR. **Tidak ada PIN.**
- **When** "Akses Laci = Hanya Laci Pribadi", anggota tidak dapat membuka Laci Bersama.

### F-03: Layar "Siapa yang Jaga?" [R1a]
- **Given** Tablet Kasir terbuka tanpa sesi aktif. **Then** tampil kartu besar berfoto dan nama tiap anggota aktif.
- **When** kasir menyentuh kartunya. **Then** sistem langsung ke layar MULAI JAGA (atau dialog pilihan laci hanya bila "Bebas Pilih"). Semua pilihan di tablet berhak setingkat kasir. **Keputusan Pemilik: tanpa PIN penjaga**; risiko memilih nama orang lain diterima dan ditutup oleh Audit Log serta laporan shift.

## Modul 2: Kasir, Multi-Keranjang & Input Barang
### F-04: Input Barang Multi-Metode [R1a]
- **When** barcode scanner membaca kode. **Then** barang masuk keranjang aktif < 100 ms.
- **When** kasir menyentuh kartu produk / mencari nama / memindai dengan kamera. **Then** barang masuk keranjang.
- **When** produk bertipe curah dipilih. **Then** muncul Modal Takaran Curah (1/4, 1/2, 1, 2 kg + input timbangan).
- Grid utama berisi **6 sampai 8 barang favorit** (kartu besar: foto, nama, harga) dengan posisi tetap. **When** kasir menggeser ke samping (atau menekan panah besar ">"). **Then** muncul halaman barang lain, satu halaman per kategori dengan judul kategori besar dan 6-8 kartu urut abjad. Titik indikator halaman selalu tampil.
- Susunan grid tidak berubah sendiri. Kartu hanya menampilkan status stok bila MENIPIS, KRITIS, atau HABIS.
- **When** barcode kardus (`box_barcode`) dipindai atau "JUAL KARDUS" ditekan. **Then** satu kardus masuk keranjang (`is_box_sale`) dan stok eceran berkurang sesuai isi kardus.

### F-05: Multi-Keranjang (5) [R1a]
- **When** "+ Keranjang Baru" ditekan. **Then** pindah ke keranjang kosong tanpa menghapus yang lama.
- **When** aplikasi ditutup paksa atau daya mati. **Then** kelima keranjang pulih utuh dari Dexie.js.

### F-06: Barang Lain (Harga Bebas) [R1a]
- **When** kasir menekan "BARANG LAIN". **Then** muncul modal: nama (opsional, bawaan "Barang Lain") dan harga dengan keypad jumbo.
- **Then** butir masuk keranjang; stok tidak berubah; omzet tercatat dan muncul di laporan sebagai "Barang Lain".

### F-07: Batalkan Barang Terakhir & Anti Tekan Ganda [R1a]
- **When** barang baru ditambahkan. **Then** muncul tombol besar "BATALKAN BARANG TERAKHIR" dan **tetap tampil** sampai barang berikutnya ditambahkan atau BAYAR ditekan (tidak hilang sendiri).
- **When** tombol Bayar/Selesai ditekan. **Then** tombol nonaktif sampai proses selesai; transaksi tidak tersimpan dua kali.

### F-08: Buka Shift dan Modal Awal [R1a]
- **Given** tidak ada sesi laci aktif untuk penjaga ini. **Then** layar MULAI JAGA muncul otomatis dan layar transaksi terkunci.
- Tersedia keypad jumbo dan tombol cepat (Rp50.000, 100.000, 150.000, 200.000, 300.000, 500.000).
- **When** modal dikonfirmasi. **Then** sesi `open`, `opening_balance` tersimpan di IndexedDB dan tercatat sebagai `cash_movements` jenis `opening`; kasir dialihkan ke layar penjualan.
- **Given** sesi anggota ini masih terbuka dari hari sebelumnya (Sesi Yatim). **Then** layar SELESAI JAGA ditampilkan lebih dulu dengan pesan "Giliran jaga kemarin belum selesai"; giliran baru baru dapat dimulai setelah itu.
- **Given** laci yang dipilih masih punya sesi terbuka di tablet lain (diketahui lokal atau dari server). **Then** pembukaan ditolak dengan pesan jelas; bila tidak diketahui (offline), server menandai `SESSION_OVERLAP` saat sinkronisasi.

### F-09: Indikator Status Sinkronisasi [R1a]
- Bilah atas selalu menampilkan teks besar: "TERSAMBUNG - SEMUA TERKIRIM", "TANPA INTERNET - n nota menunggu", atau "n NOTA BERMASALAH - hubungi Pemilik". Tidak hanya warna.
- Angka "n NOTA BERMASALAH" hanya menghitung nota **ditolak**. Nota Bertanda tidak tampil di kasir.
- Bilah atas juga menampilkan **BATERAI n%**; bila <= 20% muncul pita "COLOK PENGISI DAYA". Bila peramban tidak menyediakan data baterai, indikator disembunyikan (tanpa galat).

## Modul 3: Pembayaran, Kembalian & Void
Layar utama hanya punya satu tombol hijau besar **BAYAR**; metode dipilih di modal berikutnya.

### F-10: Pembayaran Tunai & Kembalian Bersuara [R1a]
- **Given** total Rp45.000. **When** tunai Rp50.000. **Then** kembalian Rp5.000 dan, bila toggle suara ON, terdengar "Kembalian lima ribu rupiah" (offline).
- Saat aplikasi pertama dipasang, ada tombol uji suara untuk memastikan paket suara bahasa Indonesia tersedia.

### F-11: Tukar Kembalian 1-Klik [R1a]
- **When** dipilih. **Then** sistem memilih barang bertanda `is_change_item` (paling banyak 5 butir) yang jumlahnya pas dengan kembalian, ditambahkan sebagai baris biasa sehingga kembalian Rp0. Bila tidak ada kombinasi yang pas, dipakai kombinasi terbesar yang tidak melebihi kembalian; sisanya tetap diberikan tunai dan layar menyatakannya jelas (mis. "Sisa kembalian tunai Rp200"). Tombol disembunyikan bila belum ada produk receh.

### F-12: Pembayaran Split Tunai + Kasbon [R1b]
- **Given** total Rp100.000, tunai Rp40.000, kasbon Rp60.000. **Then** laci +Rp40.000 dan saldo kasbon pelanggan +Rp60.000. Jika kasbon melewati limit, lihat F-30.

### F-13: Pembayaran QRIS Statis [R2]
- **Given** gambar QRIS sudah diunggah. **Then** tombol "QRIS" muncul; ditekan menampilkan gambar QRIS dan nominal besar.
- Gambar belum diunggah: tombol tersembunyi total.
- **When** pembayaran dirasa masuk. **Then** kasir wajib menekan "UANG SUDAH MASUK" setelah mengecek notifikasi pembayaran; nota baru selesai setelahnya, waktunya dicatat.

### F-14: Pakai Sendiri [R2]
- **When** kasir memilih "PAKAI SENDIRI" di modal bayar. **Then** muncul QR Persetujuan (`PERSONAL_USE`). Setelah disetujui, stok berkurang dengan alasan `personal_use`, tanpa uang masuk.

### F-15: Cetak Struk 58mm & Kirim Nota via WhatsApp [R2]
- **When** "CETAK STRUK" ditekan. **Then** perintah cetak dikirim ke printer thermal (Web Bluetooth/WebUSB).
- **When** "KIRIM LEWAT WHATSAPP" ditekan. **Then** WhatsApp terbuka dengan teks nota siap kirim (`wa.me`).
- Tanpa menekan keduanya, transaksi tetap sah.
- Catatan: printer harus mendukung Bluetooth LE atau USB; uji perangkat sebelum dibeli (lihat Tech Stack).

### F-16: Penomoran Nota Unik [R1a]
- **Given** perangkat `DEV1` bertransaksi pertama pada 26 November 2026 (menurut jam perangkat). **Then** nomor nota `kw/0001/DEV1/26112026`; transaksi berikutnya `0002`, dst.; hari baru kembali ke `0001`.
- Counter disimpan di `appSettings` dan tidak terpengaruh purge.
- Bila server menemukan nomor sama dengan `id` berbeda, nota diterima dengan akhiran `~xxxx` dan ditandai `INVOICE_NO_CONFLICT` (Aturan Bisnis bagian 4).

### F-17: Void Nota & Retur Barang [R1a untuk shift sendiri; R1b untuk nota lama]
- **When** kasir menekan "BATALKAN / KEMBALIKAN" pada nota atau keranjang **dari shift aktifnya**. **Then** wajib pilih alasan: "Salah Tekan", "Pembeli Batal", "Barang Dikembalikan". Stok kembali, saldo laci dan kasbon dikoreksi lewat catatan pergerakan, tercatat di Audit Log. Void dikirim sebagai catatan `voids` di `push_bundle` (juga untuk nota yang sudah tersinkron).
- **Retur sebagian**: kasir memilih butir dan jumlah yang dikembalikan (alasan "Barang Dikembalikan"); hanya bagian itu yang dikoreksi (stok, laci, kasbon). Dikirim sebagai `returns`.
- **When** nota bukan dari shift aktif milik kasir itu (shift lain atau shift sudah ditutup). **Then** butuh Persetujuan QR (`VOID_LATE`) dan internet; data nota selalu diambil dari server. Berlaku untuk void maupun retur sebagian.
- "Barang rusak" bukan alasan void; gunakan F-23.

### F-18: Tutup Shift Blind Count [R1a]
- **When** kasir mengisi jumlah lembar (100rb, 50rb, 20rb, 10rb, 5rb, 2rb, 1rb) dan keping koin (1000, 500, 200, 100). **Then** total fisik dihitung otomatis.
- **When** "SELESAI JAGA & KUNCI" ditekan. **Then** kasir hanya melihat "Selesai Jaga, Terima Kasih".
- **Then** HP Pengelola menerima laporan: uang fisik, saldo sistem, selisih (kurang/lebih), dan rincian pecahan, serta **rekap QRIS** shift itu (jumlah nota dan total) agar dapat dicocokkan dengan mutasi pembayaran QRIS.
- **Then** selisih dinyatakan "sementara" sampai semua catatan shift itu sudah diterima server (`is_complete`); setelah itu berlabel final.

## Modul 4: Kasbon
### F-19: Buku Kasbon & Cicilan oleh Kasir [R1b]
- Kasir memilih pelanggan dari kartu bernama besar, menginput cicilan (misal Rp50.000 dari hutang Rp150.000), menekan "Terima Pembayaran". **Then** hutang jadi Rp100.000 (catatan buku kasbon), laci aktif +Rp50.000, bukti setor dapat dicetak/dikirim WhatsApp (nomor diketik kasir; telepon pelanggan tidak ada di tablet).
- **Pelanggan baru**: kasir menekan "TAMBAH PELANGGAN" dan mengisi nama (telepon opsional). Pelanggan dibuat dengan limit Rp0; kasbon pertamanya memicu F-30 (`DEBT_OVERLIMIT`) sampai Pengelola menetapkan limit.

### F-20: Pengingat Kasbon via WhatsApp [R2]
- **Given** pelanggan punya hutang dan nomor telepon. **When** Pengelola menekan "INGATKAN". **Then** WhatsApp terbuka dengan kalimat sopan, tanpa denda atau tenggat. Tidak ada pengiriman otomatis.

## Modul 5: Inventori & Kulakan
### F-21: Penerimaan Barang Kulakan oleh Kasir [R2]
- **When** sales datang, kasir memilih supplier, menginput jumlah fisik per barang, (opsional) tanggal kadaluwarsa untuk barang yang dilacak, dan uang yang dibayar dari laci. **Then** faktur berstatus "Menunggu Konfirmasi Harga"; laci berkurang seketika (`purchase_payment`); **stok jual bertambah seketika** (`purchase_receive`, Penerimaan Sementara). Kekurangan bayar dari Kas Simpanan dicatat saat Pengelola mengonfirmasi harga (total faktur baru diketahui saat itu). Kasir tidak melihat harga modal.

### F-22: Konfirmasi Faktur & HPP oleh Pengelola [R2]
- **When** Pengelola membuka "Faktur Menunggu" di HP-nya dan memasukkan harga per barang (atau memotret faktur untuk diisi otomatis), lalu menekan "SETUJUI". **Then** HPP Moving Average dihitung ulang (Aturan Bisnis 5.3), kekurangan bayar dicatat dari Kas Simpanan, kenaikan harga ditandai titik merah, kelebihan/kekurangan jumlah dibanding hitungan kasir ditandai. Stok tidak berubah saat konfirmasi.
- **When** Pengelola menekan "TOLAK". **Then** pergerakan Penerimaan Sementara dibalik (`purchase_reverse`), uang laci dikembalikan lewat catatan koreksi, dan alasan wajib dicatat di Audit Log.
- Foto Faktur [R3]: hasil bacaan AI hanya mengisi kolom; Pengelola wajib memeriksa sebelum menyetujui.

### F-23: Penyesuaian Stok Rusak/Kadaluwarsa [R2]
- **Given** 2 bks minyak bocor. **When** kasir memilih produk, jumlah 2, alasan "Rusak/Bocor" (atau "Kadaluwarsa"). **Then** QR Persetujuan (`DAMAGED_STOCK`) muncul; stok terpotong hanya setelah disetujui. Penyesuaian punya `id` dari klien dan satu persetujuan hanya berlaku untuk satu penyesuaian.

### F-24: Stok Opname [R2]
- Hanya di HP Pengelola: hitung fisik per produk, sistem membuat `stock_movements` jenis `opname` untuk selisih, tercatat di Audit Log.
- Selisih dihitung dengan `counted_at` (Aturan Bisnis 5.11) sehingga penjualan offline yang terlambat tiba tidak menggandakan koreksi.

### F-25: Gas Elpiji & Isi Ulang [R2]
- Produk bertipe `refill` menampilkan pilihan "TUKAR TABUNG KOSONG" (bawaan) atau "BELI TABUNG BARU".
- **When** tukar dipilih. **Then** stok isi -1, stok kosong +1.
- Saat kulakan dari agen, kasir menginput juga jumlah tabung kosong yang dikembalikan (stok isi +N, kosong -N).

### F-26: Barang Konsinyasi (Titipan) [R2]
- Penerimaan titipan hanya menambah stok (`consignment_in`), tanpa pembayaran.
- **Then** HP Pengelola menampilkan "Utang ke Penitip" (terjual x harga setor - pembayaran). **When** "BAYAR PENITIP" ditekan dengan nominal. **Then** tercatat sebagai pengurang laci/Kas Simpanan.
- Barang tak laku dikembalikan lewat penyesuaian stok `consignment_return`.

### F-27: Pelacakan Kadaluwarsa [R2]
- Untuk produk dengan `track_expiry`, tanggal terdekat disimpan. **Then** Pengelola menerima pemberitahuan H-30 (angka dapat diatur) **sekali** per produk per tanggal, dan daftar "Segera Habis Masa Berlaku".
- **When** batch lama habis. **Then** Pengelola menekan "ED TERBARU" pada produk itu untuk mengganti tanggal; tanggal juga dikosongkan otomatis bila stok <= 0.

### F-28: Daftar Belanja Pasar + Kirim ke Sales [R2]
- Daftar disusun menurut velocity tertinggi. **When** "KIRIM KE WHATSAPP" ditekan. **Then** WhatsApp terbuka berisi daftar barang dan jumlah.

### F-29: Pendaftaran Produk Mudah [R1a untuk (a) dan (b); R3 untuk (c)]
- Pengelola dapat: (a) mengimpor file CSV dari templat; (b) memindai barcode dengan kamera, memotret kemasan (dikompresi WebP), mengisi nama dan harga; (c) opsional [R3] mengisi nama otomatis dari foto kemasan oleh AI.
- Pengelola juga dapat menambah/mengubah produk satu per satu di HP-nya (nama, harga, barcode, kategori, foto, batas stok).
- **Favorit**: Pengelola memilih 6 sampai 8 barang (maksimal 8) dan mengatur urutannya; urutan itu tetap sampai Pengelola mengubahnya. Bila belum dipilih, halaman 1 berisi 8 produk pertama menurut abjad.

## Modul 6: Persetujuan QR
### F-30: Persetujuan QR Online [R1b]
- **Given** kasbon melewati limit (atau F-14, F-17, F-23). **When** kasir menekan "MINTA PERSETUJUAN". **Then** tablet menampilkan QR dan teks "Minta Pemilik memindai".
- **When** Pengelola memindai QR (tombol "PINDAI PERMINTAAN") atau membuka notifikasi, lalu menekan "SETUJUI". **Then** tablet menutup modal dengan status disetujui dan transaksi dapat dilanjutkan.
- **When** "TOLAK". **Then** transaksi tidak dapat dilanjutkan dengan cara itu.
- **Dua jalur untuk permintaan yang sama**: (A) Pemilik memindai QR di tablet; (B) permintaan masuk ke HP Pengelola sebagai notifikasi, diketuk, lalu menekan "SETUJUI" (jika `allow_remote_approval` aktif). Tablet terbuka otomatis pada kedua jalur.
- QR kedaluwarsa 5 menit; dapat dibuat ulang. Jika tablet offline: pesan jelas "Tidak ada internet, persetujuan belum bisa."
- Satu persetujuan hanya dapat dipakai sekali. Isi yang diikat hash dan cara membuatnya ada di Spesifikasi API bagian 6; mengubah isi setelah disetujui membuat persetujuan tidak berlaku dan tablet meminta ulang.
- R1b: jalur notifikasi memakai Web Push khusus permintaan persetujuan; daftar "Permintaan Menunggu" selalu tersedia sebagai cadangan.

### F-31: Persetujuan QR Offline (QR Dua Arah) [R3]
- Tablet menampilkan QR permintaan; HP Pengelola memindai tanpa internet dan menampilkan QR jawaban bertanda tangan; tablet memindainya dan memverifikasi. Bukti tersinkron nanti.

## Modul 7: Pengelola
### F-32: Dasbor Ringkas & Bacakan Laporan [R2]
- Beranda lengkap (untuk yang tidak memakai Beranda Sederhana, F-44) menampilkan omzet, estimasi laba, jumlah nota, saldo laci, peringatan stok, nota kulakan menunggu, permintaan persetujuan menunggu, nota bermasalah dan Nota Bertanda menunggu tinjauan, serta rekap QRIS per shift. Definisi omzet/laba mengikuti Aturan Bisnis 4.15.
- **When** "BACAKAN LAPORAN" ditekan. **Then** ringkasan dibacakan suara (Web Speech API, offline).

### F-33: Notifikasi Push [R1b untuk permintaan persetujuan; R2 untuk jenis lain]
- Pengelola menerima notifikasi: permintaan persetujuan baru, shift ditutup (dengan selisih), stok habis, kadaluwarsa, nota bermasalah. Dapat dimatikan per jenis.

## Modul 8: Fitur AI Gratis
### F-34: Voice Quick Order [R3]
- **Given** toggle ON dan perangkat online. **When** kasir menekan mikrofon dan berkata "Gula pasir satu kilo dan telur dua kilo". **Then** teks dikenali peramban dan dicocokkan ke katalog oleh pencocok lokal (tanpa AI); kasir mengonfirmasi daftar sebelum masuk keranjang.
- Offline atau toggle OFF: tombol mikrofon redup/tersembunyi.

### F-35: Smart Insight [R2 aturan lokal, R3 poles AI]
- Ambang kejanggalan memakai nilai di pengaturan toko (bawaan: >= 3 void per shift; selisih kas > Rp20.000 atau 2 shift beruntun; kenaikan HPP >= 10%).
- Setelah shift ditutup, Pengelola melihat 2-3 kalimat: produk terlaris, perbandingan omzet, barang menipis, serta tanda kejanggalan (void berulang, selisih kas berulang, harga modal naik). Dibuat oleh aturan lokal; pemolesan AI opsional.

## Modul 9: Data Lokal, Ketahanan & Aksesibilitas
### F-36: Retensi 1 Hari & Auto-Purge [R1a]
- Dijalankan saat aplikasi dibuka dan setelah sinkronisasi sukses.
- `is_synced = 1` dan `synced_at <= sekarang - 72 jam` (3 hari; `local_retention_days`) dihapus dari IndexedDB. `is_synced = 0` tidak pernah dihapus. Data server tidak terpengaruh.

### F-37: Mode Latihan [R1a]
- **When** Pengelola/kasir memilih "MODE LATIHAN". **Then** aplikasi memakai database lokal terpisah, ada pita kuning besar "LATIHAN - TIDAK TERSIMPAN"; tidak ada sinkronisasi atau perubahan stok/kas nyata.

### F-38: Penyimpanan Persisten & Pembaruan Aman [R1a]
- Saat pertama dibuka, aplikasi meminta `navigator.storage.persist()` dan memandu pemasangan PWA.
- Pembaruan aplikasi hanya diterapkan saat tidak ada keranjang berisi dan transaksi berjalan, lewat tombol besar "PERBARUI SEKARANG".

### F-39: Pengaturan Aksesibilitas [R1a]
- Pengguna dapat memilih ukuran huruf 3 tingkat (normal, besar, sangat besar); pilihan tersimpan per perangkat.
- Bunyi dan getar singkat saat barang masuk keranjang (dapat dimatikan).
- Toggle "Ucapkan nama barang" mengucapkan nama barang yang baru masuk (Web Speech API, offline).

## Modul 10: Kas & Peninjauan (tambahan v2.3)
### F-40: Setoran & Transfer Kas [R2]
- **Given** kasir ingin menyetor uang ke Kas Simpanan. **When** menekan "SETOR KE KAS SIMPANAN" dan mengisi nominal. **Then** laci aktif berkurang seketika (`transfer_out`); transfer berstatus "Menunggu Konfirmasi"; Kas Simpanan belum bertambah.
- **When** Pengelola membuka "Setoran Menunggu" dan menekan "UANG DITERIMA". **Then** Kas Simpanan bertambah (`transfer_in`); bila "TIDAK DITERIMA", transfer ditolak, laci kasir dikembalikan lewat catatan koreksi, dan Audit Log mencatatnya.
- **When** Pengelola memindahkan uang antar laci di HP-nya. **Then** kedua catatan langsung dibuat tanpa konfirmasi tambahan.
- **When** Pengelola menghitung ulang Kas Simpanan (opname kas). **Then** selisih dicatat sebagai `adjustment` dengan alasan wajib di Audit Log.

### F-41: Tinjauan Nota Bertanda & Antrean Bermasalah [R1b]
- Pengelola melihat dua daftar di layar **PERLU DICEK** (diterima, ada tanda seperti `OVER_LIMIT_NO_APPROVAL`, `PRODUCT_DELETED`, `INVOICE_NO_CONFLICT`, `PRICE_DEVIATION`, `OPENING_MISMATCH`, `SESSION_OVERLAP`, `CLOCK_SKEW`) dan **NOTA BERMASALAH** (ditolak atau karantina; Nota Bertanda = Perlu Dicek, Antrean Bermasalah = Nota Bermasalah).
- **When** Pengelola menekan "SUDAH DICEK" pada nota di daftar Perlu Dicek dengan catatan. **Then** nota ditandai ditinjau; tidak ada perubahan angka.
- **When** pada daftar Nota Bermasalah Pengelola menekan "COBA LAGI" (mis. setelah produk dipulihkan). **Then** server memproses ulang isinya; bila "BUANG", alasan wajib dan dicatat di Audit Log (data tetap disimpan sebagai `discarded`).
- Setiap tanda menampilkan penjelasan satu kalimat dalam bahasa sederhana dan tindakan yang disarankan.

## Modul 11: Ramah Lansia & Kesiapan Awal (tambahan v2.4)
### F-42: Konfirmasi Angka Tidak Wajar [R1a]
- **Given** salah satu kondisi berikut sebelum transaksi disimpan: kembalian >= Rp100.000; jumlah satu barang non-curah > 20; harga Barang Lain > Rp500.000; total nota > Rp1.000.000 (ambang di pengaturan toko). **Then** muncul pertanyaan besar, mis. "KEMBALIAN Rp450.000. BENAR?" dengan tombol "BENAR, LANJUT" dan "UBAH LAGI".
- **When** "UBAH LAGI". **Then** kembali ke isian tanpa kehilangan keranjang. **When** "BENAR, LANJUT". **Then** transaksi lanjut dan dicatat biasa.
- Di bawah semua ambang, tidak ada pertanyaan tambahan.

### F-43: Tombol Bantuan [R1a]
- Tombol "BANTUAN" selalu ada di bilah atas. **When** ditekan. **Then** muncul pilihan: "HUBUNGI PEMILIK LEWAT WHATSAPP" (membuka `wa.me` ke `help_phone` dengan teks "Mohon bantuan di kasir K_War (Tablet TAB1)"), dan panduan bergambar "CARA MENJUAL", "CARA MEMBAYAR", "CARA SELESAI JAGA".
- Panduan tersimpan di perangkat dan terbaca tanpa internet. Bila `help_phone` kosong, tombol WhatsApp disembunyikan.

### F-44: Beranda Sederhana HP Pemilik [R1a]
- **Given** `members.simple_mode = true` (bawaan untuk Pemilik). **Then** beranda hanya memuat tombol besar yang tersedia di rilis itu: "SETUJUI PERMINTAAN (n)" [R1b], "LAPORAN HARI INI" (dibacakan suara di R2), "DAFTAR BELANJA" [R2], dan "MENU LAINNYA". Fitur lain berada di "MENU LAINNYA".
- Tombol yang belum tersedia di rilis berjalan disembunyikan, bukan dibiarkan mati.
- Pemilik atau Kerabat dapat mematikan mode ini per anggota; Beranda Lengkap lalu tampil (F-32).

### F-45: Pemulihan Akses HP Pengelola [R1a]
- Login memakai tautan masuk, bukan kata sandi. **When** tautan hilang atau HP diganti. **Then** Kerabat atau Administrator Teknis mengirim tautan baru (`send_login_link`).
- **When** HP Pemilik diganti. **Then** Administrator Teknis mencabut HP lama, membuat token pairing baru lewat skrip admin, dan HP baru didaftarkan (F-01). HP lama langsung kehilangan hak (TC-54).

### F-46: Setup Awal oleh Administrator Teknis [R1a]
- Skrip seed dan langkah tertulis (Tech Stack bagian 9) membuat: pengaturan toko, Laci Bersama, Kas Simpanan, butir sistem "Barang Lain", akun dan HP Pemilik (`simple_mode = true`), tablet pertama, `help_phone`. Setelah itu produk diimpor (F-29) dan favorit dipilih.
- Setup dinyatakan selesai bila TC-56 (RLS) dan TC-67 (heartbeat backup) lulus.
