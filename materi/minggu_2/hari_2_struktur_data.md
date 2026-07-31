---
title: Hari 2 - Struktur Data
parent: Minggu 2 - Python Fundamentals
nav_order: 3
---

# Hari 2 — Struktur Data: list, dict, set, tuple + Comprehension

*Selasa, 2 jam. Setup & dataset: lihat `00_overview.md`.*

## Tujuan Belajar

- [ ] Memilih struktur data yang tepat: `list`, `tuple`, `set`, `dict` — dan menjelaskan kenapa
- [ ] Menulis list/dict/set comprehension, dan menghubungkannya ke cara berpikir SQL (`SELECT ... WHERE`) dari Minggu 1
- [ ] Memakai `enumerate` dan `zip` untuk pola iterasi umum
- [ ] Menghindari bug klasik: memodifikasi list saat looping di atasnya

## Untuk Instruktur: Mindset Shift

Ini sesi kedua yang juga lebih soal "sintaks baru untuk konsep lama" (array, hash map dari bahasa lain), **kecuali comprehension** — dan comprehension inilah yang layak dapat waktu ekstra karena ini titik temu yang sangat kuat dengan pelajaran Minggu 1.

**Poin penting untuk ditekankan eksplisit**: list comprehension itu **structurally sama** dengan `SELECT ... FROM ... WHERE` di SQL.

```python
[p["product_name"] for p in products if p["unit_price"] > 20]
#      ^SELECT              ^FROM         ^WHERE
```
```sql
SELECT product_name FROM products WHERE unit_price > 20;
```

Ingat pembahasan "SQL itu declarative" di `materi/minggu_1/hari_1_select_where.md`? List comprehension adalah **versi Python dari cara berpikir yang sama**: kamu deklarasikan bentuk hasil akhir (`expr for item in iterable if condition`), bukan menulis langkah-langkah imperatif (`buat list kosong, loop, append kalau syarat terpenuhi`). Ini kesempatan bagus menyambungkan kembali insight minggu lalu — developer yang connect kedua hal ini biasanya langsung lancar pakai comprehension setelahnya.

## Konsep & Sintaks

### 4 Struktur Data Inti

| Python | Padanan di bahasa lain | Sifat |
|---|---|---|
| `list` | Array (dynamic) | Urutan terjaga (ordered), **mutable**, boleh duplikat |
| `tuple` | Array immutable / fixed record | Urutan terjaga, **immutable**, boleh duplikat |
| `set` | `Set`/`HashSet` | **Tidak** ada urutan pasti, elemen unik, mutable |
| `dict` | Object (JS) / `Map`/`HashMap` | Key-value, key harus unique & hashable |

```python
product_ids = [1, 2, 3, 1]          # list — boleh duplikat, urutan terjaga
coordinate = (3, 4)                  # tuple — biasa dipakai untuk data majemuk yang gak akan diubah
unique_categories = {"Electronics", "Furniture", "Electronics"}  # jadi {"Electronics", "Furniture"} — duplikat otomatis hilang
product = {"product_id": 1, "product_name": "Wireless Mouse"}    # dict
```

**Kapan pakai yang mana**: `list` untuk koleksi yang bisa berubah & urutannya penting. `tuple` untuk data majemuk yang "selesai dibentuk" dan tidak boleh berubah (mis. koordinat, atau return value majemuk dari function seperti di Hari 1). `set` saat yang penting cuma **keunikan** dan butuh operasi himpunan (`&` irisan, `|` gabungan, `-` selisih) — mis. cek negara mana yang ada di kedua sumber data. `dict` untuk lookup cepat by key (analog hash map — O(1) rata-rata, dibanding `list` yang harus di-scan O(n) untuk cari elemen tertentu).

### List/Dict/Set Comprehension

```python
[expr for item in iterable if condition]      # list comprehension
{key_expr: value_expr for item in iterable}   # dict comprehension
{expr for item in iterable}                   # set comprehension
```

```python
# List comprehension: nama produk di atas $20
expensive = [p["product_name"] for p in products if p["unit_price"] > 20]

# Dict comprehension: lookup product_id -> product_name (dipakai lagi Hari 4 sebagai alternatif merge)
id_to_name = {p["product_id"]: p["product_name"] for p in products}

# Set comprehension: kategori unik
categories = {p["category"] for p in products}

# Dict comprehension dengan filter: harga produk Electronics saja
electronics_prices = {p["product_name"]: p["unit_price"] for p in products if p["category"] == "Electronics"}
```

### `enumerate` dan `zip`

```python
for i, p in enumerate(products, start=1):     # butuh index + elemen
    print(i, p["product_name"])

names = [p["product_name"] for p in products]
prices = [p["unit_price"] for p in products]
for name, price in zip(names, prices):        # gabungkan 2 list sejajar jadi pasangan
    print(name, price)
```

### Unpacking

```python
cheapest, priciest = min(prices), max(prices)     # tuple unpacking
first, *rest = [1, 2, 3, 4]                        # first=1, rest=[2, 3, 4]
```

## Kesalahan Umum

1. **Memodifikasi list saat sedang di-loop di atasnya — bug diam-diam, tidak selalu error.**
   ```python
   # SALAH:
   items = [2, 4, 6, 8, 10]
   for x in items:
       if x % 2 == 0:
           items.remove(x)
   print(items)   # [4, 8] <- BUKAN [] seperti yang diharapkan!
   ```
   Menghapus elemen saat iterasi membuat index internal iterator "meleset" — sebagian elemen ter-skip karena list bergeser setelah penghapusan. Solusi: bikin list baru (paling idiomatik, pakai comprehension), atau iterasi di atas salinan (`items[:]`):
   ```python
   items = [x for x in items if x % 2 != 0]   # cara paling idiomatik
   # atau: for x in items[:]: ...
   ```

2. **Coba mengubah isi `tuple`.** `tuple` immutable — `my_tuple[0] = 5` akan `TypeError`. Kalau butuh mengubah isi, itu tandanya harusnya pakai `list`, bukan `tuple`.

3. **`dict` key harus hashable — tidak bisa pakai `list` sebagai key.** `{[1,2]: "value"}` akan error (`TypeError: unhashable type`). Kalau butuh key majemuk, pakai `tuple`: `{(1,2): "value"}` valid karena tuple immutable & hashable.

4. **Mengandalkan urutan elemen `set`.** `set` tidak menjamin urutan tertentu (walau di praktiknya sering terlihat "konsisten" untuk tipe data tertentu — jangan andalkan itu). Kalau urutan penting, pakai `list`.

5. **Nested comprehension yang terlalu padat — merusak keterbacaan.** Comprehension bagus untuk transformasi 1 langkah yang jelas. Begitu logikanya butuh lebih dari 1 kondisi bertingkat atau nested loop, `for` loop biasa (atau memecah jadi beberapa langkah) justru lebih terbaca. Aturan praktis: kalau comprehension-nya butuh dibaca 2 kali untuk dipahami, tulis ulang jadi loop biasa.

## Latihan

Pakai `products` (7 item, sama seperti Hari 1) dan data tambahan berikut:
```python
orders_summary = [
    {"order_id": 1, "customer_id": 1, "total": 40.5},
    {"order_id": 2, "customer_id": 2, "total": 61.0},
    {"order_id": 4, "customer_id": 3, "total": 128.0},
    {"order_id": 9, "customer_id": 6, "total": 240.0},
]
```

1. Buat list comprehension: nama produk dengan `category == "Electronics"`.
2. Buat dict comprehension: `order_id` → `total` dari `orders_summary`.
3. Buat set comprehension: semua `category` yang muncul di `products` (harus otomatis unik).
4. Pakai `zip` dan `enumerate` bersamaan: cetak `"1. Wireless Mouse - $15.00"`, dst, untuk semua produk (nomor urut mulai dari 1).
5. Ada 2 dict: `dict_a = {"Indonesia", "UK", "Germany"}` (set, bukan dict — sengaja) dan `dict_b = {"UK", "Germany", "Singapore"}`. Cari negara yang ada di **kedua** set itu, dan negara yang ada di `dict_a` tapi **tidak** di `dict_b` — pakai operator himpunan (`&`, `-`), bukan loop manual.

## Kunci Jawaban & Pembahasan

**1.**
```python
[p["product_name"] for p in products if p["category"] == "Electronics"]
```
Hasil: `['Wireless Mouse', 'Mechanical Keyboard', 'USB-C Hub']`.

**2.**
```python
{o["order_id"]: o["total"] for o in orders_summary}
```
Hasil: `{1: 40.5, 2: 61.0, 4: 128.0, 9: 240.0}`.

**3.**
```python
{p["category"] for p in products}
```
Hasil: `{'Electronics', 'Furniture', 'Stationery', 'Kitchen'}` (urutan tampil boleh beda tiap run — itu memang sifat `set`, bukan bug).

**4.**
```python
for i, p in enumerate(products, start=1):
    print(f'{i}. {p["product_name"]} - ${p["unit_price"]:.2f}')
```
(Tidak butuh `zip` sebenarnya kalau datanya sudah 1 list of dict — ini soal jebakan kecil: kadang solusi paling sederhana tidak butuh semua tool yang disebut di soal. Versi dengan `zip` kalau nama & harga terpisah di 2 list berbeda:)
```python
names = [p["product_name"] for p in products]
prices = [p["unit_price"] for p in products]
for i, (name, price) in enumerate(zip(names, prices), start=1):
    print(f"{i}. {name} - ${price:.2f}")
```

**5.**
```python
dict_a = {"Indonesia", "UK", "Germany"}
dict_b = {"UK", "Germany", "Singapore"}

irisan = dict_a & dict_b        # {'UK', 'Germany'}
hanya_di_a = dict_a - dict_b    # {'Indonesia'}
```
Ini kesempatan bagus menekankan: operasi himpunan (`&`, `|`, `-`) di `set` sangat berguna untuk kebutuhan data engineering sehari-hari seperti "negara yang ada di sumber data A tapi tidak di B" — konsep yang sama dengan pembahasan `FULL JOIN` untuk rekonsiliasi data di `materi/minggu_1/hari_3_join.md`, cuma di sini levelnya per-value, bukan per-tabel.
