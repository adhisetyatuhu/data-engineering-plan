---
title: Overview
parent: Minggu 3 - Arsitektur Data & Big Data Pipeline
nav_order: 1
---

# Modul Minggu 3 — Arsitektur Data & Big Data Pipeline (untuk Software Developer)

> Pendamping `minggu_3.md` (jadwal & outline). File ini konten pengajaran lengkap: penjelasan konsep, analogi kode, contoh, latihan, kunci jawaban. Lihat juga `materi/minggu_1/` (SQL) dan `materi/minggu_2/` (Python/Pandas) — minggu ini menyambung langsung dari keduanya.

## Shift Terbesar Minggu Ini: dari Analisis ke Sistem

Minggu 1–2 hasil kerjanya adalah **jawaban** — query dan notebook yang dijalankan manual, sekali, untuk menjawab pertanyaan bisnis. Minggu 3 hasil kerjanya adalah **sistem** — pipeline yang jalan sendiri, berulang, tanpa perlu ada orang yang klik "Run All" tiap hari. Ini pergeseran peran dari "orang yang menganalisis data" ke "orang yang membangun infrastruktur supaya orang lain bisa menganalisis data". Itulah inti pekerjaan data engineer, dan minggu ini adalah minggu pertama roadmap yang benar-benar terasa seperti itu.

Dua tool baru minggu ini — **Spark** dan **Airflow** — punya peran yang jangan sampai tertukar di kepala peserta:
- **Spark** menjawab "bagaimana **mentransformasi** data dalam jumlah besar secara efisien" — pengganti Pandas ketika data tidak lagi muat nyaman di memori satu mesin.
- **Airflow** menjawab "bagaimana **menjalankan & menjadwalkan** urutan langkah-langkah itu secara otomatis, dengan visibilitas kalau ada yang gagal" — pengganti kamu menekan tombol "Run" secara manual tiap hari.

Spark itu **pekerja**, Airflow itu **mandor**. Tekankan analogi ini dari hari pertama karena akan terus dipakai sampai `latihan_pipeline_mini_project.md`.

## Kenapa Bobot Materinya Begini

- **Senin–Kamis** murni konsep (arsitektur, modeling, ETL/ELT, batch/stream) — sengaja tanpa banyak coding, supaya peserta punya kerangka berpikir dulu sebelum pegang tool. Developer yang langsung loncat ke Spark/Airflow tanpa paham **kenapa** pipeline dirancang begitu biasanya berakhir bisa "menjalankan" tool tapi tidak bisa membuat keputusan desain sendiri saat kasusnya beda dikit dari tutorial.
- **Jumat** baru masuk Spark, itu pun konsep + sintaks dasar saja (local mode).
- **Sabtu–Minggu** adalah bagian terberat & paling penting: hands-on penuh, langsung jadi mini project (upgrade repo portfolio jadi pipeline otomatis).

## Setup Environment

```bash
# Python venv yang sama dari Minggu 2 bisa dipakai lagi, tambah dependency baru
source venv/bin/activate
pip install pyspark==3.5.* pandas sqlalchemy psycopg2-binary pyarrow

# PostgreSQL "pg-belajar" dari Minggu 1 tetap dipakai sebagai target "warehouse"
docker start pg-belajar   # kalau sebelumnya sudah dibuat dan sekarang berhenti

# Airflow via docker-compose resmi (dipakai mulai `latihan_pipeline_mini_project.md`)
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.9.3/docker-compose.yaml'
mkdir -p ./dags ./logs ./plugins ./config
echo -e "AIRFLOW_UID=$(id -u)" > .env
docker compose up airflow-init
docker compose up -d
```

- **PySpark local mode** tidak butuh cluster apapun — cukup `pip install pyspark`, Java (JDK 11/17) harus tersedia di mesin (`pyspark` akan error jelas kalau `JAVA_HOME` tidak ketemu, cek `java -version` dulu kalau ada masalah).
- **Airflow via Docker** dipilih (bukan install langsung) karena Airflow punya banyak dependency sistem yang gampang bentrok versi — pola yang sama seperti alasan pakai Docker untuk Postgres di Minggu 1. Ini juga jembatan pemanasan ke Minggu 7 (Containerization), yang akan mendalami Docker jauh lebih detail.
- Setelah `docker compose up -d`, Airflow UI ada di `http://localhost:8080` (default login `airflow`/`airflow` di docker-compose resmi).

## Dataset yang Dipakai Minggu Ini

### Senin–Kamis (Konsep): Dataset Mini Minggu 1

File CSV yang sama dari `materi/minggu_1/00_overview.md` / `materi/minggu_2/00_overview.md` (`customers`, `products`, `orders`, `order_items`) dipakai lagi untuk latihan **data modeling** (Hari 2) — sengaja dataset kecil & familiar supaya peserta fokus ke konsep star schema, bukan sibuk memahami data baru.

### Jumat–Minggu (Hands-on): Online Retail II

Mulai Hari 5 (pengenalan sintaks PySpark) dan penuh di Sabtu–Minggu, kembali pakai **Online Retail II** — file yang sama yang sudah dibersihkan di `materi/minggu_2/latihan_eda_dan_mini_project.md` (kolom: `Invoice`, `StockCode`, `Description`, `Quantity`, `InvoiceDate`, `Price`, `Customer ID`, `Country`). Minggu ini datanya **dinormalisasi penuh** jadi star schema (`fact_sales` + `dim_product`/`dim_customer`/`dim_date`) memakai PySpark — pekerjaan yang sengaja belum dilakukan di Minggu 2 (yang cuma pakai 1 tabel flat `retail_clean`).

## Tentang Repo Portfolio: Rename, Bukan Project Baru

Repo yang dimulai di Minggu 2 (`ecommerce-sales-analysis`) **dilanjutkan**, bukan ditinggalkan — tapi mulai minggu ini diberi nama yang lebih mencerminkan isinya: **`ecommerce-etl-pipeline`** (nama yang konsisten dipakai `minggu_3.md` dan seterusnya sampai Minggu 8). Cukup rename repo di GitHub (Settings → Repository name) dan `git remote set-url` di lokal — histori commit Minggu 1–2 tetap ada, itu justru nilai tambah portfolio (menunjukkan progres dari analisis manual ke pipeline). Detail struktur folder baru ada di `latihan_pipeline_mini_project.md`.

## Struktur Modul

| File | Sesuai Jadwal `minggu_3.md` | Topik |
|---|---|---|
| [`hari_1_arsitektur_data.md`](hari_1_arsitektur_data.md) | Senin, 2 jam | OLTP vs OLAP, Data Warehouse vs Data Lake vs Lakehouse |
| [`hari_2_data_modeling.md`](hari_2_data_modeling.md) | Selasa, 2 jam | Star schema, snowflake schema, fact & dimension table |
| [`hari_3_etl_elt.md`](hari_3_etl_elt.md) | Rabu, 2 jam | ETL vs ELT, tools (Airflow, dbt, Fivetran) |
| [`hari_4_batch_stream.md`](hari_4_batch_stream.md) | Kamis, 2 jam | Batch vs Stream processing |
| [`hari_5_spark.md`](hari_5_spark.md) | Jumat, 2 jam | Arsitektur Spark, RDD vs DataFrame, PySpark dasar |
| [`latihan_pipeline_mini_project.md`](latihan_pipeline_mini_project.md) | Sabtu (4 jam) + Minggu (4 jam) | Hands-on PySpark + Airflow, mini project upgrade jadi pipeline otomatis |

Struktur tiap file `hari_X` sama dengan Minggu 1–2: Tujuan Belajar → Untuk Instruktur → Konsep & Sintaks → Contoh → Kesalahan Umum → Latihan → Kunci Jawaban.

## Catatan Cara Mengajar

- **Senin–Kamis boleh lebih banyak diskusi/whiteboard**, bukan cuma live coding — ini materi arsitektur & desain, keputusannya sering "tergantung konteks", beda dari SQL/Pandas yang jawabannya lebih hitam-putih. Dorong peserta berargumen "kenapa pilih X", bukan cuma hafal definisi.
- **Sambungkan terus ke pengalaman developer**: OLTP ≈ database aplikasi production yang mereka pakai sehari-hari, data warehouse ≈ tempat semua data itu "dikumpulkan ulang" buat dianalisis tanpa mengganggu database production.
- **Spark & Airflow: jangan takut error di depan peserta.** Kedua tool ini terkenal dengan error message yang panjang dan kadang tidak intuitif (terutama Spark, karena errornya sering dari JVM). Justru bagus dibahas live — ini pengalaman yang akan mereka temui lagi nanti.
- Total waktu: 5 hari × 2 jam + Sabtu 4 jam + Minggu 4 jam = 18 jam, sesuai `minggu_3.md`.
