# Aturan Bisnis dan Glosarium - K_War
Versi 2.4 - 26 November 2026
Dokumen ini adalah sumber kebenaran. Aturan perubahan: lihat PRD bagian 11.

## 1. Glosarium Istilah Resmi
Istilah di bawah dipakai di dokumen dan kode. **Teks di layar memakai bahasa Indonesia sehari-hari** yang dipakai di warung (lihat UI/UX bagian 1 butir 9); tidak ada kamus terpisah, kata sehari-hari langsung dipakai di wireframe.
- **K_War**: Aplikasi kasir (POS) internal warung kelontong sembako.
- **Pengelola**: Pemilik atau Kerabat Tepercaya yang memakai HP-nya sendiri (HP Pengelola) untuk memberi persetujuan dan melihat data keuangan.
- **Administrator Teknis**: orang yang mengelola layanan teknis (akun Supabase/Cloudflare/GitHub); peran di luar aplikasi.
- **Tablet Kasir**: HP/tablet yang dipakai penjaga di meja kasir. Semua orang yang login di sini bertindak sebagai kasir.
- **Penjaga**: orang yang sedang bertugas di Tablet Kasir, dipilih lewat layar "Siapa yang jaga?".
- **Persetujuan QR**: pengesahan tindakan sensitif. Tablet Kasir menampilkan QR; Pengelola memindainya atau menekan Setujui/Tolak dari notifikasi di HP-nya. **Menggantikan PIN Master. K_War tidak memakai PIN.**
- **Laci Bersama**: kas gabungan toko, dipakai Pemilik dan Kerabat.
- **Laci Pribadi**: kas mandiri per kasir; kasir bertanggung jawab atas uang fisiknya.
- **Kas Simpanan Warung**: kas cadangan di luar laci; menutup kekurangan saat bayar kulakan.
- **Remote Revoke**: Pemilik mencabut izin perangkat dari jarak jauh.
- **Device Provisioning**: pendaftaran identitas perangkat, sekali di awal.
- **Moving Average Cost (HPP)**: harga pokok rata-rata bergerak, dihitung ulang tiap faktur kulakan disahkan.
- **Kasbon**: piutang belanja pelanggan tanpa bunga, denda, atau jatuh tempo kaku.
- **Split Payment**: sebagian tunai, sisanya kasbon.
- **Pakai Sendiri**: barang diambil keluarga untuk dipakai sendiri; stok berkurang, tanpa uang masuk, disahkan Persetujuan QR.
- **Barang Lain**: butir bebas (nama dan harga diketik kasir) untuk barang yang belum terdaftar; tidak mengubah stok.
- **Tukar Kembalian 1-Klik**: mengganti receh kembalian dengan barang kecil bernilai pas.
- **Isi Ulang / Tukar Tabung**: penjualan gas/galon dengan menyerahkan tabung/galon kosong; stok isi turun 1, stok kosong naik 1.
- **Konsinyasi (Titipan)**: barang titipan penitip; dibayar ke penitip hanya sebesar yang terjual.
- **Push Bundle**: paket data offline yang dikirim atomik per item ke Supabase saat online.
- **Pull Ledger**: penarikan perubahan katalog, harga, stok, pelanggan, dan pengaturan ke perangkat.
- **QRIS Statis Toko**: gambar QRIS yang diunggah Pengelola; kasir menunjukkannya ke pembeli.
- **Struk Thermal 58mm Opsional**: nota fisik hanya saat diminta pembeli.
- **Kode Perangkat**: pengenal ringkas (DEV1, TAB1) pada nomor nota.
- **Blind Count**: kasir menghitung uang fisik tanpa melihat saldo sistem; selisih hanya tampil di HP Pengelola.
- **Kalkulator Denominasi**: input jumlah lembar (Rp100rb, 50rb, 20rb, 10rb, 5rb, 2rb, 1rb) dan keping koin (Rp1000, 500, 200, 100).
- **Modal Awal Laci**: uang kas awal yang diinput kasir saat membuka shift.
- **Retensi Lokal 3 Hari**: transaksi yang sudah tersinkron disimpan 72 jam di perangkat lalu dibersihkan (`store_settings.local_retention_days`).
- **Antrean Bermasalah**: item `push_bundle` yang **ditolak** server karena datanya rusak (mis. total tidak cocok, produk tidak pernah ada) atau dikirim perangkat yang sudah dicabut (karantina); disimpan untuk ditinjau, tidak dibuang.
- **Mode Latihan**: database terpisah untuk berlatih; tidak menyentuh stok, kas, atau server.
- **Bacakan Laporan**: laporan dasbor dibacakan suara (Web Speech API, gratis).
- **Nota Bertanda**: nota yang **diterima** server tetapi gagal satu pemeriksaan lunak (mis. kasbon melewati limit yang usang, produk sudah dihapus, nomor nota bentrok). Tercatat penuh di kas, stok, dan kasbon; ditinjau Pengelola. Berbeda dengan nota **ditolak**.
- **Penerimaan Sementara**: saat kasir menerima barang kulakan, stok jual langsung bertambah sesuai hitungan kasir; harga dan HPP baru menyusul saat Pengelola mengonfirmasi faktur.
- **Setoran Kas**: perpindahan uang dari laci ke Kas Simpanan (atau sebaliknya), tercatat sebagai pasangan `transfer_out`/`transfer_in`, menunggu konfirmasi Pengelola bila dibuat kasir.
- **Sesi Yatim**: shift yang masih terbuka melewati `shift_max_hours` (bawaan 16 jam) tanpa Tutup Shift.
- **Favorit**: 6 sampai 8 barang yang ditetapkan Pemilik/Kerabat dan selalu tampil di halaman pertama grid kasir dengan urutan tetap (`products.favorite_order`).
- **Beranda Sederhana**: tampilan HP Pemilik dengan 3 tombol besar saja; fitur lain di "MENU LAINNYA" (`members.simple_mode`).
- **Kasir Umum**: istilah resmi untuk `regular_staff` (disebut "Pegawai Kasir Biasa" di dokumen lama).

## 2. Peran dan Hak Akses
Semua tindakan di Tablet Kasir setingkat kasir, siapa pun yang dipilih di "Siapa yang jaga?". Hak Pengelola hanya berlaku di **HP Pengelola** (login Supabase Auth, sesi panjang) atau lewat Persetujuan QR.

1. **Pemilik**: seluruh hak finansial dan operasional; tambah/hapus anggota, atur izin, daftarkan/cabut perangkat. Tidak perlu persetujuan apa pun di HP-nya sendiri.
2. **Kerabat Tepercaya**: hak finansial/operasional setara Pemilik di HP-nya (laba bersih, rekap semua laci, konfirmasi faktur/HPP, ubah harga, stok opname, Persetujuan QR). **Dilarang**: mengubah anggota, mendaftarkan/mencabut perangkat. HP Kerabat didaftarkan oleh Pemilik; Kerabat hanya menerima undangan login dari Pemilik (F-02).
3. **Kasir Umum**: transaksi penjualan, tahan keranjang, tukar receh, void/retur dengan alasan, terima cicilan kasbon, terima barang kulakan (jumlah saja), catat barang rusak/pakai sendiri (menunggu QR), tambah pelanggan baru (limit Rp0), setor kas ke Kas Simpanan (menunggu konfirmasi). Hanya Laci Pribadinya. **Dilarang**: melihat harga modal/HPP, laba, laci orang lain; mengonfirmasi faktur; mengubah harga; stok opname; menyetujui apa pun.

4. **Administrator Teknis** (di luar aplikasi, bukan anggota `members`): memegang akun Supabase, Cloudflare, GitHub; merilis pembaruan, memantau backup dan kuota, uji restore, serta mencabut perangkat darurat lewat dasbor Supabase. Tidak menyetujui transaksi kecuali terdaftar sebagai Kerabat Tepercaya.

## 3. Laci Kas & Alur Masuk
1. Anggota dengan 1 jenis laci langsung diarahkan ke kasir tanpa dialog pilihan.
2. Dialog pilihan laci hanya untuk anggota "Bebas Pilih".
3. Kasir wajib input **Modal Awal** sebelum bisa berjualan (`opening_balance`); ini dasar saldo akhir.
4. Saldo laci hanya berubah lewat catatan pergerakan kas (`cash_movements`): modal awal, penjualan tunai, cicilan kasbon, bayar kulakan, bayar titipan, void/retur, setoran/transfer kas.
5. Bayar kulakan dari laci mengurangi saldo laci seketika; kekurangan dicatat dari **Kas Simpanan Warung**.
6. Saldo sistem tidak pernah ditampilkan di layar kasir (blind count).
7. **Satu sesi terbuka per laci.** Tablet menolak membuka shift pada laci yang sesinya masih terbuka (diperiksa lokal, dan ke server bila online). Karena dua tablet offline dapat saling tidak tahu, server tetap menerima dan menandai `SESSION_OVERLAP` untuk ditinjau.
8. **Modal awal dicocokkan.** Server membandingkan `opening_balance` dengan `closing_balance_actual` sesi sebelumnya pada laci yang sama. Beda berarti ditandai `OPENING_MISMATCH` dan selisihnya ditunjukkan ke Pengelola (bukan ke kasir). Pengambilan uang antar shift harus lewat Setoran Kas (F-40) agar tidak tampak sebagai selisih.
9. **Sesi Yatim.** Bila sesi sebuah anggota masih terbuka di tablet ini, layar Buka Shift lebih dulu mewajibkan Tutup Shift sesi itu. Untuk sesi anggota lain yang yatim, Pengelola dapat **Tutup Paksa** (dicatat di Audit Log) dan memasukkan hitungan fisik.
10. **Selisih final.** Saat Tutup Shift, tablet mengirim `declared_counts` (jumlah nota, cicilan, penerimaan, setoran sesi itu). Server menghitung selisih hanya setelah jumlah yang diterima sama dengan yang dinyatakan (`is_complete`). Sebelum itu laporan ke Pengelola berlabel "selisih sementara".
11. **Setoran Kas.** Kasir dapat menyetor uang ke Kas Simpanan: laci kasir langsung berkurang (`transfer_out`), Kas Simpanan bertambah (`transfer_in`) hanya setelah Pengelola mengonfirmasi uangnya diterima. Pengelola dapat memindahkan uang dua arah antar laci langsung di HP-nya.
12. **Kas Simpanan** dapat dihitung ulang oleh Pengelola (opname kas); selisihnya dicatat sebagai `adjustment` dengan alasan di Audit Log.

## 4. Transaksi & Kasir
1. Maksimal 5 keranjang tertahan, tersimpan di IndexedDB.
2. Harga paket kelipatan otomatis (1 pcs Rp3.500; beli 3 = Rp10.000). Rumus: `subtotal = lantai(qty / bundle_qty) x bundle_price + (qty mod bundle_qty) x selling_price`. Contoh: qty 4 = 10.000 + 3.500 = 13.500. Hanya satu tingkat paket per produk.
3. Penjualan kardus memotong stok eceran sesuai isi kardus. Kardus punya barcode sendiri (`box_barcode`); memindainya otomatis menjual satu kardus (`is_box_sale`), dan kartu produk menyediakan tombol "JUAL KARDUS".
4. Kembalian bersuara 100% offline (Web Speech API), tanpa kuota AI.
5. **Void/retur** tanpa persetujuan hanya untuk nota **dari shift yang sedang aktif milik kasir itu sendiri**. Alasan baku: "Salah Tekan", "Pembeli Batal", "Barang Dikembalikan". Void/retur nota **mana pun di luar itu** (shift lain, shift sudah ditutup) butuh **Persetujuan QR (VOID_LATE)** dan internet, dan data nota selalu diambil dari server. Semua masuk Audit Log.
   - **Void** membatalkan seluruh nota. **Retur** dapat sebagian: memilih butir dan jumlah yang dikembalikan; stok kembali, saldo kasbon dikoreksi lebih dulu (`void_reversal`, sampai nilai kasbon nota habis), sisanya dikembalikan tunai dari laci (`void_refund`). Retur nota QRIS tidak menggerakkan laci; pengembalian uangnya manual dan dicatat di Audit Log.
   - Void dan retur adalah catatan baru yang dikirim lewat `push_bundle`; nota yang sudah tersinkron tidak ditimpa.
6. Barang rusak **bukan** alasan void; dicatat lewat Penyesuaian Stok (butuh Persetujuan QR).
7. **Barang Lain**: kasir mengetik nama dan harga; tidak mengubah stok; HPP dianggap tidak diketahui sehingga tidak dihitung dalam laba, tetapi omzetnya tercatat dan dilaporkan terpisah.
8. **Pakai Sendiri**: wajib Persetujuan QR; mengurangi stok (alasan `personal_use`), tanpa pemasukan, nilai modalnya dilaporkan sebagai pemakaian pribadi.
9. **QRIS Statis**: tombol hanya muncul jika gambar QRIS sudah diunggah. Kasir wajib menekan "Uang Sudah Masuk" setelah mengecek notifikasi pembayaran; waktu dan penjaga dicatat.
10. Nomor nota `kw/[urut 4 digit]/[kode perangkat]/ddmmyyyy`; urut direset per perangkat per hari, tanggal dan jam pada nota/struk mengikuti jam perangkat, counter disimpan permanen di `appSettings` (bukan dihitung dari isi `localTransactions`). Bila nomor yang sama sudah ada di server untuk perangkat itu dengan `id` berbeda (mis. counter terhapus karena pemasangan ulang), server **menerima** nota dan menambahkan akhiran `~xxxx` (4 karakter pertama `id`) pada nomornya, lalu menandai `INVOICE_NO_CONFLICT`. Nomor yang terlewat dalam urutan dilaporkan di ringkasan malam sebagai petunjuk transaksi hilang.
11. Tombol bayar tidak aktif sesaat setelah ditekan (anti tekan ganda); barang terakhir dapat dibatalkan satu langkah.
12. Stok boleh menjadi negatif akibat penjualan offline di dua perangkat; dilaporkan sebagai peringatan untuk dicek.
13. **Pembulatan.** `total_final = subtotal + rounding_adj`. Total nota dibulatkan ke kelipatan `store_settings.rounding_unit` terdekat (bawaan Rp100; 0 = tanpa pembulatan; setengah ke atas). Subtotal tiap baris dibulatkan ke rupiah terdekat (mis. 1/4 kg x Rp17.500 = Rp4.375). **QRIS statis tidak dibulatkan** (`rounding_adj = 0`).
14. **Tukar Receh.** Hanya memakai produk bertanda `is_change_item`. Sistem mencari kombinasi (paling banyak 5 butir) yang jumlahnya persis sama dengan kembalian; bila tidak ada yang pas, dipakai kombinasi terbesar yang tidak melebihi kembalian dan sisanya tetap diberikan tunai. Butir ditambahkan sebagai baris biasa (stok berkurang, omzet bertambah) sehingga `total_final` dan kembalian dihitung ulang.
15. **Definisi laporan.** Omzet = jumlah `total_final` nota `completed`, dikurangi retur, tidak termasuk nota `personal_use`. Laba = jumlah `(harga jual - cost_price_unit) x qty` butir selain "Barang Lain" dan `personal_use`, dikurangi retur. Pemakaian pribadi dilaporkan terpisah pada nilai modalnya. "Barang Lain" dilaporkan terpisah (hanya omzet).
16. **Konfirmasi angka tidak wajar.** Sebelum transaksi disimpan, layar bertanya "... BENAR?" (tombol BENAR, LANJUT dan UBAH LAGI) bila: kembalian >= `confirm_change_over` (bawaan Rp100.000); jumlah satu barang non-curah > `confirm_qty_over` (20); harga Barang Lain > `confirm_price_over` (Rp500.000); total nota > `confirm_total_over` (Rp1.000.000). Di bawah ambang tidak ada pertanyaan tambahan, agar kasir tidak terbiasa menekan tanpa membaca.
17. **Tombol BANTUAN** selalu ada di bilah atas: membuka WhatsApp ke `help_phone` (Pemilik) dengan teks siap kirim, dan panduan bergambar 3 langkah yang tersimpan offline.
18. **Baterai**: bilah atas menampilkan sisa baterai; <= 20% muncul pita "COLOK PENGISI DAYA". Bila peramban tidak menyediakan data baterai, indikator disembunyikan.

## 5. Inventori, Kulakan & HPP
1. Kasir hanya menginput **jumlah** barang datang dan uang yang dibayar dari laci, serta tanggal kadaluwarsa jika diminta. Kasir tidak melihat harga modal. **Stok jual langsung bertambah** sebesar jumlah itu (Penerimaan Sementara) agar barang yang sudah ada di rak tidak tampil HABIS.
2. Pengelola mengonfirmasi harga faktur di HP-nya (opsional bantuan Foto Faktur). Saat dikonfirmasi, HPP dihitung ulang dan kekurangan bayar diambil dari Kas Simpanan. **Stok tidak berubah lagi saat konfirmasi** (hitungan kasir sudah menjadi stok); perbedaan jumlah ditangani pada butir 4. Bila faktur **ditolak**, pergerakan Penerimaan Sementara dibalik (`purchase_reverse`) dan uang dari laci dikembalikan lewat catatan koreksi.
3. `Stok Lama = max(0, stok sekarang - Stok Masuk faktur ini)`; `HPP Baru = ((Stok Lama x HPP Lama) + (Stok Masuk x Harga Kulakan Baru)) / (Stok Lama + Stok Masuk)`. `Stok Masuk` harus > 0 (penyebut selalu > 0). Bila stok sudah negatif atau habis terjual, `Stok Lama = 0` dan HPP Baru = harga kulakan baru. Dihitung ulang tiap faktur disahkan..
4. Jika jumlah di faktur berbeda dari jumlah yang dihitung kasir, selisih ditandai untuk dicek Pengelola. Stok **tidak** diubah otomatis; hitungan fisik kasir dipercaya sampai ada opname.
5. Kenaikan harga modal (HPP Baru > HPP Lama) hanya ditandai titik merah di kartu produk (Pengelola), tanpa pop-up. Titik hilang saat Pengelola menekan "SUDAH DILIHAT" atau pada konfirmasi berikutnya yang tidak menaikkan HPP.
6. Status stok, selalu ditulis dengan teks dan ikon, tidak hanya warna: **HABIS** (stok <= 0, merah), **KRITIS** (stok <= `critical_stock_alert`, merah), **MENIPIS** (stok <= `min_stock_alert`, kuning, masuk daftar belanja), **AMAN** (biru).
7. Barang dengan velocity tertinggi di posisi teratas Daftar Belanja Pasar.
8. **Gas/galon (Isi Ulang)**: penjualan standar menukar tabung kosong (stok isi -1, kosong +1). Tabung baru adalah produk terpisah. Pembelian dari agen: stok isi +N, kosong -N.
9. **Konsinyasi**: stok masuk tanpa harga/pembayaran; harga setor ditetapkan Pengelola; utang ke penitip = jumlah terjual x harga setor - pembayaran sebelumnya. Barang tidak laku dikembalikan ke penitip lewat penyesuaian stok.
10. **Kadaluwarsa**: untuk produk bertanda `track_expiry`, sistem menyimpan tanggal kadaluwarsa **terdekat** (bukan per batch). Saat penerimaan: bila stok lama > 0, `nearest_expiry = min(lama, baru)`; bila tidak, = tanggal baru. Bila stok mencapai <= 0, tanggal dikosongkan. Pengelola dapat mengoreksinya ("ED TERBARU") setelah batch lama habis. Pemberitahuan H-30 (dapat diatur) dikirim **sekali** per produk per tanggal.
11. Stok opname hanya oleh Pengelola di HP-nya. Hitungan membawa `counted_at`. Selisih = `counted - (stok server - pergerakan yang berlaku dengan created_at > counted_at)`. Pergerakan ber-`created_at` sebelum `counted_at` yang baru tiba setelah opname **dicatat tetapi tidak diterapkan** ke stok (`applied_to_cache = false`) karena fisiknya sudah terhitung.

## 6. Kasbon Pelanggan
1. Atas nama pelanggan, tanpa tanggal jatuh tempo kaku, denda, atau bunga.
2. Setiap pelanggan punya limit yang ditetapkan Pengelola.
3. Kasbon yang membuat saldo melewati limit wajib **Persetujuan QR (DEBT_OVERLIMIT)**. Pelanggan baru yang ditambah kasir berlimit Rp0, sehingga kasbon pertamanya selalu butuh persetujuan sampai Pengelola menetapkan limit.
4. Kasir boleh menerima cicilan; uang masuk ke laci aktif kasir itu.
5. Saldo kasbon adalah hasil catatan buku kasbon (`customer_debt_ledger`), bukan satu angka yang ditimpa.
6. Pengingat kasbon hanya berupa tautan WhatsApp berisi kalimat sopan yang dikirim Pengelola; tidak ada pengiriman otomatis.
7. Tablet hanya menerima **nama** pelanggan, limit, dan saldo; telepon dan alamat hanya di HP Pengelola (minimisasi data pribadi). Nomor untuk bukti setor via WhatsApp diketik kasir saat itu.
8. Kasbon offline yang ternyata melewati limit karena data tablet usang **diterima dan ditandai** `OVER_LIMIT_NO_APPROVAL` (barang sudah keluar); Pengelola meninjau dan dapat menaikkan limit atau menagih.

## 7. Persetujuan QR
1. Jenis permintaan: `DEBT_OVERLIMIT`, `DAMAGED_STOCK`, `PERSONAL_USE`, `VOID_LATE`.
2. Hanya akun Pemilik atau Kerabat (login di HP-nya) yang dapat menyetujui; identitas diambil dari login server, bukan dari isi permintaan.
3. Token acak 128-bit, sekali pakai, berlaku 5 menit, terikat pada isi permintaan (nominal, pelanggan, barang, dan id transaksi/penyesuaian). Isi berubah berarti token tidak berlaku. Isi dan cara hash ditetapkan di Spesifikasi API bagian 6.
4. Cara menyetujui (dua jalur untuk permintaan yang sama): (A) pindai QR di tablet; (B) ketuk notifikasi atau buka daftar "Permintaan Menunggu" di HP Pengelola lalu tekan Setujui (jarak jauh; dapat dimatikan Pemilik). Keduanya butuh internet di tablet dan HP Pengelola.
5. Tablet offline tidak dapat menerima persetujuan online. Alternatif: **QR dua arah offline** (Rilis 3). Sebelum itu tersedia, kasbon over-limit saat offline ditolak.
6. Konfirmasi faktur, ubah harga, dan stok opname dilakukan langsung di HP Pengelola; tidak butuh QR.
7. **Satu persetujuan = satu penggunaan**: satu `approval_request_id` hanya boleh menempel pada satu transaksi/penyesuaian/void (dijaga indeks unik).

## 8. Retensi Data Lokal & Sinkronisasi
1. Transaksi `is_synced = 1` disimpan 72 jam (3 hari) di IndexedDB lalu dihapus otomatis. `synced_at` memakai waktu server dari respons `push_bundle`.
2. `is_synced = 0` tidak boleh dihapus dalam kondisi apa pun sebelum server mengonfirmasi.
3. Sebuah item hanya ditandai tersinkron setelah ada konfirmasi per-id dari server.
4. **Dua kelas hasil.** (a) **Ditolak** (data rusak: `MALFORMED`, `TOTAL_MISMATCH`, `ITEM_SUBTOTAL_MISMATCH`, `UNKNOWN_PRODUCT`, `UNKNOWN_SESSION`, `VOID_TARGET_UNKNOWN`, `DEVICE_REVOKED`): masuk Antrean Bermasalah, tampil di kasir sebagai "n NOTA BERMASALAH", dan dilaporkan ke Pengelola, yang dapat mencoba ulang atau membuang dengan alasan. (b) **Diterima dengan tanda** (kegagalan lunak, lihat Spesifikasi API bagian 2): tercatat penuh dan hanya terlihat oleh Pengelola, bukan oleh kasir.
5. Perangkat yang dicabut tetap boleh mengirim data tertunda; server menaruhnya di karantina untuk ditinjau Pemilik, bukan membuangnya.
6. Aplikasi meminta penyimpanan persisten (`navigator.storage.persist()`) dan harus dipasang sebagai PWA.
7. Void/retur nota di luar shift aktif sendiri selalu mengambil data dari server dan butuh internet, ada atau tidaknya salinan lokal.
8. Harga modal (HPP) tidak pernah dikirim ke Tablet Kasir.
9. Seluruh riwayat permanen ada di Supabase Postgres.
10. **Push dulu, baru pull.** Satu siklus sinkronisasi mengirim outbox lebih dulu, baru menarik `pull_ledger`. Pull tidak boleh menimpa data yang masih ada di outbox.
11. **Stok dan hutang lokal diturunkan**: stok tampil = `stock_qty` dari server - jumlah pada outbox yang belum tersinkron; saldo kasbon tampil = saldo server + selisih kasbon pada outbox.
12. Satu `push_bundle` maksimal 50 item; antrean lebih besar dipecah klien (hindari batas waktu statement server).
13. Jam perangkat dibandingkan dengan jam server tiap sinkronisasi. Selisih > 10 menit membuat peringatan ke Pengelola; nota yang `created_at`-nya berselisih > 24 jam dari waktu server saat diterima ditandai `CLOCK_SKEW`.

## 9. Empat Mode Input & Barang Curah
1. **Grid Favorit**: 6 sampai 8 kartu ekstra besar (foto, nama, harga) pilihan Pemilik dengan posisi tetap; geser (atau tekan panah besar) untuk halaman barang lain. Satu sentuhan masuk keranjang.
2. **Scan Barcode Fisik** USB/Bluetooth.
3. **Scan Kamera HP**.
4. **Pencarian Teks Cepat**.
5. Tanpa kode PLU atau angka hafalan. Barang curah (telur, beras, gula, cabai, bawang) memicu **Modal Takaran Curah** (1/4, 1/2, 1, 2 kg + input timbangan).
6. **Susunan grid tetap dan tidak berubah sendiri** (lansia mengandalkan ingatan letak). Halaman 1 = Favorit sesuai `favorite_order` (maksimal 8; bila belum dipilih, 8 produk pertama menurut abjad). Halaman berikutnya: satu halaman per kategori dengan judul kategori besar, 6-8 kartu per halaman, urut abjad. Kartu menampilkan status stok hanya bila MENIPIS/KRITIS/HABIS. Indikator halaman (titik) dan panah besar selalu tampil sebagai cadangan bagi yang sulit menggeser.
7. Barang yang tidak ada di katalog: gunakan **Barang Lain**.

## 10. Fitur AI & Notifikasi (Gratis)
1. Semua fitur AI memakai kuota gratis; ada sakelar on/off dan batas pemakaian harian.
2. **Voice Quick Order**: dikte memakai SpeechRecognition peramban (butuh internet), lalu dicocokkan ke katalog oleh pencocok lokal tanpa AI. Offline: tombol mikrofon redup.
3. **Smart Insight**: teks dibuat dari aturan lokal (offline). Pemolesan kalimat oleh AI bersifat opsional. Tanda kejanggalan memakai ambang pada `store_settings` (bawaan): >= 3 void dalam satu shift; selisih kas > Rp20.000 atau selisih pada 2 shift beruntun untuk penjaga yang sama; kenaikan HPP >= 10%.
4. **Foto Faktur**: AI membaca foto faktur menjadi daftar item; hasil selalu dikonfirmasi manusia.
5. Notifikasi memakai Web Push (gratis) ke HP Pengelola.
6. WhatsApp hanya lewat tautan `wa.me` (nota, pengingat kasbon, daftar belanja); tidak ada API WhatsApp.
