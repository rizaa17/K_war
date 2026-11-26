# UI/UX dan Wireframe - K_War
Versi 2.4 - 26 November 2026
Dokumen ini adalah sumber kebenaran. Aturan perubahan: lihat PRD bagian 11.

## 1. Standar Desain Ramah Usia Lanjut
1. **Palet kontras tinggi (WCAG AAA ≥ 7:1 di atas putih)**:
   - Latar: Putih `#FFFFFF` / Abu Terang `#F8FAFC`.
   - Teks & angka: Hitam Pekat `#0F172A`.
   - Aksi positif / Bayar: Hijau `#166534`.
   - Aksi hati-hati / Batalkan / Kritis: Merah `#991B1B`.
   - Status Aman: Biru `#1E40AF`.
   - Status Menipis / Peringatan: Kuning Tua `#92400E`.
   - **Aturan pemakaian**: warna-warna di atas dipakai sebagai teks hanya di atas Putih `#FFFFFF` (atau sebagai latar tombol dengan teks putih). Abu Terang `#F8FAFC` hanya untuk latar bidang yang berisi teks Hitam Pekat (rasio hijau/kuning tua di atas abu hanya 6,81/6,78, di bawah 7:1).
2. **Tipografi**: label dan tombol ≥ 18 px tebal; harga ≥ 22 px; total belanja dan kembalian 32-40 px. Pengaturan ukuran huruf 3 tingkat.
3. **Tidak ada ikon tanpa teks**; status tidak hanya dengan warna (selalu teks "HABIS/MENIPIS/AMAN").
4. **Satu tombol utama per layar**; layar utama kasir hanya punya satu tombol hijau **BAYAR**.
5. Target sentuh ≥ 56 x 56 dp; bunyi dan getar singkat sebagai umpan balik.
6. Pita status sinkronisasi dan mode selalu terlihat di bagian atas, bersama sisa baterai dan tombol BANTUAN.
7. Dialog berurutan linear maksimal 3 modal (Bayar -> Cara Bayar -> Tunai); tiap modal punya tombol KEMBALI besar; tidak ada modal di dalam modal.
8. Status stok memakai empat label: **HABIS** (<= 0), **KRITIS**, **MENIPIS**, **AMAN**, selalu dengan teks dan ikon.
9. **Bahasa sehari-hari langsung di layar**, tanpa kamus: "Mulai Jaga", "Selesai Jaga", "Batalkan Nota", "Perlu Dicek", "Nota Bermasalah", "Hitung Stok", "Nota Kulakan", "Beranda", "HP & Tablet". Istilah teknis tidak muncul di layar.
10. **Tata letak tetap dan tanpa batas waktu**: kartu tidak berpindah sendiri; tombol dan pesan penting tidak hilang sendiri. Setiap gerakan geser punya panah besar sebagai cadangan.

---

## 2. Layar "Siapa yang Jaga?"
```
+-------------------------------------------------------------+
| K_War        SIAPA YANG JAGA SEKARANG?     [TERSAMBUNG - OK] |
+-------------------------------------------------------------+
|  +-----------+   +-----------+   +-----------+              |
|  |  [FOTO]   |   |  [FOTO]   |   |  [FOTO]   |              |
|  |  BUDE     |   |  BPK.     |   |  ADI      |              |
|  |  TITIN    |   |  SLAMET   |   |           |              |
|  +-----------+   +-----------+   +-----------+              |
|                                                             |
|  [ MODE LATIHAN ]                                           |
+-------------------------------------------------------------+
```

## 3. Layar Mulai Jaga & Modal Awal
```
+-------------------------------------------------------------+
| MULAI JAGA        Penjaga: Bpk. Slamet     Laci: Pribadi     |
+-------------------------------------------------------------+
|              MASUKKAN MODAL AWAL LACI KAS                   |
|              +-----------------------------+                |
|              |  Rp [ 150.000             ] |                |
|              +-----------------------------+                |
|  [ Rp 50.000 ] [ Rp 100.000 ] [ Rp 150.000 ]                |
|  [ Rp 200.000] [ Rp 300.000 ] [ Rp 500.000 ]                |
|  Keypad jumbo:  [1] [2] [3]  [4] [5] [6]  [7] [8] [9]       |
|                 [C] [0] [000]                               |
|  [          BUKA LACI & MULAI JUALAN (HIJAU)           ]    |
+-------------------------------------------------------------+
```

## 4. Layar Kasir Utama (Tablet Lanskap)
```
+----------------------------------------------------------------------------------------------------+
| Bude Titin | Laci Bersama | [KERANJANG 1*][2][3][4][5] | TERSAMBUNG | BATERAI 80% | [? BANTUAN]    |
+------------------------------------------------------+---------------------------------------------+
| [ KETIK NAMA ]  [ SCAN KAMERA ]  [ BARANG LAIN ]     | KERANJANG AKTIF (#1)                        |
|                                                      | 1. Beras Rojolele 5kg                       |
| FAVORIT  (halaman 1 dari 4)                          |    1 x Rp68.000          [ - ] [ + ]        |
| +---------+ +---------+ +---------+ +---------+      | 2. Minyak Goreng 1L                         |
| | [FOTO]  | | [FOTO]  | | [FOTO]  | | [FOTO]  |      |    2 x Rp16.000          [ - ] [ + ]        |
| |BERAS    | |MINYAK   | |GULA     | |TELUR    |      | 3. Telur (Curah 1/2 kg)                     |
| |Rp68.000 | |Rp16.000 | |Rp17.500 | |Rp30.000 |      |    1 x Rp15.000          [ - ] [ + ]        |
| +---------+ +---------+ +---------+ +---------+      |                                             |
| +---------+ +---------+ +---------+ +---------+      | [ BATALKAN BARANG TERAKHIR ]                |
| | [FOTO]  | | [FOTO]  | | [FOTO]  | | [FOTO]  |      | TOTAL BELANJA: Rp115.000                    |
| |KOPI     | |TEH      | |SUSU     | |SABUN    |      |                                             |
| |Rp3.000  | |Rp5.000  | |Rp12.000 | |Rp3.500  |      | [        BAYAR (HIJAU JUMBO)        ]       |
| +---------+ +---------+ +---------+ +---------+      | [ BATALKAN NOTA ]  (merah)                  |
|                                                      |                                             |
| [ < ]   GESER UNTUK BARANG LAIN   [ > ]   (o . . .)  |                                             |
+------------------------------------------------------+---------------------------------------------+
```
Aturan layar:
- **Grid favorit**: halaman 1 berisi 6 sampai 8 kartu besar (foto, nama, harga) pilihan Pemilik dengan posisi **tetap**. Geser ke samping (atau tekan panah besar) untuk halaman berikutnya: satu halaman per kategori (judul kategori besar, urut abjad). Titik di bawah menunjukkan halaman.
- Kartu menampilkan status stok (MENIPIS/KRITIS/HABIS) hanya bila stok tidak aman, agar kartu tetap bersih.
- Takaran curah (1/4, 1/2, 1, 2 kg + input) muncul sebagai jendela setelah barang curah disentuh, bukan terus tampil.
- Bilah atas: sambungan, **baterai** (pita kuning "COLOK PENGISI DAYA" saat <= 20%), dan tombol **BANTUAN**.
- Tombol "BATALKAN BARANG TERAKHIR" tetap tampil sampai barang berikutnya ditambahkan atau BAYAR ditekan.
- Kasir tidak melihat harga modal atau titik merah kenaikan HPP (hanya Pengelola).

## 5. Modal Pilih Cara Bayar
```
+---------------------------------------------------+
| TOTAL: Rp115.000          PILIH CARA BAYAR         |
+---------------------------------------------------+
| [ TUNAI ]        (hijau)                          |
| [ QRIS ]         (tampil hanya jika QRIS diunggah)|
| [ KASBON / SEBAGIAN TUNAI ]                       |
| [ PAKAI SENDIRI ] (butuh persetujuan Pemilik)     |
| [ KEMBALI ]                                       |
+---------------------------------------------------+
```

## 6. Modal Pembayaran Tunai & Kembalian
```
+---------------------------------------------------+
| PEMBAYARAN TUNAI - TOTAL: Rp115.000               |
| UANG DITERIMA: [ Rp 120.000 ]                     |
| [Uang Pas] [Rp120.000] [Rp150.000] [Rp200.000]    |
+---------------------------------------------------+
| KEMBALIAN:  Rp5.000                               |
| (Suara: "Kembalian lima ribu rupiah")             |
| [ TUKAR RECEH DENGAN PERMEN/BARANG LAIN ]         |
+---------------------------------------------------+
| [ CETAK STRUK 58mm ] [ KIRIM LEWAT WHATSAPP ]     |
| [        SELESAI & SIMPAN TRANSAKSI            ]  |
+---------------------------------------------------+
```

## 7. Modal Barang Lain (Harga Bebas)
```
+---------------------------------------------------+
| BARANG LAIN (BELUM TERDAFTAR)                      |
| Nama (boleh kosong): [ Bumbu pecel             ]  |
| Harga:               [ Rp 5.000                ]  |
| Keypad jumbo: [1][2][3] [4][5][6] [7][8][9] [C][0][000] |
| [ MASUKKAN KE KERANJANG ]                         |
+---------------------------------------------------+
```

## 8. Modal Selesai Jaga (Hitung Uang)
```
+-------------------------------------------------------------+
| SELESAI JAGA - HITUNG UANG DI LACI                          |
| Pecahan Rp100.000 : [ 12 ] lembar = Rp 1.200.000            |
| Pecahan Rp 50.000 : [  8 ] lembar = Rp   400.000            |
| Pecahan Rp 20.000 : [ 15 ] lembar = Rp   300.000            |
| Pecahan Rp 10.000 : [ 20 ] lembar = Rp   200.000            |
| Pecahan Rp  5.000 : [ 30 ] lembar = Rp   150.000            |
| Pecahan Rp  2.000 : [ 25 ] lembar = Rp    50.000            |
| Pecahan Rp  1.000 : [  0 ] lembar = Rp         0            |
| Koin Rp1.000 [ 5 ]  Rp500 [ 4 ]  Rp200 [ 0 ]  Rp100 [ 0 ]   |
+-------------------------------------------------------------+
| TOTAL UANG FISIK : Rp 2.307.000   (saldo sistem dirahasiakan)|
| [   SELESAI JAGA & KUNCI (KIRIM KE PEMILIK)       ]         |
+-------------------------------------------------------------+
```

## 9. Persetujuan QR
```
TABLET KASIR:                          HP PENGELOLA:
+------------------------------+       +------------------------------+
| BUTUH PERSETUJUAN PEMILIK    |       | PERSETUJUAN KASIR            |
| Aksi: Kasbon Lewat Limit     |       | Dari: Kasir Budi (TAB1)      |
| Pelanggan: Bpk. Joko         |       | Aksi: Kasbon Lewat Limit     |
| Limit Rp100.000              |       | Total Belanja: Rp145.000     |
| Total hutang menjadi Rp145.000| ---> | Limit: Rp100.000             |
|   +------------------+       | scan  | Kelebihan: Rp45.000          |
|   |   [ KODE QR ]    |       | atau  |                              |
|   +------------------+       | tekan | [  SETUJUI  (HIJAU JUMBO) ]  |
| Minta Pemilik memindai QR    | notif | [  TOLAK    (MERAH)       ]  |
| atau tekan Setujui di HP-nya.|       +------------------------------+
| Berlaku 5 menit. [BUAT ULANG]|
| (Tanpa internet: belum bisa) |
+------------------------------+
```

## 10. Beranda Lengkap HP Pengelola
```
+---------------------------------------------------+
| K_War - BERANDA           [BACAKAN LAPORAN]       |
+---------------------------------------------------+
| HARI INI: OMZET Rp3.420.000  LABA Rp480.000       |
| NOTA SELESAI: 84   Barang Lain: Rp35.000          |
+---------------------------------------------------+
| PERMINTAAN MENUNGGU: 1  -> [ LIHAT & SETUJUI ]    |
| NOTA KULAKAN MENUNGGU: Unilever Rp850.000         |
|   -> [ PERIKSA & SETUJUI ]  [ FOTO NOTA ]       |
| NOTA BERMASALAH: 0                                |
+---------------------------------------------------+
| LACI: Bersama Rp2.850.000 | Pribadi Adi Rp570.000 |
+---------------------------------------------------+
| STOK: [HABIS] 2  [KRITIS] 1  [MENIPIS] 5 barang   |
|   -> [ DAFTAR BELANJA ]  [ KIRIM KE WHATSAPP ]    |
| [MASA BERLAKU] 3 barang segera habis              |
+---------------------------------------------------+
| [ BUKU KASBON ] [ ANGGOTA ] [ HP & TABLET ]       |
| [ TITIPAN ] [ PRODUK ] [ HITUNG STOK ]            |
+---------------------------------------------------+
```

## 11. Konfirmasi Nota Kulakan (HP Pengelola)
```
+---------------------------------------------------+
| NOTA KULAKAN: Sales Unilever  [ FOTO NOTA ]       |
| Barang       Hitungan Kasir  Di Nota    Harga     |
| Sabun 80g        24            24     [ 2.800 ]   |
| Sampo sachet     50            48 (!) [   950 ]   |
| (!) = jumlah beda dengan hitungan kasir           |
| TOTAL Rp850.000                                   |
| [  SETUJUI HARGA  ]  [ TOLAK ]                    |
+---------------------------------------------------+
```

## 12. Mode Latihan
```
+-------------------------------------------------------------+
|  LATIHAN - TIDAK TERSIMPAN, TIDAK MENGUBAH STOK/KAS  (kuning) |
|  ... tampilan kasir seperti biasa ...        [ KELUAR LATIHAN ]|
+-------------------------------------------------------------+
```

## 13. Perlu Dicek & Nota Bermasalah (HP Pengelola) - F-41
```
+---------------------------------------------------+
| PERLU DICEK                                       |
+---------------------------------------------------+
| PERLU DICEK (2)                                   |
| kw/0042/DEV1/26112026   Rp145.000   Bpk. Joko     |
|  (!) KASBON MELEWATI LIMIT - tanpa persetujuan    |
|  Penjelasan: tablet belum tahu limit terbaru.     |
|  [ SUDAH DICEK ]   [ UBAH LIMIT ]                 |
+---------------------------------------------------+
| NOTA BERMASALAH (1)                               |
| kw/0007/TAB1/26112026   Rp30.000                  |
|  (!) BARANG TIDAK DIKENAL                         |
|  [ COBA LAGI ]   [ BUANG (WAJIB ALASAN) ]         |
+---------------------------------------------------+
```

## 14. Setoran Uang ke Kas Simpanan - F-40
```
TABLET KASIR:                          HP PENGELOLA:
+------------------------------+       +------------------------------+
| SETOR UANG KE SIMPANAN       |       | SETORAN MENUNGGU             |
| Jumlah: Rp [ 500.000 ]       | ----> | Dari: Bude Titin (TAB1)      |
| [ KIRIM SETORAN ]            |       | Rp500.000                    |
| Status: MENUNGGU DICEK       |       | [ UANG DITERIMA (HIJAU) ]    |
+------------------------------+       | [ TIDAK DITERIMA (MERAH) ]   |
                                       +------------------------------+
```

## 15. Giliran Jaga Kemarin Belum Selesai
```
+-------------------------------------------------------------+
| GILIRAN JAGA KEMARIN BELUM SELESAI                          |
| Hitung uang di laci dulu, lalu tekan tombol di bawah.       |
| (tampilan Selesai Jaga - Hitung Uang seperti biasa)         |
| [   SELESAI JAGA KEMARIN & KUNCI   ]                        |
+-------------------------------------------------------------+
```

## 16. Tambah Pelanggan Baru (Kasir) - F-19
```
+---------------------------------------------------+
| TAMBAH PELANGGAN BARU                              |
| Nama:    [ Bpk. Joko                          ]   |
| Telepon: [ (boleh kosong)                     ]   |
| Limit kasbon awal: Rp0 (Pemilik menetapkan nanti) |
| [ SIMPAN & PILIH PELANGGAN INI ]                  |
+---------------------------------------------------+
```

## 17. Konfirmasi Angka Tidak Wajar - F-42
```
+---------------------------------------------------+
| KEMBALIAN Rp450.000. BENAR?                       |
| Total belanja: Rp50.000                           |
| Uang diterima: Rp500.000                          |
| [  BENAR, LANJUT (HIJAU)  ]  [ UBAH LAGI (MERAH) ]|
+---------------------------------------------------+
```
Pertanyaan yang sama berlaku untuk jumlah barang ganjil ("100 bungkus. BENAR?"), harga Barang Lain besar, dan total nota besar.

## 18. Bantuan - F-43
```
+---------------------------------------------------+
| BANTUAN                                           |
| [ HUBUNGI PEMILIK LEWAT WHATSAPP ]                |
| [ CARA MENJUAL ]   [ CARA MEMBAYAR ]              |
| [ CARA SELESAI JAGA ]          [ KEMBALI ]        |
+---------------------------------------------------+
```

## 19. Beranda Sederhana HP Pemilik - F-44
```
+---------------------------------------------------+
| K_War                  Selamat pagi, Pak Slamet   |
+---------------------------------------------------+
| [  SETUJUI PERMINTAAN (1)   ]   (hijau jumbo)     |
| [  BACAKAN LAPORAN HARI INI ]                     |
| [  DAFTAR BELANJA           ]                     |
| [  MENU LAINNYA             ]                     |
+---------------------------------------------------+
```
Tombol yang belum tersedia di rilis berjalan disembunyikan (R1a: LAPORAN HARI INI dan MENU LAINNYA; R1b menambah SETUJUI PERMINTAAN; R2 menambah suara dan DAFTAR BELANJA).

Catatan: tombol [FOTO NOTA] pada Beranda dan Konfirmasi Nota Kulakan tersembunyi sampai R3.
