---
title: Hari 3 - JOIN
parent: Minggu 1 - SQL Fundamentals
nav_order: 4
---

# Hari 3 — JOIN (INNER, LEFT, RIGHT, FULL)

*Rabu, 2 jam — bagian paling krusial minggu ini. Dataset & setup: lihat `00_overview.md`.*

> Dataset punya 2 baris "sengaja kosong" untuk sesi ini: customer **Hassan Ali** yang belum pernah bikin order, dan produk **Desk Lamp** yang belum pernah terjual. Keduanya jadi contoh utama `LEFT`/`RIGHT`/`FULL JOIN`.

## Tujuan Belajar

- [ ] Menjelaskan kenapa data dipecah ke banyak tabel (normalisasi), bukan satu tabel besar
- [ ] Menulis `INNER JOIN`, `LEFT JOIN`, `RIGHT JOIN`, `FULL OUTER JOIN` dan menjelaskan bedanya
- [ ] Mengenali kapan `LEFT JOIN` "berubah jadi" `INNER JOIN` secara tidak sengaja (jebakan `WHERE` vs `ON`)
- [ ] Menggabungkan lebih dari 2 tabel dalam satu query

## Untuk Instruktur: Mindset Shift

JOIN adalah konsep yang **tidak punya padanan langsung** di kode sehari-hari developer (beda dari `WHERE`≈`.filter()` atau `GROUP BY`≈`.reduce()`) — makanya ini paling sering jadi titik macet. Analogi paling dekat:

```js
// JOIN manual di kode: cari data terkait dari 2 array lewat lookup
const ordersWithCustomer = orders.map(o => ({
  ...o,
  customer: customers.find(c => c.customer_id === o.customer_id)
}));
```

`INNER JOIN` ≈ `.map().find()` di atas tapi **buang row yang `find()`-nya `undefined`**. `LEFT JOIN` ≈ versi yang **tetap simpan row meski `find()` gak ketemu**, cuma field customer-nya jadi `null`.

Tekankan satu hal dari awal: **JOIN itu soal menggabungkan baris berdasarkan kecocokan nilai kolom, bukan soal urutan/posisi row** — beda dari `zip()` di Python yang menggabungkan berdasarkan index/posisi. Developer yang datang dari mindset array sering awalnya mengira JOIN itu seperti `zip()`.

**Kenapa data dipecah ke banyak tabel?** Tanya balik: *"Kalau semua data (customer, order, product) digabung jadi 1 tabel flat, apa masalahnya?"* Arahkan ke: data customer akan terduplikasi di tiap baris order-nya (pemborosan), dan kalau nama customer berubah, harus update di banyak baris (risiko inkonsistensi). Ini fondasi konsep **normalisasi** yang akan mereka pakai terus di data engineering.

## Konsep & Sintaks

### Kenapa Butuh `ON`

```sql
SELECT ...
FROM table_a a
JOIN table_b b ON a.some_key = b.some_key
```

`ON` menentukan **syarat kecocokan** antar baris dua tabel. Tanpa `ON` (atau kalau kondisinya salah), hasilnya bisa jadi **cartesian product** — tiap baris di `table_a` dikombinasikan dengan **semua** baris di `table_b`. Ini kesalahan #1 yang bikin hasil query meledak jadi jutaan baris tanpa disadari (lihat "Kesalahan Umum").

### 4 Jenis JOIN

| Jenis | Hasil |
|---|---|
| `INNER JOIN` | Hanya baris yang **cocok di kedua tabel** |
| `LEFT JOIN` | Semua baris tabel **kiri**, dilengkapi data tabel kanan kalau cocok (kalau tidak cocok → `NULL`) |
| `RIGHT JOIN` | Kebalikan `LEFT JOIN` — semua baris tabel **kanan** |
| `FULL OUTER JOIN` | Semua baris dari **kedua tabel**, cocok atau tidak |

```
INNER          LEFT           RIGHT          FULL
 A∩B            A (semua)      B (semua)      A∪B
```

Catatan praktis: `RIGHT JOIN` jarang dipakai di dunia nyata karena `A RIGHT JOIN B` = `B LEFT JOIN A` — kebanyakan tim (termasuk konvensi yang akan kita pakai sepanjang roadmap ini) lebih suka **selalu tulis `LEFT JOIN` dan tukar urutan tabel**, supaya query lebih konsisten dibaca dari kiri ke kanan.

## Contoh Query

**1. `INNER JOIN` — order lengkap dengan nama customer**
```sql
SELECT c.customer_name, o.order_id, o.order_date, o.status
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
ORDER BY o.order_id;
```
Hasil: 12 baris (semua order, karena tiap order pasti punya customer valid — FK constraint menjamin ini). **Hassan Ali tidak muncul sama sekali** karena dia tidak punya order yang cocok.

**2. `LEFT JOIN` + `GROUP BY` — jumlah order per customer, termasuk yang 0**
```sql
SELECT c.customer_name, COUNT(o.order_id) AS jumlah_order
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_name
ORDER BY jumlah_order DESC;
```
| customer_name | jumlah_order |
|---|---|
| Andi Wijaya | 3 |
| Budi Santoso | 3 |
| Citra Lestari | 2 |
| Emma Watson | 2 |
| Frank Muller | 1 |
| Grace Tan | 1 |
| Hassan Ali | 0 |

Perhatikan: dipakai `COUNT(o.order_id)`, **bukan** `COUNT(*)`. Untuk Hassan, `LEFT JOIN` tetap menghasilkan satu baris (dengan semua kolom `orders` bernilai `NULL`) — `COUNT(*)` akan menghitung baris itu jadi `1` (salah!), sedangkan `COUNT(o.order_id)` mengabaikan NULL sehingga hasilnya benar `0`. Ini sambungan langsung dari pembahasan `COUNT(*)` vs `COUNT(column)` di Hari 2.

**3. `LEFT JOIN` — cari produk yang belum pernah terjual**
```sql
SELECT p.product_name
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
WHERE oi.order_item_id IS NULL;
```
Hasil: `Desk Lamp`. Pola `LEFT JOIN ... WHERE <kolom_kanan> IS NULL` adalah idiom umum untuk **"cari data yang ada di A tapi tidak ada relasinya di B"** — sangat sering dipakai di data engineering untuk cek data quality (mis. "cari transaksi yang customer_id-nya tidak ada di master data customer").

**4. `RIGHT JOIN` — query yang sama, arah dibalik (buat perbandingan)**
```sql
SELECT p.product_name
FROM order_items oi
RIGHT JOIN products p ON oi.product_id = p.product_id
WHERE oi.order_item_id IS NULL;
```
Hasil sama persis: `Desk Lamp`. Ini untuk menunjukkan `RIGHT JOIN` bukan konsep baru — cuma `LEFT JOIN` dibalik.

**5. `FULL OUTER JOIN`**
```sql
SELECT c.customer_name, o.order_id
FROM customers c
FULL OUTER JOIN orders o ON c.customer_id = o.customer_id
WHERE c.customer_id IS NULL OR o.order_id IS NULL;
```
Hasil: hanya `Hassan Ali` (dengan `order_id = NULL`). Di dataset ini, hasil `FULL OUTER JOIN` sebenarnya sama dengan `LEFT JOIN` karena foreign key `orders.customer_id` **dijamin** selalu merujuk ke customer yang valid — tidak mungkin ada order "yatim" (orphan) di sisi kanan.

**Kapan `FULL JOIN` benar-benar dibutuhkan?** Saat kamu **tidak punya jaminan** integritas relasi — misalnya rekonsiliasi dua sumber data mentah (raw CSV export dari sistem A vs sistem B) yang belum tentu sinkron. Ini sangat relevan untuk pekerjaan data engineering sehari-hari: bagian terbesar tugas data quality justru mencari baris yang "hilang" di salah satu sisi.

**6. JOIN 3 tabel sekaligus**
```sql
SELECT c.customer_name, p.product_name, oi.quantity, oi.quantity * oi.unit_price AS subtotal
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status = 'completed'
ORDER BY subtotal DESC
LIMIT 5;
```
Ini pola yang akan sangat sering dipakai: rangkai beberapa `JOIN` untuk "meratakan" (flatten) data relasional jadi satu tabel hasil yang siap dianalisis — persis yang akan dilakukan berulang-ulang di mini project Minggu 2.

## Kesalahan Umum

1. **Lupa `ON` / kondisi JOIN salah → cartesian product.**
   ```sql
   -- BAHAYA: tanpa ON yang benar, tiap order dikalikan tiap customer
   SELECT * FROM customers, orders;  -- comma join tanpa WHERE = cartesian product
   ```
   Kalau tiba-tiba jumlah baris hasil query jauh lebih banyak dari yang diharapkan, curigai ini duluan.

2. **Taruh filter tabel kanan di `WHERE`, bukan di `ON`, saat pakai `LEFT JOIN` — diam-diam berubah jadi `INNER JOIN`.**
   ```sql
   -- SALAH (niatnya "semua customer, tampilkan order completed kalau ada"):
   SELECT c.customer_name, o.order_id
   FROM customers c
   LEFT JOIN orders o ON c.customer_id = o.customer_id
   WHERE o.status = 'completed';
   -- Masalah: customer tanpa order completed (termasuk Hassan) akan HILANG,
   -- karena WHERE dieksekusi setelah JOIN dan membuang baris ber-NULL.

   -- BENAR: syarat status masuk ke ON, bukan WHERE
   SELECT c.customer_name, o.order_id
   FROM customers c
   LEFT JOIN orders o ON c.customer_id = o.customer_id AND o.status = 'completed';
   ```
   Ini salah satu bug paling umum & paling sering lolos code review karena query tetap "jalan" tanpa error, cuma hasilnya diam-diam salah.

3. **Gaya lama `FROM a, b WHERE a.id = b.a_id`.** Masih valid secara sintaks tapi hindari — tidak eksplisit soal jenis JOIN, dan gampang lupa kondisinya (jadi cartesian product tanpa sengaja). Selalu pakai `JOIN ... ON` eksplisit.

4. **Ambiguous column error** saat 2 tabel punya nama kolom sama (mis. `order_id` ada di `orders` dan `order_items`). Selalu pakai alias tabel (`o.order_id`, `oi.order_id`) begitu query melibatkan lebih dari 1 tabel.

## Latihan

1. Tampilkan `order_id`, `order_date`, dan nama customer-nya, untuk semua order dengan status `'cancelled'` atau `'refunded'`.
2. Cari customer yang **belum pernah** bikin order sama sekali (harus dapat "Hassan Ali" tanpa hardcode nama-nya).
3. Untuk tiap kategori produk, hitung total revenue (`quantity * unit_price`) dari `order_items`, hanya dari order berstatus `'completed'`. (Butuh JOIN 3 tabel: `products`, `order_items`, `orders`.)
4. Tampilkan semua produk beserta total quantity terjual — termasuk produk yang belum pernah terjual (harus tampil dengan `0`, bukan hilang).
5. Cari kesalahan di query berikut, jelaskan kenapa salah, lalu perbaiki:
   ```sql
   SELECT c.customer_name, COUNT(o.order_id) AS jumlah_order
   FROM customers c
   LEFT JOIN orders o ON c.customer_id = o.customer_id
   WHERE o.status = 'completed'
   GROUP BY c.customer_name;
   ```

## Kunci Jawaban & Pembahasan

**1.**
```sql
SELECT o.order_id, o.order_date, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status IN ('cancelled', 'refunded');
```
Hasil: order 6 (Budi Santoso, cancelled), order 11 (Emma Watson, refunded).

**2.**
```sql
SELECT c.customer_name
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```
Hasil: Hassan Ali. Ini idiom "cari yang tidak punya pasangan" — sama seperti contoh #3 di atas tapi arah tabelnya dibalik.

**3.**
```sql
SELECT p.category, SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY p.category
ORDER BY total_revenue DESC;
```
Hasil: Furniture (360.00 — dari Office Chair saja, Desk Lamp menyumbang 0 karena tidak pernah terjual), Electronics (320.00), Kitchen (64.00), Stationery (63.00).

**4.**
```sql
SELECT p.product_name, COALESCE(SUM(oi.quantity), 0) AS total_terjual
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_name
ORDER BY total_terjual DESC;
```
Hasil: Desk Lamp muncul dengan `total_terjual = 0`. Poin penting: **`SUM` dari kumpulan yang isinya cuma `NULL` hasilnya `NULL`, bukan `0`** — makanya butuh `COALESCE(..., 0)` untuk mengubah `NULL` jadi `0` secara eksplisit. Ini variasi lanjutan dari jebakan `AVG`/`SUM` dengan `NULL` yang dibahas di Hari 2.

**5.**
Salah karena `WHERE o.status = 'completed'` dieksekusi **setelah** `LEFT JOIN`, dan membuang semua baris di mana `o.status` bukan `'completed'` — termasuk baris `NULL` milik Hassan (NULL tidak pernah `= 'completed'`). Hasilnya, `LEFT JOIN` yang dimaksudkan untuk tetap menampilkan semua customer, diam-diam berperilaku seperti `INNER JOIN`.

Perbaikan — pindahkan syarat status ke `ON`:
```sql
SELECT c.customer_name, COUNT(o.order_id) AS jumlah_order
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id AND o.status = 'completed'
GROUP BY c.customer_name;
```
Sekarang Hassan tetap muncul dengan `jumlah_order = 0`.
