# Product Requirement Document (PRD) - K_War
Versi 2.4 - 26 November 2026

## 1. Ringkasan Eksekutif
- **Nama Produk**: K_War
- **Tipe**: Aplikasi kasir (POS) internal warung kelontong/sembako, berbentuk PWA.
- **Tujuan**: Kasir cepat, transaksi offline-first, laci kas modular, dan buku kasbon fleksibel yang mudah dipakai pengguna lanjut usia (pemilik dan penjaga warung).
- **Prinsip biaya**: seluruh komponen wajib gratis (free tier). Fitur yang butuh biaya tidak dibangun.

## 2. Profil Pengguna & Masalah Utama
1. **Pemilik Warung (Owner)**: lansia; butuh antarmuka ringkas, angka kontras tinggi, tanpa grafik rumit; butuh laba bersih, kontrol HPP, dan otoritas atas perangkat serta anggota.
2. **Kerabat Tepercaya**: keluarga inti; wewenang operasional dan finansial setara Pemilik, tanpa wewenang administratif (anggota/perangkat). Memiliki HP sendiri yang terdaftar.
3. **Kasir Umum** (pegawai kasir biasa; di database `regular_staff`): butuh aplikasi cepat, mudah dipelajari, tahan mati listrik/sinyal, tanpa risiko merusak data finansial toko.
4. **Administrator Teknis**: peran di luar aplikasi (bukan anggota di `members`). Memegang akun Supabase, Cloudflare, GitHub; merilis pembaruan, memantau backup dan kuota, uji restore, mencabut perangkat darurat lewat dasbor Supabase. Tidak menyetujui transaksi kecuali terdaftar sebagai Kerabat Tepercaya.

## 3. Nilai Unggulan
- **Offline-first**: alur penjualan, kas, dan kasbon 100% berjalan tanpa internet (IndexedDB/Dexie.js) dengan penyimpanan lokal yang dikunci agar tidak dihapus peramban. Pengecualian ada di bagian 6.
- **Multi-perangkat simultan**: tiap penjaga memegang HP/tablet sendiri.
- **Ramah lansia**: elemen besar, kontras tinggi (WCAG AAA), navigasi satu arah, semua ikon disertai teks, **teks layar memakai bahasa Indonesia sehari-hari** (mis. "Mulai Jaga", "Selesai Jaga", "Batalkan Nota"), tata letak tetap yang tidak berubah sendiri, dan tidak ada elemen yang hilang sendiri dalam waktu singkat.
- **Tanpa PIN**: tindakan sensitif disahkan dengan **Persetujuan QR** (pindai QR atau ketuk notifikasi di HP Pemilik/Kerabat, lalu Setujui/Tolak).
- **Laci kas modular**: Laci Bersama, Laci Pribadi, dan Kas Simpanan.
- **Kasbon tradisional**: tanpa bunga, denda, atau jatuh tempo kaku.
- **AI gratis dan opsional**: bisa dimatikan; dibatasi agar tidak melewati kuota gratis.

## 4. Lingkup Fitur MVP (Disetujui)
Tanda rilis: **[R1]** Inti Kasir, **[R2]** Stok & Operasional, **[R3]** AI & Persetujuan Offline (rincian di Roadmap). **R1 dibagi dua tahap**: **R1a** (jual tunai offline, sinkronisasi, shift, backup) dan **R1b** (kasbon, Persetujuan QR, void/retur nota lama, notifikasi persetujuan). Tanda [R1] di bawah berarti R1a atau R1b; pembagian per fitur ada di Roadmap dan di tanda rilis Spesifikasi Fungsional.

**Inti kasir**
1. Pendaftaran perangkat 1 kali (Device Provisioning) + Remote Revoke. [R1]
2. Pilih penjaga ("Siapa yang jaga?") dengan kartu foto besar, lalu routing laci otomatis. [R1]
3. Empat mode input tanpa hafalan: grid favorit 6-8 barang (kartu besar berisi foto, nama, harga; geser untuk barang lain), scan barcode fisik, scan kamera HP, pencarian teks; Modal Takaran Curah otomatis. [R1]
4. **Barang Lain (harga bebas)** untuk barang yang belum terdaftar. [R1]
5. Multi-keranjang (5) + **Batalkan barang terakhir** + perlindungan tekan ganda. [R1]
6. Pembayaran tunai, kasbon penuh, split tunai+kasbon [R1]; QRIS statis dan **Pakai Sendiri** (disahkan QR) [R2].
7. Tukar kembalian 1-klik; kembalian bersuara offline. [R1]
8. Void/retur dengan alasan wajib + audit log; void nota lama disahkan QR. [R1]
9. Buka Shift (modal awal) dan Tutup Shift Blind Count dengan kalkulator pecahan. [R1]
10. Indikator status sinkronisasi selalu terlihat. [R1]
11. Pengaturan aksesibilitas: ukuran huruf 3 tingkat, bunyi/getar, ucapkan nama barang. [R1]

**Kasbon & persetujuan**
12. Buku kasbon, limit per pelanggan, cicilan diterima kasir; pelanggan baru dapat ditambah kasir (limit awal Rp0, sehingga kasbon pertamanya butuh Persetujuan QR). [R1b]
13. **Persetujuan QR online** (dua jalur: pindai QR atau notifikasi) untuk kasbon over-limit, barang rusak, pakai sendiri, dan void nota lama. [R1b; barang rusak dan pakai sendiri aktif di R2]. Notifikasi khusus permintaan persetujuan ikut R1b.
14. Pengingat kasbon sopan via tautan WhatsApp (`wa.me`). [R2]
15. **Persetujuan QR offline dua arah.** [R3]

**Inventori**
16. Penerimaan fisik kulakan oleh kasir; konfirmasi harga/HPP oleh Pemilik/Kerabat di HP-nya. [R2] Foto Faktur (AI). [R3]
17. HPP Moving Average, penanda merah kenaikan harga modal, status stok Merah/Kuning/Biru. [R2]
18. Penyesuaian stok rusak/kadaluwarsa disahkan QR; pelacakan kadaluwarsa terdekat; stok opname. [R2]
19. **Gas elpiji & isi ulang** (tukar tabung/galon kosong) dan **barang konsinyasi (titipan)**. [R2]
20. Daftar Belanja Pasar berdasarkan velocity, dapat dikirim via WhatsApp ke sales. [R2]
21. Pendaftaran dan penataan produk: impor CSV, tambah/ubah manual, scan barcode + foto produk, memilih barang favorit. **[R1a]** (kasir tidak bisa berjualan tanpa katalog). Isi nama dari foto kemasan (AI). [R3]

**Pengelola (HP Pemilik/Kerabat)**
22. Dasbor ringkas dengan tombol **Bacakan Laporan** (suara). [R2]
23. Notifikasi push (permintaan persetujuan, shift ditutup, stok habis, kadaluwarsa, selisih kas, nota bermasalah) dan ringkasan malam. [R2; khusus notifikasi permintaan persetujuan sudah di R1b]

**Infrastruktur**
24. Cetak struk thermal 58mm opsional dan kirim nota via WhatsApp (`wa.me`). [R2]
25. Format nota baku `kw/[urut]/[kode perangkat]/ddmmyyyy`. [R1]
26. Retensi lokal transaksi tersinkron 3 hari + auto-purge; penyimpanan persisten; **Mode Latihan**. [R1]
27. Fitur AI gratis dengan sakelar on/off dan batas harian: Smart Insight aturan lokal [R2] dan poles AI [R3]; Voice Quick Order [R3]; Foto Faktur dan foto kemasan [R3].
28. Backup harian terenkripsi, keep-alive Supabase, dan peringatan otomatis bila backup berhenti berjalan (heartbeat). [R1a]

**Kas (tambahan v2.3)**
29. **Setoran & Transfer Kas** antar laci dan Kas Simpanan, dengan konfirmasi Pengelola; Kas Simpanan dapat dihitung ulang (opname kas). [R2]

**Ramah lansia (tambahan v2.4)**
30. **Tombol BANTUAN**: hubungi Pemilik lewat WhatsApp satu ketukan dan panduan bergambar 3 langkah (jual, bayar, selesai jaga). [R1a]
31. **Konfirmasi angka tidak wajar** (kembalian besar, jumlah barang ganjil, harga Barang Lain besar, total nota besar) dan **indikator baterai** dengan peringatan colok pengisi daya. [R1a]
32. **Beranda Sederhana** di HP Pemilik (3 tombol besar) dan **Pemulihan Akses** bila lupa login atau ganti HP. [R1a]
33. **Setup Awal oleh Administrator Teknis** (skrip seed dan langkah tertulis). [R1a]

## 5. Di Luar Lingkup (Keputusan Sadar)
- **API WhatsApp resmi (Meta)**: berbayar. Hanya tautan `wa.me`.
- **PPOB** (pulsa, token listrik, dll.): ditunda sampai pengguna sudah lancar memakai aplikasi.
- **QRIS dinamis/terintegrasi**: belum ada QRIS merchant; sementara QRIS statis. Integrasi perangkat/agregator dipertimbangkan nanti.
- **Katalog web untuk pelanggan**: tidak dibuat.
- **Fitur AI berbayar**: tidak dibuat.
- **PIN** (Master maupun penjaga): tidak dipakai.

## 6. Pengecualian Offline
Alur penjualan, shift, kasbon, dan penerimaan kulakan berjalan offline. Yang **membutuhkan internet** (layar menampilkan pesan jelas):
| Fitur | Keterangan |
| :--- | :--- |
| Persetujuan QR online | Kasbon over-limit saat offline ditolak sampai Rilis 3 |
| Void/retur nota yang bukan dari shift aktif sendiri | Data diambil dari server; butuh Persetujuan QR |
| Voice Quick Order (dikte) | SpeechRecognition peramban butuh internet |
| Foto Faktur, foto kemasan, poles Smart Insight | Memanggil Gemini lewat server |
| Notifikasi push, sinkronisasi, backup | Butuh internet |

## 7. Asumsi, Batasan & Risiko
**Asumsi**
- Perangkat target: Chrome Android versi terkini, PWA terpasang.
- HP Pengelola boleh Android atau iOS 16.4+ (push butuh PWA terpasang di iOS; Web Bluetooth/WebUSB tidak tersedia di iOS).
- Semua layanan memakai paket gratis: Supabase, Cloudflare Pages/R2, GitHub Actions, Gemini free tier, Web Push.
- Tanggal dan jam pada nota/struk mengikuti jam perangkat; jadwal backup otomatis memakai WIB (UTC+7). Zona waktu toko diatur di `store_settings.timezone`; contoh di dokumen memakai WIB, dan jadwal backup/ringkasan malam harus disesuaikan bila warung berada di WITA/WIT.

**Batasan**
- Aplikasi inti tidak boleh bergantung pada AI.
- Kuota gratis dapat berubah; batas harian diatur di `ai_usage`.
- Data foto faktur/kemasan diproses layanan gratis Google (diterima Pemilik).

**Risiko**
- Printer thermal murah memakai Bluetooth klasik sehingga tidak terdeteksi Web Bluetooth; uji printer sebelum membeli.
- Penjaga dapat memilih nama orang lain di layar "Siapa yang jaga?" (tanpa PIN, diterima); ditutup oleh Audit Log dan laporan shift.
- Jam perangkat meleset; server mencatat dan melaporkan.
- Lansia sulit memindai QR; tersedia jalur notifikasi.
- HP Pengelola hilang; dicabut oleh Pemilik dari HP lain atau oleh Administrator Teknis lewat dasbor Supabase. Pencabutan juga menghapus sesi login HP itu (Tech Stack bagian 3).
- Tablet hilang/dicuri: data lokal tidak dienkripsi di perangkat. Yang tersimpan hanya katalog tanpa HPP, nama pelanggan (tanpa telepon/alamat), dan nota maksimal 3 hari. Mitigasi: Remote Revoke.

## 8. Alur Pengguna Harian
1. **Pagi**: penjaga membuka tablet, menyentuh kartu namanya, memasukkan modal awal, lalu mulai berjualan. Jika sales datang: penjaga mencatat jumlah barang dan uang yang dibayar dari laci; Pengelola mengonfirmasi harga di HP-nya.
2. **Siang**: pembeli dilayani lewat grid, scan, atau ketik; barang curah memakai takaran cepat; barang tak terdaftar lewat Barang Lain. Penjaga menekan BAYAR lalu memilih tunai, QRIS, atau kasbon. Kasbon melewati limit: tablet meminta persetujuan; Pemilik memindai QR atau mengetuk notifikasi dan menekan Setujui; transaksi lanjut. Salah tekan: Batalkan Barang Terakhir atau Void.
3. **Sore/malam**: penjaga menutup shift dengan menghitung pecahan uang, tanpa melihat selisih. HP Pengelola menerima laporan shift; Pemilik menekan Bacakan Laporan, melihat daftar belanja, dan mengirimnya ke sales lewat WhatsApp.
4. **Sepanjang hari**: sinkronisasi berjalan di latar; bilah atas menunjukkan status; nota bermasalah dilaporkan ke Pengelola.
5. **Berkala**: Administrator Teknis memeriksa backup, kuota, dan pembaruan.

## 9. Ukuran Keberhasilan
Target awal, disesuaikan setelah UAT 1 dengan persetujuan Pemilik.
- Penjaga lansia menyelesaikan transaksi tunai tanpa bantuan setelah 1 hari latihan.
- Waktu satu transaksi tunai 3 barang ≤ 60 detik pada UAT, diukur dengan stopwatch dari barang pertama disentuh/dipindai sampai tombol "SELESAI & SIMPAN" ditekan (termasuk memilih uang diterima dan melihat kembalian; tidak termasuk waktu mengobrol dengan pembeli), rata-rata dari transaksi uji.
- Bantuan kepada penjaga ≤ 3 kali per hari di minggu pertama dan ≤ 1 kali per hari di minggu keempat.
- Nol transaksi hilang selama 30 hari pemakaian nyata.
- Nota bermasalah (ditolak atau bertanda) ≤ 1% per minggu, semuanya ditinjau dalam 24 jam.
- Nomor nota terlewat (celah urutan per perangkat per hari) yang tidak dapat dijelaskan = 0.
- Backup harian berhasil ≥ 29 dari 30 hari, tidak pernah dua hari gagal beruntun, dan uji restore berhasil.
- Selisih kas hanya terlihat di HP Pengelola, tidak di layar kasir.

## 10. Parameter yang Belum Ditetapkan
- Jumlah perangkat (Tablet Kasir + HP Pengelola): perkiraan 5-10, belum pasti.
- Perkiraan transaksi per hari: belum diketahui; dicatat selama UAT 1.
- Data awal (pemasok, penitip, pelanggan kasbon, dll.): diisi setelah aplikasi dapat dipakai, bukan sekarang.
- Zona waktu warung (WIB/WITA/WIT): menentukan `store_settings.timezone` dan jadwal backup.
- Penyedia URL publik foto produk (lihat Tech Stack bagian 2).

## 11. Aturan Dokumen
1. Kesepuluh dokumen `.md` K_War adalah **sumber kebenaran mutlak**. Kode yang berbeda dari dokumen dianggap salah.
2. Ide baru, perubahan, atau penghapusan fitur harus lebih dulu disetujui Pemilik proyek, lalu dimasukkan ke dokumen yang terkait sebelum dikerjakan.
3. **Satu aturan, satu dokumen pemilik.** Dokumen lain merujuk, tidak menyalin angka atau rumusnya. Bila terpaksa ada ringkasan, perubahan dilakukan di dokumen pemilik lebih dulu, lalu semua ringkasan diperbarui dalam perubahan yang sama.

| Aturan | Dokumen pemilik |
| :--- | :--- |
| Peran, laci kas, shift, transaksi, kulakan/HPP, kasbon, persetujuan, retensi, sinkronisasi | Aturan Bisnis dan Glosarium |
| Perilaku layar dan kriteria penerimaan | Spesifikasi Fungsional |
| Tabel, kolom, RLS, trigger, penyimpanan lokal | Model Data |
| Endpoint, validasi, kode kesalahan | Spesifikasi API |
| Teknologi, identitas perangkat, backup, kapasitas | Tech Stack dan Arsitektur |
| Angka performa, aksesibilitas, keamanan umum | Dokumen Non-Fungsional |
| Palet dan wireframe | UI/UX dan Wireframe |
| Urutan rilis | Roadmap |
| Kasus uji dan rilisnya | Rencana Pengujian |

## 12. Riwayat Perubahan
**v2.4 (26 November 2026)** - penyesuaian untuk pengguna lansia dan kesiapan R1a:
- **Disetujui Pemilik**: (a) tidak ada kamus label; teks layar langsung memakai bahasa sehari-hari; (b) grid utama = 6-8 barang favorit bertombol besar (foto, nama, harga) dengan geser untuk barang lain, menggantikan grid adaptif "terlaris"; (c) rincian pecahan kembalian tidak dibuat; (d) rekomendasi lain disesuaikan oleh penyusun; (e) R1a disesuaikan.
- **R1a bertambah**: pendaftaran/penataan produk (impor CSV, tambah manual, foto, favorit), Beranda Sederhana dan Pemulihan Akses HP Pemilik, Setup Awal, Bantuan, konfirmasi angka tidak wajar, indikator baterai. R1a menjadi 5 minggu; rilis berikutnya bergeser.
- **Tombol "Batalkan Barang Terakhir"** tidak hilang sendiri; label layar diganti bahasa sehari-hari (Mulai Jaga, Selesai Jaga, Batalkan Nota, Perlu Dicek, Nota Bermasalah, Hitung Stok, Nota Kulakan, Beranda).
- Label "SETUJUI & TAMBAH STOK" diganti "SETUJUI HARGA" karena stok sudah bertambah saat barang diterima (v2.3).
- Kasus uji: TC-45 ditulis ulang (grid favorit), TC-41 pindah ke R1a, TC-73 sampai TC-76 baru.

**v2.3 (26 November 2026)** - hasil analisis mendalam v2.2:
- **Disetujui Pemilik**: (a) nota offline yang sah secara fisik diterima dan ditandai, bukan ditolak; (b) stok kulakan bertambah sementara saat kasir menerima, HPP baru dihitung saat konfirmasi; (c) Rilis 1 dipecah R1a/R1b dan notifikasi persetujuan masuk R1b; (d) retensi lokal 3 hari.
- **Keamanan**: RLS default-deny dan matriks izin (Model Data); Pengelola menulis hanya lewat RPC; pencabutan HP ikut menghapus sesi login; masa berlaku JWT, token pairing, dan undangan Kerabat ditetapkan; `audit_logs` dapat mencatat aksi sistem/admin.
- **Sinkronisasi**: jalur void/retur dan retur sebagian; stok/hutang lokal diturunkan dari server dikurangi outbox; `pull_ledger` memuat laci; batas 50 item per bundle; penanganan jam meleset.
- **Persetujuan QR**: satu persetujuan = satu penggunaan; isi payload dan kanonisasi hash ditetapkan.
- **Kas**: satu sesi terbuka per laci, pencocokan modal awal, sesi yatim, selisih baru final setelah data lengkap, setoran/transfer kas, rekap QRIS.
- **Aturan baru**: pembulatan, harga paket, barcode kardus, Tukar Receh, definisi laporan, HPP aman terhadap stok negatif, opname terhadap nota terlambat, aturan reset penanda kenaikan modal dan kadaluwarsa, label stok HABIS/KRITIS/MENIPIS/AMAN.
- **Operasional**: heartbeat backup, repo privat, kunci enkripsi backup, perkiraan kapasitas.
- **Perbaikan dokumen**: kontras latar, total wireframe Tutup Shift, kolom rilis pada Rencana Pengujian dan 19 kasus uji baru, peta kepemilikan aturan.

## 13. Keputusan yang Masih Menunggu Pemilik
Usulan di bawah sudah dimasukkan ke dokumen sebagai nilai bawaan; ubah bila tidak setuju.
1. Pembulatan total nota ke Rp100 terdekat (kecuali QRIS); dapat diatur 0/100/500 di pengaturan toko.
2. Kasir boleh menambah pelanggan baru dengan limit Rp0.
3. Fitur Setoran & Transfer Kas (F-40), termasuk opname Kas Simpanan.
4. Tablet tidak menerima telepon/alamat pelanggan (pengingat kasbon dikirim Pengelola).
5. Sesi shift dianggap tertinggal setelah 16 jam (`shift_max_hours`).
6. Ambang kejanggalan Smart Insight: 3 void per shift, selisih kas > Rp20.000, kenaikan HPP >= 10%.
7. Foto produk memakai URL R2 bawaan (`r2.dev`) selama MVP, tanpa membeli domain.
8. Ambang konfirmasi angka tidak wajar: kembalian >= Rp100.000, jumlah > 20 per barang (selain curah), harga Barang Lain > Rp500.000, total nota > Rp1.000.000 (dapat diatur di pengaturan toko).
9. Beranda Sederhana aktif bawaan untuk Pemilik; dapat dimatikan per anggota.
10. Nomor WhatsApp bantuan (`help_phone`) diisi Pemilik saat setup awal.
