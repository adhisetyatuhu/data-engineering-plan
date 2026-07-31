---
title: Hari 3 - Pandas Dasar
parent: Minggu 2 - Python Fundamentals
nav_order: 4
---

# Hari 3 — Pandas Dasar: DataFrame, read_csv, Filtering, Selection

*Rabu, 2 jam. Setup & dataset: lihat `00_overview.md`.*

> Ini titik mulai skill yang **benar-benar baru** buat peserta — beda dari 2 hari sebelumnya. Alokasikan waktu lebih banyak di sini dibanding Hari 1–2.

## Tujuan Belajar

- [ ] Menjelaskan `DataFrame` sebagai representasi tabel SQL di Python
- [ ] Memuat CSV, eksplorasi awal (`head`, `info`, `describe`, `dtypes`)
- [ ] Memilih kolom & baris dengan `[]`, `.loc`, `.iloc`
- [ ] Filtering pakai boolean indexing (padanan `WHERE`)
- [ ] Sorting & membatasi jumlah baris (padanan `ORDER BY` + `LIMIT`)

## Untuk Instruktur: Mindset Shift

`DataFrame` = **tabel SQL yang hidup di memori Python**, dengan `Series` (1 kolom) sebagai unit dasarnya. Tekankan pemetaan ini dari awal — sudah disinggung di `00_overview.md`, sekarang saatnya dipraktikkan langsung:

| SQL (Minggu 1) | Pandas |
|---|---|
| `SELECT col1, col2 FROM t` | `df[['col1', 'col2']]` |
| `SELECT * FROM t WHERE cond` | `df[cond]` |
| `ORDER BY col DESC` | `df.sort_values('col', ascending=False)` |
| `LIMIT n` | `df.head(n)` |

**Peringatan penting untuk instruktur**: developer yang baru pindah dari SQL/array sering kaget dengan `.loc` vs `.iloc`. Jangan buru-buru — ini salah satu sumber bug paling umum di kode Pandas pemula. Sisihkan waktu khusus untuk ini (lihat bagian Konsep di bawah).

## Konsep & Sintaks

### Memuat & Eksplorasi Data

```python
import pandas as pd

products = pd.read_csv("data/products.csv")

products.head()        # 5 baris pertama (padanan LIMIT 5)
products.info()        # tipe data tiap kolom + jumlah non-null (deteksi missing value cepat)
products.describe()    # statistik ringkas kolom numerik (count, mean, std, min, max, quartile)
products.shape         # (jumlah_baris, jumlah_kolom) — bukan method, tanpa ()
products.columns       # daftar nama kolom
products.dtypes        # tipe data tiap kolom
```

`products.info()` adalah kebiasaan **pertama** yang harus dilakukan tiap buka dataset baru — langsung kelihatan ada berapa baris, kolom apa saja, tipe datanya apa, dan **paling penting**: kolom mana yang punya nilai `null` (dari selisih "Non-Null Count" dengan total baris).

### Memilih Kolom

```python
products["product_name"]              # 1 kolom -> Series (mirip list 1 dimensi dengan index)
products[["product_name", "unit_price"]]  # banyak kolom -> tetap DataFrame (perhatikan: double bracket!)
```

Double bracket `[[...]]` itu bukan typo — bracket luar untuk "akses ke DataFrame", bracket dalam adalah **list** nama kolom yang dipilih.

### Memilih Baris: `.loc` vs `.iloc`

| | Berbasis apa | Slicing |
|---|---|---|
| `.loc[]` | **Label** (nama index/kolom) | Batas akhir **inklusif** |
| `.iloc[]` | **Posisi integer** (seperti index array biasa) | Batas akhir **eksklusif** (seperti Python biasa) |

```python
products.loc[0:2, ["product_name", "unit_price"]]   # baris index 0, 1, DAN 2 (inklusif!) -> 3 baris
products.iloc[0:2, [1, 3]]                            # baris posisi 0 dan 1 saja (eksklusif) -> 2 baris
```

Ini **beda dari slicing list Python biasa** (`my_list[0:2]` cuma ambil index 0 dan 1) — `.loc` sengaja inklusif karena bekerja dengan label, bukan posisi, dan label bisa berupa apa saja (termasuk string), jadi konsep "eksklusif" tidak selalu masuk akal untuk label.

### Filtering — Boolean Indexing (padanan `WHERE`)

```python
products[products["unit_price"] > 20]

# Beberapa kondisi: WAJIB pakai & (dan) / | (atau), BUKAN and/or, dan WAJIB kurung tiap kondisi
products[(products["category"] == "Electronics") & (products["unit_price"] > 20)]
```

### Sorting & Membatasi Baris

```python
products.sort_values("unit_price", ascending=False)   # ORDER BY unit_price DESC
products.sort_values("unit_price", ascending=False).head(3)   # + LIMIT 3
```

## Contoh Kode

```python
import pandas as pd

customers = pd.read_csv("data/customers.csv")
products = pd.read_csv("data/products.csv")
orders = pd.read_csv("data/orders.csv")
```

**1. Padanan `SELECT customer_name, country FROM customers WHERE country = 'Indonesia'`**
```python
customers[customers["country"] == "Indonesia"][["customer_name", "country"]]
```
Hasil: Andi Wijaya, Budi Santoso, Citra Lestari (semua "Indonesia") — identik dengan contoh #2 di `materi/minggu_1/hari_1_select_where.md`.

**2. Padanan `SELECT ... ORDER BY unit_price DESC LIMIT 3`**
```python
products.sort_values("unit_price", ascending=False).head(3)[["product_name", "unit_price"]]
```
Hasil: Office Chair (120.00), Mechanical Keyboard (45.00), Desk Lamp (30.00) — identik dengan contoh #4 (setelah update dataset) di `hari_1_select_where.md` Minggu 1.

**3. Padanan `WHERE category = 'Electronics' AND unit_price > 20`**
```python
products[(products["category"] == "Electronics") & (products["unit_price"] > 20)]
```
Hasil: Mechanical Keyboard, USB-C Hub.

**4. Padanan `WHERE status != 'completed'`**
```python
orders[orders["status"] != "completed"]
```
Hasil: order 6 (cancelled), order 11 (refunded).

## Kesalahan Umum

1. **Pakai `and`/`or` Python biasa, bukan `&`/`|`, di boolean indexing.**
   ```python
   # SALAH — akan ValueError ("truth value of a Series is ambiguous"):
   products[products["category"] == "Electronics" and products["unit_price"] > 20]
   # BENAR:
   products[(products["category"] == "Electronics") & (products["unit_price"] > 20)]
   ```
   `and`/`or` Python didesain untuk 1 nilai boolean tunggal, bukan array/Series berisi banyak boolean sekaligus — Pandas sengaja meng-override `&`/`|` (operator bitwise) untuk kebutuhan ini. **Kurung tiap kondisi wajib** karena `&`/`|` punya prioritas operator lebih tinggi dari `==`/`>`, tanpa kurung hasilnya akan salah/error.

2. **Lupa bedanya `.loc` slicing (inklusif) vs `.iloc` slicing (eksklusif).** Ini sumber off-by-one error paling umum untuk pemula Pandas. Aturan aman: kalau tidak yakin, pakai `.iloc` untuk akses berbasis posisi (perilakunya konsisten dengan slicing Python biasa) dan `.loc` khusus untuk akses berbasis label/kondisi boolean.

3. **`SettingWithCopyWarning`.** Muncul saat kamu mencoba assign nilai ke hasil filter tanpa `.copy()` eksplisit — Pandas tidak selalu bisa memastikan apakah hasil filter itu "view" (referensi ke data asli) atau "copy" (salinan independen).
   ```python
   # Berpotensi warning/bug:
   expensive = products[products["unit_price"] > 20]
   expensive["on_sale"] = True   # SettingWithCopyWarning

   # Aman:
   expensive = products[products["unit_price"] > 20].copy()
   expensive["on_sale"] = True
   ```

4. **`df.shape` dan `df.columns` bukan method — jangan tulis `df.shape()`.** Ini `attribute`, bukan `method`, jadi tanpa tanda kurung.

## Latihan

Pakai `data/customers.csv`, `data/products.csv`, `data/orders.csv`.

1. Tampilkan `product_name` dan `unit_price` untuk produk dengan harga di bawah 20 (padanan Latihan #1 Hari 1 Minggu 1).
2. Tampilkan `customer_name` dan `signup_date` untuk customer yang mendaftar setelah `'2023-03-01'`, diurutkan dari yang paling lama daftar (padanan Latihan #2 Hari 1 Minggu 1 — hati-hati, `signup_date` terbaca sebagai string oleh `read_csv`, perbandingan string ISO-format `YYYY-MM-DD` kebetulan tetap benar secara urutan, tapi diskusikan kenapa ini rapuh).
3. Tampilkan semua order dengan `status == 'completed'` yang `order_date`-nya di bulan Maret 2024 (boleh tetap pakai perbandingan string dulu, isu tipe data `datetime` akan dibahas Hari 5).
4. Tampilkan 3 produk termurah beserta kolom `category`.
5. Pakai `.info()` pada ketiga DataFrame (`customers`, `products`, `orders`) — sebutkan apakah ada kolom dengan nilai `null` di salah satu dari ketiganya (jawaban: seharusnya tidak ada, karena data ini masih "bersih" — akan berubah di Hari 5).

## Kunci Jawaban & Pembahasan

**1.**
```python
products[products["unit_price"] < 20][["product_name", "unit_price"]]
```
Hasil: Wireless Mouse (15.00), Notebook A5 (3.50), Coffee Mug (8.00).

**2.**
```python
customers[customers["signup_date"] > "2023-03-01"].sort_values("signup_date")[["customer_name", "signup_date"]]
```
Hasil: Frank Muller, Grace Tan, Hassan Ali. Perbandingan string `> "2023-03-01"` kebetulan benar di sini **karena** format tanggalnya ISO 8601 (`YYYY-MM-DD`) yang urutan alfabetisnya kebetulan sama dengan urutan kronologisnya. Ini **rapuh**: kalau formatnya `DD/MM/YYYY`, perbandingan string akan salah total. Solusi tepat: konversi ke `datetime` dulu (`pd.to_datetime`) — dibahas detail di Hari 5.

**3.**
```python
orders[(orders["status"] == "completed") &
       (orders["order_date"] >= "2024-03-01") &
       (orders["order_date"] <= "2024-03-31")]
```
Hasil: order 8, 9, 10, 12.

**4.**
```python
products.sort_values("unit_price").head(3)[["product_name", "category", "unit_price"]]
```
Hasil: Notebook A5 (Stationery, 3.50), Coffee Mug (Kitchen, 8.00), Wireless Mouse (Electronics, 15.00).

**5.**
```python
customers.info()
products.info()
orders.info()
```
Ketiganya menunjukkan "Non-Null Count" = jumlah total baris di tiap kolom → tidak ada `null`. Ini akan jadi pembanding langsung dengan `data/sales_raw_dirty.csv` di Hari 5, yang `.info()`-nya akan menunjukkan gap antara non-null count dan total baris di beberapa kolom.
