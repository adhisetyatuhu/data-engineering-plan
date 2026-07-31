---
title: Overview
parent: Minggu 1 - SQL Fundamentals
nav_order: 1
---

# Modul Minggu 1 — SQL Fundamentals (untuk Software Developer)

> Materi ajar ini adalah pendamping `minggu_1.md` (jadwal & outline). File ini berisi konten pengajaran lengkap: penjelasan konsep, analogi ke dunia software development, contoh query, latihan, dan kunci jawaban — siap dipakai untuk sesi mengajar (live coding) atau belajar mandiri.

## Untuk Siapa Materi Ini

Peserta: software developer yang **sudah bisa** programming (variabel, function, array/object, loop, kondisional) tapi **belum terbiasa** dengan SQL/database relasional secara mendalam. Pendekatan mengajar di seluruh modul ini selalu memakai jembatan: *"kalau di kode kamu biasa nulis X, di SQL ini setara Y"*.

## Filosofi Mengajar Minggu Ini

Shift mental terbesar buat developer yang belajar SQL: **dari imperative ke declarative.**

- Kode (imperative): "Ambil array `users`, loop tiap elemen, kalau `age > 18` push ke array baru."
- SQL (declarative): "Saya mau `users` yang `age > 18`." — Kamu bilang **apa** yang kamu mau, bukan **bagaimana** cara komputer menghitungnya. Query planner yang urus eksekusinya.

Tekankan ini di hari pertama (lihat catatan instruktur di `hari_1_select_where.md`). Kalau mental model ini klik dari awal, seluruh minggu ini akan lebih cepat dicerna karena developer akan terus mencari padanan mental "ini kayak `.filter()`", "ini kayak `.reduce()`", dst.

## Setup Environment

Rekomendasi: **PostgreSQL via Docker**, bukan SQLite — karena (a) developer biasanya sudah familiar Docker, (b) PostgreSQL mendukung semua fitur yang akan dipakai sepanjang roadmap ini (RIGHT/FULL JOIN, window function lengkap, dsb) tanpa quirk versi, dan (c) ini konsisten dengan tools yang akan dipakai di project sungguhan nanti (Minggu 3+ pakai Postgres/warehouse sungguhan).

```bash
# jalankan PostgreSQL lokal
docker run --name pg-belajar -e POSTGRES_PASSWORD=belajar -p 5432:5432 -d postgres:16

# konek pakai psql (bawaan image) atau client GUI (DBeaver / TablePlus / pgAdmin)
docker exec -it pg-belajar psql -U postgres
```

Alternatif tanpa install apa-apa: [DB Fiddle](https://www.db-fiddle.com/) (pilih PostgreSQL) — cocok untuk demo cepat saat sesi mengajar, tapi untuk latihan mandiri tetap disarankan pakai Docker supaya terbiasa dengan environment kerja beneran.

## Dataset Latihan: Toko Elektronik & Furnitur Mini

Dataset ini **dipakai konsisten di semua modul harian Minggu 1** (hari 1–5 + latihan akhir pekan), supaya peserta tidak perlu mikir ulang skema tiap hari — fokus ke konsep SQL yang baru.

> Catatan: ini bukan dataset "Online Retail II" yang dipakai untuk mini project portfolio (lihat `minggu_2.md`). Dataset di bawah sengaja kecil & bersih supaya hasil query gampang diverifikasi manual saat belajar konsep dasar. Online Retail II baru masuk saat mini project gabungan SQL+Python di Minggu 2.

### Skema

```
customers (customer_id PK, customer_name, country, signup_date)
products  (product_id PK, product_name, category, unit_price)
orders    (order_id PK, customer_id FK, order_date, status)
order_items (order_item_id PK, order_id FK, product_id FK, quantity, unit_price)
```

Relasi: `customers 1—N orders`, `orders 1—N order_items`, `products 1—N order_items`.

### Setup Script

Jalankan sekali di awal minggu (simpan sebagai `schema.sql` dan jalankan lewat `psql -U postgres -f schema.sql` atau paste ke client GUI):

```sql
DROP TABLE IF EXISTS order_items, orders, products, customers CASCADE;

CREATE TABLE customers (
    customer_id   INT PRIMARY KEY,
    customer_name VARCHAR(50),
    country       VARCHAR(50),
    signup_date   DATE
);

CREATE TABLE products (
    product_id   INT PRIMARY KEY,
    product_name VARCHAR(50),
    category     VARCHAR(30),
    unit_price   NUMERIC(10,2)
);

CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),
    order_date  DATE,
    status      VARCHAR(20)  -- 'completed', 'cancelled', 'refunded'
);

CREATE TABLE order_items (
    order_item_id INT PRIMARY KEY,
    order_id      INT REFERENCES orders(order_id),
    product_id    INT REFERENCES products(product_id),
    quantity      INT,
    unit_price    NUMERIC(10,2)
);

INSERT INTO customers VALUES
(1, 'Andi Wijaya',   'Indonesia', '2023-01-10'),
(2, 'Budi Santoso',  'Indonesia', '2023-02-15'),
(3, 'Citra Lestari', 'Indonesia', '2023-03-01'),
(4, 'Emma Watson',   'UK',        '2023-01-20'),
(5, 'Frank Muller',  'Germany',   '2023-04-05'),
(6, 'Grace Tan',     'Singapore', '2023-05-12'),
(7, 'Hassan Ali',    'UAE',       '2023-06-01');

INSERT INTO products VALUES
(1, 'Wireless Mouse',      'Electronics', 15.00),
(2, 'Mechanical Keyboard', 'Electronics', 45.00),
(3, 'USB-C Hub',           'Electronics', 25.00),
(4, 'Office Chair',        'Furniture',   120.00),
(5, 'Notebook A5',         'Stationery',  3.50),
(6, 'Coffee Mug',          'Kitchen',     8.00),
(7, 'Desk Lamp',           'Furniture',   30.00);

INSERT INTO orders VALUES
(1,  1, '2024-01-05', 'completed'),
(2,  2, '2024-01-08', 'completed'),
(3,  1, '2024-01-20', 'completed'),
(4,  3, '2024-02-02', 'completed'),
(5,  4, '2024-02-10', 'completed'),
(6,  2, '2024-02-15', 'cancelled'),
(7,  5, '2024-02-18', 'completed'),
(8,  1, '2024-03-01', 'completed'),
(9,  6, '2024-03-05', 'completed'),
(10, 3, '2024-03-12', 'completed'),
(11, 4, '2024-03-20', 'refunded'),
(12, 2, '2024-03-25', 'completed');

INSERT INTO order_items VALUES
(1,  1, 1, 2,  15.00),
(2,  1, 5, 3,  3.50),
(3,  2, 2, 1,  45.00),
(4,  2, 6, 2,  8.00),
(5,  3, 3, 1,  25.00),
(6,  4, 4, 1,  120.00),
(7,  4, 6, 1,  8.00),
(8,  5, 1, 1,  15.00),
(9,  5, 2, 1,  45.00),
(10, 6, 3, 2,  25.00),
(11, 7, 5, 10, 3.50),
(12, 8, 1, 3,  15.00),
(13, 8, 6, 1,  8.00),
(14, 9, 4, 2,  120.00),
(15, 10, 2, 2, 45.00),
(16, 10, 5, 5, 3.50),
(17, 11, 1, 1, 15.00),
(18, 12, 3, 1, 25.00),
(19, 12, 6, 4, 8.00);
```

**Ringkasan isi dataset:**

- **7 customer** — satu di antaranya, **Hassan Ali**, belum pernah bikin order sama sekali
- **7 produk** — satu di antaranya, **Desk Lamp**, belum pernah terjual
- **12 order** — termasuk 1 `cancelled` dan 1 `refunded` (sengaja, buat latihan filtering & data quality)
- **19 baris** `order_items`

Customer dan produk yang "kosong" itu sengaja ditaruh — nanti jadi bahan utama latihan `LEFT`/`RIGHT`/`FULL JOIN` di Hari 3, karena baris seperti ini cuma muncul di satu sisi JOIN.

Ukurannya sengaja dijaga pas: cukup kecil untuk dihitung manual saat verifikasi jawaban, tapi cukup besar untuk menghasilkan JOIN/GROUP BY/window function yang bermakna.

## Struktur Modul

| File | Sesuai Jadwal di `minggu_1.md` | Topik |
|---|---|---|
| [`hari_1_select_where.md`](hari_1_select_where.md) | Senin, 2 jam | Database relasional, SELECT, WHERE, ORDER BY, LIMIT |
| [`hari_2_agregasi_groupby.md`](hari_2_agregasi_groupby.md) | Selasa, 2 jam | COUNT/SUM/AVG, GROUP BY, HAVING |
| [`hari_3_join.md`](hari_3_join.md) | Rabu, 2 jam | INNER/LEFT/RIGHT/FULL JOIN |
| [`hari_4_subquery_cte.md`](hari_4_subquery_cte.md) | Kamis, 2 jam | Subquery & CTE (WITH) |
| [`hari_5_window_function.md`](hari_5_window_function.md) | Jumat, 2 jam | ROW_NUMBER, RANK, LAG/LEAD |
| [`latihan_studi_kasus.md`](latihan_studi_kasus.md) | Sabtu (4 jam) + Minggu (4 jam) | Studi kasus gabungan + mini project pemanasan + EXPLAIN/indexing dasar |

Tiap file `hari_X` punya struktur yang sama:

1. **Tujuan Belajar** — checklist target hari itu
2. **Untuk Instruktur** — mindset shift & pertanyaan yang biasanya muncul dari developer
3. **Konsep & Sintaks** — penjelasan + analogi kode
4. **Contoh Query** — pakai dataset di atas, lengkap dengan output
5. **Kesalahan Umum** — jebakan yang sering bikin developer stuck
6. **Latihan** — soal makin sulit
7. **Kunci Jawaban & Pembahasan**

## Catatan Cara Mengajar (Instructor Notes Umum)

- **Live coding, bukan slide.** Buka client SQL, ketik query bareng-bareng, biarkan error muncul dan dibahas — developer belajar SQL paling cepat lewat trial-error di terminal/GUI, sama seperti mereka belajar bahasa pemrograman baru.
- **Selalu tanya balik**: "kalau ini array of objects di JS, gimana kamu nulis logic yang sama?" sebelum kasih jawaban SQL-nya. Ini memaksa mereka connect ke pengetahuan lama.
- **Jangan skip urutan eksekusi logis** (`FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT`) — ini dijelaskan detail di `hari_1_select_where.md` dan jadi rujukan terus di hari-hari berikutnya. Developer yang paham ini akan jauh lebih jarang bingung soal "kenapa alias ini gak bisa dipakai di WHERE".
- **Total waktu**: 5 hari x 2 jam (materi + live coding) + Sabtu 4 jam (studi kasus) + Minggu 4 jam (review + mini project pemanasan) = 18 jam, sesuai kapasitas di `minggu_1.md`.
