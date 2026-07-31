---
title: Hari 4 - Subquery & CTE
parent: Minggu 1 - SQL Fundamentals
nav_order: 5
---

# Hari 4 — Subquery & CTE (WITH clause)

*Kamis, 2 jam. Dataset & setup: lihat `00_overview.md`.*

## Tujuan Belajar

- [ ] Menulis subquery scalar, `IN`, dan `EXISTS`/`NOT EXISTS`
- [ ] Membedakan subquery biasa vs **correlated subquery**, dan tahu implikasi performanya
- [ ] Menulis CTE (`WITH`) untuk memecah query kompleks jadi langkah-langkah yang terbaca
- [ ] Menghindari jebakan `NOT IN` + `NULL`

## Untuk Instruktur: Mindset Shift

Subquery ≈ **nested function call**. CTE ≈ **variabel/helper function bernama** yang mewakili hasil intermediate — persis alasan developer menulis `const activeUsers = ...` alih-alih menumpuk semua logic dalam satu ekspresi raksasa.

```js
// Nested (subquery-style): sulit dibaca
const result = getTopSpenders(filterActive(getAllOrders()));

// Dipecah dengan nama (CTE-style): lebih terbaca
const allOrders = getAllOrders();
const activeOrders = filterActive(allOrders);
const result = getTopSpenders(activeOrders);
```

Sama seperti developer disarankan extract nested expression jadi variabel bernama untuk readability, **CTE ada terutama untuk readability**, bukan untuk fitur baru — hampir semua yang bisa ditulis pakai CTE, bisa juga ditulis pakai subquery bersarang. Bedanya keterbacaan.

**Poin penting soal correlated subquery**: ini analog persis dengan **masalah N+1 query** yang mungkin sudah familiar dari kerja dengan ORM (Rails/Django/Sequelize) — subquery yang merujuk ke tabel luar (`WHERE o.customer_id = c.customer_id`) dieksekusi **ulang untuk tiap baris** di query luar. Kalau developer pernah kena masalah N+1 di ORM, mereka akan langsung ngeh soal implikasi performanya di sini.

## Konsep & Sintaks

### 1. Scalar Subquery — subquery yang hasilnya 1 nilai
```sql
SELECT ... WHERE column > (SELECT AVG(column) FROM table);
```

### 2. `IN` Subquery — subquery yang hasilnya list nilai
```sql
SELECT ... WHERE column IN (SELECT column FROM table WHERE ...);
```

### 3. `EXISTS` / `NOT EXISTS` — cek keberadaan, bukan ambil nilai
```sql
SELECT ... WHERE EXISTS (SELECT 1 FROM table WHERE correlated_condition);
```
`EXISTS` cuma peduli **ada/tidaknya baris**, jadi konvensinya pakai `SELECT 1` (angka apapun, gak dipakai) — bukan `SELECT *`.

### 4. Correlated Subquery — subquery yang "menyentuh" query luar
Subquery **correlated** kalau di dalamnya ada referensi ke tabel dari query luar (mis. `WHERE o.customer_id = c.customer_id` dengan `c` didefinisikan di query luar). Ini beda dari subquery biasa yang bisa dijalankan berdiri sendiri tanpa tahu apa-apa soal query luar.

### 5. CTE (`WITH`)
```sql
WITH nama_cte AS (
    SELECT ...
)
SELECT ... FROM nama_cte ...;
```
Bisa lebih dari satu CTE, dan CTE berikutnya boleh pakai CTE sebelumnya:
```sql
WITH cte_a AS (SELECT ...),
     cte_b AS (SELECT ... FROM cte_a ...)
SELECT ... FROM cte_b;
```

## Contoh Query

**1. Scalar subquery — produk di atas harga rata-rata**
```sql
SELECT product_name, unit_price
FROM products
WHERE unit_price > (SELECT AVG(unit_price) FROM products);
```
Rata-rata harga 7 produk ≈ 35.21. Hasil: Mechanical Keyboard (45.00), Office Chair (120.00).

**2. `IN` subquery — customer yang punya minimal 1 order completed**
```sql
SELECT customer_name
FROM customers
WHERE customer_id IN (SELECT customer_id FROM orders WHERE status = 'completed');
```
Hasil: Andi, Budi, Citra, Emma, Frank, Grace (6 dari 7 — Hassan Ali otomatis tidak ikut karena tidak pernah ada di tabel `orders` sama sekali).

**3. `NOT EXISTS` — produk yang belum pernah terjual (bandingkan dengan LEFT JOIN di Hari 3)**
```sql
SELECT product_name
FROM products p
WHERE NOT EXISTS (
    SELECT 1 FROM order_items oi WHERE oi.product_id = p.product_id
);
```
Hasil: Desk Lamp — **sama persis** dengan hasil query `LEFT JOIN ... WHERE ... IS NULL` di Hari 3. Ini sengaja: tunjukkan ke peserta bahwa banyak masalah SQL bisa diselesaikan dengan beberapa pendekatan berbeda (JOIN vs subquery vs EXISTS), dan bagian dari jadi mahir adalah tahu kapan tiap pendekatan lebih terbaca/lebih cepat.

**4. Correlated subquery — order terakhir tiap customer**
```sql
SELECT c.customer_name,
       (SELECT MAX(o.order_date) FROM orders o WHERE o.customer_id = c.customer_id) AS last_order_date
FROM customers c;
```
| customer_name | last_order_date |
|---|---|
| Andi Wijaya | 2024-03-01 |
| Budi Santoso | 2024-03-25 |
| Citra Lestari | 2024-03-12 |
| Emma Watson | 2024-03-20 |
| Frank Muller | 2024-02-18 |
| Grace Tan | 2024-03-05 |
| Hassan Ali | NULL |

Subquery ini dieksekusi ulang untuk **tiap baris** `customers` (7 kali) — untuk dataset kecil ini tidak masalah, tapi di tabel jutaan baris, pola ini bisa jadi sangat lambat. Ini akan dibahas lagi di `latihan_studi_kasus.md` soal `EXPLAIN`.

**5. Subquery di `FROM` (derived table) — revenue per bulan**
```sql
SELECT month, revenue
FROM (
    SELECT DATE_TRUNC('month', o.order_date) AS month,
           SUM(oi.quantity * oi.unit_price) AS revenue
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status = 'completed'
    GROUP BY 1
) AS monthly
ORDER BY month;
```

**6. Query yang sama, ditulis ulang pakai CTE — bandingkan keterbacaannya**
```sql
WITH monthly_revenue AS (
    SELECT DATE_TRUNC('month', o.order_date) AS month,
           SUM(oi.quantity * oi.unit_price) AS revenue
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status = 'completed'
    GROUP BY 1
)
SELECT * FROM monthly_revenue ORDER BY month;
```
| month | revenue |
|---|---|
| 2024-01-01 | 126.50 |
| 2024-02-01 | 223.00 |
| 2024-03-01 | 457.50 |

Hasilnya identik dengan #5 — bedanya murni keterbacaan. Tekankan ke peserta: begitu query mulai butuh subquery di `FROM`, **hampir selalu lebih baik ditulis sebagai CTE.**

**7. Chained CTE — customer dengan revenue di atas rata-rata**
```sql
WITH customer_revenue AS (
    SELECT c.customer_id, c.customer_name,
           SUM(oi.quantity * oi.unit_price) AS total_revenue
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status = 'completed'
    GROUP BY c.customer_id, c.customer_name
),
avg_revenue AS (
    SELECT AVG(total_revenue) AS avg_rev FROM customer_revenue
)
SELECT cr.customer_name, cr.total_revenue
FROM customer_revenue cr, avg_revenue ar
WHERE cr.total_revenue > ar.avg_rev
ORDER BY cr.total_revenue DESC;
```
Rata-rata revenue 6 customer aktif = 807 / 6 = 134.5. Hasil: Grace Tan (240.00), Citra Lestari (235.50) — keduanya di atas rata-rata.

## Kesalahan Umum

1. **`NOT IN` + subquery yang bisa menghasilkan `NULL` = jebakan diam-diam mengembalikan hasil kosong.** Kalau subquery di dalam `NOT IN` menghasilkan **satu saja** nilai `NULL`, seluruh kondisi `NOT IN` jadi `UNKNOWN` untuk semua baris, dan query mengembalikan **nol baris** — tanpa error apapun.
   ```sql
   -- BAHAYA kalau order_items.product_id bisa NULL:
   SELECT * FROM products
   WHERE product_id NOT IN (SELECT product_id FROM order_items);

   -- AMAN: NOT EXISTS tidak kena masalah ini
   SELECT * FROM products p
   WHERE NOT EXISTS (SELECT 1 FROM order_items oi WHERE oi.product_id = p.product_id);
   ```
   Rule of thumb yang aman dipakai terus: **default ke `NOT EXISTS`, bukan `NOT IN`**, kecuali yakin 100% kolom di subquery tidak pernah `NULL`.

2. **Scalar subquery di `SELECT` untuk tiap baris = potensi masalah N+1.** Kalau query punya banyak scalar subquery ter-correlated di `SELECT` (bukan cuma 1), performanya bisa jauh lebih lambat dibanding `JOIN` + `GROUP BY` yang setara. Kalau ada pilihan, biasanya `JOIN` lebih cepat daripada correlated subquery untuk kasus yang sama.

3. **CTE bukan jaminan "dihitung sekali lalu di-cache".** Di PostgreSQL modern (versi 12+), CTE non-recursive **bisa di-inline** oleh query planner (diperlakukan seperti subquery biasa demi optimasi) — beda dari versi lama Postgres yang selalu materialize CTE apa adanya. Jangan asumsikan CTE otomatis lebih cepat; itu murni alat keterbacaan, bukan alat optimasi (kecuali eksplisit pakai `MATERIALIZED`).

## Latihan

1. Cari produk dengan harga di bawah rata-rata harga produk di kategorinya sendiri (petunjuk: perlu correlated subquery, bandingkan `p.unit_price` dengan rata-rata `WHERE category = p.category`).
2. Tulis ulang soal #1 di atas memakai CTE, bandingkan mana yang lebih gampang dibaca.
3. Cari customer yang **tidak pernah** membeli produk kategori `'Furniture'` (pakai `NOT EXISTS`).
4. Buat CTE `order_totals` (total revenue per order dari `order_items`), lalu dari CTE itu cari order dengan total revenue tertinggi per customer (petunjuk: gabungkan CTE dengan `orders` dan `customers`).
5. Jelaskan dengan kata-kata sendiri: kenapa `WHERE product_id NOT IN (SELECT product_id FROM order_items WHERE quantity IS NULL)` berisiko mengembalikan hasil kosong meskipun sebenarnya banyak produk yang seharusnya lolos filter.

## Kunci Jawaban & Pembahasan

**1.**
```sql
SELECT product_name, category, unit_price
FROM products p
WHERE unit_price < (
    SELECT AVG(unit_price) FROM products p2 WHERE p2.category = p.category
);
```
Hasil: hanya `USB-C Hub` (25.00 < rata-rata Electronics 28.33). Kategori dengan cuma 1 produk (Furniture punya 2: Office Chair & Desk Lamp — Desk Lamp 30 < rata-rata (120+30)/2=75, jadi Desk Lamp juga lolos) tidak menghasilkan produk yang lolos untuk Stationery/Kitchen karena rata-ratanya sama dengan satu-satunya nilai di kategori itu (tidak ada yang "di bawah rata-rata dirinya sendiri"). Hasil akhir: USB-C Hub, Desk Lamp.

**2.**
```sql
WITH category_avg AS (
    SELECT category, AVG(unit_price) AS avg_price
    FROM products
    GROUP BY category
)
SELECT p.product_name, p.category, p.unit_price
FROM products p
JOIN category_avg ca ON p.category = ca.category
WHERE p.unit_price < ca.avg_price;
```
Hasil sama dengan #1. Diskusikan: versi CTE ini juga lebih murah secara komputasi karena rata-rata per kategori dihitung **sekali** (via `GROUP BY`), bukan berulang tiap baris seperti correlated subquery di #1 — bonus poin soal performa, bukan cuma keterbacaan.

**3.**
```sql
SELECT c.customer_name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    WHERE o.customer_id = c.customer_id AND p.category = 'Furniture'
);
```
Hasil: Andi Wijaya, Budi Santoso, Frank Muller, Hassan Ali (Hassan otomatis lolos karena dia bahkan tidak punya order sama sekali — `NOT EXISTS` benar untuk kasus ini).

**4.**
```sql
WITH order_totals AS (
    SELECT order_id, SUM(quantity * unit_price) AS total_revenue
    FROM order_items
    GROUP BY order_id
)
SELECT c.customer_name, o.order_id, ot.total_revenue
FROM order_totals ot
JOIN orders o ON ot.order_id = o.order_id
JOIN customers c ON o.customer_id = c.customer_id
WHERE (c.customer_id, ot.total_revenue) IN (
    SELECT o2.customer_id, MAX(ot2.total_revenue)
    FROM order_totals ot2
    JOIN orders o2 ON ot2.order_id = o2.order_id
    GROUP BY o2.customer_id
)
ORDER BY ot.total_revenue DESC;
```
Ini soal yang cukup menantang secara sengaja — kombinasi CTE + subquery `IN` dengan tuple. Instruktur: ini kesempatan bagus untuk preview window function (`RANK() OVER (PARTITION BY customer_id ORDER BY total_revenue DESC)`) sebagai cara yang **jauh lebih bersih** untuk soal seperti ini — jembatan sempurna ke materi Hari 5 besok.

**5.**
Kalau ada **satu saja** baris di `order_items` dengan `quantity IS NULL`, maka subquery `SELECT product_id FROM order_items WHERE quantity IS NULL` akan menghasilkan `product_id` tersebut. Tapi masalahnya lebih dalam: kalau kolom yang dipakai di subquery (di soal ini `product_id`-nya sendiri, dalam skenario lain) mengandung `NULL` di salah satu barisnya, `NOT IN` akan membandingkan tiap `product_id` di tabel luar dengan `NULL` itu — dan `x = NULL` selalu `UNKNOWN`, bukan `TRUE`/`FALSE`. Karena `NOT IN` secara logika berarti "AND dari semua NOT EQUAL", begitu salah satu perbandingan itu `UNKNOWN`, hasil keseluruhan gak pernah bisa jadi pasti `TRUE` — jadi seluruh `WHERE` mengembalikan `UNKNOWN` (dianggap gagal filter) untuk **setiap** baris, meski secara logis banyak yang seharusnya lolos.
