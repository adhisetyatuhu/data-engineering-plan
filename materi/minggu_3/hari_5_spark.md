---
title: Hari 5 - Pengantar Apache Spark
parent: Minggu 3 - Arsitektur Data & Big Data Pipeline
nav_order: 6
---

# Hari 5 — Pengantar Apache Spark: Arsitektur, RDD vs DataFrame, PySpark Dasar

*Jumat, 2 jam. Dataset: dataset mini Minggu 1 (lihat `materi/minggu_1/00_overview.md`) — Online Retail II baru dipakai penuh mulai Sabtu.*

> Hari ini local mode saja, dataset kecil, tujuannya membangun **kebiasaan sintaks** PySpark sebelum dipakai di volume data sungguhan besok. Jangan buru-buru ke Online Retail II hari ini — dataset kecil dari Minggu 1 justru pas karena hasilnya sudah diketahui & bisa diverifikasi manual (sama seperti alasan dataset itu dipakai terus sepanjang Minggu 1–2).

## Tujuan Belajar

- [ ] Menjelaskan komponen arsitektur Spark: driver, executor, cluster manager
- [ ] Menjelaskan beda RDD dan DataFrame, dan kenapa DataFrame jadi default pilihan untuk data terstruktur
- [ ] Menjelaskan konsep **lazy evaluation**: transformation vs action
- [ ] Membaca CSV, melakukan filter/select/groupBy/join dasar dengan PySpark DataFrame API
- [ ] Memetakan sintaks Pandas (Minggu 2) ke PySpark

## Untuk Instruktur: Mindset Shift

Trik utama hari ini sama seperti Minggu 2 (Pandas dijembatani dari SQL): **PySpark DataFrame API dijembatani dari Pandas**, karena sengaja dirancang mirip. Kalau peserta sudah lancar Pandas, sintaks PySpark akan terasa familiar — yang benar-benar baru adalah **model eksekusinya** (lazy evaluation, terdistribusi), bukan sintaksnya.

Analogi arsitektur yang efektif: bayangkan **load balancer + worker pool** di sistem backend yang sudah familiar buat developer.
- **Driver** ≈ proses utama yang menerima "request" (kode Spark kamu), menyusun rencana kerja, membagi tugas.
- **Cluster Manager** ≈ orchestrator (mirip Kubernetes scheduler) yang mengalokasikan resource untuk worker.
- **Executor** ≈ worker process yang benar-benar mengerjakan potongan tugas, paralel satu sama lain.

Di **local mode** (yang dipakai sepanjang minggu ini), semua peran ini jalan di **1 mesin** — driver dan beberapa executor sama-sama proses lokal, cuma memanfaatkan banyak core CPU, bukan banyak mesin fisik. Ini penting ditekankan supaya peserta tidak salah kira sedang "pakai cluster sungguhan" — local mode murni untuk belajar & development, cluster mode (banyak mesin fisik) baru relevan untuk data produksi skala besar (di luar cakupan roadmap 8 minggu ini).

## Konsep & Sintaks

### Kenapa Spark, Bukan Pandas Saja?

Pandas memuat **seluruh data ke memori 1 mesin** — bekerja baik untuk data yang muat di RAM (ratusan ribu–jutaan baris, seperti Online Retail II yang dipakai Minggu 2). Begitu data terlalu besar untuk RAM 1 mesin (miliaran baris, atau butuh diproses lebih cepat dari yang bisa dilakukan 1 mesin), Spark membagi data & pekerjaan ke **banyak mesin (cluster) sekaligus**, memproses secara paralel.

Untuk mini project minggu ini, datanya (Online Retail II, ratusan ribu baris) **sebenarnya masih muat nyaman di Pandas** — dipilih Spark murni untuk **latihan tool dan pola pikir** yang dipakai di skala data yang jauh lebih besar di dunia kerja nyata, bukan karena datanya benar-benar butuh Spark di skala ini. Penting untuk jujur soal ini ke peserta — jangan sampai kesan "Spark selalu lebih baik dari Pandas", padahal untuk data yang muat di 1 mesin, Pandas sering kali lebih simpel dan lebih cepat (tidak ada overhead distribusi).

### Arsitektur Spark

```
                    Driver Program
                (kode Spark kamu jalan di sini,
                 menyusun rencana eksekusi/DAG)
                          |
                  Cluster Manager
           (alokasikan resource ke executor —
         di local mode: cuma "pura-pura" di 1 mesin)
                          |
        +---------+-------+-------+---------+
        |         |               |         |
    Executor 1  Executor 2  Executor 3  Executor N
   (proses potongan   (proses potongan   dst.
    data paralel)       data paralel)
```

- **Driver** — menjalankan kode Spark kamu (`SparkSession`, definisi transformasi), menyusun **DAG** (directed acyclic graph — urutan langkah kerja, konsepnya sama dengan DAG di Airflow, dibahas lagi Sabtu–Minggu), lalu membagi kerja ke executor.
- **Executor** — proses yang benar-benar menjalankan tugas (baca partisi data, filter, agregasi parsial), melapor balik ke driver.
- **Cluster Manager** — mengatur alokasi resource (CPU/memory) untuk executor. Di local mode, ini disederhanakan; di production, biasanya YARN, Kubernetes, atau Spark Standalone.

### RDD vs DataFrame

| | RDD (Resilient Distributed Dataset) | DataFrame |
|---|---|---|
| Level abstraksi | Rendah — koleksi objek Python terdistribusi, tanpa skema | Tinggi — tabel dengan skema (nama & tipe kolom), mirip Pandas DataFrame / tabel SQL |
| API | Mirip operasi functional Python (`.map()`, `.filter()`, `.reduce()`) | Mirip Pandas/SQL (`.select()`, `.filter()`, `.groupBy()`) |
| Optimasi | Tidak ada optimasi otomatis — kamu yang bertanggung jawab menulis kode efisien | **Catalyst Optimizer** otomatis menyusun ulang rencana eksekusi jadi lebih efisien sebelum benar-benar dijalankan |
| Kapan dipakai | Data tidak terstruktur, butuh kontrol sangat low-level | **Default pilihan** untuk data terstruktur/semi-terstruktur (kasus kita) |

**Rekomendasi jelas untuk roadmap ini dan kebanyakan pekerjaan data engineering modern: pakai DataFrame API, bukan RDD.** RDD tetap ada "di bawah" DataFrame (DataFrame dibangun di atas RDD), tapi jarang ditulis langsung kecuali kebutuhan sangat spesifik (custom logic yang tidak bisa diekspresikan lewat operasi DataFrame). Analogi: RDD itu seperti menulis SQL manual query plan sendiri, DataFrame itu seperti menulis SQL biasa dan biarkan query optimizer database yang urus efisiensinya — hampir selalu lebih baik percaya ke optimizer kecuali ada alasan kuat tidak.

### Lazy Evaluation: Transformation vs Action

Ini konsep paling penting & paling sering bikin bingung pemula Spark, karena **berbeda dari Pandas** (yang dieksekusi baris kode langsung saat itu juga, disebut *eager evaluation*).

- **Transformation** (`select`, `filter`, `groupBy`, `join`, dst.) — **tidak langsung dieksekusi**. Spark cuma mencatat "rencana" yang harus dilakukan (membangun DAG), belum benar-benar memproses data.
- **Action** (`show`, `count`, `collect`, `write`) — **memicu eksekusi sungguhan** dari semua transformation yang sudah "diantre" sebelumnya.

```python
df_filtered = df.filter(df.category == "Electronics")   # transformation -> BELUM jalan
df_selected = df_filtered.select("product_name", "unit_price")  # transformation -> BELUM jalan
df_selected.show()   # action -> BARU SEKARANG semuanya benar-benar dieksekusi
```

Analogi kode paling pas buat developer: ini persis seperti **JavaScript Promise/generator yang belum di-`await`**, atau **query builder** (mis. Knex/SQLAlchemy) yang menyusun query tapi belum dieksekusi sampai `.execute()`/`.all()` dipanggil. Keuntungannya: Spark bisa melihat **seluruh rencana** sebelum eksekusi, lalu mengoptimalkan (mis. menggabungkan beberapa filter jadi satu pass, membuang kolom yang tidak dipakai lebih awal) — sesuatu yang tidak mungkin dilakukan kalau tiap baris kode dieksekusi terpisah begitu ditulis (gaya Pandas).

### Setup & Baca Data

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("belajar-spark-minggu-3") \
    .master("local[*]") \
    .getOrCreate()

# local[*]  -> jalankan di local mode, pakai semua core CPU yang tersedia

products = spark.read.csv("data/products.csv", header=True, inferSchema=True)
products.printSchema()   # lihat skema (nama kolom + tipe data) yang berhasil dideteksi
products.show(5)         # action -> tampilkan 5 baris pertama (padanan df.head())
```

`inferSchema=True` membuat Spark membaca sebagian data dulu untuk menebak tipe kolom — nyaman untuk belajar, tapi **untuk pipeline production sebaiknya skema didefinisikan eksplisit** (`StructType`) supaya bisa dipastikan mendadak tidak jadi lambat/salah kalau tipe data di file sumber berubah tanpa disadari. Poin ini akan relevan lagi saat bahas Data Quality di Minggu 4.

### Pemetaan Sintaks: Pandas → PySpark

| Pandas (Minggu 2) | PySpark |
|---|---|
| `pd.read_csv(path)` | `spark.read.csv(path, header=True, inferSchema=True)` |
| `df.head(n)` | `df.show(n)` |
| `df.shape` | `(df.count(), len(df.columns))` |
| `df[['col1','col2']]` | `df.select('col1', 'col2')` |
| `df[df.col > 20]` | `df.filter(df.col > 20)` atau `df.where(df.col > 20)` |
| `df.sort_values('col', ascending=False)` | `df.orderBy(df.col.desc())` |
| `df.groupby('col').agg(...)` | `df.groupBy('col').agg(...)` |
| `df1.merge(df2, on='key')` | `df1.join(df2, on='key')` |
| `df['new_col'] = ...` | `df.withColumn('new_col', ...)` |

**Beda penting yang wajib ditekankan**: di Pandas, `df['new_col'] = expr` **mengubah** `df` yang ada (mutasi in-place, atau assignment ke variabel yang sama). Di PySpark, `df.withColumn(...)` **mengembalikan DataFrame baru** — DataFrame itu **immutable** (tidak bisa diubah setelah dibuat), sama seperti string di Python yang immutable. Ini alasan pola penulisan PySpark selalu terlihat seperti *method chaining* (`df.filter(...).select(...).withColumn(...)`) — tiap langkah menghasilkan DataFrame baru, bukan memodifikasi yang lama.

## Contoh Kode

Memakai dataset `products.csv`, `order_items.csv` dari Minggu 1 (skema & isi identik dengan `materi/minggu_1/00_overview.md`).

**1. Select + Filter — padanan `WHERE category = 'Electronics' AND unit_price > 20`**
```python
products = spark.read.csv("data/products.csv", header=True, inferSchema=True)

result = products.filter(
    (products.category == "Electronics") & (products.unit_price > 20)
).select("product_name", "unit_price")

result.show()
```
```
+-------------------+----------+
|product_name       |unit_price|
+-------------------+----------+
|Mechanical Keyboard|45.00     |
|USB-C Hub          |25.00     |
+-------------------+----------+
```
Sama persis hasilnya dengan contoh #3 di `materi/minggu_2/hari_3_pandas_dasar.md` — bukti langsung ke peserta bahwa logikanya identik, cuma sintaksnya beda.

**2. GroupBy + Agregasi — revenue per kategori**
```python
from pyspark.sql import functions as F

products = spark.read.csv("data/products.csv", header=True, inferSchema=True)
order_items = spark.read.csv("data/order_items.csv", header=True, inferSchema=True)

revenue_per_category = (
    products.join(order_items, on="product_id", how="left")
    .withColumn("revenue", F.coalesce(order_items.quantity * order_items.unit_price, F.lit(0)))
    .groupBy("category")
    .agg(F.sum("revenue").alias("total_revenue"))
    .orderBy(F.col("total_revenue").desc())
)

revenue_per_category.show()
```
```
+-----------+-------------+
|category   |total_revenue|
+-----------+-------------+
|Furniture  |360.00       |
|Electronics|320.00       |
|Kitchen    |64.00        |
|Stationery |63.00        |
+-----------+-------------+
```
Hasil ini **identik** dengan menjumlahkan angka per produk di contoh #2 `materi/minggu_1/hari_5_window_function.md` (Furniture: Office Chair 360 + Desk Lamp 0 = 360; Electronics: Mechanical Keyboard 180 + Wireless Mouse 90 + USB-C Hub 50 = 320) — cara ketiga (SQL, Pandas, sekarang PySpark) untuk pertanyaan yang sama, hasilnya wajib sama sebagai self-check.

**3. Import fungsi (`pyspark.sql.functions`)** — perhatikan `F.col()`, `F.sum()`, `F.coalesce()` di atas: hampir semua fungsi non-trivial di PySpark (agregasi, fungsi tanggal/string, kondisional) diimpor dari modul `pyspark.sql.functions` (konvensi alias `F`) — **bukan** fungsi Python biasa. Ini beda dari Pandas yang sering pakai method langsung di `Series`/fungsi NumPy.

## Kesalahan Umum

1. **Lupa `pyspark.sql.functions` dan mencoba pakai fungsi Python biasa** (mis. `sum()`, `len()`) langsung ke kolom Spark — akan error, karena kolom Spark (`Column` object) bukan list/array Python biasa, dan operasinya harus lewat fungsi Spark yang tahu cara menjalankannya **terdistribusi** di banyak executor.
2. **Kaget kenapa kode "tidak jalan" padahal tidak error** — ini gejala lazy evaluation: transformation yang ditulis tanpa action setelahnya **tidak benar-benar dieksekusi**, jadi tidak ada error tapi juga tidak ada hasil terlihat. Ingatkan: harus ada `.show()`/`.count()`/`.write()` di akhir chain untuk memicu eksekusi.
3. **Menganggap DataFrame PySpark bisa dimutasi seperti Pandas.** `df['col'] = ...` **tidak valid** di PySpark (ini sintaks Pandas). Harus pakai `df.withColumn('col', ...)` yang mengembalikan DataFrame baru — kalau ingin "menyimpan" hasilnya, harus di-assign ulang: `df = df.withColumn(...)`.
4. **Terlalu sering memanggil action (`.show()`, `.count()`) di tengah chain panjang untuk "debugging".** Tiap action memicu eksekusi ulang **dari awal DAG** (kecuali data di-cache eksplisit) — untuk dataset besar, ini bisa jadi lambat kalau dilakukan berulang-ulang tanpa perlu. Cukup panggil action di titik yang benar-benar perlu dilihat hasilnya.

## Latihan

1. Baca `customers.csv`, filter customer dengan `country == 'Indonesia'`, tampilkan kolom `customer_name` dan `country` saja. Bandingkan hasilnya dengan contoh #1 di `materi/minggu_2/hari_3_pandas_dasar.md`.
2. Baca `products.csv`, urutkan berdasarkan `unit_price` menurun, tampilkan 3 baris teratas (padanan `ORDER BY unit_price DESC LIMIT 3`).
3. Jelaskan dengan kata-kata sendiri (tidak perlu kode): kalau kamu menulis 5 baris `.filter()`/`.select()` berantai lalu **tidak** memanggil `.show()`/`.count()` di baris ke-6, apa yang terjadi saat script dijalankan? Kenapa?
4. Hitung total quantity terjual per produk (`order_items` di-`groupBy` `product_id`, `agg` `F.sum("quantity")`), lalu join dengan `products` untuk menampilkan `product_name`-nya. Urutkan dari quantity terbanyak. (Petunjuk: hasilnya harus cocok dengan tabel `total_qty` di contoh #1 `materi/minggu_1/hari_5_window_function.md`.)

## Kunci Jawaban & Pembahasan

**1.**
```python
customers = spark.read.csv("data/customers.csv", header=True, inferSchema=True)
customers.filter(customers.country == "Indonesia").select("customer_name", "country").show()
```
Hasil: Andi Wijaya, Budi Santoso, Citra Lestari — identik dengan versi Pandas di `materi/minggu_2/hari_3_pandas_dasar.md` contoh #1.

**2.**
```python
products = spark.read.csv("data/products.csv", header=True, inferSchema=True)
products.orderBy(products.unit_price.desc()).select("product_name", "unit_price").show(3)
```
Hasil: Office Chair (120.00), Mechanical Keyboard (45.00), Desk Lamp (30.00) — identik dengan hasil versi SQL/Pandas yang sudah diverifikasi di minggu-minggu sebelumnya.

**3.** **Tidak terjadi apa-apa yang terlihat** — script akan selesai jalan tanpa error dan tanpa output apapun terkait 5 baris transformation itu. Ini karena `.filter()`/`.select()` adalah **transformation**, yang cuma dicatat sebagai rencana (DAG) di driver, bukan langsung dieksekusi. Tanpa action (`.show()`/`.count()`/`.write()`/dst.) yang memicu eksekusi, rencana itu tidak pernah benar-benar dijalankan terhadap data — mirip menyusun query builder tapi tidak pernah memanggil `.execute()`.

**4.**
```python
order_items = spark.read.csv("data/order_items.csv", header=True, inferSchema=True)
products = spark.read.csv("data/products.csv", header=True, inferSchema=True)

qty_per_product = (
    order_items.groupBy("product_id")
    .agg(F.sum("quantity").alias("total_qty"))
)

result = (
    products.join(qty_per_product, on="product_id", how="left")
    .withColumn("total_qty", F.coalesce(F.col("total_qty"), F.lit(0)))
    .select("product_name", "total_qty")
    .orderBy(F.col("total_qty").desc())
)
result.show()
```
Hasil: Notebook A5 (18), Coffee Mug (8), Wireless Mouse (7), Mechanical Keyboard (4), USB-C Hub (4), Office Chair (3), Desk Lamp (0) — persis tabel `total_qty` di contoh #1 `materi/minggu_1/hari_5_window_function.md`. `how="left"` dan `F.coalesce(..., F.lit(0))` dipakai supaya Desk Lamp (belum pernah terjual, tidak muncul di `order_items`) tetap muncul dengan `total_qty = 0`, bukan hilang dari hasil — pola yang sama persis dengan `LEFT JOIN` + `COALESCE` yang sudah dipelajari di Minggu 1.
