---
title: Hari 4 - Pandas Lanjutan
parent: Minggu 2 - Python Fundamentals
nav_order: 5
---

# Hari 4 — Pandas Lanjutan: groupby, merge/join, pivot_table

*Kamis, 2 jam. Setup & dataset: lihat `00_overview.md`.*

## Tujuan Belajar

- [ ] Memakai `.groupby()` + agregasi, termasuk multi-aggregation dengan `.agg()`
- [ ] Memakai `.merge()` dengan `how='inner'/'left'/'right'/'outer'`, dan mengenali risiko row multiplication
- [ ] Memakai `.pivot_table()` untuk ringkasan 2 dimensi
- [ ] Terbiasa `.reset_index()` setelah `groupby` agar hasilnya gampang dipakai lagi

## Untuk Instruktur: Mindset Shift

Ini sesi **penyambung paling eksplisit** ke Minggu 1 — `.groupby()` ≈ `GROUP BY`, `.merge()` ≈ `JOIN`. Kalau ada waktu, paling efektif: ambil ulang 2-3 query SQL dari `materi/minggu_1/hari_2_agregasi_groupby.md` dan `hari_3_join.md`, minta peserta tulis versi Pandas-nya sendiri **sebelum** ditunjukkan caranya — mereka sudah punya jawaban SQL sebagai "spek" yang harus direplikasi hasilnya.

**Kesalahan yang paling penting diantisipasi**: `.merge()` bisa **diam-diam memperbanyak baris** kalau key join-nya tidak unik di salah satu sisi — persis versi Pandas dari peringatan "cartesian product" di `materi/minggu_1/hari_3_join.md`. Developer yang tidak sadar row count-nya membengkak setelah merge adalah salah satu bug paling umum & paling sulit dilacak di pipeline data nyata (biasanya ketauannya belakangan, waktu angka agregat tiba-tiba dobel).

## Konsep & Sintaks

### `groupby` — padanan `GROUP BY`

```python
order_items.groupby("order_id")["subtotal"].sum()          # 1 agregasi
order_items.groupby("order_id")["subtotal"].agg(["sum", "count"])   # beberapa agregasi sekaligus
order_items.groupby("order_id").agg(
    total_revenue=("subtotal", "sum"),
    jumlah_item=("subtotal", "count")
)   # named aggregation — hasil kolom langsung dikasih nama, lebih terbaca
```

**Penting**: hasil `groupby` punya kolom yang di-`groupby` sebagai **index**, bukan kolom biasa. Kalau mau dipakai lagi seperti DataFrame normal (misal buat di-merge lagi), panggil `.reset_index()`:
```python
revenue_per_order = order_items.groupby("order_id")["subtotal"].sum().reset_index()
# sekarang order_id jadi kolom biasa lagi, bukan index
```

### `merge` — padanan `JOIN`

```python
pd.merge(df_kiri, df_kanan, on="key_column", how="inner")   # atau df_kiri.merge(df_kanan, ...)
```

| Parameter `how=` | Padanan SQL |
|---|---|
| `'inner'` (default) | `INNER JOIN` |
| `'left'` | `LEFT JOIN` |
| `'right'` | `RIGHT JOIN` |
| `'outer'` | `FULL OUTER JOIN` |

```python
customers.merge(orders, on="customer_id", how="inner")   # cuma customer yang punya order
customers.merge(orders, on="customer_id", how="left")    # semua customer, order-nya NaN kalau gak ada
```

Kalau nama kolom key beda di kedua tabel: `left_on="customer_id", right_on="cust_id"`. Kalau ada kolom lain (bukan key) yang namanya sama di kedua tabel, Pandas otomatis kasih suffix `_x`/`_y` — lebih baik kasih suffix eksplisit yang jelas maknanya: `suffixes=("_order", "_product")`.

### `pivot_table` — ringkasan 2 dimensi

```python
df.pivot_table(index="category", columns="month", values="revenue", aggfunc="sum", fill_value=0)
```
Ini seperti bikin tabel `GROUP BY category, month` lalu "diputar" supaya `month` jadi kolom, bukan baris — mirip pivot table di spreadsheet. `fill_value=0` mengisi kombinasi yang tidak ada datanya dengan `0`, bukan `NaN`.

## Contoh Kode

```python
import pandas as pd

customers = pd.read_csv("data/customers.csv")
products = pd.read_csv("data/products.csv")
orders = pd.read_csv("data/orders.csv")
order_items = pd.read_csv("data/order_items.csv")
order_items["subtotal"] = order_items["quantity"] * order_items["unit_price"]
```

**1. Padanan query `GROUP BY order_id` (Hari 2 Minggu 1, contoh #3)**
```python
order_items.groupby("order_id")["subtotal"].sum()
```
Hasil identik dengan tabel di `materi/minggu_1/hari_2_agregasi_groupby.md`: order 9 = 240.00 tertinggi, dst.

**2. Padanan revenue per customer (JOIN 3 tabel + GROUP BY, completed saja)**
```python
merged = (order_items
    .merge(orders, on="order_id")
    .merge(customers, on="customer_id"))
completed = merged[merged["status"] == "completed"]
completed.groupby("customer_name")["subtotal"].sum().sort_values(ascending=False)
```
Hasil: Grace Tan (240.00), Citra Lestari (235.50), Andi Wijaya (118.50), Budi Santoso (118.00), Emma Watson (60.00), Frank Muller (35.00) — **identik persis** dengan jawaban Latihan #1 di `materi/minggu_1/hari_5_window_function.md`.

**3. `LEFT` merge — cari produk yang belum pernah terjual (padanan `LEFT JOIN ... WHERE ... IS NULL`)**
```python
prod_check = products.merge(order_items, on="product_id", how="left")
prod_check[prod_check["order_item_id"].isna()][["product_name"]]
```
Hasil: Desk Lamp — identik dengan contoh #3 `hari_3_join.md` Minggu 1. Pola `.merge(..., how="left")` lalu filter `[kolom_kanan].isna()` adalah padanan langsung idiom SQL "cari yang tidak ada pasangannya".

**4. `pivot_table` — revenue per kategori per bulan (completed saja)**
```python
completed = completed.copy()
completed["order_date"] = pd.to_datetime(completed["order_date"])
completed["month"] = completed["order_date"].dt.to_period("M")

completed.pivot_table(index="category", columns="month", values="subtotal", aggfunc="sum", fill_value=0)
```
| category | 2024-01 | 2024-02 | 2024-03 |
|---|---|---|---|
| Electronics | 100.0 | 60.0 | 160.0 |
| Furniture | 0.0 | 120.0 | 240.0 |
| Kitchen | 16.0 | 8.0 | 40.0 |
| Stationery | 10.5 | 35.0 | 17.5 |

Total tiap kolom bulan harus sama dengan hasil `GROUP BY month` di `materi/minggu_1/hari_4_subquery_cte.md` contoh #6 (126.5 / 223.0 / 457.5) — coba jumlahkan tiap kolom sebagai latihan verifikasi silang.

## Kesalahan Umum

1. **`merge` memperbanyak baris diam-diam kalau key tidak unik di salah satu sisi.**
   ```python
   discounts = pd.DataFrame({
       "category": ["Electronics", "Electronics", "Furniture"],
       "campaign": ["Ramadan Sale", "Year End Sale", "Ramadan Sale"],
   })
   result = products.merge(discounts, on="category", how="left")
   print(len(products), "->", len(result))   # 7 -> 10 !!
   ```
   `category` bukan key unik di `discounts` (Electronics muncul 2x) — tiap produk Electronics jadi digandakan sesuai jumlah campaign yang match. Ini **bukan bug Pandas**, ini konsekuensi logis dari JOIN/merge relasional (sama seperti cartesian product di SQL), tapi sangat mudah tidak disadari kalau tidak dicek. **Kebiasaan wajib**: selalu bandingkan `len(df)` sebelum dan sesudah `merge`, terutama untuk merge yang bukan jelas-jelas 1:1.

2. **Lupa `.reset_index()` setelah `groupby`, lalu bingung kenapa kolom groupby-nya "hilang".** Kolom hasil `groupby` jadi index, bukan kolom biasa — kalau langsung dipakai untuk `merge` lagi tanpa `reset_index()`, akan error atau perilakunya tidak sesuai ekspektasi.

3. **`how="inner"` (default) diam-diam membuang baris yang tidak match, sama seperti lupa pilih jenis JOIN di SQL.** Selalu putuskan `how=` secara sadar, jangan biarkan default `inner` dipakai tanpa dipikir — terutama kalau maksudnya "semua data kiri harus tetap muncul" (butuh `how="left"`, bukan default).

4. **Overlapping column name tanpa `suffixes` eksplisit bikin bingung `col_x`/`col_y` yang mana.** Kasih `suffixes=("_kiri", "_kanan")` yang deskriptif kalau tahu bakal ada tabrakan nama kolom.

## Latihan

1. Hitung total quantity dan total revenue per `product_id` dari `order_items` (padanan Latihan #2 Hari 2 Minggu 1).
2. Gabungkan `customers` dan `orders` dengan `how="left"`, lalu hitung jumlah order per customer (termasuk yang 0) — padanan contoh #2 `hari_3_join.md` Minggu 1.
3. Cari kategori produk dengan total revenue tertinggi (completed saja) — butuh merge 3 tabel.
4. Buat `pivot_table` jumlah order (`count`, bukan `sum`) per `customer_id` per `status`.
5. Ada 2 DataFrame kecil: `orders_a = pd.DataFrame({"order_id":[1,2,3], "amount":[100,200,300]})` dan `orders_b = pd.DataFrame({"order_id":[2,3,4], "shipped":[True, False, True]})`. Merge dengan `how="outer"`, lalu identifikasi baris mana yang `NaN` di kolom `amount` dan baris mana yang `NaN` di kolom `shipped` — jelaskan artinya masing-masing.

## Kunci Jawaban & Pembahasan

**1.**
```python
order_items.groupby("product_id").agg(
    total_qty=("quantity", "sum"),
    total_revenue=("subtotal", "sum")
).sort_values("total_revenue", ascending=False)
```
Hasil teratas: product_id 4 (Office Chair, 360.00), product_id 2 (Keyboard, 180.00) — sama seperti jawaban Hari 2 Minggu 1.

**2.**
```python
cust_orders = customers.merge(orders, on="customer_id", how="left")
cust_orders.groupby("customer_name")["order_id"].count().sort_values(ascending=False)
```
Hasil: Andi Wijaya (3), Budi Santoso (3), Citra Lestari (2), Emma Watson (2), Frank Muller (1), Grace Tan (1), **Hassan Ali (0)** — identik dengan tabel di `hari_3_join.md` contoh #2. Perhatikan dipakai `.count()` pada kolom `order_id` (bukan `.size()`) — `.count()` mengabaikan `NaN` (persis kenapa `COUNT(o.order_id)` dipakai, bukan `COUNT(*)`, di versi SQL-nya).

**3.**
```python
merged = (order_items
    .merge(orders, on="order_id")
    .merge(products, on="product_id"))
completed = merged[merged["status"] == "completed"]
completed.groupby("category")["subtotal"].sum().sort_values(ascending=False)
```
Hasil: Furniture (360.00), Electronics (320.00), Kitchen (64.00), Stationery (63.00) — identik dengan `hari_3_join.md` Latihan #3.

**4.**
```python
cust_orders.pivot_table(index="customer_id", columns="status", values="order_id", aggfunc="count", fill_value=0)
```
Hasil: tiap baris customer, kolom `completed`/`cancelled`/`refunded` isinya jumlah order per status, `0` untuk kombinasi yang tidak ada. **Tapi perhatikan: Hassan Ali (customer_id 7) tidak muncul sama sekali di hasil pivot**, bukan muncul dengan semua kolom `0` seperti dugaan intuitif. Penyebabnya: `status` milik Hassan bernilai `NaN` (dia tidak punya order sama sekali), dan `pivot_table` **secara default membuang baris yang nilai kolom `columns=`-nya `NaN`** — beda dari `fill_value` yang cuma mengisi *kombinasi* yang kosong, bukan baris yang datanya sendiri `NaN` di dimensi `columns`. Ini gotcha nyata yang gampang lolos code review: kalau butuh Hassan tetap muncul dengan `0`, harus `reindex` manual dengan daftar lengkap `customer_id` setelah pivot.

**5.**
```python
result = orders_a.merge(orders_b, on="order_id", how="outer")
print(result)
```
| order_id | amount | shipped |
|---|---|---|
| 1 | 100.0 | NaN |
| 2 | 200.0 | True |
| 3 | 300.0 | False |
| 4 | NaN | True |

`order_id=1` punya `shipped = NaN` — artinya order ini ada di `orders_a` tapi **tidak** ada datanya di `orders_b` (mungkin belum pernah di-cek status pengirimannya). `order_id=4` punya `amount = NaN` — ada di `orders_b` tapi **tidak** ada di `orders_a` (mungkin order ini belum tercatat di sistem billing). Ini pola yang sama persis dengan diskusi `FULL OUTER JOIN` untuk **rekonsiliasi 2 sumber data** di `materi/minggu_1/hari_3_join.md` — kali ini dipraktikkan dengan data yang benar-benar tidak dijamin sinkron satu sama lain (`orders_b` tidak punya foreign key constraint apapun ke `orders_a`).
