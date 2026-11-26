# Model Data - K_War
Versi 2.4 - 26 November 2026
Dokumen ini adalah sumber kebenaran. Aturan perubahan: lihat PRD bagian 11.

Prinsip: saldo (stok, kas, kasbon) adalah **hasil jumlah catatan pergerakan**, bukan angka yang ditimpa. Kolom saldo di tabel induk hanya cache yang diperbarui trigger.

## 1. Skema PostgreSQL Supabase (schema.sql)

```sql
-- 1. Perangkat terdaftar
CREATE TABLE devices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_code VARCHAR(20) NOT NULL UNIQUE,            -- DEV1, TAB1
    device_name VARCHAR(100) NOT NULL,
    device_role VARCHAR(20) NOT NULL CHECK (device_role IN ('hp_manager','tablet_cashier')),
    install_id UUID NOT NULL UNIQUE,                    -- dibuat saat provisioning (bukan fingerprint)
    auth_user_id UUID UNIQUE,                           -- khusus hp_manager: akun Supabase Auth yang terikat ke HP ini
    secret_hash TEXT,                                   -- hash device_secret (untuk refresh_device_token); tidak pernah dikirim balik
    is_active BOOLEAN NOT NULL DEFAULT true,
    registered_at TIMESTAMPTZ DEFAULT NOW(),
    last_seen_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ
);

-- 2. Anggota (tidak ada PIN)
CREATE TABLE members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nickname VARCHAR(50) NOT NULL,
    photo_r2_path TEXT,                                 -- kartu "Siapa yang jaga?"
    role VARCHAR(30) NOT NULL CHECK (role IN ('owner','trusted_family','regular_staff')),
    drawer_access VARCHAR(20) NOT NULL CHECK (drawer_access IN ('shared_only','individual_only','both_flexible')),
    auth_user_id UUID UNIQUE,                           -- akun Supabase Auth; hanya owner & trusted_family
    approval_public_key TEXT,                           -- kunci publik QR offline (Rilis 3)
    simple_mode BOOLEAN NOT NULL DEFAULT false,         -- Beranda Sederhana di HP Pengelola (true untuk Pemilik saat setup awal)
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 3. Laci kas (reserve = Kas Simpanan Warung)
CREATE TABLE cash_drawers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    drawer_type VARCHAR(20) NOT NULL CHECK (drawer_type IN ('shared','individual','reserve')),
    drawer_name VARCHAR(50) NOT NULL,
    assigned_member_id UUID REFERENCES members(id),     -- null jika shared/reserve
    current_cash_balance NUMERIC(15,2) NOT NULL DEFAULT 0,  -- cache dari cash_movements
    created_at TIMESTAMPTZ DEFAULT NOW(),
    CHECK ((drawer_type = 'individual') = (assigned_member_id IS NOT NULL))
);

-- 4. Sesi laci (shift)
CREATE TABLE drawer_sessions (
    id UUID PRIMARY KEY,                                -- dibuat klien
    drawer_id UUID NOT NULL REFERENCES cash_drawers(id),
    member_id UUID NOT NULL REFERENCES members(id),
    device_id UUID NOT NULL REFERENCES devices(id),
    opened_at TIMESTAMPTZ NOT NULL,
    opening_balance NUMERIC(15,2) NOT NULL,
    closed_at TIMESTAMPTZ,
    closing_balance_system NUMERIC(15,2),               -- dihitung server dari cash_movements
    closing_balance_actual NUMERIC(15,2),
    cash_denominations JSONB,                           -- {"100000":12,...,"coin_500":4}
    difference_amount NUMERIC(15,2),                    -- hanya terbaca oleh pengelola
    is_difference_reviewed BOOLEAN DEFAULT false,
    status VARCHAR(20) DEFAULT 'open' CHECK (status IN ('open','closed')),
    expected_opening NUMERIC(15,2),                     -- diisi server: closing_balance_actual sesi sebelumnya di laci ini
    declared_counts JSONB,                              -- diisi tablet saat tutup: {"transactions":n,"debt_payments":n,"purchases":n,"transfers":n}
    is_complete BOOLEAN NOT NULL DEFAULT false,         -- true bila jumlah diterima server = declared_counts; selisih final hanya setelah ini
    review_flags TEXT[] NOT NULL DEFAULT '{}',          -- OPENING_MISMATCH, SESSION_OVERLAP, CLOCK_SKEW
    force_closed_by_member_id UUID REFERENCES members(id)  -- Tutup Paksa oleh Pengelola (sesi yatim)
);

-- 5. Pergerakan kas (sumber kebenaran saldo laci)
CREATE TABLE cash_movements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    drawer_id UUID NOT NULL REFERENCES cash_drawers(id),
    session_id UUID REFERENCES drawer_sessions(id),
    amount NUMERIC(15,2) NOT NULL,                      -- + masuk, - keluar
    kind VARCHAR(30) NOT NULL CHECK (kind IN
        ('opening','sale_cash','debt_payment','purchase_payment','consignment_payment','void_refund','adjustment',
         'transfer_out','transfer_in')),   -- void_refund dipakai juga untuk retur; transfer_* untuk Setoran Kas
    ref_id UUID,
    device_id UUID REFERENCES devices(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Pengaman idempotensi: satu jenis catatan kas per rujukan.
CREATE UNIQUE INDEX ux_cash_mv_ref ON cash_movements(kind, ref_id) WHERE ref_id IS NOT NULL;

-- 6. Penitip (konsinyasi)
CREATE TABLE consignors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(30),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 7. Produk
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    barcode VARCHAR(50) UNIQUE,
    box_barcode VARCHAR(50) UNIQUE,                     -- barcode kardus (opsional); memindainya menjual 1 kardus
    name VARCHAR(150) NOT NULL,
    category VARCHAR(50),
    unit VARCHAR(20) DEFAULT 'pcs',
    product_kind VARCHAR(20) NOT NULL DEFAULT 'regular'
        CHECK (product_kind IN ('regular','bulk','refill','consignment','open_price')),
    consignor_id UUID REFERENCES consignors(id),        -- wajib jika consignment
    cost_price_hpp NUMERIC(15,2) NOT NULL DEFAULT 0,    -- konsinyasi: harga setor. TIDAK dikirim ke tablet kasir
    selling_price NUMERIC(15,2) NOT NULL DEFAULT 0,
    box_selling_price NUMERIC(15,2),
    box_conversion_qty NUMERIC(10,2),
    bundle_qty INT,
    bundle_price NUMERIC(15,2),
    stock_qty NUMERIC(12,2) NOT NULL DEFAULT 0,         -- cache dari stock_movements
    empty_stock_qty NUMERIC(12,2) NOT NULL DEFAULT 0,   -- tabung/galon kosong (refill)
    min_stock_alert NUMERIC(12,2) DEFAULT 5,
    critical_stock_alert NUMERIC(12,2) DEFAULT 2,
    track_expiry BOOLEAN NOT NULL DEFAULT false,
    is_change_item BOOLEAN NOT NULL DEFAULT false,      -- boleh dipakai Tukar Receh
    last_opname_at TIMESTAMPTZ,                         -- counted_at opname terakhir (lihat Aturan Bisnis 5.11)
    favorite_order SMALLINT CHECK (favorite_order BETWEEN 1 AND 8),  -- 1..8 = tampil di halaman Favorit; null = bukan favorit
    nearest_expiry DATE,    -- shortcut: hanya ED terdekat, bukan per batch; upgrade ke tabel batch jika perlu akurasi per batch
    has_price_hike_indicator BOOLEAN DEFAULT false,
    image_r2_path TEXT,
    deleted_at TIMESTAMPTZ,                             -- soft delete agar tersinkron ke perangkat
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()                -- diisi trigger BEFORE UPDATE
);
-- Baris sistem: satu produk 'Barang Lain' dengan product_kind = 'open_price' (id tetap).

-- 8. Pergerakan stok (sumber kebenaran stok)
CREATE TABLE stock_movements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id),
    qty_delta NUMERIC(12,2) NOT NULL,
    empty_qty_delta NUMERIC(12,2) NOT NULL DEFAULT 0,
    reason VARCHAR(30) NOT NULL CHECK (reason IN
        ('sale','void_return','purchase_receive','purchase_reverse','consignment_in','consignment_return',
         'damaged','expired','personal_use','opname')),
    ref_type VARCHAR(30),
    ref_id UUID,
    device_id UUID REFERENCES devices(id),
    applied_to_cache BOOLEAN NOT NULL DEFAULT true,     -- false bila created_at < products.last_opname_at (sudah terhitung opname)
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 9. Faktur kulakan
CREATE TABLE purchase_invoices (
    id UUID PRIMARY KEY,                                -- dibuat klien
    invoice_number VARCHAR(100),
    supplier_name VARCHAR(100),
    received_by_member_id UUID NOT NULL REFERENCES members(id),
    confirmed_by_member_id UUID REFERENCES members(id),
    session_id UUID REFERENCES drawer_sessions(id),
    total_amount NUMERIC(15,2) NOT NULL DEFAULT 0,      -- diisi saat dikonfirmasi
    paid_from_drawer_id UUID REFERENCES cash_drawers(id),
    paid_from_drawer_amount NUMERIC(15,2) DEFAULT 0,    -- diinput kasir saat barang datang
    paid_from_reserve_amount NUMERIC(15,2) DEFAULT 0,   -- dari Kas Simpanan
    invoice_photo_r2_path TEXT,
    status VARCHAR(30) DEFAULT 'pending_confirmation' CHECK (status IN ('pending_confirmation','confirmed','rejected')),
    created_at TIMESTAMPTZ NOT NULL,
    confirmed_at TIMESTAMPTZ
);

CREATE TABLE purchase_invoice_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id UUID NOT NULL REFERENCES purchase_invoices(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id),
    received_qty NUMERIC(12,2) NOT NULL,                -- hitungan kasir
    invoice_qty NUMERIC(12,2),                          -- jumlah di faktur (isi pengelola)
    empty_returned_qty NUMERIC(12,2) DEFAULT 0,         -- tabung/galon kosong dikembalikan ke agen
    expiry_date DATE,
    purchase_price_per_unit NUMERIC(15,2)               -- null sampai dikonfirmasi pengelola
);

-- 10. Pembayaran penitip
CREATE TABLE consignment_settlements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    consignor_id UUID NOT NULL REFERENCES consignors(id),
    amount NUMERIC(15,2) NOT NULL,
    paid_from_drawer_id UUID REFERENCES cash_drawers(id),
    settled_by_member_id UUID NOT NULL REFERENCES members(id),
    settled_at TIMESTAMPTZ DEFAULT NOW()
);

-- 11. Pelanggan & buku kasbon
CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(30),                                  -- format 62xxxxxxxx untuk wa.me
    address TEXT,
    credit_limit NUMERIC(15,2) NOT NULL DEFAULT 0,      -- pelanggan baru dari kasir = 0 (kasbon pertama butuh Persetujuan QR); form Pengelola menyarankan Rp500.000
    created_by_member_id UUID REFERENCES members(id),
    current_debt NUMERIC(15,2) NOT NULL DEFAULT 0,      -- cache dari customer_debt_ledger
    deleted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE customer_debt_ledger (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customers(id),
    delta NUMERIC(15,2) NOT NULL,                       -- + utang bertambah, - berkurang
    kind VARCHAR(20) NOT NULL CHECK (kind IN ('sale_debt','payment','void_reversal','adjustment')),
    ref_id UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 12. Permintaan Persetujuan QR (sebelum transactions karena direferensikan)
CREATE TABLE qr_approval_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id UUID NOT NULL REFERENCES devices(id),
    requested_by_member_id UUID NOT NULL REFERENCES members(id),
    request_type VARCHAR(30) NOT NULL CHECK (request_type IN ('DEBT_OVERLIMIT','DAMAGED_STOCK','PERSONAL_USE','VOID_LATE')),
    qr_token VARCHAR(64) UNIQUE NOT NULL,               -- 128-bit acak (base64url), sekali pakai
    payload JSONB NOT NULL,
    payload_hash CHAR(64) NOT NULL,                     -- sha256 payload; persetujuan terikat ke hash ini
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending','approved','rejected','expired')),
    approved_by_member_id UUID REFERENCES members(id),  -- diambil dari login server
    proof JSONB,                                        -- tanda tangan (QR offline, Rilis 3)
    consumed_at TIMESTAMPTZ,                            -- diisi saat dipakai transaksi/penyesuaian/void; satu persetujuan = satu penggunaan
    consumed_ref UUID,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    responded_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ DEFAULT (NOW() + INTERVAL '5 minutes')
);

-- 13. Transaksi penjualan
CREATE TABLE transactions (
    id UUID PRIMARY KEY,                                -- UUID v4 dari klien
    device_id UUID NOT NULL REFERENCES devices(id),
    member_id UUID NOT NULL REFERENCES members(id),
    drawer_id UUID NOT NULL REFERENCES cash_drawers(id),
    session_id UUID NOT NULL REFERENCES drawer_sessions(id),
    customer_id UUID REFERENCES customers(id),
    invoice_no VARCHAR(50) NOT NULL,
    subtotal NUMERIC(15,2) NOT NULL,
    rounding_adj NUMERIC(10,2) DEFAULT 0,
    total_final NUMERIC(15,2) NOT NULL,
    payment_type VARCHAR(20) NOT NULL CHECK (payment_type IN
        ('cash_full','debt_full','split_cash_debt','qris_static','personal_use')),
    cash_tendered NUMERIC(15,2) DEFAULT 0,
    cash_change NUMERIC(15,2) DEFAULT 0,
    debt_amount NUMERIC(15,2) DEFAULT 0,
    qris_confirmed_at TIMESTAMPTZ,                      -- saat kasir menekan "Uang Sudah Masuk"
    approval_request_id UUID REFERENCES qr_approval_requests(id),  -- wajib untuk over-limit / pakai sendiri
    review_flags TEXT[] NOT NULL DEFAULT '{}',          -- Nota Bertanda: OVER_LIMIT_NO_APPROVAL, APPROVAL_MISSING, PRODUCT_DELETED, INVOICE_NO_CONFLICT, PRICE_DEVIATION, MEMBER_DRAWER_MISMATCH, CLOCK_SKEW
    review_status VARCHAR(10) NOT NULL DEFAULT 'none' CHECK (review_status IN ('none','open','reviewed')),
    reviewed_by_member_id UUID REFERENCES members(id),
    reviewed_at TIMESTAMPTZ,
    status VARCHAR(20) DEFAULT 'completed' CHECK (status IN ('completed','voided','returned')),  -- returned = seluruh butir diretur
    void_reason VARCHAR(30) CHECK (void_reason IN ('salah_tekan','pembeli_batal','barang_dikembalikan')),
    voided_at TIMESTAMPTZ,
    voided_by_member_id UUID REFERENCES members(id),
    void_approval_request_id UUID REFERENCES qr_approval_requests(id),  -- wajib bila VOID_LATE
    created_at TIMESTAMPTZ NOT NULL,
    synced_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (device_id, invoice_no),
    CHECK (total_final = subtotal + rounding_adj)
);

CREATE TABLE transaction_items (
    id UUID PRIMARY KEY,                                -- dibuat klien agar retur offline dapat merujuk butir
    transaction_id UUID NOT NULL REFERENCES transactions(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id),
    custom_name VARCHAR(150),                           -- nama untuk 'Barang Lain'
    qty NUMERIC(10,2) NOT NULL,
    selling_price_unit NUMERIC(15,2) NOT NULL,
    cost_price_unit NUMERIC(15,2) NOT NULL,             -- DIISI SERVER saat push_bundle, bukan klien
    is_box_sale BOOLEAN DEFAULT false,
    exchange_empty BOOLEAN DEFAULT false,               -- isi ulang dengan tukar tabung kosong
    subtotal NUMERIC(15,2) NOT NULL
);

-- 14. Pembayaran cicilan kasbon
CREATE TABLE debt_payments (
    id UUID PRIMARY KEY,
    customer_id UUID NOT NULL REFERENCES customers(id),
    member_id UUID NOT NULL REFERENCES members(id),
    drawer_id UUID NOT NULL REFERENCES cash_drawers(id),
    session_id UUID NOT NULL REFERENCES drawer_sessions(id),
    amount_paid NUMERIC(15,2) NOT NULL,
    payment_date TIMESTAMPTZ NOT NULL,
    synced_at TIMESTAMPTZ DEFAULT NOW()
);

-- View v_consignment_payable didefinisikan di bagian 19 (bergantung pada tabel retur).

-- 15. Pengaturan toko
CREATE TABLE store_settings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    store_name VARCHAR(100) NOT NULL DEFAULT 'Warung Sembako K_War',
    store_address TEXT,
    store_phone VARCHAR(30),
    timezone VARCHAR(40) NOT NULL DEFAULT 'Asia/Pontianak',  -- hanya untuk batas hari laporan server; nota/struk memakai jam perangkat
    qris_image_r2_url TEXT,                             -- null = tombol QRIS disembunyikan
    receipt_footer_text TEXT DEFAULT 'Terima kasih atas kunjungan Anda',
    ai_features_enabled BOOLEAN NOT NULL DEFAULT false,
    voice_order_enabled BOOLEAN NOT NULL DEFAULT false,
    smart_insight_enabled BOOLEAN NOT NULL DEFAULT true,
    allow_remote_approval BOOLEAN NOT NULL DEFAULT true,
    expiry_alert_days INT NOT NULL DEFAULT 30,
    rounding_unit INT NOT NULL DEFAULT 100 CHECK (rounding_unit IN (0,100,500)),  -- 0 = tanpa pembulatan; QRIS tidak pernah dibulatkan
    local_retention_days INT NOT NULL DEFAULT 3,        -- retensi transaksi tersinkron di tablet
    shift_max_hours INT NOT NULL DEFAULT 16,            -- lewat ini = Sesi Yatim
    void_alert_per_shift INT NOT NULL DEFAULT 3,        -- ambang Smart Insight
    cash_diff_alert_amount NUMERIC(15,2) NOT NULL DEFAULT 20000,
    price_hike_alert_pct INT NOT NULL DEFAULT 10,
    confirm_change_over NUMERIC(15,2) NOT NULL DEFAULT 100000,   -- konfirmasi angka tidak wajar (Aturan Bisnis 4.16)
    confirm_qty_over INT NOT NULL DEFAULT 20,
    confirm_price_over NUMERIC(15,2) NOT NULL DEFAULT 500000,    -- harga Barang Lain
    confirm_total_over NUMERIC(15,2) NOT NULL DEFAULT 1000000,
    help_phone VARCHAR(30),                             -- WhatsApp bantuan (format 62...), diisi Pemilik
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 16. Audit log (tambah-saja)
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type VARCHAR(50) NOT NULL,  -- VOID_NOTE, RETURN_ITEM, PRICE_CHANGE, STOCK_OPNAME,
                                      -- APPROVAL_DECISION, DAMAGED_STOCK, PERSONAL_USE, QRIS_CONFIRMED, DEVICE_REVOKED,
                                      -- SESSION_FORCE_CLOSED, REJECTION_RESOLVED, TX_REVIEWED, CASH_TRANSFER, RESERVE_COUNT, CREDIT_LIMIT_CHANGE
    member_id UUID REFERENCES members(id),             -- null bila aksi sistem/admin
    device_id UUID REFERENCES devices(id),
    actor_kind VARCHAR(10) NOT NULL DEFAULT 'member' CHECK (actor_kind IN ('member','system','admin')),
    source VARCHAR(10) NOT NULL DEFAULT 'server' CHECK (source IN ('server','device')),  -- 'device' = dilaporkan tablet, bukan bukti utama
    reason TEXT NOT NULL,
    reference_id UUID,
    metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
-- Hak akses: hanya INSERT dan SELECT; REVOKE UPDATE, DELETE, TRUNCATE untuk semua peran aplikasi (lihat bagian 1b). Penulisan hanya lewat RPC/trigger.

-- 17. Sinkronisasi: idempotensi & item bermasalah
CREATE TABLE processed_bundles (
    client_bundle_id UUID PRIMARY KEY,
    device_id UUID NOT NULL REFERENCES devices(id),
    result JSONB NOT NULL,                              -- respons yang sama dikembalikan bila bundle dikirim ulang
    processed_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE sync_rejections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id UUID NOT NULL REFERENCES devices(id),
    client_bundle_id UUID,
    item_type VARCHAR(30) NOT NULL,
    item_id UUID,
    code VARCHAR(40) NOT NULL,                          -- kode penolakan: lihat Spesifikasi API bagian 9
    payload JSONB NOT NULL,
    status VARCHAR(20) DEFAULT 'open' CHECK (status IN ('open','resolved','discarded')),
    resolved_by_member_id UUID REFERENCES members(id),
    resolved_at TIMESTAMPTZ,
    resolution_note TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 18. Notifikasi & pemakaian AI
CREATE TABLE push_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    member_id UUID NOT NULL REFERENCES members(id),
    endpoint TEXT NOT NULL UNIQUE,
    keys JSONB NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE ai_usage (
    day DATE PRIMARY KEY,
    request_count INT NOT NULL DEFAULT 0                -- Edge Function menolak bila melewati batas harian
);

-- 19. Tambahan v2.3
-- Penyesuaian stok rusak/kadaluwarsa dari kasir (butuh Persetujuan QR)
CREATE TABLE stock_adjustments (
    id UUID PRIMARY KEY,                                -- dibuat klien
    product_id UUID NOT NULL REFERENCES products(id),
    qty NUMERIC(12,2) NOT NULL CHECK (qty > 0),
    reason VARCHAR(30) NOT NULL CHECK (reason IN ('damaged','expired')),
    member_id UUID NOT NULL REFERENCES members(id),
    device_id UUID NOT NULL REFERENCES devices(id),
    approval_request_id UUID REFERENCES qr_approval_requests(id),
    created_at TIMESTAMPTZ NOT NULL
);
-- Pengembalian konsinyasi ke penitip dilakukan Pengelola lewat RPC `return_consignment` (reason 'consignment_return').

-- Retur (sebagian atau seluruhnya)
CREATE TABLE transaction_returns (
    id UUID PRIMARY KEY,                                -- dibuat klien
    transaction_id UUID NOT NULL REFERENCES transactions(id),
    reason VARCHAR(30) NOT NULL DEFAULT 'barang_dikembalikan',
    returned_by_member_id UUID NOT NULL REFERENCES members(id),
    session_id UUID REFERENCES drawer_sessions(id),
    approval_request_id UUID REFERENCES qr_approval_requests(id),   -- wajib bila VOID_LATE
    refund_cash NUMERIC(15,2) NOT NULL DEFAULT 0,       -- bagian tunai yang dikembalikan (void_refund)
    reversed_debt NUMERIC(15,2) NOT NULL DEFAULT 0,     -- bagian kasbon yang dikoreksi (void_reversal); dikoreksi lebih dulu
    created_at TIMESTAMPTZ NOT NULL
);
CREATE TABLE transaction_return_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    return_id UUID NOT NULL REFERENCES transaction_returns(id) ON DELETE CASCADE,
    transaction_item_id UUID NOT NULL REFERENCES transaction_items(id),
    qty NUMERIC(10,2) NOT NULL CHECK (qty > 0),
    refund_amount NUMERIC(15,2) NOT NULL
);

-- Setoran & transfer kas (F-40)
CREATE TABLE cash_transfers (
    id UUID PRIMARY KEY,                                -- dibuat klien
    from_drawer_id UUID NOT NULL REFERENCES cash_drawers(id),
    to_drawer_id UUID NOT NULL REFERENCES cash_drawers(id),
    amount NUMERIC(15,2) NOT NULL CHECK (amount > 0),
    created_by_member_id UUID NOT NULL REFERENCES members(id),
    session_id UUID REFERENCES drawer_sessions(id),
    device_id UUID REFERENCES devices(id),
    status VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending','confirmed','rejected')),
    confirmed_by_member_id UUID REFERENCES members(id),
    created_at TIMESTAMPTZ NOT NULL,
    confirmed_at TIMESTAMPTZ,
    CHECK (from_drawer_id <> to_drawer_id)
);

-- Urutan favorit unik
CREATE UNIQUE INDEX ux_fav_order ON products(favorite_order) WHERE favorite_order IS NOT NULL AND deleted_at IS NULL;

-- Satu persetujuan = satu penggunaan
CREATE UNIQUE INDEX ux_tx_approval     ON transactions(approval_request_id)      WHERE approval_request_id IS NOT NULL;
CREATE UNIQUE INDEX ux_tx_void_appr    ON transactions(void_approval_request_id) WHERE void_approval_request_id IS NOT NULL;
CREATE UNIQUE INDEX ux_return_approval ON transaction_returns(approval_request_id) WHERE approval_request_id IS NOT NULL;
CREATE UNIQUE INDEX ux_adj_approval    ON stock_adjustments(approval_request_id) WHERE approval_request_id IS NOT NULL;

-- Token pairing perangkat (dibuat Pemilik; sekali pakai; 10 menit)
CREATE TABLE pairing_tokens (
    token_hash CHAR(64) PRIMARY KEY,                    -- sha256 dari token acak 128-bit; token asli tampil sekali di layar Pemilik
    device_role VARCHAR(20) NOT NULL CHECK (device_role IN ('hp_manager','tablet_cashier')),
    created_by_member_id UUID NOT NULL REFERENCES members(id),
    expires_at TIMESTAMPTZ NOT NULL DEFAULT (NOW() + INTERVAL '10 minutes'),
    used_at TIMESTAMPTZ
);

-- Detak pekerjaan terjadwal (backup, digest, uji restore); diperiksa pg_cron tiap 6 jam
CREATE TABLE ops_heartbeats (
    job TEXT PRIMARY KEY,                               -- 'backup' | 'keepalive' | 'daily_digest' | 'restore_test'
    last_ok_at TIMESTAMPTZ,
    last_status TEXT,
    max_age_hours INT NOT NULL DEFAULT 36               -- restore_test: 2200 (kira-kira satu kuartal)
);

-- Utang ke penitip = terjual x harga setor - retur - pembayaran
CREATE VIEW v_consignment_payable WITH (security_invoker = true) AS
SELECT c.id AS consignor_id, c.name,
       COALESCE(s.sold_cost,0) - COALESCE(r.returned_cost,0) - COALESCE(p.paid,0) AS payable
FROM consignors c
LEFT JOIN (SELECT pr.consignor_id, SUM(ti.qty * ti.cost_price_unit) AS sold_cost
           FROM transaction_items ti
           JOIN transactions t ON t.id = ti.transaction_id AND t.status IN ('completed','returned')
           JOIN products pr ON pr.id = ti.product_id
           WHERE pr.consignor_id IS NOT NULL GROUP BY pr.consignor_id) s ON s.consignor_id = c.id
LEFT JOIN (SELECT pr.consignor_id, SUM(tri.qty * ti.cost_price_unit) AS returned_cost
           FROM transaction_return_items tri
           JOIN transaction_items ti ON ti.id = tri.transaction_item_id
           JOIN transactions t ON t.id = ti.transaction_id AND t.status IN ('completed','returned')
           JOIN products pr ON pr.id = ti.product_id
           WHERE pr.consignor_id IS NOT NULL GROUP BY pr.consignor_id) r ON r.consignor_id = c.id
LEFT JOIN (SELECT consignor_id, SUM(amount) AS paid FROM consignment_settlements GROUP BY consignor_id) p
       ON p.consignor_id = c.id;
```

### Trigger & aturan server (ringkas)
- `updated_at` diperbarui trigger pada `products`, `customers`, `members`, `store_settings`.
- `AFTER INSERT` pada `stock_movements` memperbarui `products.stock_qty`/`empty_stock_qty` **hanya bila `applied_to_cache = true`** (server mengisi false bila `created_at` < `products.last_opname_at` dan reason bukan `opname`); pada `cash_movements` memperbarui `cash_drawers.current_cash_balance`; pada `customer_debt_ledger` memperbarui `customers.current_debt`.
- Saat penerimaan kulakan diproses (`push_bundle`): `purchase_receive` menambah stok seketika (Penerimaan Sementara) dan `nearest_expiry` diperbarui. Saat konfirmasi faktur: HPP dihitung ulang (Aturan Bisnis 5.3), `has_price_hike_indicator` diatur, kekurangan bayar dicatat dari Kas Simpanan; stok tidak berubah. Faktur ditolak: `purchase_reverse`.
- RLS: perangkat `tablet_cashier` **tidak** punya akses `SELECT` ke tabel `products` langsung; data katalog hanya lewat RPC `pull_ledger` yang membuang `cost_price_hpp`.
- Stok negatif diperbolehkan tetapi ditandai pada ringkasan harian.
- Bila `stock_qty` <= 0 pada produk `track_expiry`, `nearest_expiry` dikosongkan.
- Trigger pada `devices`: perubahan `is_active`/`revoked_at` (termasuk lewat dasbor Supabase/service role) mengisi `revoked_at` dan menulis `audit_logs` (`actor_kind = 'admin'` bila tanpa identitas anggota). Bila `is_active` menjadi false pada `hp_manager`, sesi login pengguna terikat dihapus (`auth.sessions`).
- Tiap item `push_bundle` diproses dalam satu sub-transaksi (savepoint) sehingga catatan kas/stok/kasbon dan item induknya atomik; `ux_cash_mv_ref` menjadi pengaman terakhir terhadap duplikasi.
- Server mengisi `drawer_sessions.expected_opening` dan menandai `OPENING_MISMATCH`/`SESSION_OVERLAP`; mengisi `is_complete` bila jumlah catatan sesi sama dengan `declared_counts`; `difference_amount` baru dihitung saat `is_complete = true`.
- Pelanggan baru dari kasir: `credit_limit = 0` dan `created_by_member_id` terisi.

## 1b. Keamanan Database (wajib selesai di R1a)
Skema di atas hanya mendefinisikan data. Sebelum ada data nyata, hal berikut harus tertulis di migrasi dan lolos pemeriksaan Supabase Security Advisor:
1. **Semua tabel `public`**: `ENABLE ROW LEVEL SECURITY`; `REVOKE ALL ... FROM anon, authenticated, PUBLIC`; lalu hanya pemberian hak di matriks bawah. Tidak ada tabel tanpa RLS.
2. **Tablet Kasir** tidak punya akses tabel langsung; hanya RPC `SECURITY DEFINER` (dengan `search_path` dikunci) yang membaca klaim `device_id` dari JWT (bukan dari isi permintaan) dan memeriksa `devices.is_active`.
3. **Pengelola**: PostgREST hanya untuk `SELECT`. **Semua penulisan lewat RPC** yang memeriksa peran dari `members.auth_user_id`, memeriksa bahwa permintaan datang dari `hp_manager` aktif yang terikat ke akun itu (`x-install-id`), dan menulis `audit_logs`.
4. `audit_logs`: `REVOKE UPDATE, DELETE, TRUNCATE` dari semua peran aplikasi.
5. View memakai `security_invoker = true`; `service_role` hanya dipakai Edge Function dan Administrator Teknis.

| Tabel | Tablet (JWT perangkat) | Kerabat Tepercaya | Pemilik |
| :--- | :--- | :--- | :--- |
| `products` | tidak ada; katalog lewat `pull_ledger` tanpa HPP | SELECT; ubah lewat RPC | SELECT; ubah lewat RPC |
| `members` | subset publik lewat `pull_ledger` | SELECT | SELECT; ubah lewat RPC |
| `devices`, `pairing_tokens` | tidak ada | SELECT `devices` | SELECT; daftar/cabut lewat RPC |
| `cash_drawers`, `drawer_sessions`, `cash_movements`, `cash_transfers` | tulis lewat `push_bundle`; tidak boleh membaca saldo | SELECT | SELECT |
| `transactions`, `transaction_items`, `transaction_returns*` | tulis lewat `push_bundle` | SELECT | SELECT |
| `customers`, `customer_debt_ledger`, `debt_payments` | nama, limit, saldo lewat `pull_ledger`; tulis lewat `push_bundle` | SELECT; ubah limit lewat RPC | sama |
| `purchase_invoices*`, `stock_adjustments`, `consignment_settlements` | tulis lewat `push_bundle` | SELECT; konfirmasi lewat RPC | sama |
| `qr_approval_requests` | hanya RPC `create_qr_approval`, `get_approval_status` | hanya RPC `get_qr_approval`, `confirm_qr_approval` | sama |
| `audit_logs`, `sync_rejections` | tidak ada | SELECT | SELECT |
| `store_settings` | subset lewat `pull_ledger` | SELECT | SELECT; ubah lewat RPC |
| `ai_usage`, `push_subscriptions`, `ops_heartbeats`, `processed_bundles` | tidak ada | milik sendiri (`push_subscriptions`) | sama |

Hak Pemilik atas anggota, perangkat, dan pengaturan toko **tidak** diberikan ke Kerabat pada RPC terkait (`invite_member`, `register_device`, `revoke_device`, `update_settings`).

## 2. Struktur Penyimpanan Lokal Dexie.js (IndexedDB)
Boolean **tidak** dapat diindeks di IndexedDB, jadi `is_synced` disimpan sebagai angka `0 | 1`.

```typescript
import Dexie, { Table } from 'dexie';

export class KWarDatabase extends Dexie {
  products!: Table<LocalProduct, string>;          // tanpa cost_price_hpp; stock_qty = nilai server. Stok TAMPIL = stock_qty - jumlah di outbox yang belum tersinkron (Aturan Bisnis 8.11)
  customers!: Table<LocalCustomer, string>;
  members!: Table<LocalMember, string>;            // kartu "Siapa yang jaga?"
  cashDrawers!: Table<LocalCashDrawer, string>;   // id laci dari pull_ledger; tanpa saldo (blind count)
  activeCarts!: Table<LocalCart, number>;          // id 1..5
  localTransactions!: Table<LocalTransaction, string>;
  outboxQueue!: Table<OutboxItem, string>;         // transaksi, cicilan, sesi laci, penerimaan kulakan, penyesuaian stok, void, retur, setoran kas, pelanggan baru
  cashDrawerSession!: Table<LocalDrawerSession, string>;
  appSettings!: Table<LocalSetting, string>;       // termasuk counter nota: key 'invoice_seq:ddmmyyyy'

  constructor(name = 'KWarLocalDB') {
    super(name);                                   // Mode Latihan memakai nama 'KWarPracticeDB'
    this.version(1).stores({
      products: 'id, barcode, name, category, nearest_expiry, favorite_order',   // favorite_order angka/null (boolean tidak bisa diindeks)
      customers: 'id, name',                           // tanpa telepon/alamat (minimisasi data pribadi)
      cashDrawers: 'id, drawer_type',
      members: 'id, role',
      activeCarts: 'id, updatedAt',
      localTransactions: 'id, invoice_no, createdAt, is_synced, synced_at',
      outboxQueue: 'id, type, createdAt, status',
      cashDrawerSession: 'id, drawer_id, status',
      appSettings: 'key'
    });
    // Aturan: perubahan skema = version(2), version(3), ... dengan .upgrade(); jangan ubah version yang sudah dirilis.
  }
}
```

### Counter nomor nota (tidak dihitung dari isi tabel)
```typescript
export async function nextInvoiceNo(deviceCode: string) {
  const ddmmyyyy = formatDate(new Date(), 'ddMMyyyy');   // tanggal dari jam perangkat
  const key = `invoice_seq:${ddmmyyyy}`;
  return db.transaction('rw', db.appSettings, async () => {
    const row = await db.appSettings.get(key);
    const seq = (row?.value ?? 0) + 1;
    await db.appSettings.put({ key, value: seq });
    return `kw/${String(seq).padStart(4, '0')}/${deviceCode}/${ddmmyyyy}`;
  });
}
```

### Pembersihan data lokal (purge)
```typescript
// Dijalankan saat aplikasi dibuka dan setelah sinkronisasi sukses.
export async function purgeOldSyncedTransactions() {
  const days = (await db.appSettings.get('local_retention_days'))?.value ?? 3;   // dari store_settings.local_retention_days
  const cutoff = new Date(Date.now() - days * 24 * 60 * 60 * 1000).toISOString();
  await db.localTransactions
    .where('is_synced').equals(1)                    // 1 = sudah dikonfirmasi server
    .filter(tx => tx.synced_at !== null && tx.synced_at < cutoff)  // synced_at = server_timestamp dari respons
    .delete();
}
```
