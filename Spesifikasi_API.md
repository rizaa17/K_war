# Spesifikasi API dan Protokol Sinkronisasi - K_War
Versi 2.4 - 26 November 2026
Dokumen ini adalah sumber kebenaran. Aturan perubahan: lihat PRD bagian 11.

## 1. Arsitektur Komunikasi & Autentikasi
- **Tablet Kasir**: memakai **JWT perangkat** yang diterbitkan Edge Function `provision_device` (klaim `device_id`, `device_role`). RLS dan setiap RPC memeriksa klaim ini dan `devices.is_active`.
  - **Masa berlaku 30 hari**, diperpanjang otomatis oleh tablet pada setiap sinkronisasi sukses. Bila kedaluwarsa (offline lama), tablet memanggil `refresh_device_token(install_id, device_secret)`; `device_secret` dibuat saat pairing, disimpan di tablet, dan hanya hash-nya (`devices.secret_hash`) di server. Perangkat yang dicabut ditolak di sini.
  - **Klaim JWT selalu menang atas isi permintaan.** `device_id` di body harus sama dengan klaim; beda berarti `403`.
- **HP Pengelola** (Pemilik/Kerabat): login Supabase Auth sekali, sesi panjang. Identitas (`members.auth_user_id`) diambil server dari token; **tidak pernah dari isi permintaan**.
  - HP Pengelola juga terdaftar sebagai perangkat (`device_role = hp_manager`, `devices.auth_user_id` terikat). Setiap RPC Pengelola mengirim header `x-install-id` dan server menolak bila perangkat itu tidak aktif atau tidak terikat ke akun yang login. Akibatnya mencabut perangkat langsung memutus hak menyetujui.
  - Pencabutan HP juga menghapus sesi Auth-nya (trigger `devices`, Model Data bagian 1). Masa berlaku access token Auth diatur singkat (<= 15 menit) agar pencabutan berlaku cepat.
- **Pairing**: Pemilik membuat token pairing (acak 128-bit, sekali pakai, 10 menit; hanya hash disimpan) lewat RPC `create_pairing_token(role)`. Perangkat baru memanggil `provision_device(token, install_id)`; token yang kedaluwarsa atau terpakai ditolak (`PAIRING_INVALID`).
- **Undangan Kerabat**: Pemilik memanggil `invite_member(...)` yang mengirim tautan undangan Supabase Auth sekali pakai; akun Auth yang jadi dihubungkan ke `members.auth_user_id` oleh server.
- **Pemulihan akses**: login HP Pengelola memakai tautan masuk (tanpa kata sandi yang bisa lupa). Bila tautan hilang atau HP diganti: Kerabat atau Administrator Teknis mengirim tautan baru (`send_login_link`); untuk HP baru Administrator Teknis mencabut HP lama dan membuat token pairing lewat skrip admin (service role, tidak diekspos sebagai RPC). Langkahnya ada di Tech Stack bagian 9.
- Komponen: RPC `push_bundle`, RPC `pull_ledger`, RPC Persetujuan QR, RPC Pengelola, PostgREST (hanya `SELECT` untuk Pengelola; lihat Model Data bagian 1b), dan Edge Function `provision_device`, `r2_presign`, `ai_proxy`, `send_push`, `daily_digest`, `report_job`.

---

## 2. RPC `push_bundle`
- **Fungsi**: memasukkan antrean offline. Diproses **per item** dalam sub-transaksi sendiri; satu item buruk tidak menggagalkan item lain.
- `POST /rest/v1/rpc/push_bundle` - Header: `apikey`, `Authorization: Bearer <JWT perangkat>`.
- **Maksimal 50 item per bundle** (jumlah semua array). Klien memecah antrean lebih besar menjadi beberapa bundle berurutan; bundle > 50 ditolak `BUNDLE_TOO_LARGE`.
- Klien **tidak** mengirim HPP; server mengisi `cost_price_unit` dari `products.cost_price_hpp` saat memproses (shortcut: HPP saat diproses, bukan saat transaksi; selisih kecil bila faktur dikonfirmasi di antaranya).
- **Urutan pemrosesan dalam bundle**: `new_customers` -> `drawer_sessions` -> `transactions` -> `debt_payments` -> `purchase_receipts` -> `stock_adjustments` -> `voids` -> `returns` -> `cash_transfers` -> `audit_logs`. Rujukan ke item yang dikirim lebih awal (bundle sebelumnya) maupun di bundle yang sama diizinkan.

**Request**
```json
{
  "device_id": "7b8e1f54-8c43-4e36-9b5a-4e209193e2b1",
  "client_bundle_id": "90e0c83a-4467-4a0b-8d02-e2c7a33bc912",
  "new_customers": [],
  "drawer_sessions": [],
  "transactions": [
    {
      "id": "1d2a45e0-8a12-4f33-b40a-112233445566",
      "member_id": "c3e98124-7711-4560-a890-bcdef0123456",
      "drawer_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "session_id": "3a9c1d20-0b7e-4c55-8d1f-9a8b7c6d5e4f",
      "customer_id": "550e8400-e29b-41d4-a716-446655440000",
      "invoice_no": "kw/0042/DEV1/26112026",
      "subtotal": 50000,
      "rounding_adj": 0,
      "total_final": 50000,
      "payment_type": "split_cash_debt",
      "cash_tendered": 20000,
      "cash_change": 0,
      "debt_amount": 30000,
      "approval_request_id": null,
      "status": "completed",
      "created_at": "2026-11-26T00:30:00+07:00",
      "items": [
        {
          "id": "6f1c2b7a-3d4e-4a55-9b60-0a1b2c3d4e5f",
          "product_id": "a1b2c3d4-0000-1111-2222-333344445555",
          "custom_name": null,
          "qty": 2,
          "selling_price_unit": 25000,
          "is_box_sale": false,
          "exchange_empty": false,
          "subtotal": 50000
        }
      ]
    }
  ],
  "debt_payments": [],
  "purchase_receipts": [],
  "stock_adjustments": [],
  "voids": [
    { "id": "…", "transaction_id": "…", "reason": "salah_tekan", "voided_by_member_id": "…",
      "voided_at": "2026-11-26T01:10:00+07:00", "approval_request_id": null }
  ],
  "returns": [
    { "id": "…", "transaction_id": "…", "reason": "barang_dikembalikan", "returned_by_member_id": "…",
      "session_id": "…", "approval_request_id": null, "refund_cash": 15000, "reversed_debt": 0,
      "created_at": "2026-11-26T01:20:00+07:00",
      "items": [ { "transaction_item_id": "…", "qty": 1, "refund_amount": 15000 } ] }
  ],
  "cash_transfers": [],
  "audit_logs": []
}
```
- Nomor nota memakai tanggal dari **jam perangkat** (contoh: 00:30 WIB tanggal 26 sama dengan 17:30 UTC tanggal 25).
- `drawer_sessions` memuat pembukaan sesi dan, saat ditutup, `closing_balance_actual`, `cash_denominations`, dan `declared_counts` (jumlah transaksi, cicilan, penerimaan, setoran sesi itu). Server menandai `is_complete` dan baru menghitung selisih setelah jumlah diterima sama.
- `new_customers`: `{id, name, phone?, created_by_member_id}`; server menetapkan `credit_limit = 0`.
- `voids` menunjuk transaksi yang sudah ada (di bundle ini atau sebelumnya, juga yang sudah tersinkron). Void yang sudah berstatus `voided` dianggap sukses. Tanpa `approval_request_id`, server memeriksa void itu dibuat oleh pemilik shift yang sedang berjalan; bila tidak, **diterima dengan tanda** `VOID_NOT_OWN_SHIFT`. Dengan `approval_request_id` bertipe `VOID_LATE`, hash harus cocok (bagian 6).
- `returns`: bagian kasbon dikoreksi lebih dulu (`reversed_debt <= debt_amount` nota dikurangi koreksi sebelumnya), sisanya tunai (`refund_cash`); jumlah keduanya harus sama dengan jumlah `refund_amount`. Nota QRIS: `refund_cash` mencatat Audit Log saja, tanpa `cash_movements`. Bila seluruh butir telah diretur, `transactions.status = 'returned'`. Jumlah retur kumulatif tidak boleh melebihi yang terjual (`RETURN_EXCEEDS_SOLD`).

**Pemeriksaan keras (item ditolak, kode di bagian 9)** - data rusak yang tidak dapat dipercaya:
| Tipe | Aturan |
| :--- | :--- |
| semua nota | `total_final = subtotal + rounding_adj`; jumlah `items.subtotal = subtotal`; kolom bayar sesuai `payment_type` di bawah; sesi dan laci dikenal server |
| `cash_full` | `cash_tendered - cash_change = total_final`, `debt_amount = 0` |
| `debt_full` | `debt_amount = total_final`, `customer_id` wajib |
| `split_cash_debt` | `(cash_tendered - cash_change) + debt_amount = total_final`, `customer_id` wajib |
| `qris_static` | tunai dan kasbon = 0, `rounding_adj = 0`, `qris_confirmed_at` wajib |
| `personal_use` | tunai dan kasbon = 0 |
| butir | `product_id` harus pernah ada (`UNKNOWN_PRODUCT` bila tidak), `qty > 0` |

Pelanggaran aturan di atas ditolak dengan `TOTAL_MISMATCH` (atau kode butir yang sesuai) dan masuk `sync_rejections`.

**Pemeriksaan lunak (item DITERIMA, diberi tanda di `review_flags`)** - barang dan uang sudah berpindah di dunia nyata, jadi kas, stok, dan kasbon tetap dicatat:
| Tanda | Kondisi |
| :--- | :--- |
| `OVER_LIMIT_NO_APPROVAL` | kasbon melewati limit server dan tidak ada persetujuan `DEBT_OVERLIMIT` yang sah (mis. data limit tablet usang atau pelanggan baru) |
| `APPROVAL_MISSING` | `personal_use` atau butir yang mewajibkan persetujuan tanpa `approval_request_id` berstatus `approved` dengan `payload_hash` cocok, atau persetujuan sudah terpakai |
| `PRODUCT_DELETED` | produk sudah soft-delete saat diproses |
| `INVOICE_NO_CONFLICT` | nomor nota sudah ada untuk perangkat dengan `id` berbeda; nomor diberi akhiran `~xxxx` |
| `PRICE_DEVIATION` | `subtotal` butir tidak sama dengan harga katalog server saat diproses (memakai rumus paket/kardus/curah); tidak ditolak karena katalog tablet bisa usang |
| `MEMBER_DRAWER_MISMATCH` | anggota tidak berhak atas laci itu (`drawer_access`) atau tidak aktif |
| `OPENING_MISMATCH` / `SESSION_OVERLAP` | pada `drawer_sessions` (Aturan Bisnis 3.7 dan 3.8) |
| `CLOCK_SKEW` | `created_at` berselisih > 24 jam dari waktu server saat diterima |
| `VOID_NOT_OWN_SHIFT` | void tanpa persetujuan dari bukan pemilik shift berjalan |

Nota bertanda menaikkan `transactions.review_status` menjadi `open` dan muncul di F-41; tidak tampil di kasir. Stok negatif tidak ditandai per nota, tetapi di ringkasan harian.

**Respons (200 OK)** - per item:
```json
{
  "status": "partial",
  "server_timestamp": "2026-11-25T17:30:02.124Z",
  "accepted_ids": ["1d2a45e0-8a12-4f33-b40a-112233445566"],
  "flagged": [
    { "type": "transaction", "id": "1d2a45e0-8a12-4f33-b40a-112233445566", "flags": ["PRICE_DEVIATION"] }
  ],
  "rejected": [
    { "type": "transaction", "id": "…", "code": "UNKNOWN_PRODUCT", "message": "Produk tidak dikenal" }
  ]
}
```
- `status`: `success` (tidak ada `rejected`) atau `partial`. Item `flagged` termasuk dalam `accepted_ids`.
- Item `rejected` disimpan di `sync_rejections` dan ditampilkan ke Pengelola; klien memindahkannya ke "Antrean Bermasalah", tidak menghapusnya.
- **Idempotensi**: `client_bundle_id` yang sudah ada di `processed_bundles` mengembalikan `result` yang sama tanpa memproses ulang. Item yang `id`-nya sudah ada dianggap sukses (void, retur, dan transfer punya `id` sendiri sehingga tidak ikut tertimpa). `cash_movements` punya indeks unik `(kind, ref_id)` sebagai pengaman.
- **Perangkat dicabut** (`is_active = false`): server tetap menerima isi bundle ke `sync_rejections` dengan kode `DEVICE_REVOKED` (karantina), lalu membalas `403 DEVICE_REVOKED`. Klien mengunci layar ke aktivasi tetapi **tidak menghapus** data lokal.
- Klien menandai `is_synced = 1` dan mengisi `synced_at = server_timestamp` hanya untuk id di `accepted_ids`.
- `audit_logs` dari klien disimpan dengan `source = 'device'`; bukti utama adalah catatan yang dibuat server (`source = 'server'`).
- Efek di server per jenis item: nota -> `cash_movements` (`sale_cash`), `stock_movements` (`sale`), `customer_debt_ledger` (`sale_debt`); cicilan -> `debt_payment`; penerimaan -> `purchase_receive` dan `purchase_payment`; void/retur -> `void_return`, `void_refund`, `void_reversal`; setoran -> `transfer_out`. Butir "Barang Lain" tidak membuat `stock_movements`.

---

## 3. RPC `pull_ledger`
- `POST /rest/v1/rpc/pull_ledger`, payload `{ "last_synced_at": "2026-11-25T12:00:00Z" }`.
- **Urutan siklus sinkronisasi: push dulu, baru pull.** Pull tidak boleh menimpa data yang masih ada di outbox (Aturan Bisnis 8.10 dan 8.11).
- Server memakai selisih dengan toleransi 2 menit ke belakang (upsert di klien bersifat idempoten). Klien menyimpan `server_timestamp` dari respons, bukan jam perangkat.
- Respons untuk **Tablet Kasir** tidak memuat `cost_price_hpp` maupun `has_price_hike_indicator`, serta tidak memuat telepon/alamat pelanggan; untuk HP Pengelola semuanya disertakan.
```json
{
  "server_timestamp": "2026-11-25T17:30:05.000Z",
  "products_updated": [
    { "id": "a1b2c3d4-0000-1111-2222-333344445555", "barcode": "8991234567890", "box_barcode": null,
      "name": "Beras Rojolele 5kg", "category": "Sembako", "unit": "pcs",
      "product_kind": "regular", "selling_price": 68000,
      "box_selling_price": null, "box_conversion_qty": null, "bundle_qty": null, "bundle_price": null,
      "stock_qty": 24, "empty_stock_qty": 0, "min_stock_alert": 5, "critical_stock_alert": 2,
      "track_expiry": false, "nearest_expiry": null, "is_change_item": false, "favorite_order": 1,
      "image_url": "https://pub-xxxx.r2.dev/products/a1b2c3d4-0000-1111-2222-333344445555.webp",
      "deleted_at": null, "updated_at": "2026-11-25T15:00:00Z" }
  ],
  "customers_updated": [
    { "id": "550e8400-e29b-41d4-a716-446655440000", "name": "Bpk. Joko", "credit_limit": 100000,
      "current_debt": 80000, "deleted_at": null, "updated_at": "2026-11-25T15:00:00Z" }
  ],
  "members_updated": [],
  "drawers_updated": [
    { "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479", "drawer_type": "shared", "drawer_name": "Laci Bersama",
      "assigned_member_id": null }
  ],
  "store_settings": { "timezone": "Asia/Pontianak", "qris_image_r2_url": null,
                      "ai_features_enabled": false, "voice_order_enabled": false,
                      "rounding_unit": 100, "local_retention_days": 3,
                      "help_phone": "62812xxxxxxx", "confirm_change_over": 100000, "confirm_qty_over": 20,
                      "confirm_price_over": 500000, "confirm_total_over": 1000000 },
  "member_public_keys": []
}
```
- Data yang dihapus dikirim sebagai baris dengan `deleted_at` terisi (soft delete).
- `drawers_updated` tidak memuat saldo (blind count). `current_debt` dan `stock_qty` adalah nilai server; tablet menampilkan nilai itu dikurangi/ditambah isi outbox.
- Batas kasbon pelanggan di perangkat dapat tertinggal (data offline). Server menjadi pemutus akhir saat `push_bundle`, dengan memberi tanda `OVER_LIMIT_NO_APPROVAL`, bukan menolak.

---

## 4. RPC Pengelola (HP Pemilik/Kerabat)
Semua RPC memeriksa peran dari login dan perangkat dari `x-install-id` (bagian 1), dan menulis `audit_logs` dengan identitas dari token.

| RPC | Siapa | Tugas |
| :--- | :--- | :--- |
| `confirm_invoice(invoice_id, items[], total_amount)` | Pemilik, Kerabat | menulis harga; menghitung ulang HPP (Aturan Bisnis 5.3); mencatat kekurangan bayar dari Kas Simpanan; menandai selisih jumlah dan kenaikan harga. Tidak mengubah stok |
| `reject_invoice(invoice_id, reason)` | Pemilik, Kerabat | membalik penerimaan sementara (`purchase_reverse`) dan uang laci |
| `settle_consignor(consignor_id, amount, drawer_id)` | Pemilik, Kerabat | bayar penitip |
| `return_consignment(product_id, qty)` | Pemilik, Kerabat | `consignment_return` |
| `stock_opname(items[{product_id, counted_qty, counted_at}])` | Pemilik, Kerabat | selisih dihitung per Aturan Bisnis 5.11; mengisi `products.last_opname_at` |
| `update_price(product_id, ...)`, `set_credit_limit(customer_id, amount)` | Pemilik, Kerabat | |
| `upsert_product(...)`, `import_products(rows[])`, `set_favorites(product_ids[])` | Pemilik, Kerabat | tambah/ubah produk, impor CSV, dan memilih 6-8 favorit berurutan (maksimal 8, urutan = `favorite_order`) |
| `send_login_link(member_id)` | Pemilik, Kerabat | mengirim tautan masuk baru (pemulihan akses); Kerabat boleh mengirimnya ke Pemilik |
| `ack_price_hike(product_id)`, `set_expiry(product_id, date)` | Pemilik, Kerabat | menghapus titik merah; mengoreksi ED terdekat |
| `review_transaction(transaction_id, note)` | Pemilik, Kerabat | menandai Nota Bertanda ditinjau |
| `resolve_rejection(rejection_id, action, note)` | Pemilik, Kerabat | `action` = `retry` (proses ulang payload) atau `discard` (alasan wajib; data tetap disimpan) |
| `force_close_session(session_id, counted_amount, denominations?)` | Pemilik, Kerabat | Tutup Paksa sesi yatim |
| `transfer_cash(from, to, amount)`, `confirm_cash_transfer(id, decision)`, `count_reserve(amount, note)` | Pemilik, Kerabat | F-40 |
| `create_pairing_token(role)`, `register_device`, `revoke_device(device_id)` | **hanya Pemilik** | pencabutan juga menghapus sesi Auth HP |
| `invite_member(...)`, `update_member(...)`, `update_settings(...)` | **hanya Pemilik** | |

- Administrator Teknis (di luar aplikasi) dapat mencabut perangkat darurat langsung lewat dasbor Supabase (service role); trigger `devices` membuat hasilnya sama dengan `revoke_device`, termasuk catatan Audit Log (`actor_kind = 'admin'`).

---

## 5. Foto Produk, Faktur, dan QRIS (Cloudflare R2)
- Unggah lewat Edge Function `r2_presign` (hanya HP Pengelola) yang mengembalikan URL unggah sementara. Foto dikompresi di klien menjadi WebP maksimal sekitar 200 KB.
- Foto produk/QRIS di bucket publik: `https://<URL publik R2>/products/{product_id}.webp`. Selama MVP memakai URL `r2.dev` bawaan (tanpa membeli domain); foto di-cache Service Worker sehingga batas laju `r2.dev` cukup. Domain sendiri bersifat opsional dan berbayar, di luar prinsip gratis.
- Foto anggota (kartu "Siapa yang jaga?") di bucket publik dengan nama acak `members/{uuid-acak}.webp` (tidak dapat ditebak, tanpa nama asli).
- Foto faktur di bucket **privat** (`purchase_invoices.invoice_photo_r2_path`).

---

## 6. Protokol Persetujuan QR
1. **Buat permintaan (Tablet Kasir, online)** - `POST /rest/v1/rpc/create_qr_approval`
   - Body: `{ "request_type": "DEBT_OVERLIMIT", "requested_by_member_id": "…", "payload": { … } }`. Server memeriksa bahwa anggota aktif dan boleh memakai perangkat itu.
   - Isi `payload` per jenis (semua id dibuat klien lebih dulu):
     | Jenis | Isi `payload` |
     | :--- | :--- |
     | `DEBT_OVERLIMIT` | `{customer_id, tx_id, debt_before, debt_amount, debt_after, limit}` |
     | `DAMAGED_STOCK` | `{adjustment_id, product_id, qty, reason}` |
     | `PERSONAL_USE` | `{tx_id, items:[{product_id, qty}]}` |
     | `VOID_LATE` | `{transaction_id, action: "void" atau "return", items: daftar {transaction_item_id, qty} atau null}` |
   - **Kanonisasi hash**: `payload_hash = sha256(JSON dengan kunci diurutkan alfabet di semua tingkat, tanpa spasi, angka tanpa pecahan yang tidak perlu)` dalam hex huruf kecil. Klien dan server memakai aturan yang sama. Pada tampilan Pengelola, server menambahkan angka versi server (mis. saldo kasbon server) hanya sebagai informasi; angka itu tidak ikut hash.
   - Server membuat `qr_token` acak 128-bit, berlaku 5 menit, dan mengirim notifikasi push ke Pengelola.
   - Tablet menampilkan QR berisi `qr_token`, lalu memantau status lewat `get_approval_status(request_id)` setiap 2 detik (Realtime hanya mempercepat; polling selalu jalan).
2. **Persetujuan (HP Pengelola)**
   - Pindai QR di aplikasi (tombol "PINDAI PERMINTAAN"), atau buka notifikasi/daftar "Permintaan Menunggu" (jika `allow_remote_approval`).
   - `POST /rest/v1/rpc/get_qr_approval` (token) menampilkan rincian; `POST /rest/v1/rpc/confirm_qr_approval`:
```json
{ "qr_token": "k3J9x…(22+ karakter acak)", "decision": "approved" }
```
   - Server memeriksa: token pending dan belum kedaluwarsa, pengguna adalah `owner`/`trusted_family`, perangkat pemanggil aktif dan terikat, lalu mengisi `approved_by_member_id` dari token login.
3. **Keterikatan dan sekali pakai**: transaksi, penyesuaian stok, void, atau retur menyimpan `approval_request_id`. Server menolak/menandai bila `payload_hash` tidak cocok dengan isi sebenarnya (id transaksi/penyesuaian, pelanggan, jumlah). Satu persetujuan hanya dapat menempel pada **satu** penggunaan: `consumed_at` diisi dan indeks unik pada tabel pemakai mencegah pemakaian kedua (`APPROVAL_ALREADY_USED`).
4. **Offline (QR dua arah) - Rilis 3**
   - Tablet menampilkan QR permintaan berisi `{request_id, type, payload_hash, nonce, expires}`. HP Pengelola memindainya (tanpa internet), lalu menampilkan QR jawaban berisi tanda tangan ECDSA (kunci privat non-ekstraktabel di HP Pengelola) atas `{request_id, decision, payload_hash, nonce}`. Tablet memindai QR jawaban dan memverifikasi memakai `member_public_keys` hasil `pull_ledger`.
   - `proof` disimpan di `qr_approval_requests`/transaksi dan diverifikasi ulang server saat `push_bundle`.
   - Sebelum Rilis 3, tindakan yang butuh persetujuan **tidak tersedia saat offline** (kasbon over-limit ditolak; barang rusak menunggu online).

---

## 7. Edge Function
| Fungsi | Pemanggil | Tugas |
| :--- | :--- | :--- |
| `provision_device` | HP Pemilik (pairing) | Memvalidasi token pairing, membuat perangkat, menerbitkan JWT perangkat dan `device_secret` |
| `r2_presign` | HP Pengelola | URL unggah foto produk/QRIS/faktur |
| `ai_proxy` | HP Pengelola, Tablet (Voice) | Meneruskan ke Google Gemini API (free tier). Kunci API hanya di server. Memeriksa `ai_features_enabled`, menaikkan `ai_usage`, menolak dengan pesan ramah jika melewati batas harian |
| `send_push` | Database Webhook | Web Push ke HP Pengelola. **R1b**: permintaan persetujuan baru; **R2**: shift ditutup, stok habis, kadaluwarsa, nota bermasalah, dll. |
| `daily_digest` | GitHub Actions (malam hari) | Ringkasan harian, stok kritis, kadaluwarsa terdekat, item `sync_rejections` terbuka, Nota Bertanda belum ditinjau, **nomor nota terlewat** per perangkat, sesi yatim, selisih jam perangkat |
| `report_job` | GitHub Actions (backup, digest, uji restore) | Menulis `ops_heartbeats` (`job`, `status`). Dipanggil dengan kunci rahasia khusus |

- `ai_proxy` menerima `task` = `invoice_ocr` | `package_ocr` | `insight_polish`. Hasil selalu berupa saran yang dikonfirmasi manusia. Batas kuota Gemini free tier berubah-ubah; cek dokumentasi resmi sebelum memakai dan atur batas harian di `ai_usage` di bawah kuota tersebut.
- **Pengawas heartbeat**: pekerjaan `pg_cron` di Supabase berjalan tiap 6 jam dan memeriksa `ops_heartbeats`; bila `last_ok_at` lebih tua dari `max_age_hours` (36 jam untuk backup) atau status terakhir gagal, ia memicu `send_push` ke Pengelola. Pemeriksaan sengaja dari sisi Supabase agar tetap berjalan walau GitHub Actions berhenti.

---

## 8. Format Tautan WhatsApp (`wa.me`)
- Tanpa API; hanya membuka WhatsApp dengan teks siap kirim:
  `https://wa.me/62812xxxxxxx?text=<teks di-URL-encode>`
- Tanpa nomor (`https://wa.me/?text=…`) berarti pengguna memilih kontak sendiri.
- Tiga templat: nota digital, pengingat kasbon sopan, daftar belanja untuk sales. Nomor diubah ke format `62…`.

---

## 9. Kode Kesalahan dan Tanda
**Ditolak (masuk `sync_rejections`)**: `MALFORMED`, `BUNDLE_TOO_LARGE`, `TOTAL_MISMATCH`, `ITEM_SUBTOTAL_MISMATCH`, `UNKNOWN_PRODUCT`, `UNKNOWN_SESSION`, `UNKNOWN_DRAWER`, `VOID_TARGET_UNKNOWN`, `RETURN_EXCEEDS_SOLD`, `DEVICE_REVOKED` (juga `403`).

**Diterima dengan tanda (`review_flags`)**: `OVER_LIMIT_NO_APPROVAL`, `APPROVAL_MISSING`, `PRODUCT_DELETED`, `INVOICE_NO_CONFLICT`, `PRICE_DEVIATION`, `MEMBER_DRAWER_MISMATCH`, `OPENING_MISMATCH`, `SESSION_OVERLAP`, `CLOCK_SKEW`, `VOID_NOT_OWN_SHIFT`.

**Kesalahan permintaan (HTTP)**: `PAIRING_INVALID`, `APPROVAL_ALREADY_USED`, `APPROVAL_EXPIRED`, `APPROVAL_HASH_MISMATCH`, `NOT_AUTHORIZED_ROLE`, `DEVICE_NOT_BOUND`, `AI_QUOTA_EXCEEDED`.
