---
title: Hari 2 - Pengantar Great Expectations
parent: Minggu 4 - Data Quality, Orchestration & Streaming
nav_order: 3
---

# Hari 2 — Tools Data Quality: Pengantar Great Expectations

*Selasa, 2 jam. Dataset: dataset mini Minggu 1 (`products.csv`) — dipilih supaya hasil validasi mudah diverifikasi manual sebelum dipakai ke data sungguhan di mini project.*

> Versi yang dipakai di seluruh modul ini: `great_expectations==0.18.*` (Fluent API). Great Expectations termasuk tool yang API-nya berubah cukup sering antar versi major — kalau ada method yang tidak ketemu persis, cek [docs resmi](https://docs.greatexpectations.io/) untuk versi yang terpasang, konsepnya (Expectation, Suite, Validator, Checkpoint) tetap sama walau detail pemanggilannya bisa sedikit berbeda.

## Tujuan Belajar

- [ ] Menjelaskan 4 konsep inti GE: **Expectation**, **Expectation Suite**, **Validator**, **Checkpoint**
- [ ] Menerjemahkan aturan data quality (dari `hari_1_data_quality_dimensions.md`) jadi Expectation GE yang konkret
- [ ] Menjalankan validasi terhadap sebuah DataFrame dan membaca hasilnya (`success`/`unexpected_count`)
- [ ] Menjelaskan kenapa dipakai tool khusus (GE), bukan sekadar `assert` manual di kode Python

## Untuk Instruktur: Mindset Shift

Analogi paling pas: **Great Expectations adalah "pytest" untuk data**, bukan untuk kode.

```python
# pytest -> menguji KODE
def test_calculate_total():
    assert calculate_total(quantity=2, price=5) == 10

# Great Expectations -> menguji DATA
validator.expect_column_values_to_not_be_null("customer_id")
validator.expect_column_values_to_be_between("unit_price", min_value=0, exclusive_min=True)
```

Kenapa tidak cukup `assert` manual (`assert df["unit_price"].min() > 0`)? Beberapa alasan yang perlu ditekankan:
1. **Hasil terstruktur & bisa dibaca ulang** — GE menghasilkan report (`success: True/False`, jumlah baris yang gagal, contoh baris yang gagal) yang bisa disimpan, ditampilkan di UI, atau dipakai untuk alerting — bukan cuma `AssertionError` polos yang menghentikan program.
2. **Expectation Suite = dokumentasi hidup** — sekumpulan expectation yang tersimpan (`ecommerce_suite.json` di mini project) sekaligus **mendokumentasikan** kontrak kualitas data yang disepakati, bisa dibaca siapa saja tanpa baca kode.
3. **Reusable** — satu suite yang sama bisa dijalankan ulang tiap pipeline run, tanpa menulis ulang logic pemeriksaan tiap kali.

## Konsep & Sintaks

### 4 Konsep Inti

| Konsep | Analog di Software Testing | Definisi |
|---|---|---|
| **Expectation** | 1 `assert` statement | 1 aturan spesifik tentang data (mis. "kolom ini tidak boleh null") |
| **Expectation Suite** | 1 file test (`test_*.py`) berisi banyak `assert` | Kumpulan Expectation untuk 1 dataset/tabel |
| **Validator** | Test runner yang tahu **data apa** yang mau ditest | Objek yang menghubungkan Expectation Suite dengan data sungguhan (DataFrame/tabel) untuk dijalankan |
| **Checkpoint** | CI job yang menjalankan test suite + melapor hasil | Konfigurasi siap pakai untuk menjalankan validasi + aksi setelahnya (simpan hasil, dsb) — dipakai berulang, terutama dari Airflow |

### Setup & Membuat Validator

```python
import great_expectations as gx
import pandas as pd

products = pd.read_csv("data/products.csv")

context = gx.get_context()  # Data Context: pusat konfigurasi GE (mode ephemeral/in-memory untuk belajar)

# datasource bawaan untuk validasi langsung dari Pandas DataFrame
validator = context.sources.pandas_default.read_dataframe(products)
```

`context.sources.pandas_default` adalah cara cepat untuk memvalidasi DataFrame Pandas yang sudah ada di memori tanpa perlu setup datasource manual — cocok untuk kasus kita (data sudah di-load lewat script Python/PySpark sebelumnya), beda dari setup GE yang lebih penuh (menyambung langsung ke file/database sebagai *datasource* permanen), yang lebih relevan untuk tim yang mengelola banyak dataset sekaligus dalam jangka panjang.

### Menulis Expectation

```python
# Completeness
validator.expect_column_values_to_not_be_null("product_name")
validator.expect_column_values_to_not_be_null("category")

# Validity
validator.expect_column_values_to_be_between("unit_price", min_value=0, strict_min=True)
validator.expect_column_values_to_be_in_set(
    "category", ["Electronics", "Furniture", "Stationery", "Kitchen"]
)

# Uniqueness
validator.expect_column_values_to_be_unique("product_id")
```

Tiap `validator.expect_...()` langsung mengembalikan hasil untuk expectation **itu saja** (berguna untuk eksplorasi cepat) sekaligus **menambahkannya** ke suite yang sedang dibangun di `validator`.

### Menjalankan Semua Sekaligus & Membaca Hasil

```python
results = validator.validate()

print(results.success)              # True kalau SEMUA expectation lolos, False kalau ada yang gagal
for result in results.results:
    expectation_type = result.expectation_config.expectation_type
    column = result.expectation_config.kwargs.get("column")
    print(f"{expectation_type} ({column}): success={result.success}, gagal={result.result.get('unexpected_count', 0)} baris")
```

### Menyimpan Suite

```python
validator.save_expectation_suite(discard_failed_expectations=False)
```

`discard_failed_expectations=False` penting: secara default GE **membuang** expectation yang gagal saat validasi pertama kali (asumsinya: kalau gagal, mungkin itu salah tulis aturan). Untuk kebutuhan kita — expectation yang **memang harusnya** bisa gagal (itu tujuannya, mendeteksi data buruk) — expectation yang gagal tetap harus **disimpan** di suite, supaya validasi berikutnya tetap memeriksanya.

### Checkpoint — untuk Dipanggil Berulang dari Airflow

```python
checkpoint = context.add_or_update_checkpoint(
    name="products_checkpoint",
    validator=validator,
)

checkpoint_result = checkpoint.run()
print(checkpoint_result.success)
```

Checkpoint membungkus "jalankan suite ini terhadap data ini" jadi satu objek yang bisa dipanggil ulang — ini yang akan dipanggil dari task Airflow di mini project (`latihan_dq_streaming_mini_project.md`), supaya task Airflow-nya sendiri singkat (`checkpoint.run()` lalu cek `.success`), bukan menulis ulang semua expectation di dalam kode DAG.

## Contoh: Dataset dengan Pelanggaran Sengaja

Memakai dataset yang sengaja dikotori (pola sama dengan `sales_raw_dirty.csv` di `materi/minggu_2/hari_5_data_cleaning.md`):

```python
import pandas as pd

dirty_products = pd.DataFrame({
    "product_id": [1, 2, 3, 3],                     # 3 duplikat
    "product_name": ["Wireless Mouse", "Keyboard", None, "USB-C Hub"],  # None di baris ke-3
    "category": ["Electronics", "Electronics", "Electronics", "Toys"],  # "Toys" tidak ada di list valid
    "unit_price": [15.00, 45.00, -5.00, 25.00],      # -5.00 tidak valid
})

context = gx.get_context()
validator = context.sources.pandas_default.read_dataframe(dirty_products)

validator.expect_column_values_to_not_be_null("product_name")
validator.expect_column_values_to_be_unique("product_id")
validator.expect_column_values_to_be_between("unit_price", min_value=0, strict_min=True)
validator.expect_column_values_to_be_in_set("category", ["Electronics", "Furniture", "Stationery", "Kitchen"])

results = validator.validate()
print(results.success)   # False
```

Hasil: `results.success` bernilai `False`, dan tiap expectation melaporkan detail kegagalannya sendiri —
- `expect_column_values_to_not_be_null("product_name")` → gagal, 1 baris (`product_id=3`)
- `expect_column_values_to_be_unique("product_id")` → gagal, 2 baris (kedua baris `product_id=3`)
- `expect_column_values_to_be_between("unit_price", ...)` → gagal, 1 baris (`unit_price=-5.00`)
- `expect_column_values_to_be_in_set("category", ...)` → gagal, 1 baris (`category="Toys"`)

Perhatikan: **empat** expectation gagal secara **independen** — ini bukti langsung kenapa 5 dimensi di `hari_1_data_quality_dimensions.md` memang perlu diperiksa terpisah, bukan digabung jadi satu pemeriksaan besar; laporan GE otomatis memberi tahu **persis** dimensi mana yang gagal dan di baris mana, sesuatu yang harus ditulis manual kalau tidak pakai tool seperti ini.

## Kesalahan Umum

1. **Lupa `save_expectation_suite()` setelah menulis expectation.** Expectation yang sudah ditulis ke `validator` **tidak otomatis tersimpan permanen** — tanpa `save_expectation_suite()`, expectation itu hilang begitu sesi Python berakhir, dan run berikutnya harus menulis ulang dari nol.
2. **Menulis expectation yang terlalu longgar** (mis. `expect_column_values_to_not_be_null` untuk semua kolom tanpa mikir kolom mana yang **memang boleh** kosong) atau **terlalu ketat** (mis. `expect_column_values_to_be_in_set` dengan daftar yang lupa meng-update kategori baru yang sah) — keduanya menghasilkan alert yang salah (*false positive*/*false negative*), yang lama-lama bikin orang berhenti percaya ke hasil validasi ("data quality check-nya selalu merah, biasanya juga cuma false alarm" — ini kegagalan tool yang paling berbahaya, karena bikin orang mengabaikan alert yang sungguhan).
3. **Menaruh expectation langsung di kode DAG** (menulis ulang semua `validator.expect_...()` di dalam file Airflow) alih-alih menyimpan sebagai suite dan memanggilnya lewat Checkpoint. Ini bikin kode DAG jadi panjang dan aturan kualitas data tercampur dengan logic orkestrasi — pisahkan keduanya (pola ini dipraktikkan di `latihan_dq_streaming_mini_project.md`).
4. **Mengira `results.success == False` berarti pipeline harus otomatis berhenti.** GE sendiri **tidak** menghentikan pipeline — dia cuma melaporkan hasil. Yang menentukan "kalau gagal, hentikan task" adalah kode di sekitarnya (mis. `if not checkpoint_result.success: raise ValueError(...)` di task Airflow) — ini poin penting untuk fail-fast, dibahas lagi di `latihan_dq_streaming_mini_project.md`.

## Latihan

1. Untuk `dim_customer` Minggu 3 (kolom: `customer_id`, `country`), tulis expectation untuk: (a) `customer_id` unik, (b) `country` tidak null, (c) `country` merupakan salah satu dari negara yang memang ada di Online Retail II (boleh pakai daftar bebas 5-10 negara sebagai contoh).
2. Untuk `fact_sales` Minggu 3, tulis expectation untuk aturan **consistency** "`revenue` harus sama dengan `quantity × unit_price`" — cari nama expectation yang sesuai di dokumentasi GE untuk membandingkan 2 kolom (petunjuk: cari expectation dengan kata `pair` di namanya).
3. Jalankan validasi terhadap `dirty_products` di atas tapi **hapus** parameter `strict_min=True` dari `expect_column_values_to_be_between`. Apakah hasilnya berubah untuk kasus `unit_price = -5.00`? Jelaskan bedanya `min_value=0` dengan dan tanpa `strict_min=True`.
4. Jelaskan ke rekan kerja yang skeptis ("kenapa tidak `assert` biasa saja?") — kapan menurutmu `assert` manual di kode Python **cukup**, dan kapan memang butuh tool seperti GE? Beri 1 contoh konkret masing-masing.

## Kunci Jawaban & Pembahasan

**1.**
```python
validator.expect_column_values_to_be_unique("customer_id")
validator.expect_column_values_to_not_be_null("country")
validator.expect_column_values_to_be_in_set(
    "country", ["United Kingdom", "Germany", "France", "Australia", "Netherlands", "Spain"]
)
```

**2.**
```python
validator.expect_column_pair_values_to_be_equal("revenue", "computed_total")
```
Karena GE membandingkan 2 kolom yang **sudah ada** di data, biasanya perlu ditambahkan dulu kolom bantu `computed_total = quantity * unit_price` (lewat Pandas, sebelum data masuk ke validator) supaya bisa dibandingkan langsung dengan `revenue` — GE sendiri tidak menghitung ekspresi matematis di dalam expectation-nya, cuma membandingkan nilai kolom yang sudah ada.

**3.** **Ya, berubah** — tanpa `strict_min=True`, `min_value=0` berarti batas bawahnya **inklusif** (`unit_price >= 0` dianggap valid), jadi `unit_price = -5.00` **tetap gagal** (karena tetap di bawah 0), tapi `unit_price = 0.00` (persis di batas) akan **lolos**. Dengan `strict_min=True`, batasnya **eksklusif** (`unit_price > 0`), jadi `unit_price = 0.00` **juga gagal**. Untuk kasus harga produk, `strict_min=True` lebih tepat karena harga `0.00` biasanya menandakan data error/adjustment (bukan produk yang sungguhan dijual gratis), sama seperti keputusan cleaning `Price > 0` (bukan `>= 0`) yang sudah diambil di `materi/minggu_2/latihan_eda_dan_mini_project.md` dan `materi/minggu_3/latihan_pipeline_mini_project.md`.

**4.** `assert` manual **cukup** untuk pemeriksaan sekali pakai, cepat, di skrip eksplorasi lokal yang tidak dijalankan otomatis berulang (mis. "aku baru mau cek sekilas apakah dataset yang baru didownload ini ada kolom yang aneh, sebelum mulai analisis") — tidak butuh laporan terstruktur, tidak butuh disimpan, tidak dipakai orang lain. Tool seperti GE **dibutuhkan** begitu pemeriksaan itu (a) dijalankan **berulang otomatis** sebagai bagian pipeline produksi (persis kasus `data_quality_check` di DAG Airflow), (b) hasilnya perlu **dilihat/didiagnosis** orang lain tanpa baca kode Python (laporan yang terstruktur, bukan traceback error), atau (c) aturannya perlu **didokumentasikan** sebagai kontrak kualitas data yang disepakati tim, bukan cuma logic yang terkubur di dalam skrip.
