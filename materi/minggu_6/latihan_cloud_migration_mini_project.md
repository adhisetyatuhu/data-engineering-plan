---
title: Hands-on Cloud Migration + Mini Project v4
parent: Minggu 6 - Cloud Platform Fundamentals
nav_order: 7
---

# Sabtu–Minggu — Hands-on: Migrasi Pipeline ke Cloud Storage & BigQuery

*Sabtu 4 jam + Minggu 4 jam. Kelanjutan langsung `minggu_6.md` bagian "Mini Project: Migrasi Pipeline E-Commerce ke Cloud Data Warehouse" — upgrade dari `materi/minggu_5/latihan_catalog_lineage_mini_project.md`.*

> **Repo yang sama**: tetap `ecommerce-etl-pipeline`, tambah folder `cloud/`. Pipeline **lokal** (Postgres `pg-belajar`, Airflow, Spark) **tidak dihapus** — cloud jadi target **tambahan**, supaya kamu tetap punya cara menjalankan & mendemokan semuanya secara lokal tanpa bergantung akun cloud aktif setiap saat.

## Tujuan Belajar

- [ ] Setup GCP project dengan budget alert **sebelum** membuat resource apapun
- [ ] Upload data (raw + processed) ke Cloud Storage dengan struktur folder & lifecycle policy
- [ ] Membuat service account least-privilege untuk pipeline (menerapkan `hari_4_iam.md`)
- [ ] Load star schema Parquet ke BigQuery dan menjalankan ulang query analisis Minggu 1-2
- [ ] (Bonus) Memodifikasi DAG Airflow supaya bisa memuat data ke BigQuery, bukan cuma Postgres lokal

## Untuk Instruktur

**Ini sesi dengan risiko finansial nyata** — beda dari semua sesi hands-on sebelumnya. Wajibkan Langkah 0 (budget alert) selesai **sebelum** peserta membuat resource apapun, tanpa kecuali. Kalau resource/waktu terbatas, prioritaskan Bagian 1 Tahap 1-2 (storage + IAM, relatif murah & aman) dan BigQuery Sandbox untuk latihan query (`hari_5_cloud_data_warehouse.md`) — bagian yang paling berisiko biaya adalah lupa membersihkan resource di akhir, jadi Langkah pembersihan di akhir file ini **sama pentingnya** dengan langkah setup di awal.

---

## Langkah 0 (Wajib, Sebelum Apapun): Setup Project & Budget Alert (±30 menit)

```bash
gcloud projects create ecommerce-etl-pipeline-belajar --name="Ecommerce ETL Belajar"
gcloud config set project ecommerce-etl-pipeline-belajar

# Aktifkan billing (butuh billing account -- pakai free trial credit $300/90 hari untuk akun baru)
gcloud billing projects link ecommerce-etl-pipeline-belajar --billing-account=BILLING_ACCOUNT_ID
```

**Set budget alert** (Console → Billing → Budgets & alerts → Create Budget):
- Set budget nominal kecil (mis. Rp 150.000 / $10) untuk project ini
- Alert di 50%, 90%, 100% dari budget itu — kamu akan dapat email peringatan **sebelum** biaya membengkak tak terduga

Ini bukan langkah opsional — sesuai peringatan eksplisit `minggu_6.md`: "Pastikan pakai free tier / set budget alert untuk menghindari biaya tak terduga."

---

## Bagian 1 (Sabtu, 4 jam): Cloud Storage & IAM

### Tahap 1: Setup Bucket sebagai Data Lake (±2 jam)

```bash
# Nama bucket harus unik global -- tambahkan project ID sebagai suffix
gcloud storage buckets create gs://ecommerce-data-lake-ecommerce-etl-pipeline-belajar \
  --location=asia-southeast1 \
  --uniform-bucket-level-access
```

`--uniform-bucket-level-access` mengunci kontrol akses cuma lewat IAM (bukan ACL per-object lama yang lebih gampang salah konfigurasi jadi publik tidak sengaja) — praktik yang direkomendasikan, langsung menghindari kesalahan umum #4 di `hari_2_object_storage.md`.

**Upload data** dengan struktur folder sesuai `hari_2_object_storage.md` Latihan #1:

```bash
BUCKET=gs://ecommerce-data-lake-ecommerce-etl-pipeline-belajar

gcloud storage cp data/raw/online_retail_II.csv ${BUCKET}/raw/

for table in dim_customer dim_product dim_date fact_sales; do
  gcloud storage cp data/warehouse/${table}/*.parquet ${BUCKET}/processed/${table}/
done
```

**Setup lifecycle policy** (persis contoh `hari_2_object_storage.md`, simpan sebagai `cloud/lifecycle.json`):

```json
{
  "lifecycle": {
    "rule": [
      {
        "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
        "condition": {"age": 30, "matchesPrefix": ["raw/"]}
      },
      {
        "action": {"type": "Delete"},
        "condition": {"age": 730, "matchesPrefix": ["raw/"]}
      }
    ]
  }
}
```

```bash
gcloud storage buckets update ${BUCKET} --lifecycle-file=cloud/lifecycle.json
```

### Tahap 2: Setup IAM & Service Account (±2 jam)

Ikuti persis rancangan least-privilege dari `hari_4_iam.md` Latihan #1:

```bash
gcloud iam service-accounts create etl-pipeline-sa \
  --display-name="ETL Pipeline Service Account"

SA_EMAIL=etl-pipeline-sa@ecommerce-etl-pipeline-belajar.iam.gserviceaccount.com

gcloud storage buckets add-iam-policy-binding ${BUCKET} \
  --member=serviceAccount:${SA_EMAIL} \
  --role=roles/storage.objectAdmin   # baca+tulis, dibatasi HANYA ke bucket ini

gcloud projects add-iam-policy-binding ecommerce-etl-pipeline-belajar \
  --member=serviceAccount:${SA_EMAIL} \
  --role=roles/bigquery.dataEditor

gcloud projects add-iam-policy-binding ecommerce-etl-pipeline-belajar \
  --member=serviceAccount:${SA_EMAIL} \
  --role=roles/bigquery.jobUser
```

Simpan ringkasan binding ini sebagai `cloud/iam_policy.json` (dokumentasi, sesuai struktur repo `minggu_6.md`) — bisa berupa hasil `gcloud projects get-iam-policy ecommerce-etl-pipeline-belajar --format=json` yang difilter ke binding relevan saja.

**Untuk dipakai dari Airflow container** (Bagian 2 Bonus), attached service account tidak berlaku (container Docker lokal bukan resource GCP) — dalam kasus ini JSON key **tetap dibutuhkan**, tapi **wajib** ditambahkan ke `.gitignore` segera setelah dibuat:

```bash
gcloud iam service-accounts keys create cloud/etl-pipeline-sa-key.json \
  --iam-account=${SA_EMAIL}
echo "cloud/etl-pipeline-sa-key.json" >> .gitignore
```

### Deliverable Sabtu

- [ ] Budget alert aktif (Langkah 0)
- [ ] Bucket berisi `raw/online_retail_II.csv` dan `processed/{dim_customer,dim_product,dim_date,fact_sales}/*.parquet`
- [ ] Lifecycle policy aktif (`gcloud storage buckets describe ${BUCKET} --format="default(lifecycle_config)"` menunjukkan rule yang sudah dibuat)
- [ ] Service account `etl-pipeline-sa` dengan role least-privilege (bukan `Owner`/`Editor`)

---

## Bagian 2 (Minggu, 4 jam): Load ke BigQuery & Jalankan Ulang Query

### Tahap 3: Load ke BigQuery (±1.5 jam)

```bash
bq mk --dataset --location=asia-southeast1 ecommerce-etl-pipeline-belajar:ecommerce_warehouse

BUCKET=gs://ecommerce-data-lake-ecommerce-etl-pipeline-belajar

for table in dim_customer dim_product dim_date fact_sales; do
  bq load --source_format=PARQUET \
    ecommerce_warehouse.${table} \
    "${BUCKET}/processed/${table}/*.parquet"
done
```

BigQuery **otomatis mendeteksi skema** dari file Parquet (Parquet menyimpan skema di dalam file itu sendiri — salah satu alasan format ini dipilih sejak `materi/minggu_3/latihan_pipeline_mini_project.md`, terbayar lagi di sini tanpa perlu definisi skema manual).

### Jalankan Ulang Query Analisis Minggu 1-2 (±1.5 jam)

Simpan di `cloud/bigquery_queries/rfm_analysis.sql` (persis struktur `minggu_6.md`) — pakai query dari `hari_5_cloud_data_warehouse.md` (top produk, RFM), ganti `PROJECT_ID` dengan project id kamu:

```bash
bq query --use_legacy_sql=false < cloud/bigquery_queries/rfm_analysis.sql
```

**Bandingkan performa** dengan versi lokal (SQLite/Postgres, Minggu 1-2) — catat di `cloud/setup_notes.md`: waktu eksekusi, dan **volume data yang di-scan** (`bq query --dry_run` menunjukkan estimasi bytes processed **sebelum** benar-benar menjalankan query — kebiasaan baik untuk selalu cek ini dulu di tabel besar, sesuai `hari_5_cloud_data_warehouse.md` Kesalahan Umum #1-2).

Untuk data seukuran Online Retail II (ratusan ribu baris), **wajar** kalau hasilnya tidak jauh berbeda atau bahkan BigQuery terasa sedikit lebih lambat untuk query sederhana (ada overhead job scheduling di baliknya) — sama seperti catatan jujur soal benchmark PySpark vs Pandas di `materi/minggu_3/latihan_pipeline_mini_project.md`, manfaat BigQuery baru terasa jelas di **volume data yang jauh lebih besar** dari yang dipakai roadmap belajar ini.

### Bonus: Update DAG Airflow untuk Load ke BigQuery (±1 jam, opsional)

Tambahkan 2 task baru di `dags/ecommerce_etl_dag.py` (setelah `data_quality_check`, sebelum/menggantikan `load_to_warehouse` yang lama — boleh pertahankan keduanya, load ke Postgres **dan** BigQuery, atau ganti total tergantung preferensi):

```python
@task
def upload_to_gcs(warehouse_dir: str) -> str:
    from pathlib import Path
    from google.cloud import storage

    client = storage.Client()
    bucket = client.bucket("ecommerce-data-lake-ecommerce-etl-pipeline-belajar")
    gcs_prefix = "processed"

    for table in ["dim_customer", "dim_product", "dim_date", "fact_sales"]:
        for local_file in Path(f"{warehouse_dir}/{table}").glob("*.parquet"):
            blob = bucket.blob(f"{gcs_prefix}/{table}/{local_file.name}")
            blob.upload_from_filename(str(local_file))

    return f"gs://ecommerce-data-lake-ecommerce-etl-pipeline-belajar/{gcs_prefix}"


@task
def load_to_bigquery(gcs_processed_prefix: str) -> None:
    from google.cloud import bigquery

    client = bigquery.Client()
    dataset_id = "ecommerce_warehouse"

    for table in ["dim_customer", "dim_product", "dim_date", "fact_sales"]:
        table_id = f"{client.project}.{dataset_id}.{table}"
        job_config = bigquery.LoadJobConfig(
            source_format=bigquery.SourceFormat.PARQUET,
            write_disposition=bigquery.WriteDisposition.WRITE_TRUNCATE,
        )
        uri = f"{gcs_processed_prefix}/{table}/*.parquet"
        load_job = client.load_table_from_uri(uri, table_id, job_config=job_config)
        load_job.result()
        print(f"Loaded {table} ke BigQuery")
```

Sambungkan ke DAG existing:
```python
checked_dir = data_quality_check(warehouse_dir)
gcs_prefix = upload_to_gcs(checked_dir)
load_to_bigquery(gcs_prefix)
loaded_dir = load_to_warehouse(checked_dir)   # tetap load ke Postgres lokal juga, kalau dipertahankan
notify(loaded_dir)
```

**Kredensial di container Airflow**: mount `cloud/etl-pipeline-sa-key.json` sebagai volume, set environment variable `GOOGLE_APPLICATION_CREDENTIALS=/opt/airflow/cloud/etl-pipeline-sa-key.json` di `docker-compose.yml` — `google-cloud-storage`/`google-cloud-bigquery` client otomatis memakainya untuk autentikasi tanpa kode tambahan.

**Kenapa ini "bonus", bukan wajib**: memindahkan **seluruh** target load pipeline ke cloud adalah keputusan arsitektural yang signifikan (menyambung `hari_5_cloud_data_warehouse.md` bagian ELT) — untuk tujuan belajar, memahami **cara kerjanya** (dua task baru di atas) sudah mencapai tujuan pembelajaran minggu ini, tanpa harus berkomitmen mengganti total pipeline yang sudah berjalan baik secara lokal.

### Update README & Struktur Repo Final

```
ecommerce-etl-pipeline/
├── README.md
├── GOVERNANCE.md
├── dags/
│   └── ecommerce_etl_dag.py          # update: tambah upload_to_gcs, load_to_bigquery (opsional)
├── spark_jobs/
├── great_expectations/
├── streaming-demo/
├── governance/
├── cloud/                              # baru
│   ├── iam_policy.json
│   ├── lifecycle.json
│   ├── bigquery_queries/
│   │   └── rfm_analysis.sql
│   └── setup_notes.md
├── data/
├── Dockerfile
├── docker-compose.yml
└── diagrams/
    ├── star_schema.png
    ├── pipeline_architecture_v2.png
    ├── data_lineage.png
    └── cloud_architecture.png          # baru
```

README section baru **"Cloud Migration"**:
- Diagram before/after (pipeline lokal murni vs pipeline + cloud storage/warehouse)
- Screenshot BigQuery console dengan tabel ter-load, hasil query
- Catatan perbandingan biaya/performa (dari `cloud/setup_notes.md`)

### Langkah Pembersihan (Wajib, Setelah Selesai)

```bash
# Hapus resource yang tidak perlu terus menyala, supaya tidak ada biaya berkelanjutan tak terduga
bq rm -r -d ecommerce-etl-pipeline-belajar:ecommerce_warehouse
gcloud storage rm -r gs://ecommerce-data-lake-ecommerce-etl-pipeline-belajar
# (opsional, kalau sudah benar-benar selesai dengan project ini)
gcloud projects delete ecommerce-etl-pipeline-belajar
```

Sama pentingnya dengan setup — resource cloud yang dibiarkan menyala/tersimpan setelah tidak dipakai adalah sumber biaya tak terduga paling umum, persis peringatan yang dibahas `hari_1_konsep_cloud.md` Kesalahan Umum #4.

### Kriteria "Selesai" untuk Minggu 6

- [ ] Budget alert aktif sebelum resource apapun dibuat
- [ ] Data (raw + processed) ter-upload ke Cloud Storage dengan struktur folder rapi + lifecycle policy aktif
- [ ] Service account least-privilege dipakai (bukan akun personal) untuk semua operasi pipeline ke cloud
- [ ] Star schema ter-load ke BigQuery, query top produk & RFM berhasil dijalankan ulang di sana
- [ ] `cloud/setup_notes.md` berisi catatan perbandingan biaya/performa lokal vs cloud
- [ ] Bisa menjelaskan ke orang lain: kenapa separation of storage & compute (`hari_5_cloud_data_warehouse.md`) relevan untuk kasus pipeline ini, dan bagian mana dari `GOVERNANCE.md` (Minggu 5) yang sekarang benar-benar ditegakkan lewat IAM (`hari_4_iam.md` Latihan #4)
- [ ] Resource cloud yang tidak lagi dipakai sudah dibersihkan (langkah pembersihan di atas)

Kalau semua tercentang, lanjut ke `minggu_7.md` — pipeline yang sama akan di-containerize dengan Docker image custom & di-deploy ke Kubernetes, melengkapi sisi "bagaimana kode pipeline ini sendiri dikemas & dijalankan" (beda dari minggu ini yang fokus "di mana data disimpan & diproses").
