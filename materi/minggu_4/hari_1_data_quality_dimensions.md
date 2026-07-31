---
title: Hari 1 - Dimensi Data Quality
parent: Minggu 4 - Data Quality, Orchestration & Streaming
nav_order: 2
---

# Hari 1 — Data Quality dalam Pipeline: 5 Dimensi Kualitas Data

*Senin, 2 jam. Konsep murni — pakai tabel `fact_sales`/`dim_customer`/`dim_product`/`dim_date` hasil pipeline Minggu 3 sebagai contoh kasus.*

> Materi ini fondasi literal untuk Data Governance Minggu 5 (lihat `minggu_5.md`) — istilah "dimensi kualitas data" akan terus dipakai sampai akhir roadmap, bukan cuma jargon 1 minggu.

## Tujuan Belajar

- [ ] Menyebutkan dan menjelaskan 5 dimensi kualitas data: completeness, uniqueness, validity, consistency, timeliness
- [ ] Untuk data di `fact_sales`/dimension table Minggu 3, merumuskan aturan konkret per dimensi
- [ ] Menjelaskan kenapa "pipeline tidak error" tidak sama dengan "data valid"
- [ ] Menjelaskan konsep **fail-fast** dalam konteks data quality gate

## Untuk Instruktur: Mindset Shift

Analogi paling langsung buat developer: dimensi data quality itu setara **assertion/test case**, tapi untuk **data**, bukan untuk **kode**. Developer sudah terbiasa menulis `assert response.status == 200` atau unit test yang mengecek "fungsi ini menghasilkan output yang benar". Data quality check melakukan hal yang sama persis, tapi objeknya baris-baris data: *"apakah baris ini masuk akal?"*

Poin yang wajib ditekankan di awal sesi: **Airflow (Minggu 3) tahu apakah kode-nya error, tapi tidak tahu apakah datanya benar.** Task `load_to_warehouse` di DAG Minggu 3 akan tetap **sukses** (hijau) walau isinya `Quantity = -99999` atau `Customer ID` kosong semua — karena secara teknis, `pd.read_parquet()` dan `.to_sql()` tidak error menjalankan itu. Data quality check adalah lapisan **tambahan** yang secara eksplisit memeriksa isi datanya, bukan cuma "apakah kodenya jalan tanpa exception".

## Konsep & Sintaks

### 5 Dimensi Kualitas Data

| Dimensi | Pertanyaan yang Dijawab | Contoh Aturan untuk `fact_sales`/dimension Minggu 3 |
|---|---|---|
| **Completeness** | Apakah data yang seharusnya ada, benar-benar ada (tidak `NULL`/kosong)? | `customer_id`, `invoice`, `stock_code` di `fact_sales` tidak boleh `NULL` |
| **Uniqueness** | Apakah tidak ada duplikasi yang tidak seharusnya ada? | Kombinasi `invoice` + `stock_code` tidak boleh duplikat (1 produk cuma muncul 1x per invoice) |
| **Validity** | Apakah nilai data sesuai format/range/tipe yang diharapkan? | `unit_price` dan `quantity` harus `> 0`; `country` di `dim_customer` harus ada di daftar negara valid |
| **Consistency** | Apakah data konsisten secara logis antar kolom/tabel? | `revenue` harus sama dengan `quantity × unit_price`; `customer_id` di `fact_sales` harus ada padanannya di `dim_customer` |
| **Timeliness** | Apakah data cukup "segar" sesuai kebutuhan (tidak basi)? | `dim_date`/`fact_sales` hasil run hari ini harus berisi data sampai kemarin, bukan data 3 bulan lalu (pipeline gagal jalan diam-diam) |

Cara mudah mengingat: 4 dimensi pertama (completeness, uniqueness, validity, consistency) menjawab **"apakah isi datanya benar?"**, dimensi terakhir (timeliness) menjawab **"apakah datanya masih relevan waktunya?"** — dimensi ini sering terlewat karena tidak terlihat dari **isi** satu baris data, tapi dari **kapan** data itu terakhir diperbarui dibanding kapan seharusnya.

### Kenapa Bukan Cuma "Data Bersih vs Kotor"

Developer sering menyederhanakan data quality jadi 1 pertanyaan besar ("apakah datanya bersih?"), padahal itu 5 pertanyaan **independen** — data bisa lolos 1 dimensi tapi gagal di dimensi lain:

- Baris dengan `customer_id` terisi (lolos **completeness**) tapi `customer_id`-nya tidak ada di `dim_customer` (gagal **consistency**).
- Baris dengan `unit_price = 15.00` (lolos **validity**, angka positif wajar) tapi `revenue` tersimpan `999999.00` padahal `quantity × unit_price` harusnya jauh lebih kecil (gagal **consistency**).
- Tidak ada baris duplikat sama sekali (lolos **uniqueness**) tapi data terakhir yang masuk 2 bulan lalu (gagal **timeliness**, karena pipeline diam-diam berhenti jalan tanpa ada yang sadar).

Memisahkan jadi 5 dimensi memaksa pemeriksaan yang **spesifik dan terukur**, bukan perasaan umum "kelihatannya oke".

### Fail-Fast: Kenapa Urutan Pemeriksaan Penting

**Fail-fast** = begitu ditemukan pelanggaran, pipeline **berhenti segera** di titik itu — tidak melanjutkan ke tahap berikutnya dengan data yang sudah diketahui bermasalah. Ini prinsip yang sama dengan validasi input di awal fungsi (`guard clause`) di software development — developer sudah terbiasa: cek validitas argumen **di awal** fungsi, bukan di tengah-tengah setelah sebagian efek samping sudah terjadi.

Untuk pipeline data, ini berarti data quality check **wajib ditaruh sebelum** data menyentuh sistem yang dipakai orang lain (warehouse, dashboard). Kalau check ditaruh **setelah** load (seperti pola sederhana di `materi/minggu_3/latihan_pipeline_mini_project.md`), data yang salah **sudah telanjur** ada di warehouse saat masalahnya baru ketahuan — analyst atau dashboard yang kebetulan mengakses warehouse di jendela waktu itu bisa saja sudah melihat/memakai data yang salah. Ini alasan DAG Minggu 4 (`latihan_dq_streaming_mini_project.md`) mengubah urutan jadi `extract → transform → data_quality_check → load → notify` — check dipindah **sebelum** load, persis prinsip fail-fast.

## Contoh Kasus

Bayangkan `fact_sales` hari ini menerima baris-baris berikut dari pipeline Minggu 3. Untuk tiap baris, dimensi apa yang dilanggar?

| Baris | `invoice` | `stock_code` | `customer_id` | `quantity` | `unit_price` | `revenue` |
|---|---|---|---|---|---|---|
| A | 536365 | 85123A | `NULL` | 6 | 2.55 | 15.30 |
| B | 536366 | 71053 | 17850 | 6 | 3.39 | 20.34 |
| B (lagi) | 536366 | 71053 | 17850 | 6 | 3.39 | 20.34 |
| C | 536367 | 84406B | 17850 | 8 | -2.75 | -22.00 |
| D | 536368 | 22752 | 17850 | 2 | 7.65 | 999.00 |

- **Baris A**: `customer_id` kosong → pelanggaran **completeness**.
- **Baris B (duplikat)**: kombinasi `invoice`+`stock_code` (536366, 71053) muncul 2x → pelanggaran **uniqueness**.
- **Baris C**: `unit_price` negatif → pelanggaran **validity** (harga tidak masuk akal secara bisnis, beda dari `quantity` negatif yang sudah dikategorikan sebagai retur di Minggu 2–3).
- **Baris D**: `revenue` (999.00) tidak sama dengan `quantity × unit_price` (2 × 7.65 = 15.30) → pelanggaran **consistency**.

Latihan serupa dengan skenario timeliness ada di bagian Latihan.

## Kesalahan Umum

1. **Menganggap "tidak ada error saat pipeline jalan" berarti "data valid".** Ini miskonsepsi inti minggu ini — sudah dibahas di atas, tapi wajar diulang karena sangat mudah terlewat oleh developer yang terbiasa berpikir dalam kerangka "ada exception vs tidak ada exception".
2. **Mencampur 5 dimensi jadi satu pemeriksaan besar yang samar** ("cek apakah data OK") alih-alih aturan spesifik per dimensi. Aturan yang samar susah diotomasi dan susah didiagnosis kalau gagal — bandingkan `assert data_ok(df)` (tidak jelas apa yang salah kalau gagal) vs 5 assertion terpisah yang jelas dimensi mana yang gagal.
3. **Fokus cuma ke completeness/validity, lupa consistency.** Completeness (null check) dan validity (range check) paling gampang ditulis, jadi sering jadi satu-satunya yang diperiksa — padahal bug data yang paling berbahaya justru sering muncul di **consistency** (angka yang masing-masing "terlihat wajar" sendiri-sendiri, tapi tidak nyambung logikanya satu sama lain, seperti Baris D di atas).
4. **Meletakkan data quality check setelah data sudah dipakai sistem lain** (bukan fail-fast). Deteksi telat tetap lebih baik dari tidak ada deteksi sama sekali, tapi konsekuensinya beda jauh dari mencegah data buruk masuk sejak awal.

## Latihan

1. Untuk skenario "pipeline `ecommerce_etl_pipeline` seharusnya jalan tiap hari jam 2 pagi, tapi karena container Airflow down 3 hari, tidak ada run baru sejak 3 hari lalu — namun kalau dicek, semua data yang **ada** di `fact_sales` sebenarnya valid (tidak ada null, tidak ada duplikat, angka semua masuk akal)" — dimensi apa yang dilanggar? Kenapa 4 dimensi lain tidak cukup untuk menangkap masalah ini?
2. Rumuskan aturan **completeness** dan **validity** untuk `dim_date` (kolom: `date_id`, `full_date`, `day`, `month`, `quarter`, `year`, `is_weekend` — lihat `materi/minggu_3/hari_2_data_modeling.md`).
3. Seorang rekan bilang "consistency check itu berlebihan, kalau completeness dan validity sudah lolos, datanya pasti sudah benar." Bantah pernyataan ini dengan contoh konkret dari `fact_sales`.
4. Rancang aturan **uniqueness** untuk `dim_customer` dan `dim_product` (bukan `fact_sales`) — apa yang seharusnya tidak boleh duplikat di masing-masing tabel ini, dan kenapa itu penting untuk `JOIN` yang benar ke `fact_sales`?

## Kunci Jawaban & Pembahasan

**1.** Ini pelanggaran **timeliness** — datanya sendiri (yang sudah ada) valid di 4 dimensi lain, tapi data yang **seharusnya sudah ada** (3 hari terbaru) tidak pernah masuk. Completeness cuma memeriksa apakah **kolom** di baris yang ada terisi, bukan apakah **baris/periode waktu** yang seharusnya ada memang ada — ini beda level pemeriksaan. Uniqueness, validity, consistency juga sama-sama cuma memeriksa baris yang **sudah ada**, tidak bisa mendeteksi baris yang **hilang seluruhnya**. Ini kenapa timeliness butuh jenis pemeriksaan berbeda: biasanya "apakah ada data baru sejak timestamp X" atau "apakah `MAX(invoice_date)` cukup dekat dengan hari ini" — bukan pemeriksaan per-baris seperti 4 dimensi lainnya.

**2.**
```
Completeness: date_id, full_date, day, month, quarter, year, is_weekend semuanya tidak boleh NULL
              (dim_date biasanya di-generate penuh, jadi seharusnya tidak ada NULL sama sekali —
               kalau ada, kemungkinan bug di logic generate-nya sendiri)
Validity:     month antara 1-12
              quarter antara 1-4
              day antara 1-31 (dan idealnya konsisten dengan month, misal bukan 30 Februari)
              is_weekend harus boolean (True/False), bukan tipe lain
              full_date harus tipe date/timestamp valid, bukan string sembarang
```

**3.** Contoh: baris dengan `quantity = 100`, `unit_price = 50.00` — keduanya **lolos validity** (sama-sama angka positif, masuk akal secara individual). Tapi kalau `revenue` yang tersimpan di baris itu ternyata `5.00` (bukan `100 × 50 = 5000`), itu jelas ada yang salah — mungkin bug di formula perhitungan `revenue` saat transform, atau kolom yang salah ke-mapping. Completeness dan validity, sekeras apapun diperiksa, **tidak akan pernah menangkap** masalah ini karena masing-masing kolom (`quantity`, `unit_price`, `revenue`) secara individual tetap terlihat "valid" — masalahnya ada di **hubungan logis** antar kolom, dan cuma consistency check yang secara eksplisit memeriksa hubungan itu.

**4.** `dim_customer`: `customer_id` harus unik (1 baris per customer — kalau ada 2 baris `customer_id = 17850` dengan `country` berbeda, `JOIN` dari `fact_sales` ke `dim_customer` bisa menghasilkan baris ganda/ambigu, bikin agregasi revenue jadi salah karena baris fact ter-duplikasi lewat join). `dim_product`: `stock_code` harus unik (alasan sama persis — 1 produk harusnya 1 baris deskripsi). Ini kenapa uniqueness dimension table sama pentingnya dengan uniqueness fact table: pelanggaran di dimension table **menular** ke hasil query manapun yang melakukan `JOIN` ke tabel itu, walau fact table-nya sendiri sudah bersih.
