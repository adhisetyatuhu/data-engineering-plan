---
title: Hari 5 - Cloud Data Warehouse
parent: Minggu 6 - Cloud Platform Fundamentals
nav_order: 6
---

# Hari 5 — Cloud Data Warehouse: BigQuery, Redshift, Snowflake

*Jumat, 2 jam. Konsep + contoh query BigQuery — hands-on load data sungguhan baru Minggu (`latihan_cloud_migration_mini_project.md`).*

> Ini titik pertemuan dari banyak konsep minggu-minggu sebelumnya: data warehouse (`materi/minggu_3/hari_1_arsitektur_data.md`), star schema (`materi/minggu_3/hari_2_data_modeling.md`), ELT (`materi/minggu_3/hari_3_etl_elt.md`), dan Parquet columnar storage (`materi/minggu_3/latihan_pipeline_mini_project.md`) — semuanya sekarang bertemu di 1 layanan cloud.

## Tujuan Belajar

- [ ] Menjelaskan **separation of storage & compute** dan kenapa ini perubahan arsitektural fundamental dari database tradisional
- [ ] Membandingkan BigQuery, Redshift, Snowflake pada karakteristik utamanya
- [ ] Menjelaskan kembali kenapa cloud data warehouse adalah alasan utama ELT populer (menyambung `materi/minggu_3/hari_3_etl_elt.md`)
- [ ] Menulis query BigQuery yang setara dengan query yang sudah dibuat di Minggu 1-2

## Untuk Instruktur: Mindset Shift

Sepanjang Minggu 1-5, Postgres yang dipakai (`pg-belajar`) punya **storage dan compute yang menyatu di 1 mesin** — kalau butuh query lebih cepat, satu-satunya cara adalah memperbesar **seluruh** mesin itu (lebih banyak CPU, lebih banyak RAM, lebih banyak disk — semuanya naik bersamaan, walau mungkin yang benar-benar dibutuhkan cuma salah satunya). Cloud data warehouse **memisahkan** keduanya: storage (tempat data disimpan) dan compute (tempat query diproses) jadi 2 layer independen yang bisa di-scale terpisah.

Analogi yang membantu: ini seperti **arsitektur microservice vs monolith** yang sudah dikenal developer — monolith (Postgres tradisional) menyatukan semua concern jadi 1 unit yang harus di-scale bersamaan; microservice (cloud data warehouse) memisahkan concern (storage vs compute) jadi komponen independen yang masing-masing bisa di-scale sesuai kebutuhannya sendiri, tanpa saling mempengaruhi.

## Konsep & Sintaks

### Separation of Storage & Compute

```
Database Tradisional (Postgres di 1 mesin):
┌─────────────────────────┐
│   1 MESIN                │
│   Storage + Compute      │  <- keduanya naik/turun BERSAMAAN
│   (menyatu, tidak bisa   │
│    dipisah scale-nya)    │
└─────────────────────────┘

Cloud Data Warehouse (BigQuery, dkk):
┌──────────────┐         ┌──────────────────────┐
│   STORAGE     │  <--->  │   COMPUTE              │
│  (object       │         │  (cluster query engine, │
│   storage,     │         │   di-scale otomatis     │
│   scale tak    │         │   PER QUERY, lepas dari  │
│   terbatas)    │         │   ukuran data)           │
└──────────────┘         └──────────────────────┘
```

**Konsekuensi praktis dari pemisahan ini**:
1. **Storage bisa tumbuh nyaris tanpa batas** dengan biaya relatif murah (di baliknya memang object storage, `hari_2_object_storage.md`) — tidak perlu khawatir "kehabisan disk" seperti di database tradisional.
2. **Compute di-scale otomatis PER QUERY** — query berat mendapat lebih banyak resource sementara, lalu resource itu dilepas begitu query selesai (kembali ke elastisitas & pay-as-you-go, `hari_1_konsep_cloud.md`). Kamu **tidak perlu** provisioning cluster ukuran tetap yang harus cukup untuk beban puncak (beda dari Postgres yang mesinnya tetap sama ukurannya, dipakai atau tidak).
3. **Banyak query bisa jalan bersamaan** tanpa saling berebut resource compute yang sama, karena masing-masing bisa mendapat alokasi compute terpisah — ini secara langsung mengurangi risiko *resource contention* yang dibahas `materi/minggu_3/hari_1_arsitektur_data.md` (kenapa tidak query analitik berat langsung di OLTP production).

### BigQuery vs Redshift vs Snowflake

| | BigQuery (GCP) | Redshift (AWS) | Snowflake (multi-cloud) |
|---|---|---|---|
| Model compute | **Serverless** sepenuhnya — tidak ada cluster untuk dikonfigurasi, auto-scale penuh | **Cluster-based** — kamu pilih ukuran cluster (node type & count), bisa auto-scale tapi tetap ada unit "cluster" | **Virtual Warehouse** — mirip cluster, tapi lebih mudah di-resize/pause dibanding Redshift tradisional |
| Model biaya | Per **volume data yang di-scan** per query (atau flat-rate untuk pemakaian besar) | Per **jam cluster menyala** (atau serverless mode yang lebih baru, mirip BigQuery) | Per **detik compute terpakai** (virtual warehouse), terpisah dari biaya storage |
| Setup awal | Paling minim — langsung bisa query begitu ada data (termasuk BigQuery Sandbox tanpa billing) | Perlu provisioning cluster terlebih dahulu | Perlu setup warehouse & account, tapi relatif cepat |
| Ekosistem | Terintegrasi kuat dengan GCP (Cloud Storage, Dataflow, dsb) | Terintegrasi kuat dengan AWS (S3, Glue, dsb) | Multi-cloud (jalan di atas AWS/GCP/Azure), tidak terikat 1 provider |

**Kenapa BigQuery paling ramah untuk belajar** (alasan yang sama dengan `00_overview.md`): model serverless penuh berarti **tidak ada keputusan "ukuran cluster apa yang harus dipilih"** yang harus dipahami di awal — kamu langsung menulis query, BigQuery yang mengurus semua alokasi compute di baliknya. Redshift dan Snowflake tetap sangat relevan di dunia kerja (banyak perusahaan pakai salah satunya), konsepnya tetap **sama** (separation of storage & compute, kolumnar, elastis) — cuma model operasionalnya sedikit lebih eksplisit soal ukuran cluster.

### Menyambung Balik ke ELT (Minggu 3)

Ingat kembali `materi/minggu_3/hari_3_etl_elt.md`: ELT jadi masuk akal ketika **compute tujuan murah & elastis**. Sekarang konsep itu punya wujud nyata — BigQuery **adalah** contoh compute murah & elastis yang dimaksud. Kalau pipeline `ecommerce-etl-pipeline` dipindah sepenuhnya ke pola ELT (bukan ETL seperti sekarang), transformasi yang sekarang dilakukan PySpark **sebelum** load (`clean_transform.py`, `build_star_schema.py`) bisa dipindah jadi **SQL yang dijalankan di dalam BigQuery setelah** raw data di-load — memakai compute BigQuery yang memang dirancang murah untuk beban seperti ini, bukan compute Spark yang perlu di-provision terpisah. Ini **bukan** perubahan wajib untuk mini project minggu ini (`latihan_cloud_migration_mini_project.md` tetap mempertahankan pola ETL yang sudah ada — Spark tetap transform sebelum load), tapi baik dipahami sebagai **opsi arsitektur** yang sekarang benar-benar tersedia, bukan cuma teori.

### Contoh Query: Setara Minggu 1-2, Sekarang di BigQuery

Sintaks BigQuery Standard SQL **hampir identik** dengan PostgreSQL yang sudah dipakai sejak Minggu 1 — bedanya utamanya di penamaan tabel (format `project.dataset.table`, dikutip dengan backtick) dan sebagian fungsi tanggal.

```sql
-- Top 10 produk terlaris by revenue (padanan materi/minggu_2/latihan_eda_dan_mini_project.md #1)
SELECT
  stock_code,
  description,
  SUM(quantity) AS total_qty,
  SUM(revenue) AS total_revenue
FROM `PROJECT_ID.ecommerce_warehouse.fact_sales` f
JOIN `PROJECT_ID.ecommerce_warehouse.dim_product` p USING (stock_code)
WHERE is_return = FALSE
GROUP BY stock_code, description
ORDER BY total_revenue DESC
LIMIT 10;

-- RFM Analysis (padanan materi/minggu_2/latihan_eda_dan_mini_project.md #4) -- NTILE tetap tersedia
WITH rfm_raw AS (
  SELECT
    customer_id,
    DATE_DIFF(CURRENT_DATE(), MAX(PARSE_DATE('%Y%m%d', CAST(date_id AS STRING))), DAY) AS recency_days,
    COUNT(DISTINCT invoice) AS frequency,
    SUM(revenue) AS monetary
  FROM `PROJECT_ID.ecommerce_warehouse.fact_sales`
  WHERE is_return = FALSE
  GROUP BY customer_id
)
SELECT
  *,
  NTILE(5) OVER (ORDER BY recency_days DESC) AS r_score,
  NTILE(5) OVER (ORDER BY frequency ASC) AS f_score,
  NTILE(5) OVER (ORDER BY monetary ASC) AS m_score
FROM rfm_raw;
```

Perhatikan: `JOIN ... USING (stock_code)`, `GROUP BY`, `WITH` (CTE), `NTILE() OVER (...)` — **semuanya** konsep yang sudah dipelajari penuh di `materi/minggu_1/`. Ini bukti langsung kenapa fondasi SQL yang kuat di Minggu 1 tetap terbayar sampai Minggu 6 — pindah ke cloud data warehouse **tidak** berarti belajar bahasa query dari nol, cuma menyesuaikan sedikit dialek & cara referensi tabel.

## Kesalahan Umum

1. **Mengira "serverless" berarti "gratis" atau "tidak ada biaya sama sekali".** BigQuery serverless tetap dikenakan biaya berdasarkan volume data yang di-scan per query — query `SELECT *` tanpa filter di tabel besar bisa jadi mahal justru **karena** kemudahannya (tidak ada "cluster nyala/mati" yang terlihat jelas seperti Redshift, jadi biaya terasa kurang nyata sampai tagihan datang).
2. **Tidak memfilter kolom yang di-scan** (`SELECT *` padahal cuma butuh 3 kolom). Karena BigQuery adalah **kolumnar** (sama seperti alasan Parquet dipilih di `materi/minggu_3/latihan_pipeline_mini_project.md`) dan biaya dihitung dari **volume data yang di-scan**, memilih kolom secara eksplisit (`SELECT stock_code, revenue` bukan `SELECT *`) bisa mengurangi biaya secara signifikan — kolom yang tidak diminta tidak perlu dibaca sama sekali.
3. **Menyamakan model biaya BigQuery dan Redshift secara langsung tanpa penyesuaian.** Membandingkan "biaya per jam" (Redshift) dengan "biaya per query" (BigQuery) butuh perhitungan berbeda tergantung pola beban kerja — keduanya bisa lebih murah tergantung situasi (kembali ke pembahasan elastisitas `hari_1_konsep_cloud.md` Latihan #4).
4. **Mengabaikan partitioning/clustering tabel BigQuery untuk tabel besar.** Sama seperti index di Postgres (`materi/minggu_1/latihan_studi_kasus.md` bagian `EXPLAIN`), tabel BigQuery yang di-partition (mis. berdasarkan tanggal) dan di-cluster (berdasarkan kolom yang sering difilter) bisa mengurangi volume data yang di-scan secara drastis — untuk `fact_sales` yang terus bertambah, partitioning berdasarkan `date_id` adalah optimasi yang sangat berdampak.

## Latihan

1. Jelaskan dengan kata-kata sendiri kenapa "separation of storage & compute" membuat resource contention (`materi/minggu_3/hari_1_arsitektur_data.md`) jadi masalah yang jauh lebih kecil dibanding database tradisional.
2. Tim ingin menjalankan query berat (scan seluruh histori 3 tahun `fact_sales`) 1x per bulan untuk laporan, dan ratusan query kecil setiap hari untuk dashboard. Bandingkan kecocokan BigQuery (per-query) vs Redshift cluster yang menyala 24/7 untuk pola beban kerja campuran ini.
3. Query berikut dijalankan di BigQuery: `SELECT * FROM fact_sales WHERE date_id = 20260115`. Tabel `fact_sales` **tidak** di-partition. Jelaskan kenapa ini kemungkinan men-scan (dan membebankan biaya untuk) jauh lebih banyak data dibanding yang sebenarnya dibutuhkan, dan bagaimana partitioning memperbaikinya.
4. Rancang 1 query BigQuery (boleh pseudocode/kerangka, tidak perlu 100% sintaks sempurna) yang setara dengan query "growth rate bulanan" dari `materi/minggu_2/latihan_eda_dan_mini_project.md` #5 — tunjukkan bagian mana yang identik dengan versi PostgreSQL, dan bagian mana (kalau ada) yang perlu disesuaikan.

## Kunci Jawaban & Pembahasan

**1.** Di database tradisional (storage+compute menyatu), query analitik berat dan beban transaksi harian **berebut resource fisik yang sama** (CPU, RAM, disk I/O di 1 mesin) — inilah akar resource contention. Dengan separation of storage & compute, tiap query (atau kelompok beban kerja) bisa mendapat **alokasi compute terpisah** yang membaca dari storage yang sama — storage-nya dibagi bersama (dan memang dirancang untuk dibaca banyak pihak sekaligus, sifat object storage yang dibahas `hari_2_object_storage.md`), tapi **pemrosesannya** tidak lagi berebut 1 CPU/RAM fisik yang sama. Ini tidak menghilangkan resource contention 100% (masih ada batasan kuota/concurrent query tergantung provider), tapi mengurangi drastis dibanding arsitektur yang menyatukan semuanya di 1 mesin.

**2.** Untuk pola beban kerja **campuran & tidak konstan** ini, **BigQuery (per-query) kemungkinan lebih hemat**: laporan bulanan berat cuma dikenakan biaya **1x sebulan** saat benar-benar dijalankan (bukan biaya cluster yang menyala terus demi 1 query besar sebulan sekali), dan ratusan query kecil harian untuk dashboard biasanya men-scan volume data kecil (biaya rendah per query). Redshift cluster 24/7 berarti membayar **penuh** untuk kapasitas cluster yang harus cukup menangani laporan bulanan yang berat itu, padahal kapasitas sebesar itu **menganggur** di 29 hari lainnya. Catatan: kalau volume **query kecil harian** itu sangat banyak dan konsisten sepanjang hari (bukan cuma "ratusan", tapi jutaan), titik keseimbangan bisa bergeser — inilah kenapa keputusan ini selalu perlu dihitung berdasarkan pola beban kerja aktual, bukan aturan umum "serverless selalu menang".

**3.** Tanpa partitioning, BigQuery **tidak tahu** bagian mana dari tabel yang berisi `date_id = 20260115` tanpa membaca (scan) **seluruh** tabel terlebih dahulu untuk mengevaluasi kondisi `WHERE` itu di setiap baris — persis seperti `Seq Scan` yang dibahas `materi/minggu_1/latihan_studi_kasus.md` (baca semua baris, baru filter). Dengan **partitioning berdasarkan `date_id`** (atau kolom tanggal turunannya), BigQuery bisa langsung **melompat** ke partisi yang relevan (mirip index di Postgres) dan mengabaikan partisi lain sepenuhnya — untuk tabel yang datanya terus bertambah tiap hari (seperti `fact_sales`), ini bisa mengurangi volume data yang di-scan dari "seluruh histori 3 tahun" jadi "cuma 1 hari itu saja", dengan penghematan biaya yang proporsional.

**4.**
```sql
WITH monthly AS (
  SELECT
    DATE_TRUNC(PARSE_DATE('%Y%m%d', CAST(date_id AS STRING)), MONTH) AS month,
    SUM(revenue) AS revenue
  FROM `PROJECT_ID.ecommerce_warehouse.fact_sales`
  WHERE is_return = FALSE
  GROUP BY 1
)
SELECT
  month,
  revenue,
  LAG(revenue) OVER (ORDER BY month) AS prev_month,
  ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
        / LAG(revenue) OVER (ORDER BY month), 1) AS growth_pct
FROM monthly
ORDER BY month;
```
**Identik** dengan versi PostgreSQL: struktur CTE (`WITH`), `GROUP BY`, `LAG() OVER (ORDER BY ...)`, dan bahkan formula growth percentage-nya sama persis — semua konsep window function dari `materi/minggu_1/hari_5_window_function.md` transfer penuh tanpa perubahan logika. **Yang disesuaikan**: `DATE_TRUNC('month', ...)` (Postgres) menjadi `DATE_TRUNC(..., MONTH)` (BigQuery, urutan argumen & keyword `MONTH` tanpa kutip berbeda), dan konversi `date_id` (integer `YYYYMMDD`, mis. `20260115`, dari `materi/minggu_3/hari_2_data_modeling.md`) balik ke tipe `DATE` butuh `PARSE_DATE('%Y%m%d', CAST(date_id AS STRING))` — **bukan** `TIMESTAMP_SECONDS(date_id)` (itu untuk mengonversi Unix epoch dalam detik, salah kalau dipakai untuk `date_id` yang formatnya `YYYYMMDD`, akan menghasilkan tanggal yang sama sekali salah). Pendekatan paling praktis sebenarnya tetap menyimpan `full_date` asli dari `dim_date` dan `JOIN` ke situ, bukan mengonversi `date_id` berulang di tiap query.
