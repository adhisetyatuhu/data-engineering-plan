# Hari 2 — Fungsi Agregasi (COUNT, SUM, AVG), GROUP BY, HAVING

*Selasa, 2 jam. Dataset & setup: lihat `00_overview.md`.*

## Tujuan Belajar

- [ ] Memakai fungsi agregasi: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`
- [ ] Mengelompokkan data dengan `GROUP BY` dan memahami aturan "kolom di `SELECT` harus ada di `GROUP BY` atau dibungkus agregasi"
- [ ] Memahami beda `WHERE` (filter row sebelum grouping) vs `HAVING` (filter grup setelah agregasi)
- [ ] Sadar bagaimana `NULL` diperlakukan fungsi agregasi

## Untuk Instruktur: Mindset Shift

`GROUP BY` + agregasi = versi SQL dari **`.reduce()` yang dikelompokkan dulu berdasarkan key**. Kalau developer sudah paham `.reduce()`, jembatannya gampang:

```js
// "total quantity per product_id" di JS:
const totalPerProduct = orderItems.reduce((acc, item) => {
  acc[item.product_id] = (acc[item.product_id] || 0) + item.quantity;
  return acc;
}, {});
```
```sql
-- versi SQL:
SELECT product_id, SUM(quantity) AS total_quantity
FROM order_items
GROUP BY product_id;
```
Sama-sama: **kelompokkan berdasarkan key, lalu akumulasi nilai per grup.** Bedanya SQL menyembunyikan mekanisme reduce-nya di balik fungsi agregasi.

**Pertanyaan yang sering muncul**: *"Kenapa saya nggak bisa `SELECT product_name, SUM(quantity) FROM order_items GROUP BY product_id`?"* — ini jebakan klasik. Jawab dengan menyambung ke urutan eksekusi dari Hari 1: `GROUP BY` mengeksekusi *sebelum* `SELECT`, dan setelah grouping, database cuma tahu "grup mana", bukan lagi row individual — jadi kolom yang tidak di-`GROUP BY` dan tidak dibungkus agregasi itu **ambigu** (grup itu punya banyak row, `product_name` yang mana yang mau ditampilkan?). Aturannya: **tiap kolom di `SELECT` wajib ada di `GROUP BY`, atau dibungkus fungsi agregasi** — tidak ada pengecualian.

## Konsep & Sintaks

### Urutan Eksekusi (update dari Hari 1)

```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

Ini kunci untuk paham `WHERE` vs `HAVING`:
- **`WHERE`**: filter row **sebelum** dikelompokkan. Tidak bisa pakai fungsi agregasi (`SUM`, `COUNT`, dst) di sini — karena saat `WHERE` jalan, agregasi belum dihitung.
- **`HAVING`**: filter grup **setelah** agregasi dihitung. Di sinilah kamu boleh (dan biasanya harus) pakai `SUM(...) > x`, `COUNT(...) > x`, dst.

### Fungsi Agregasi Dasar

| Fungsi | Kegunaan | Analog di kode |
|---|---|---|
| `COUNT(*)` | Jumlah row | `array.length` |
| `COUNT(column)` | Jumlah row dengan `column` **tidak NULL** | `array.filter(x => x.column != null).length` |
| `SUM(column)` | Total nilai | `array.reduce((a,b) => a + b.column, 0)` |
| `AVG(column)` | Rata-rata (mengabaikan NULL) | jumlah dibagi count yang non-null |
| `MIN` / `MAX` | Nilai terkecil/terbesar | `Math.min(...)` / `Math.max(...)` |

## Contoh Query

**1. Agregasi tanpa `GROUP BY` (satu angka untuk seluruh tabel)**
```sql
SELECT COUNT(*) AS total_orders,
       COUNT(DISTINCT customer_id) AS unique_customers
FROM orders;
```
| total_orders | unique_customers |
|---|---|
| 12 | 6 |

**2. `GROUP BY` satu kolom — jumlah order per status**
```sql
SELECT status, COUNT(*) AS jumlah
FROM orders
GROUP BY status;
```
| status | jumlah |
|---|---|
| completed | 10 |
| cancelled | 1 |
| refunded | 1 |

**3. `GROUP BY` + `SUM` — total revenue per order (tanpa join dulu, langsung dari `order_items`)**
```sql
SELECT order_id,
       SUM(quantity * unit_price) AS total_revenue
FROM order_items
GROUP BY order_id
ORDER BY total_revenue DESC
LIMIT 5;
```
| order_id | total_revenue |
|---|---|
| 9 | 240.00 |
| 4 | 128.00 |
| 10 | 107.50 |
| 8 | 53.00 |
| 6 | 50.00 |

> Catatan: `order_id` 6 (status `cancelled`) dan `order_id` 11 (status `refunded`, tidak masuk top 5 di atas) tetap terhitung revenue-nya di sini karena kita belum filter berdasarkan status order — `order_items` sendiri tidak tahu status ordernya. Untuk exclude order batal/refund, kita butuh gabungkan ke tabel `orders` — itu topik besok (`JOIN`).

**4. `HAVING` — hanya produk yang total terjualnya lebih dari 5 unit**
```sql
SELECT product_id, SUM(quantity) AS total_terjual
FROM order_items
GROUP BY product_id
HAVING SUM(quantity) > 5
ORDER BY total_terjual DESC;
```
| product_id | total_terjual |
|---|---|
| 5 | 18 |
| 6 | 8 |
| 1 | 7 |

**5. `WHERE` + `GROUP BY` + `HAVING` sekaligus — rata-rata harga produk per kategori, hanya kategori dengan lebih dari 1 produk**
```sql
SELECT category, COUNT(*) AS jumlah_produk, ROUND(AVG(unit_price), 2) AS avg_price
FROM products
WHERE unit_price > 0
GROUP BY category
HAVING COUNT(*) > 1;
```
| category | jumlah_produk | avg_price |
|---|---|---|
| Electronics | 3 | 28.33 |
| Furniture | 2 | 75.00 |

## Kesalahan Umum

1. **Pakai fungsi agregasi di `WHERE`.**
   ```sql
   -- SALAH:
   SELECT product_id, SUM(quantity)
   FROM order_items
   WHERE SUM(quantity) > 5
   GROUP BY product_id;
   -- BENAR: pakai HAVING
   ```

2. **`SELECT` kolom yang bukan di `GROUP BY` dan bukan agregasi.** Postgres akan langsung error (`column must appear in the GROUP BY clause or be used in an aggregate function`) — beberapa database lain (mis. versi lama MySQL) malah diam-diam ngasih hasil ambigu. Jangan andalkan itu.

3. **Lupa `COUNT(*)` vs `COUNT(column)` beda kalau ada NULL.** `COUNT(*)` hitung semua row, `COUNT(column)` hanya hitung row yang kolom itu tidak NULL.

4. **`AVG` mengabaikan NULL, bukan menghitungnya sebagai 0.** Kalau ada 3 row dengan nilai `10, NULL, 20`, `AVG` = `(10+20)/2 = 15`, bukan `(10+0+20)/3 = 10`. Ini sering bikin angka bisnis salah kalau tidak disadari.

## Latihan

1. Hitung jumlah customer per `country`.
2. Hitung total quantity dan total revenue (`quantity * unit_price`) per `product_id` dari `order_items`, urutkan dari revenue tertinggi.
3. Tampilkan `status` order yang jumlah order-nya lebih dari 1 (pakai `HAVING`).
4. Cari harga produk termahal dan termurah per `category`.
5. Dari `order_items`, cari `order_id` yang punya lebih dari 1 baris item (artinya order itu beli lebih dari 1 jenis produk).

## Kunci Jawaban & Pembahasan

**1.**
```sql
SELECT country, COUNT(*) AS jumlah_customer
FROM customers
GROUP BY country;
```
Hasil: Indonesia=3, UK=1, Germany=1, Singapore=1, UAE=1.

**2.**
```sql
SELECT product_id,
       SUM(quantity) AS total_qty,
       SUM(quantity * unit_price) AS total_revenue
FROM order_items
GROUP BY product_id
ORDER BY total_revenue DESC;
```
Hasil teratas: product_id 4 (Office Chair) revenue 360, product_id 2 (Keyboard) revenue 180, dst. (Angka ini sudah termasuk order yang cancelled/refunded karena belum join ke `orders` — poin diskusi bagus: "kalau mau exclude order batal, kita butuh JOIN, itu topik besok.")

**3.**
```sql
SELECT status, COUNT(*) AS jumlah
FROM orders
GROUP BY status
HAVING COUNT(*) > 1;
```
Hasil: hanya `completed` (10) — `cancelled` dan `refunded` masing-masing cuma 1, jadi tidak lolos `HAVING`.

**4.**
```sql
SELECT category, MIN(unit_price) AS termurah, MAX(unit_price) AS termahal
FROM products
GROUP BY category;
```
Hasil: Electronics (15.00 – 45.00), Furniture (30.00 – 120.00), Stationery (3.50 – 3.50), Kitchen (8.00 – 8.00). Bahas: kategori dengan 1 produk (Stationery, Kitchen), `MIN` = `MAX` — masuk akal karena cuma ada satu nilai di grup itu.

**5.**
```sql
SELECT order_id, COUNT(*) AS jumlah_item
FROM order_items
GROUP BY order_id
HAVING COUNT(*) > 1;
```
Hasil: order_id 1, 2, 4, 5, 8, 10, 12 (masing-masing 2 baris item); order 3, 6, 7, 9, 11 hanya beli 1 jenis produk jadi tidak lolos.
