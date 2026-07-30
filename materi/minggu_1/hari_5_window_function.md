# Hari 5 — Window Functions (ROW_NUMBER, RANK, LAG/LEAD)

*Jumat, 2 jam. Dataset & setup: lihat `00_overview.md`.*

> Materi ini yang paling langsung dipakai di mini project Minggu 2 (Top N produk, RFM analysis) — tekankan ke peserta bahwa ini bukan "materi ekstra", tapi tools yang akan mereka pakai tiap minggu ke depan.

## Tujuan Belajar

- [ ] Menjelaskan beda fundamental window function vs `GROUP BY`
- [ ] Memakai `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()` dan menjelaskan bedanya saat ada nilai kembar (tie)
- [ ] Memakai `PARTITION BY` untuk ranking per grup tanpa collapse rows
- [ ] Memakai `LAG`/`LEAD` untuk perbandingan antar baris (mis. growth month-over-month)
- [ ] Memakai `SUM() OVER (...)` untuk running total
- [ ] Tahu kenapa window function tidak bisa langsung difilter di `WHERE`

## Untuk Instruktur: Mindset Shift

Ini kesalahpahaman paling umum: developer yang baru kenal window function sering mengira ini "`GROUP BY` versi lain". **Bedanya fundamental:**

- `GROUP BY` → **mengecilkan** N baris jadi 1 baris per grup (collapse).
- Window function → **tetap N baris**, tapi tiap baris dapat tambahan kolom hasil hitungan "melihat ke sekeliling"-nya (baris lain di grup/urutan yang sama).

Analogi kode paling pas:

```js
// GROUP BY ≈ ini: hasilnya mengecil jadi 1 entry per key
const totals = groupBy(orders, 'customer_id'); // { 1: [...], 2: [...] }

// Window function ≈ ini: array asli TETAP utuh, tiap elemen ditambah field baru
const withRank = orders
  .sort((a, b) => b.revenue - a.revenue)
  .map((order, index) => ({ ...order, rank: index + 1 }));
```

Kalimat kunci untuk diulang ke peserta: **"window function itu kayak `GROUP BY` yang hasilnya di-JOIN balik ke tiap baris asal, otomatis."** Sebelum ada window function, orang biasanya memang literally melakukan itu: `GROUP BY` di subquery, lalu `JOIN` balik ke tabel asal — window function menggantikan pola itu jadi satu langkah.

## Konsep & Sintaks

```sql
function_name() OVER (
    [PARTITION BY column]   -- opsional: bagi jadi grup-grup terpisah
    [ORDER BY column]       -- urutan dalam tiap grup
)
```

`PARTITION BY` ≈ `GROUP BY`-nya window function, tapi tidak meng-collapse baris. Tanpa `PARTITION BY`, seluruh tabel dianggap 1 partisi besar.

### Fungsi Ranking

| Fungsi | Perilaku saat ada nilai kembar (tie) |
|---|---|
| `ROW_NUMBER()` | Selalu unik, urut 1,2,3,4,... — kalau ada tie, urutan di antara yang kembar **arbitrer** kecuali `ORDER BY` sudah unik |
| `RANK()` | Nilai kembar dapat rank sama, lalu **melompat** (1,2,2,4) |
| `DENSE_RANK()` | Nilai kembar dapat rank sama, **tidak melompat** (1,2,2,3) |

### Fungsi Antar-Baris

| Fungsi | Kegunaan |
|---|---|
| `LAG(column)` | Nilai dari baris **sebelumnya** (dalam urutan `ORDER BY`) |
| `LEAD(column)` | Nilai dari baris **berikutnya** |
| `SUM(column) OVER (ORDER BY ...)` | Running total (default frame: dari awal partisi sampai baris ini) |

## Contoh Query

**1. `ROW_NUMBER` vs `RANK` vs `DENSE_RANK` — ranking produk by total quantity terjual**
```sql
SELECT p.product_name,
       COALESCE(SUM(oi.quantity), 0) AS total_qty,
       ROW_NUMBER() OVER (ORDER BY COALESCE(SUM(oi.quantity), 0) DESC, p.product_id) AS row_num,
       RANK()       OVER (ORDER BY COALESCE(SUM(oi.quantity), 0) DESC) AS rnk,
       DENSE_RANK() OVER (ORDER BY COALESCE(SUM(oi.quantity), 0) DESC) AS dense_rnk
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name
ORDER BY total_qty DESC;
```
| product_name | total_qty | row_num | rnk | dense_rnk |
|---|---|---|---|---|
| Notebook A5 | 18 | 1 | 1 | 1 |
| Coffee Mug | 8 | 2 | 2 | 2 |
| Wireless Mouse | 7 | 3 | 3 | 3 |
| Mechanical Keyboard | 4 | 4 | 4 | 4 |
| USB-C Hub | 4 | 5 | 4 | 4 |
| Office Chair | 3 | 6 | 6 | 5 |
| Desk Lamp | 0 | 7 | 7 | 6 |

Perhatikan **Mechanical Keyboard vs USB-C Hub** (kembar, sama-sama 4): `RANK()` kasih mereka rank 4 berdua lalu **lompat ke 6** untuk Office Chair. `DENSE_RANK()` kasih rank 4 berdua lalu **lanjut ke 5** (tidak lompat). `ROW_NUMBER()` tetap kasih angka unik (4 dan 5) — makanya butuh `p.product_id` ditambahkan di `ORDER BY` supaya urutan di antara yang kembar deterministik, bukan acak.

**2. `PARTITION BY` — ranking produk by revenue, per kategori**
```sql
WITH product_revenue AS (
    SELECT p.category, p.product_name,
           COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS revenue
    FROM products p
    LEFT JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY p.category, p.product_name
)
SELECT category, product_name, revenue,
       RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS rank_in_category
FROM product_revenue
ORDER BY category, rank_in_category;
```
| category | product_name | revenue | rank_in_category |
|---|---|---|---|
| Electronics | Mechanical Keyboard | 180.00 | 1 |
| Electronics | Wireless Mouse | 90.00 | 2 |
| Electronics | USB-C Hub | 50.00 | 3 |
| Furniture | Office Chair | 360.00 | 1 |
| Furniture | Desk Lamp | 0.00 | 2 |
| Kitchen | Coffee Mug | 64.00 | 1 |
| Stationery | Notebook A5 | 63.00 | 1 |

Ranking-nya **reset di tiap kategori** — inilah kekuatan `PARTITION BY`: satu query, ranking per grup, tanpa perlu loop per kategori atau query terpisah-pisah.

**3. `LAG` — growth revenue month-over-month**
```sql
WITH monthly_revenue AS (
    SELECT DATE_TRUNC('month', o.order_date) AS month,
           SUM(oi.quantity * oi.unit_price) AS revenue
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status = 'completed'
    GROUP BY 1
)
SELECT month, revenue,
       LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
       ROUND(
         100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
         / LAG(revenue) OVER (ORDER BY month), 1
       ) AS growth_pct
FROM monthly_revenue
ORDER BY month;
```
| month | revenue | prev_month_revenue | growth_pct |
|---|---|---|---|
| 2024-01-01 | 126.50 | NULL | NULL |
| 2024-02-01 | 223.00 | 126.50 | 76.3 |
| 2024-03-01 | 457.50 | 223.00 | 105.2 |

Baris pertama selalu `NULL` untuk `LAG` — tidak ada "bulan sebelumnya" untuk data paling awal. Ini normal, bukan bug.

**4. Running total dengan `SUM() OVER`**
```sql
SELECT month, revenue,
       SUM(revenue) OVER (ORDER BY month) AS cumulative_revenue
FROM monthly_revenue
ORDER BY month;
```
| month | revenue | cumulative_revenue |
|---|---|---|
| 2024-01-01 | 126.50 | 126.50 |
| 2024-02-01 | 223.00 | 349.50 |
| 2024-03-01 | 457.50 | 807.00 |

**Gotcha yang wajib dibahas**: kalau `OVER (ORDER BY month)` **tanpa** `PARTITION BY`, default frame-nya adalah "dari baris pertama sampai baris sekarang" — makanya hasilnya running total, **bukan** total keseluruhan. Ini beda dari ekspektasi banyak orang yang baru pindah dari `GROUP BY` (di mana `SUM` selalu berarti "total semua"). Kalau memang mau total keseluruhan tanpa collapse, hilangkan `ORDER BY` di dalam `OVER()`: `SUM(revenue) OVER ()`.

## Kesalahan Umum

1. **Window function tidak bisa langsung dipakai di `WHERE` atau `HAVING`.** Ingat urutan eksekusi: window function dihitung **sebagai bagian dari `SELECT`**, setelah `WHERE`/`GROUP BY`/`HAVING` selesai. Jadi untuk filter berdasarkan hasil window function (misal "cuma ambil rank 1 tiap kategori"), **wajib** dibungkus subquery/CTE dulu:
   ```sql
   -- SALAH — akan error:
   SELECT category, product_name,
          RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS rnk
   FROM product_revenue
   WHERE rnk = 1;

   -- BENAR — bungkus dengan CTE dulu:
   WITH ranked AS (
       SELECT category, product_name,
              RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS rnk
       FROM product_revenue
   )
   SELECT * FROM ranked WHERE rnk = 1;
   ```
   Pola **"Top-N per grup"** ini adalah salah satu penggunaan window function paling umum di data engineering (dan persis yang akan dipakai di mini project Minggu 2 untuk "Top 10 produk terlaris").

2. **Lupa `PARTITION BY` padahal maksudnya ranking per grup.** Tanpa `PARTITION BY`, ranking dihitung untuk **seluruh tabel** sebagai satu grup besar — hasil "Top 1 tiap kategori" jadi salah kalau lupa ini.

3. **`ORDER BY` di dalam `OVER()` yang tidak deterministik saat ada tie.** Kalau `ORDER BY` yang dipakai bisa punya nilai kembar dan kamu pakai `ROW_NUMBER()`, tambahkan kolom tiebreaker (biasanya primary key) supaya hasil query konsisten tiap dijalankan ulang — lihat contoh #1.

4. **Ketuker `RANK()` dan `DENSE_RANK()` saat butuh "top N" yang eksak.** Kalau ada 2 produk rank 1 (tie) dan kamu filter `rnk <= 3` pakai `RANK()`, produk berikutnya yang "seharusnya" rank 2 malah jadi rank 3 (lompat) — kalau maksudnya benar-benar "3 kelompok nilai teratas", `DENSE_RANK()` biasanya yang dimaksud.

## Latihan

1. Ranking customer berdasarkan total revenue (dari order completed), pakai `RANK()`. Tampilkan `customer_name`, `total_revenue`, `rank`.
2. Cari produk dengan revenue tertinggi **per kategori** (Top 1 per kategori) — pakai pola CTE + `WHERE rnk = 1` dari bagian "Kesalahan Umum" #1.
3. Untuk tiap customer, urutkan order mereka berdasarkan `order_date`, lalu hitung selisih hari antara satu order dengan order berikutnya dari customer yang sama (petunjuk: `LEAD(order_date) OVER (PARTITION BY customer_id ORDER BY order_date)`, lalu kurangi tanggalnya).
4. Hitung persentase kontribusi tiap produk terhadap total revenue keseluruhan (petunjuk: `SUM(revenue) OVER ()` tanpa `ORDER BY` untuk dapat total keseluruhan sebagai kolom tambahan di tiap baris).
5. Ambil 2 produk dengan quantity terjual tertinggi di tiap kategori (Top 2 per kategori, bukan Top 1).

## Kunci Jawaban & Pembahasan

**1.**
```sql
WITH customer_revenue AS (
    SELECT c.customer_id, c.customer_name,
           SUM(oi.quantity * oi.unit_price) AS total_revenue
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status = 'completed'
    GROUP BY c.customer_id, c.customer_name
)
SELECT customer_name, total_revenue,
       RANK() OVER (ORDER BY total_revenue DESC) AS rank
FROM customer_revenue
ORDER BY rank;
```
Hasil: Grace Tan (240, rank 1), Citra Lestari (235.5, rank 2), Andi Wijaya (118.5, rank 3), Budi Santoso (118, rank 4), Emma Watson (60, rank 5), Frank Muller (35, rank 6). Catatan: Hassan Ali tidak muncul karena `JOIN` (bukan `LEFT JOIN`) — kalau mau dia muncul dengan revenue 0, ganti jadi `LEFT JOIN` seperti pola di contoh #1 file ini.

**2.**
```sql
WITH product_revenue AS (
    SELECT p.category, p.product_name,
           COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS revenue
    FROM products p
    LEFT JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY p.category, p.product_name
),
ranked AS (
    SELECT *, RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS rnk
    FROM product_revenue
)
SELECT category, product_name, revenue
FROM ranked
WHERE rnk = 1;
```
Hasil: Mechanical Keyboard (Electronics, 180), Office Chair (Furniture, 360), Coffee Mug (Kitchen, 64), Notebook A5 (Stationery, 63).

**3.**
```sql
SELECT c.customer_name, o.order_id, o.order_date,
       LEAD(o.order_date) OVER (PARTITION BY o.customer_id ORDER BY o.order_date) AS next_order_date,
       LEAD(o.order_date) OVER (PARTITION BY o.customer_id ORDER BY o.order_date) - o.order_date AS days_until_next
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
ORDER BY c.customer_name, o.order_date;
```
Contoh hasil Andi Wijaya: order 2024-01-05 → next 2024-01-20 (15 hari) → next 2024-03-01 (41 hari) → next `NULL` (order terakhir dia, tidak ada order berikutnya). Ini pola dasar untuk analisis **customer purchase interval**, sering dipakai untuk deteksi churn.

**4.**
```sql
WITH product_revenue AS (
    SELECT p.product_name,
           COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS revenue
    FROM products p
    LEFT JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY p.product_name
)
SELECT product_name, revenue,
       SUM(revenue) OVER () AS total_all_products,
       ROUND(100.0 * revenue / SUM(revenue) OVER (), 1) AS pct_of_total
FROM product_revenue
ORDER BY revenue DESC;
```
Hasil: Office Chair berkontribusi 360/807 ≈ 44.6%, Mechanical Keyboard 180/807 ≈ 22.3%, dst. — total semua baris `pct_of_total` harus berjumlah 100%.

**5.**
```sql
WITH product_qty AS (
    SELECT p.category, p.product_name,
           COALESCE(SUM(oi.quantity), 0) AS total_qty
    FROM products p
    LEFT JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY p.category, p.product_name
),
ranked AS (
    SELECT *, DENSE_RANK() OVER (PARTITION BY category ORDER BY total_qty DESC) AS rnk
    FROM product_qty
)
SELECT category, product_name, total_qty
FROM ranked
WHERE rnk <= 2
ORDER BY category, total_qty DESC;
```
Dipakai `DENSE_RANK()`, bukan `RANK()` — kalau kategori Electronics punya tie di posisi kedua (Mechanical Keyboard & USB-C Hub sama-sama qty 4), `DENSE_RANK() <= 2` akan mengambil **keduanya** (karena secara nilai, mereka sama-sama "kelompok tertinggi kedua"), sementara `RANK() <= 2` juga kebetulan mengambil keduanya di kasus ini — tapi diskusikan dengan peserta: kalau ada 3 produk tie di rank 1, `RANK() <= 2` akan mengembalikan 0 baris tambahan di rank "2" (karena rank 2 dilompati jadi rank 4), sedangkan `DENSE_RANK() <= 2` tetap konsisten ambil grup nilai tertinggi kedua. Ini alasan `DENSE_RANK()` biasanya pilihan lebih aman untuk kebutuhan "Top N per grup".
