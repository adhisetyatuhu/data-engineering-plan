# Sabtu–Minggu — Studi Kasus, Mini Project Pemanasan, & Dasar Query Optimization

*Sabtu 4 jam + Minggu 4 jam. Dataset & setup: lihat `00_overview.md`.*

Akhir pekan ini punya 3 bagian:
1. **Sabtu (4 jam)** — 10 studi kasus yang menggabungkan **semua** topik Senin–Jumat, plus latihan tambahan di platform eksternal.
2. **Minggu, sesi 1 (2 jam)** — dasar query optimization: `EXPLAIN` dan indexing.
3. **Minggu, sesi 2 (2 jam)** — mini project pemanasan: tulis 5–10 query kompleks sendiri sebagai deliverable pertama.

## Tujuan Belajar

- [ ] Bisa mengombinasikan `SELECT`/`WHERE`/`GROUP BY`/`JOIN`/`CTE`/window function dalam satu query tanpa lihat referensi
- [ ] Bisa membaca output `EXPLAIN` dasar dan menjelaskan kapan index membantu
- [ ] Menghasilkan file `queries.sql` mandiri berisi 5–10 query yang menjawab pertanyaan bisnis nyata

## Untuk Instruktur

Ini sesi **checkpoint**, bukan materi baru. Kalau ada peserta yang stuck di studi kasus, itu sinyal untuk balik ke modul hari terkait, bukan tanda peserta "gagal". Cara terbaik menjalankan sesi ini: **pair programming / code review antar peserta** — minta mereka saling review query satu sama lain sebelum lihat kunci jawaban. Developer sudah terbiasa dengan budaya code review, manfaatkan itu; mereka akan lebih tajam mengkritik query orang lain daripada query sendiri, dan itu proses belajar yang efektif.

---

## Bagian 1 (Sabtu): Studi Kasus Gabungan

Kerjakan berurutan — makin ke bawah makin menggabungkan banyak konsep. Semua pakai dataset di `00_overview.md`.

1. Tim marketing mau tahu **5 customer dengan jumlah order completed terbanyak**.
2. Berapa **total revenue per bulan** (order completed saja)? Urutkan berdasarkan bulan.
3. Tampilkan **semua produk** beserta kategori dan total quantity terjual — termasuk yang belum pernah terjual, urutkan dari quantity terbanyak.
4. Cari customer yang **pernah order** tapi **belum pernah** beli produk kategori `'Electronics'`.
5. Hitung **rata-rata nilai order** (Average Order Value / AOV) dari semua order completed, lalu tampilkan order mana saja yang nilainya **di atas** AOV itu.
6. Beri **rank** tiap customer berdasarkan total revenue mereka (order completed).
7. Cari **1 produk dengan revenue tertinggi di tiap kategori**.
8. Hitung **growth revenue bulanan (%)** dari bulan ke bulan.
9. Cari order-order (completed) yang total revenue-nya **lebih tinggi** dari rata-rata revenue seluruh order completed.
10. Gabungkan semuanya: buat 1 query "laporan ringkas per customer" yang menampilkan `customer_name`, total order, total revenue, rank revenue, dan tanggal order terakhir.

### Latihan Tambahan (Eksternal)

Setelah 10 soal di atas, lanjutkan jam belajar dengan platform latihan eksternal supaya terbiasa dengan skema data yang berbeda-beda (bukan cuma skema kita):
- **SQLZoo** atau **Mode Analytics SQL Tutorial** — kalau masih ingin pemanasan sebelum ke soal lebih sulit
- **StrataScratch** atau **LeetCode SQL** — cari kategori "Medium" untuk topik JOIN, subquery, window function
- Dataset publik: Kaggle "Superstore Sales", "Northwind DB"

### Kunci Jawaban & Pembahasan

**1.**
```sql
SELECT c.customer_name, COUNT(o.order_id) AS jumlah_order
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed'
GROUP BY c.customer_name
ORDER BY jumlah_order DESC
LIMIT 5;
```
Hasil: Andi Wijaya (3), lalu Budi Santoso/Citra Lestari (2, tie), lalu Emma Watson/Frank Muller/Grace Tan (1, tie — hanya 2 dari 3 yang kebagian slot `LIMIT 5`). Bahas ke peserta: ini kasus nyata kenapa `LIMIT` tanpa tiebreaker eksplisit di `ORDER BY` bisa ambigu — baris mana yang "beruntung" masuk `LIMIT 5` di antara yang nilainya sama tidak dijamin konsisten antar run, sama seperti dibahas soal `ROW_NUMBER` di Hari 5.

**2.**
```sql
SELECT DATE_TRUNC('month', o.order_date) AS month,
       SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.status = 'completed'
GROUP BY 1
ORDER BY 1;
```
Hasil: Jan 126.50, Feb 223.00, Mar 457.50 (sama seperti contoh di Hari 4 & 5).

**3.**
```sql
SELECT p.product_name, p.category, COALESCE(SUM(oi.quantity), 0) AS total_qty
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_name, p.category
ORDER BY total_qty DESC;
```
Hasil: Notebook A5 (18) ... Desk Lamp (0) di paling bawah (sama seperti tabel di Hari 5 contoh #1).

**4.**
```sql
SELECT DISTINCT c.customer_name
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o2
    JOIN order_items oi ON o2.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    WHERE o2.customer_id = c.customer_id AND p.category = 'Electronics'
);
```
Hasil: Frank Muller (order dia isinya Notebook A5, kategori Stationery — tidak pernah beli Electronics). Perhatikan `JOIN orders o` di baris awal memastikan Hassan Ali (yang memang tidak pernah order sama sekali) **tidak** ikut — soal ini spesifik minta "pernah order tapi belum pernah beli Electronics", beda dari soal `NOT EXISTS` di Hari 4 yang tidak mensyaratkan pernah order.

**5.**
```sql
WITH order_totals AS (
    SELECT o.order_id, SUM(oi.quantity * oi.unit_price) AS order_revenue
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status = 'completed'
    GROUP BY o.order_id
),
aov AS (
    SELECT AVG(order_revenue) AS avg_order_value FROM order_totals
)
SELECT ot.order_id, ot.order_revenue
FROM order_totals ot, aov
WHERE ot.order_revenue > aov.avg_order_value
ORDER BY ot.order_revenue DESC;
```
AOV = 807 / 10 order completed = 80.7. Order di atas AOV: order 9 (240), order 4 (128), order 10 (107.5).

**6.**
```sql
WITH customer_revenue AS (
    SELECT c.customer_name, SUM(oi.quantity * oi.unit_price) AS total_revenue
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status = 'completed'
    GROUP BY c.customer_name
)
SELECT customer_name, total_revenue,
       RANK() OVER (ORDER BY total_revenue DESC) AS rank
FROM customer_revenue;
```
Sama seperti jawaban Latihan #1 di `hari_5_window_function.md`.

**7.** Sama seperti jawaban Latihan #2 di `hari_5_window_function.md`.

**8.** Sama seperti contoh #3 di `hari_5_window_function.md`.

**9.**
```sql
WITH order_totals AS (
    SELECT o.order_id, SUM(oi.quantity * oi.unit_price) AS order_revenue
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status = 'completed'
    GROUP BY o.order_id
)
SELECT order_id, order_revenue
FROM order_totals
WHERE order_revenue > (SELECT AVG(order_revenue) FROM order_totals)
ORDER BY order_revenue DESC;
```
Hasil identik dengan soal #5 (dua cara berbeda untuk pertanyaan yang sama — sengaja, untuk menunjukkan CTE+cross-join vs scalar-subquery-di-WHERE bisa mencapai hasil sama).

**10.**
```sql
WITH order_totals AS (
    SELECT o.customer_id, o.order_id, SUM(oi.quantity * oi.unit_price) AS order_revenue
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status = 'completed'
    GROUP BY o.customer_id, o.order_id
),
customer_summary AS (
    SELECT c.customer_id, c.customer_name,
           COUNT(ot.order_id) AS jumlah_order,
           COALESCE(SUM(ot.order_revenue), 0) AS total_revenue,
           MAX(o_all.order_date) AS last_order_date
    FROM customers c
    LEFT JOIN order_totals ot ON c.customer_id = ot.customer_id
    LEFT JOIN orders o_all ON c.customer_id = o_all.customer_id
    GROUP BY c.customer_id, c.customer_name
)
SELECT customer_name, jumlah_order, total_revenue, last_order_date,
       RANK() OVER (ORDER BY total_revenue DESC) AS revenue_rank
FROM customer_summary
ORDER BY revenue_rank;
```
Ini query paling kompleks minggu ini secara sengaja — gabungan CTE berlapis, `LEFT JOIN`, agregasi, dan window function dalam satu query. Kalau peserta bisa mengurai query ini sendiri (baca dari `WITH` ke bawah, satu CTE dalam satu waktu), itu sinyal kuat target Minggu 1 tercapai. `last_order_date` sengaja dihitung dari **semua** order (bukan cuma completed) — diskusikan dengan peserta apakah itu keputusan bisnis yang tepat atau tidak (poin bagus: keputusan "order apa yang dihitung" itu keputusan bisnis, bukan cuma teknis).

---

## Bagian 2 (Minggu, sesi 1): Query Optimization Dasar — `EXPLAIN` & Indexing

### Kenapa Ini Penting Sekarang

Dataset kita di Minggu 1 sengaja kecil (belasan baris) supaya gampang dihitung manual. Tapi mulai Minggu 2 (Online Retail II, ratusan ribu baris) dan seterusnya, query yang "asal jalan" bisa jadi lambat. `EXPLAIN` adalah cara **melihat rencana eksekusi** query sebelum/saat dijalankan — mirip seperti profiler di dunia programming (kalau developer pernah pakai Chrome DevTools Performance tab atau `py-spy`, ini konsepnya mirip: cari tahu bagian mana yang mahal).

### Membaca `EXPLAIN`

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 3;
```

Contoh output (bentuknya, angka aktual tergantung mesin):
```
Seq Scan on orders  (cost=0.00..1.15 rows=2 width=24) (actual time=0.010..0.014 rows=2 loops=1)
  Filter: (customer_id = 3)
  Rows Removed by Filter: 10
Planning Time: 0.05 ms
Execution Time: 0.03 ms
```

Cara baca:
- **`Seq Scan`** = database membaca **semua baris** tabel satu-satu lalu filter (analog: `array.filter()` linear, O(n)).
- **`Index Scan`** (akan muncul kalau ada index yang relevan) = database langsung "melompat" ke baris yang cocok tanpa baca semua baris (analog: lookup di hash map / B-tree, jauh lebih cepat dari O(n) untuk tabel besar).
- **`cost=0.00..1.15`** = estimasi biaya (satuan arbitrer, bukan detik) dari planner sebelum eksekusi.
- **`actual time=...`** = waktu eksekusi **sungguhan** (hanya muncul dengan `EXPLAIN ANALYZE`, karena query benar-benar dijalankan — hati-hati pakai ini untuk `DELETE`/`UPDATE` di data production, karena efeknya nyata, bukan simulasi).
- **`rows=2`** vs **`Rows Removed by Filter: 10`** = dari 12 baris di tabel, cuma 2 yang cocok, 10 dibuang.

### Kenapa Index Tidak Selalu Dipakai (Bahkan Kalau Ada)

Di dataset kita yang cuma belasan baris, query planner **akan tetap pilih `Seq Scan`** meskipun index sudah dibuat — dan itu keputusan yang **benar**. Untuk tabel sekecil ini, overhead membaca struktur index justru lebih mahal daripada baca langsung semua baris. Ini poin penting: **index bukan "selalu lebih cepat"**, planner memilih strategi berdasarkan estimasi biaya, dan untuk tabel kecil, scan penuh sering kali memang lebih murah.

Peserta baru akan lihat `Index Scan` muncul secara alami kalau nanti bekerja dengan tabel yang jauh lebih besar (Minggu 2 dengan Online Retail II).

### Membuat Index

```sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);
```

**Aturan praktis kapan bikin index** (cukup untuk level fundamental, akan diperdalam lagi di Minggu 3 saat bahas performa pipeline):
- Kolom yang sering dipakai di `WHERE` dengan selektivitas tinggi (hasil filter jauh lebih kecil dari total baris)
- Kolom foreign key yang sering dipakai di `JOIN ... ON`
- Kolom yang sering dipakai di `ORDER BY` pada tabel besar

**Trade-off yang wajib dipahami**: index mempercepat `SELECT`, tapi **memperlambat** `INSERT`/`UPDATE`/`DELETE` (karena index juga harus di-update) dan **menambah ukuran storage**. Index bukan "gratis" — analog dengan menambah cache di aplikasi: mempercepat baca, menambah kompleksitas & cost di sisi tulis.

### Latihan Mandiri

Jalankan `EXPLAIN ANALYZE` di depan 3 query dari studi kasus Bagian 1 (pilih yang pakai `JOIN` + `GROUP BY`), baca output-nya, dan diskusikan dengan instruktur/rekan: bagian mana yang `Seq Scan`, apakah masuk akal untuk ukuran data ini.

---

## Bagian 3 (Minggu, sesi 2): Mini Project Pemanasan

> Ini **bukan** mini project utama portfolio (`ecommerce-etl-pipeline`) — itu baru dimulai Minggu 2 dengan dataset Online Retail II, lihat `minggu_2.md`. Ini pemanasan format: latihan menulis dan mendokumentasikan query kompleks secara mandiri, dari nol, tanpa dituntun soal per soal.

### Tugas

Buat file `queries.sql` (boleh di folder lokal mana saja dulu, tidak perlu repo GitHub — repo baru dibuat resmi Minggu 2) berisi **5–10 query kompleks** yang menjawab pertanyaan bisnis dari dataset di `00_overview.md`. Syarat:

- Minimal 1 query yang pakai `JOIN` 3 tabel
- Minimal 1 query yang pakai CTE (`WITH`)
- Minimal 1 query yang pakai window function
- Minimal 1 query yang secara eksplisit menangani `NULL` (`COALESCE`, `IS NULL`, atau setara)
- Tiap query diberi **komentar SQL** (`-- ...`) menjelaskan pertanyaan bisnis apa yang dijawab

Contoh pertanyaan bisnis yang bisa dipakai (boleh juga bikin pertanyaan sendiri):
- "Kategori produk apa yang paling banyak menyumbang revenue?"
- "Customer mana yang paling lama tidak order (kandidat churn)?"
- "Berapa rata-rata jumlah produk berbeda yang dibeli per order?"
- "Bulan apa dengan growth revenue tertinggi?"

### Format Deliverable

```sql
-- queries.sql

-- Q1: [pertanyaan bisnis dalam 1 kalimat]
SELECT ...

-- Q2: ...
SELECT ...
```

### Alokasi Waktu (±2 jam)

- Merumuskan 5–10 pertanyaan bisnis: 20 menit
- Menulis & menguji query: 80 menit
- Self-review: baca ulang tiap query, pastikan comment-nya akurat, jalankan `EXPLAIN` di 1–2 query paling kompleks: 20 menit

### Kriteria "Selesai" untuk Minggu 1

- [ ] Bisa menjelaskan tiap query di `queries.sql` ke orang lain tanpa lihat catatan
- [ ] Semua query jalan tanpa error di database lokal
- [ ] Familiar dengan istilah: primary key, foreign key, `JOIN` (4 jenis), CTE, window function, `EXPLAIN`

Kalau semua poin di atas sudah bisa dicentang, target akhir Minggu 1 di `minggu_1.md` ("bisa menulis query SQL kompleks tanpa lihat referensi") sudah tercapai — lanjut ke `minggu_2.md` untuk mulai project portfolio sungguhan.
