# Hari 1 — Konsep Database Relasional, SELECT, WHERE, ORDER BY, LIMIT

*Senin, 2 jam. Dataset & setup: lihat `00_overview.md`.*

## Tujuan Belajar

Di akhir sesi ini, peserta bisa:
- [ ] Menjelaskan apa itu tabel, row, column, primary key, foreign key
- [ ] Menulis query `SELECT` dengan filter (`WHERE`), urutan (`ORDER BY`), dan pembatasan jumlah hasil (`LIMIT`)
- [ ] Menjelaskan urutan eksekusi logis sebuah query SQL
- [ ] Menghindari 3 kesalahan umum pemula (NULL comparison, alias di WHERE, string case sensitivity)

## Untuk Instruktur: Mindset Shift

Ini sesi paling penting di minggu ini secara *fondasi mental*, walau secara sintaks paling sederhana. Kalau mental model di sini gak klik, sisa minggu akan terasa seperti hafalan sintaks doang.

**Poin utama: SQL itu declarative, kode yang biasa mereka tulis itu imperative.**

Buka dengan pertanyaan: *"Kalau saya kasih kalian array of objects `users`, dan saya minta 'ambil yang `country == 'Indonesia'`, urutkan by `signup_date` terbaru, ambil 5 teratas' — gimana kalian nulisnya di JS/Python?"*

Mereka akan jawab sesuatu seperti:
```js
users.filter(u => u.country === 'Indonesia')
     .sort((a, b) => b.signup_date - a.signup_date)
     .slice(0, 5)
```

Lalu tunjukkan versi SQL-nya (nanti di bagian contoh query), dan tekankan: **struktur mentalnya sama** — filter, sort, limit — cuma **notasinya beda arah** dan **kamu gak nulis cara komputer melakukannya**, cuma nulis hasil akhir yang kamu mau. Database engine yang cari cara paling efisien (nanti masuk lagi ke `EXPLAIN` di akhir minggu).

Pertanyaan yang biasanya muncul dari developer, siapkan jawabannya:
- *"Kenapa urutan nulis `SELECT ... FROM ... WHERE` gak sesuai urutan eksekusi?"* → jawab di bagian "Urutan Eksekusi" di bawah.
- *"Ini kayak method chaining ya?"* → ya, secara konsep. Bedanya di SQL tiap "method" (klausa) posisinya fixed, gak bisa dipanggil sembarang urutan seperti `.filter().sort()` vs `.sort().filter()`.

## Konsep & Sintaks

### Tabel = Array of Objects yang Konsisten Bentuknya

| Istilah SQL | Padanan di Kode |
|---|---|
| Table | Array/list of objects dengan struktur (schema) tetap |
| Row (record) | Satu object/elemen di array itu |
| Column (field) | Property di object itu, dengan tipe data yang sudah ditentukan |
| Primary Key (PK) | Property `id` yang unik & wajib ada, dipakai buat referensi |
| Foreign Key (FK) | Property yang isinya `id` dari object di "array" lain (relasi) |

Bedanya dengan array of objects biasa di kode: **schema-nya dipaksakan** (tiap row harus punya kolom yang sama, tipe data konsisten) — beda dari objek JS yang bisa punya field berbeda-beda antar elemen.

### Anatomi Query

```sql
SELECT column1, column2
FROM table_name
WHERE condition
ORDER BY column1 [ASC|DESC]
LIMIT n;
```

### Urutan Penulisan vs Urutan Eksekusi (PENTING)

Ini yang paling sering bikin developer bingung: **urutan kamu nulis klausa ≠ urutan database mengeksekusinya.**

| Urutan Ditulis | Urutan Dieksekusi |
|---|---|
| `SELECT` | 1. `FROM` (ambil tabel) |
| `FROM` | 2. `WHERE` (filter row) |
| `WHERE` | 3. `GROUP BY` (kalau ada — masuk Hari 2) |
| `ORDER BY` | 4. `SELECT` (baru pilih/hitung kolom) |
| `LIMIT` | 5. `ORDER BY` (urutkan hasil) |
| | 6. `LIMIT` (potong hasil) |

**Konsekuensi praktis**: kamu **tidak bisa** pakai alias yang didefinisikan di `SELECT` untuk filter di `WHERE`, karena `WHERE` dieksekusi *sebelum* `SELECT` sempat menghitung alias itu.

```sql
-- ERROR di kebanyakan database:
SELECT unit_price * 1.1 AS price_with_tax
FROM products
WHERE price_with_tax > 20;   -- alias belum "ada" saat WHERE jalan

-- Solusi: ulang ekspresinya, atau pakai subquery/CTE (nanti Hari 4)
SELECT unit_price * 1.1 AS price_with_tax
FROM products
WHERE unit_price * 1.1 > 20;
```

Tapi alias **bisa** dipakai di `ORDER BY`, karena `ORDER BY` dieksekusi setelah `SELECT`.

## Contoh Query

Semua contoh pakai dataset di `00_overview.md`.

**1. Ambil semua kolom (analog: `console.log(users)`)**
```sql
SELECT * FROM customers;
```

**2. Pilih kolom tertentu + filter (analog: `.filter()` lalu `.map()`)**
```sql
SELECT customer_name, country
FROM customers
WHERE country = 'Indonesia';
```
| customer_name | country |
|---|---|
| Andi Wijaya | Indonesia |
| Budi Santoso | Indonesia |
| Citra Lestari | Indonesia |

**3. Filter dengan operator perbandingan & logika**
```sql
SELECT order_id, order_date, status
FROM orders
WHERE status != 'completed';
```
| order_id | order_date | status |
|---|---|---|
| 6 | 2024-02-15 | cancelled |
| 11 | 2024-03-20 | refunded |

```sql
SELECT product_name, category, unit_price
FROM products
WHERE category = 'Electronics' AND unit_price > 20;
```
| product_name | category | unit_price |
|---|---|---|
| Mechanical Keyboard | Electronics | 45.00 |
| USB-C Hub | Electronics | 25.00 |

**4. `ORDER BY` + `LIMIT` (analog: `.sort().slice(0, n)`)**
```sql
SELECT product_name, unit_price
FROM products
ORDER BY unit_price DESC
LIMIT 3;
```
| product_name | unit_price |
|---|---|
| Office Chair | 120.00 |
| Mechanical Keyboard | 45.00 |
| Desk Lamp | 30.00 |

**5. `BETWEEN`, `IN`, `LIKE` — operator yang sering dipakai**
```sql
SELECT order_id, order_date FROM orders
WHERE order_date BETWEEN '2024-02-01' AND '2024-02-29';

SELECT product_name FROM products
WHERE category IN ('Electronics', 'Kitchen');

SELECT customer_name FROM customers
WHERE customer_name LIKE 'A%';  -- diawali huruf A
```

## Kesalahan Umum

1. **`= NULL` tidak pernah `TRUE`.** NULL berarti "tidak diketahui", bukan nilai. Developer biasa `if (x === null)` di kode — di SQL harus pakai `IS NULL` / `IS NOT NULL`.
   ```sql
   -- SALAH, tidak akan pernah return apapun:
   SELECT * FROM orders WHERE order_date = NULL;
   -- BENAR:
   SELECT * FROM orders WHERE order_date IS NULL;
   ```

2. **String matching case-sensitive tergantung database/collation.** `'indonesia' = 'Indonesia'` bisa `FALSE` di Postgres default. Kalau butuh case-insensitive: `WHERE LOWER(country) = 'indonesia'` atau `ILIKE`.

3. **Alias di `SELECT` tidak bisa dipakai di `WHERE`** (lihat penjelasan urutan eksekusi di atas) — tapi bisa di `ORDER BY`.

4. **Lupa `LIMIT` saat eksplorasi tabel besar.** Kebiasaan `SELECT *` tanpa `LIMIT` di tabel jutaan baris = query lambat/nge-hang. Selalu `LIMIT 10` dulu saat eksplorasi awal.

## Latihan

Pakai dataset `00_overview.md`.

1. Tampilkan semua produk dengan `unit_price` di bawah 20.
2. Tampilkan `customer_name` dan `signup_date` dari customer yang mendaftar setelah `2023-03-01`, diurutkan dari yang paling lama daftar.
3. Tampilkan semua order dengan status `'completed'` yang terjadi di bulan Maret 2024.
4. Tampilkan 3 produk termurah, tampilkan juga kolom `category`.
5. Tampilkan customer yang negaranya BUKAN `'Indonesia'` DAN bukan `'UK'`.

## Kunci Jawaban & Pembahasan

**1.**
```sql
SELECT * FROM products WHERE unit_price < 20;
```
Hasil: Wireless Mouse (15.00), Notebook A5 (3.50), Coffee Mug (8.00).

**2.**
```sql
SELECT customer_name, signup_date
FROM customers
WHERE signup_date > '2023-03-01'
ORDER BY signup_date ASC;
```
Perhatikan: `>` bukan `>=`, jadi Citra yang signup persis `2023-03-01` **tidak** ikut — ini jebakan sengaja untuk latih ketelitian operator `>` vs `>=`.
Hasil: Frank Muller (2023-04-05), Grace Tan (2023-05-12), Hassan Ali (2023-06-01).

**3.**
```sql
SELECT * FROM orders
WHERE status = 'completed'
  AND order_date BETWEEN '2024-03-01' AND '2024-03-31';
```
Hasil: order 8, 9, 10, 12 (order 11 di bulan Maret tapi statusnya `refunded`, jadi tidak ikut).

**4.**
```sql
SELECT product_name, category, unit_price
FROM products
ORDER BY unit_price ASC
LIMIT 3;
```
Hasil: Notebook A5 (Stationery, 3.50), Coffee Mug (Kitchen, 8.00), Wireless Mouse (Electronics, 15.00).

**5.**
```sql
SELECT * FROM customers
WHERE country != 'Indonesia' AND country != 'UK';
-- atau: WHERE country NOT IN ('Indonesia', 'UK')
```
Hasil: Frank Muller (Germany), Grace Tan (Singapore), Hassan Ali (UAE).
