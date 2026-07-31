---
title: Minggu 6 - Cloud Platform Fundamentals
nav_order: 7
---

# Minggu 6 — Cloud Platform Fundamentals

## Pilih 1 Provider Dulu
Rekomendasi: **GCP** atau **AWS** (paling banyak dipakai perusahaan data engineering di Indonesia). GCP biasanya punya learning curve sedikit lebih ramah (BigQuery sangat intuitif).

## Breakdown Harian (±15-20 jam)

| Hari | Jam | Materi |
|---|---|---|
| Senin | 2 jam | Konsep dasar cloud computing: IaaS/PaaS/SaaS, region & availability zone, shared responsibility model |
| Selasa | 2 jam | Storage: object storage (S3/GCS) — konsep bucket, tiering, lifecycle policy |
| Rabu | 2 jam | Compute: virtual machine dasar (EC2/Compute Engine) + serverless (Lambda/Cloud Functions) |
| Kamis | 2 jam | IAM: user, role, policy, service account — konsep security dasar |
| Jumat | 2 jam | Data warehouse cloud: pengantar BigQuery/Redshift/Snowflake |
| Sabtu | 4 jam | Hands-on: setup akun cloud (free tier), upload data ke object storage, buat IAM role sederhana |
| Minggu | 4 jam | Hands-on: load data dari object storage ke BigQuery/Redshift, jalankan query yang sama seperti Minggu 1 |

## Detail Topik
1. **Konsep Dasar Cloud** — elastisitas, pay-as-you-go, managed service, shared responsibility model
2. **Object Storage** — bucket sebagai "data lake" modern, storage class/tiering untuk cost optimization
3. **Compute** — VM dasar (pengenalan) + serverless function (lebih relevan untuk data engineering)
4. **IAM** — fondasi keamanan cloud, konsep least privilege
5. **Cloud Data Warehouse** — separation of storage & compute, auto-scaling — paling relevan untuk data engineer

## Sumber Belajar
- Google Cloud Skills Boost — "Data Engineer Learning Path" (ada free credit)
- AWS Skill Builder — "Data Engineering" fundamentals, atau freeCodeCamp AWS course (YouTube)
- Google Cloud "BigQuery Sandbox" (gratis tanpa billing account)

## Target di Akhir Minggu 6
- Menjelaskan konsep dasar cloud computing dan model layanan (IaaS/PaaS/SaaS)
- Upload & kelola data di object storage
- Membuat IAM role/service account dengan permission tepat
- Load data ke cloud data warehouse dan jalankan query analisis dasar

> Catatan: minggu ini cukup padat (15-20 jam vs kapasitas 18 jam) — kalau setup akun cloud/billing makan waktu lebih lama, boleh geser sedikit ke awal Minggu 7.

---

## Mini Project: "Migrasi Pipeline E-Commerce ke Cloud Data Warehouse"

**Kenapa case ini?** Menunjukkan kemampuan membawa pipeline dari local/on-prem ke cloud — skill paling dicari untuk role data engineer di perusahaan modern.

> Lanjutkan repo `ecommerce-etl-pipeline`, tambahkan folder cloud baru.

### Tahap 1: Setup Cloud Storage sebagai Data Lake
- Buat bucket (misal `ecommerce-data-lake`)
- Upload raw CSV dan output Parquet dari Spark job Minggu 3 dengan struktur folder rapi:
  ```
  gs://ecommerce-data-lake/raw/
  gs://ecommerce-data-lake/processed/
  ```
- Setup lifecycle policy sederhana (raw data pindah ke storage class lebih murah setelah 30 hari)

### Tahap 2: Setup IAM & Security
- Buat service account khusus untuk pipeline (`etl-pipeline-sa`)
- Assign role least privilege (hanya read/write ke bucket tertentu)
- Gunakan service account ini untuk autentikasi, bukan akun personal

### Tahap 3: Load ke Cloud Data Warehouse & Query
- Load data dari `processed/` (star schema Parquet) ke BigQuery/Redshift
- Jalankan ulang query analisis Minggu 1-2 (top produk, RFM analysis) — bandingkan performa vs SQLite lokal
- **Bonus**: modifikasi Airflow DAG supaya task `load` menulis ke BigQuery/Redshift (operator seperti `BigQueryInsertJobOperator`)

### Struktur Repo (update dari Minggu 5)
```
ecommerce-etl-pipeline/
├── README.md
├── GOVERNANCE.md
├── dags/
│   └── ecommerce_etl_dag.py         # update: load task target ke cloud warehouse
├── spark_jobs/
├── great_expectations/
├── streaming-demo/
├── governance/
├── cloud/
│   ├── terraform/ (opsional)
│   ├── iam_policy.json
│   ├── bigquery_queries/
│   │   └── rfm_analysis.sql
│   └── setup_notes.md
├── data/
└── diagrams/
    ├── star_schema.png
    ├── pipeline_architecture_v2.png
    ├── data_lineage.png
    └── cloud_architecture.png        # baru
```

### README yang Perlu Diupdate
- Section "Cloud Migration" — diagram before/after (local vs cloud pipeline)
- Screenshot BigQuery/Redshift console dengan tabel ter-load, hasil query
- Catatan singkat perbandingan biaya/performa local vs cloud

### ⚠️ Catatan Penting
Pastikan pakai free tier / set budget alert untuk menghindari biaya tak terduga. BigQuery Sandbox dan AWS Free Tier cukup untuk skala project ini.

### Alokasi Waktu (±8 jam)
- Setup bucket + upload data + lifecycle policy: 2 jam
- Setup IAM & service account: 2 jam
- Load ke warehouse + jalankan ulang query analisis: 2.5 jam
- Update README + diagram: 1.5 jam
