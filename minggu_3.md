---
title: Minggu 3 - Arsitektur Data & Big Data Pipeline
nav_order: 4
---

# Minggu 3 — Arsitektur Data & Big Data Pipeline (Dasar)

## Breakdown Harian (±18 jam)

| Hari | Jam | Materi |
|---|---|---|
| Senin | 2 jam | Konsep dasar arsitektur data: OLTP vs OLAP, data warehouse vs data lake vs lakehouse |
| Selasa | 2 jam | Data modeling: star schema, snowflake schema, fact & dimension table |
| Rabu | 2 jam | ETL vs ELT — konsep, kapan pakai yang mana, contoh tools (Airflow, dbt, Fivetran) |
| Kamis | 2 jam | Batch processing vs Stream processing — konsep dasar, use case |
| Jumat | 2 jam | Pengantar Apache Spark: arsitektur (driver, executor), RDD vs DataFrame |
| Sabtu | 4 jam | Hands-on: setup PySpark lokal, transform data dataset e-commerce (bandingkan vs Pandas) |
| Minggu | 4 jam | Hands-on: bangun pipeline sederhana pakai Apache Airflow (DAG extract → transform → load) |

## Detail Topik
1. **Data Warehouse vs Data Lake vs Lakehouse** — perbedaan struktur data, use case, contoh tools (Snowflake, BigQuery vs S3/GCS, Delta Lake)
2. **Data Modeling** — fokus star schema, latihan rancang dari dataset e-commerce (fact_sales, dim_customer, dim_product, dim_date)
3. **ETL vs ELT** — kenapa ELT makin populer dengan cloud warehouse modern
4. **Batch vs Streaming** — level konsep dulu (Kafka masuk lebih dalam di Minggu 4)
5. **Apache Spark** — fokus konsep + PySpark basic syntax, local mode saja
6. **Apache Airflow** — install via Docker, konsep DAG/task/scheduler, bikin 1 DAG sederhana

## Sumber Belajar
- "Fundamentals of Data Engineering" (Reis & Housley) — bab 1-3
- Video overview: Andreas Kretz / Seattle Data Guy (YouTube)
- Spark official docs "Quick Start", atau Databricks Community Edition
- Airflow official "Getting Started" + docker-compose quickstart

## Target di Akhir Minggu 3
- Menjelaskan perbedaan data warehouse/lake/lakehouse dan kapan masing-masing dipakai
- Merancang star schema sederhana dari data mentah
- Menjalankan transformasi data dasar dengan PySpark
- Membuat dan menjalankan 1 DAG sederhana di Airflow

---

## Mini Project: "Automated ETL Pipeline untuk E-Commerce Data Warehouse"

**Kenapa case ini?** Naik level dari mini project Minggu 1-2 — dari analisis manual jadi pipeline otomatis. Melanjutkan dataset yang sama (Online Retail II) untuk kontinuitas portfolio.

### Tahap 1: Data Modeling
Rancang star schema:
- `fact_sales` (invoice_id, product_id, customer_id, date_id, quantity, unit_price, total_amount)
- `dim_product` (product_id, description, category)
- `dim_customer` (customer_id, country)
- `dim_date` (date_id, day, month, quarter, year)

Dokumentasikan di README/diagram (draw.io atau dbdiagram.io).

### Tahap 2: Transform Data dengan PySpark
Script PySpark yang:
1. Baca raw CSV
2. Cleaning (reuse logic dari project Minggu 2)
3. Transform jadi bentuk star schema (split ke fact & dimension tables)
4. Tulis hasil ke format Parquet

### Tahap 3: Orkestrasi dengan Airflow
DAG dengan task terpisah:
```
extract_raw_data → transform_with_spark → load_to_warehouse → data_quality_check
```
- **extract**: baca raw file
- **transform**: jalankan PySpark script
- **load**: masukkan Parquet ke SQLite/PostgreSQL sebagai "warehouse"
- **data_quality_check**: validasi sederhana (row count > 0, tidak ada duplicate invoice_id) — jembatan ke topik Data Governance

Jadwalkan DAG jalan otomatis (misal daily) untuk menunjukkan pemahaman scheduling.

### Struktur Repo
```
ecommerce-etl-pipeline/
├── README.md
├── dags/
│   └── ecommerce_etl_dag.py
├── spark_jobs/
│   ├── clean_transform.py
│   └── build_star_schema.py
├── data/
│   ├── raw/
│   └── warehouse/
├── docker-compose.yml
└── diagrams/
    └── star_schema.png
```

### README yang Perlu Ditulis
- Diagram arsitektur pipeline (extract → transform → load → check)
- Diagram star schema
- Screenshot Airflow DAG graph view (sukses run)
- Penjelasan singkat: kenapa star schema, kenapa Parquet bukan CSV

### Alokasi Waktu (±8 jam)
- Design star schema: 1 jam
- PySpark transform script: 3 jam
- Airflow DAG setup + testing: 3 jam
- README + diagram: 1 jam
