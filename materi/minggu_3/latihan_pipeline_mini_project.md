---
title: Hands-on PySpark & Airflow + Mini Project
parent: Minggu 3 - Arsitektur Data & Big Data Pipeline
nav_order: 7
---

# Sabtu–Minggu — Hands-on PySpark & Airflow, Upgrade Mini Project Jadi Pipeline Otomatis

*Sabtu 4 jam + Minggu 4 jam. Ini kelanjutan langsung `minggu_3.md` bagian "Mini Project: Automated ETL Pipeline untuk E-Commerce Data Warehouse" — file ini menjabarkan tiap tahap dengan kode konkret, seperti pola `materi/minggu_2/latihan_eda_dan_mini_project.md` di minggu lalu.*

> **Rename repo, bukan repo baru**: mulai minggu ini repo portfolio `ecommerce-sales-analysis` (Minggu 1–2) di-rename jadi **`ecommerce-etl-pipeline`** — lihat `00_overview.md` bagian "Tentang Repo Portfolio". Histori commit Minggu 1–2 tetap dipertahankan.

## Tujuan Belajar

- [ ] Menjalankan transformasi data skala besar (Online Retail II) dengan PySpark, dan membandingkannya langsung dengan pendekatan Pandas
- [ ] Menerapkan rancangan star schema (`hari_2_data_modeling.md`) ke data sungguhan lewat script PySpark
- [ ] Membangun DAG Airflow dengan 4 task (`extract → transform → load → data_quality_check`) yang berjalan otomatis
- [ ] Menghasilkan upgrade nyata pada repo portfolio: dari notebook analisis manual (Minggu 1–2) jadi pipeline yang bisa dijadwalkan berjalan sendiri

## Untuk Instruktur

Ini sesi hands-on terberat sejauh ini di roadmap — dua tool baru (Spark, Airflow) sekaligus diintegrasikan jadi satu pipeline utuh dalam waktu terbatas. Realistis untuk sebagian peserta stuck di masalah environment/integrasi (Java tidak ketemu, Airflow container tidak bisa akses Postgres, dst) — ini **bagian dari pembelajaran**, bukan tanda kegagalan. Instruktur sebaiknya sudah menjalankan sendiri seluruh alur ini sebelum sesi, supaya tahu error umum yang mungkin muncul dan solusinya cepat.

---

## Bagian 1 (Sabtu, 4 jam): PySpark Hands-on — Transform Online Retail II

### Kenapa Bandingkan dengan Pandas Dulu

Sebelum langsung percaya "Spark itu lebih cepat", biarkan peserta **membuktikan sendiri** kapan itu benar dan kapan tidak. Untuk data seukuran Online Retail II (ratusan ribu baris, muat nyaman di RAM 1 laptop modern), **wajar** kalau hasilnya Pandas sama cepat atau bahkan lebih cepat dari PySpark local mode — overhead menyalakan JVM + menyusun rencana eksekusi terdistribusi (`hari_5_spark.md`) baru "terbayar" pada data yang jauh lebih besar dari itu. Ini pelajaran penting yang sengaja dibiarkan peserta temukan sendiri, bukan cuma dibacakan.

### Setup (±30 menit)

```bash
pip install pyspark==3.5.1 pandas sqlalchemy psycopg2-binary pyarrow
java -version   # pastikan JDK 11/17 terpasang, PySpark butuh ini
```

Kalau `java -version` gagal, install dulu (mis. `brew install openjdk@17` di Mac, lalu set `JAVA_HOME` sesuai instruksi output `brew`).

### Benchmark: PySpark vs Pandas (±30 menit)

```python
import time
import pandas as pd
from pyspark.sql import SparkSession, functions as F

RAW_PATH = "data/raw/online_retail_II.csv"

# --- Pandas ---
t0 = time.time()
df = pd.read_csv(RAW_PATH)
clean = df.dropna(subset=["Customer ID"]).drop_duplicates()
clean = clean[clean["Price"] > 0]
clean["revenue"] = clean["Quantity"] * clean["Price"]
revenue_per_country = clean.groupby("Country")["revenue"].sum().sort_values(ascending=False)
t_pandas = time.time() - t0
print(f"Pandas: {t_pandas:.2f} detik")

# --- PySpark ---
t0 = time.time()
spark = SparkSession.builder.appName("benchmark").master("local[*]").getOrCreate()
sdf = spark.read.csv(RAW_PATH, header=True, inferSchema=True)
sclean = (
    sdf.dropna(subset=["Customer ID"])
    .dropDuplicates()
    .filter(F.col("Price") > 0)
    .withColumn("revenue", F.col("Quantity") * F.col("Price"))
)
sclean.groupBy("Country").agg(F.sum("revenue").alias("revenue")).orderBy(F.col("revenue").desc()).collect()
t_spark = time.time() - t0
print(f"PySpark: {t_spark:.2f} detik")
```

**Tulis hasilnya** (angka aktual di laptop kamu) di `README.md` repo, plus 2–3 kalimat kesimpulan. Kemungkinan besar `t_pandas < t_spark` untuk data seukuran ini — kalau itu hasilnya, itu **hasil yang benar dan diharapkan**, bukan kesalahan setup. Diskusikan dengan instruktur/rekan: pada volume data berapa kira-kira hasilnya akan berbalik?

### Tahap 1: Finalisasi Star Schema untuk Online Retail II (±30 menit)

Terapkan rancangan dari `hari_2_data_modeling.md` ke skema kolom Online Retail II (lihat `materi/minggu_2/00_overview.md` untuk detail kolom asli):

```
fact_sales
├── invoice        (degenerate dimension — no. invoice asli)
├── stock_code  FK -> dim_product
├── customer_id FK -> dim_customer
├── date_id     FK -> dim_date
├── quantity
├── unit_price
├── revenue        (quantity * unit_price)
└── is_return       (boolean, dari invoice yang diawali 'C')

dim_customer                     dim_product                    dim_date
├── customer_id PK                ├── stock_code PK               ├── date_id PK
└── country                       └── description                 ├── full_date
                                                                    ├── day / month / quarter / year
                                                                    └── is_weekend
```

**Catatan desain (sengaja disederhanakan)**: FK di `fact_sales` memakai **natural key langsung** (`customer_id`, `stock_code`, `date_id`), bukan surrogate key auto-generate — ini pola **SCD Type 1** (replace penuh tiap pipeline jalan, tanpa histori versi), sesuai yang sudah disinggung di `hari_2_data_modeling.md`. SCD Type 2 dengan surrogate key penuh akan dipraktikkan saat topik Data Governance di Minggu 5, ketika kebutuhan melacak histori perubahan data jadi lebih relevan.

Dokumentasikan diagram ini di `diagrams/star_schema.png` (draw.io/dbdiagram.io), sesuai instruksi `minggu_3.md`.

### Tahap 2: Script PySpark (±2.5 jam)

**`spark_jobs/clean_transform.py`** — cleaning, reuse logic dari `materi/minggu_2/latihan_eda_dan_mini_project.md` (dropna Customer ID, drop duplicate, filter harga > 0, tandai retur), sekarang ditulis dengan PySpark:

```python
import sys
from pyspark.sql import SparkSession, functions as F


def main(raw_path: str, staging_path: str) -> None:
    spark = (
        SparkSession.builder
        .appName("ecommerce-etl-pipeline-clean")
        .master("local[*]")
        .getOrCreate()
    )

    raw = spark.read.csv(raw_path, header=True, inferSchema=True)
    n_before = raw.count()

    clean = (
        raw.dropna(subset=["Customer ID"])
        .dropDuplicates()
        .filter(F.col("Price") > 0)
        .withColumn("is_return", F.col("Invoice").cast("string").startswith("C"))
        .withColumn("revenue", F.col("Quantity") * F.col("Price"))
        .withColumn("invoice_date", F.to_date("InvoiceDate"))
    )

    n_after = clean.count()
    pct = 100.0 * (n_before - n_after) / n_before
    print(f"Rows: {n_before} -> {n_after} ({n_before - n_after} dibuang, {pct:.1f}%)")

    clean.write.mode("overwrite").parquet(staging_path)
    spark.stop()


if __name__ == "__main__":
    main(sys.argv[1], sys.argv[2])
```

**`spark_jobs/build_star_schema.py`** — baca hasil cleaning, pecah jadi fact & dimension table sesuai rancangan Tahap 1:

```python
import sys
from pyspark.sql import SparkSession, functions as F


def main(staging_path: str, output_dir: str) -> None:
    spark = (
        SparkSession.builder
        .appName("ecommerce-etl-pipeline-star-schema")
        .master("local[*]")
        .getOrCreate()
    )

    clean = spark.read.parquet(staging_path)

    dim_customer = (
        clean.select(
            F.col("Customer ID").cast("int").alias("customer_id"),
            F.col("Country").alias("country"),
        )
        .distinct()
    )

    dim_product = (
        clean.groupBy(F.col("StockCode").alias("stock_code"))
        .agg(F.first("Description", ignorenulls=True).alias("description"))
    )

    dim_date = (
        clean.select(F.col("invoice_date").alias("full_date"))
        .distinct()
        .withColumn("date_id", F.date_format("full_date", "yyyyMMdd").cast("int"))
        .withColumn("day", F.dayofmonth("full_date"))
        .withColumn("month", F.month("full_date"))
        .withColumn("quarter", F.quarter("full_date"))
        .withColumn("year", F.year("full_date"))
        # Spark: dayofweek 1=Minggu ... 7=Sabtu
        .withColumn("is_weekend", F.dayofweek("full_date").isin([1, 7]))
    )

    fact_sales = (
        clean.withColumn("date_id", F.date_format("invoice_date", "yyyyMMdd").cast("int"))
        .select(
            F.col("Invoice").alias("invoice"),
            F.col("StockCode").alias("stock_code"),
            F.col("Customer ID").cast("int").alias("customer_id"),
            "date_id",
            F.col("Quantity").alias("quantity"),
            F.col("Price").alias("unit_price"),
            "revenue",
            "is_return",
        )
    )

    dim_customer.write.mode("overwrite").parquet(f"{output_dir}/dim_customer")
    dim_product.write.mode("overwrite").parquet(f"{output_dir}/dim_product")
    dim_date.write.mode("overwrite").parquet(f"{output_dir}/dim_date")
    fact_sales.write.mode("overwrite").parquet(f"{output_dir}/fact_sales")

    print(f"fact_sales: {fact_sales.count()} baris")
    print(f"dim_customer: {dim_customer.count()} baris, dim_product: {dim_product.count()} baris")

    spark.stop()


if __name__ == "__main__":
    main(sys.argv[1], sys.argv[2])
```

**Jalankan manual dulu** (sebelum masuk Airflow — pastikan scriptnya benar-benar jalan berdiri sendiri lewat command line dulu, supaya kalau nanti ada error di Airflow besok, kamu sudah yakin sumber masalahnya di orkestrasi, bukan di logic Spark-nya):
```bash
python spark_jobs/clean_transform.py data/raw/online_retail_II.csv data/warehouse/_staging/retail_clean
python spark_jobs/build_star_schema.py data/warehouse/_staging/retail_clean data/warehouse
```

**Kenapa `write.mode("overwrite")` bukan `"append"`**: karena pola SCD Type 1 di atas, tiap kali pipeline jalan seluruh star schema dibangun ulang dari data sumber terbaru — bukan menambah data baru ke yang lama. Ini keputusan desain sederhana yang cocok untuk skala mini project (dan konsisten dengan sifat `daily` full-refresh DAG di Bagian 2), tapi bukan pola yang scalable untuk data yang sangat besar (di mana `append`/incremental load jadi kebutuhan nyata) — poin bagus untuk didiskusikan, meski di luar cakupan implementasi minggu ini.

### Deliverable Sabtu

- [ ] Hasil benchmark PySpark vs Pandas tertulis di `README.md`
- [ ] `diagrams/star_schema.png` untuk skema Online Retail II
- [ ] `spark_jobs/clean_transform.py` dan `spark_jobs/build_star_schema.py` jalan tanpa error dari command line, menghasilkan 4 folder Parquet di `data/warehouse/`

---

## Bagian 2 (Minggu, 4 jam): Orkestrasi dengan Airflow + Selesaikan Mini Project

### Setup Airflow (±45 menit)

Docker-compose dasar sudah disiapkan di `00_overview.md`. Karena task `transform_with_spark` nanti perlu menjalankan PySpark **di dalam** container Airflow, image resminya perlu ditambah Java + `pyspark` — bikin `Dockerfile` kecil:

```dockerfile
# Dockerfile
FROM apache/airflow:2.9.3
USER root
RUN apt-get update \
    && apt-get install -y --no-install-recommends openjdk-17-jre-headless \
    && apt-get clean
USER airflow
RUN pip install --no-cache-dir pyspark==3.5.1 pandas sqlalchemy psycopg2-binary pyarrow
```

Ubah `docker-compose.yml` hasil download di `00_overview.md`: ganti baris `image: apache/airflow:2.9.3` jadi `build: .` (supaya docker-compose build image kustom di atas, bukan pakai image resmi apa adanya), lalu tambahkan volume mount untuk kode & data project:

```yaml
# tambahkan di bagian x-airflow-common > volumes (docker-compose.yml)
volumes:
  - ./dags:/opt/airflow/dags
  - ./spark_jobs:/opt/airflow/spark_jobs
  - ./data:/opt/airflow/data
  - ./logs:/opt/airflow/logs
  - ./plugins:/opt/airflow/plugins
```

```bash
docker compose build
docker compose up airflow-init
docker compose up -d
```

**Koneksi ke Postgres `pg-belajar`**: karena `pg-belajar` dijalankan terpisah (`docker run` biasa, bukan bagian docker-compose Airflow), container Airflow tidak otomatis bisa resolve `pg-belajar` sebagai hostname. Cara paling simpel di Docker Desktop (Mac/Windows): pakai `host.docker.internal` sebagai host, karena port `5432` `pg-belajar` sudah di-publish ke mesin host:
```
postgresql://postgres:belajar@host.docker.internal:5432/postgres
```

### Tahap 3: DAG Airflow (±2.5 jam)

`dags/ecommerce_etl_dag.py` — pakai **TaskFlow API** (`@dag`/`@task`, gaya Airflow modern berbasis Python function, lebih natural buat developer dibanding definisi `Operator` manual gaya lama):

```python
from datetime import datetime

from airflow.decorators import dag, task


@dag(
    dag_id="ecommerce_etl_pipeline",
    description="Extract -> Transform (Spark) -> Load (Postgres) -> Data Quality Check",
    schedule="@daily",
    start_date=datetime(2026, 1, 1),
    catchup=False,
    tags=["ecommerce-etl-pipeline"],
)
def ecommerce_etl_pipeline():

    @task
    def extract_raw_data() -> str:
        import os

        raw_path = "/opt/airflow/data/raw/online_retail_II.csv"
        if not os.path.exists(raw_path):
            raise FileNotFoundError(f"Raw file tidak ditemukan: {raw_path}")
        return raw_path

    @task
    def transform_with_spark(raw_path: str) -> str:
        import subprocess

        staging_path = "/opt/airflow/data/warehouse/_staging/retail_clean"
        output_dir = "/opt/airflow/data/warehouse"

        subprocess.run(
            ["python", "/opt/airflow/spark_jobs/clean_transform.py", raw_path, staging_path],
            check=True,
        )
        subprocess.run(
            ["python", "/opt/airflow/spark_jobs/build_star_schema.py", staging_path, output_dir],
            check=True,
        )
        return output_dir

    @task
    def load_to_warehouse(output_dir: str) -> None:
        import pandas as pd
        from sqlalchemy import create_engine

        engine = create_engine("postgresql://postgres:belajar@host.docker.internal:5432/postgres")
        for table in ["dim_customer", "dim_product", "dim_date", "fact_sales"]:
            df = pd.read_parquet(f"{output_dir}/{table}")
            df.to_sql(table, engine, if_exists="replace", index=False)
            print(f"Loaded {len(df)} baris ke tabel {table}")

    @task
    def data_quality_check() -> None:
        from sqlalchemy import create_engine, text

        engine = create_engine("postgresql://postgres:belajar@host.docker.internal:5432/postgres")
        with engine.connect() as conn:
            row_count = conn.execute(text("SELECT COUNT(*) FROM fact_sales")).scalar()
            if row_count == 0:
                raise ValueError("Data quality check gagal: fact_sales kosong.")

            dup_count = conn.execute(text("""
                SELECT COUNT(*) FROM (
                    SELECT invoice, stock_code
                    FROM fact_sales
                    GROUP BY invoice, stock_code
                    HAVING COUNT(*) > 1
                ) dup
            """)).scalar()
            if dup_count > 0:
                raise ValueError(f"Data quality check gagal: {dup_count} baris duplikat di fact_sales.")

            print(f"Data quality check lolos: {row_count} baris, 0 duplikat.")

    raw_path = extract_raw_data()
    warehouse_dir = transform_with_spark(raw_path)
    load_result = load_to_warehouse(warehouse_dir)
    load_result >> data_quality_check()


ecommerce_etl_pipeline()
```

**Kenapa `data_quality_check` sebagai task terpisah, bukan digabung ke `load_to_warehouse`**: supaya kalau ada masalah kualitas data, itu **terlihat jelas sebagai kegagalan tersendiri** di Airflow UI (task merah terpisah dari task load) — bukan tersembunyi di dalam log task lain. Ini pola dasar yang akan diperluas jauh lebih detail di Minggu 4 (Data Quality & Alerting).

**Kenapa `transform_with_spark` memanggil `subprocess.run(["python", ...])`, bukan mengimpor & memanggil fungsi Spark langsung di dalam task Airflow**: menjalankan Spark job sebagai proses terpisah (bukan di dalam proses worker Airflow yang sama) mengisolasi siklus hidup `SparkSession` dari siklus hidup task Airflow — kalau job Spark gagal/crash, itu tidak sampai merusak proses worker Airflow itu sendiri. Pola produksi yang lebih matang biasanya memakai `SparkSubmitOperator` (mengirim job ke cluster Spark sungguhan) — di luar cakupan setup local mode minggu ini, tapi baik diketahui peserta ada di baliknya nanti kalau bekerja dengan cluster Spark sungguhan.

### Jalankan & Verifikasi (±30 menit)

1. Copy `online_retail_II.csv` ke `data/raw/` di dalam repo.
2. Buka `http://localhost:8080`, aktifkan (unpause) DAG `ecommerce_etl_pipeline`, trigger manual run.
3. Amati Graph View — pastikan urutan `extract_raw_data → transform_with_spark → load_to_warehouse → data_quality_check` sesuai rencana, semua task hijau (sukses).
4. Verifikasi data masuk ke Postgres: `docker exec -it pg-belajar psql -U postgres -c "SELECT COUNT(*) FROM fact_sales;"`
5. Screenshot Graph View (run sukses) untuk `README.md`, sesuai instruksi `minggu_3.md`.

### Struktur Repo Final

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
├── Dockerfile
├── docker-compose.yml
└── diagrams/
    └── star_schema.png
```

### README yang Perlu Ditulis (sesuai `minggu_3.md`)

- Diagram arsitektur pipeline (extract → transform → load → check)
- Diagram star schema (`diagrams/star_schema.png`)
- Screenshot Airflow DAG graph view (run sukses)
- Hasil benchmark PySpark vs Pandas (dari Bagian 1) + kesimpulan singkat
- Penjelasan singkat: kenapa star schema, kenapa Parquet bukan CSV untuk data warehouse hasil transform (petunjuk: kolumnar, terkompresi, menyimpan skema — dibanding CSV yang plain text tanpa tipe data)

### Kriteria "Selesai" untuk Minggu 3

- [ ] Repo `ecommerce-etl-pipeline` (hasil rename dari `ecommerce-sales-analysis`) berisi struktur di atas
- [ ] `spark_jobs/clean_transform.py` dan `build_star_schema.py` jalan tanpa error, menghasilkan Parquet fact + 3 dimension table
- [ ] DAG `ecommerce_etl_pipeline` di Airflow jalan end-to-end (4 task hijau), dan bisa dijelaskan kenapa urutan task-nya begitu
- [ ] `data_quality_check` benar-benar bisa **menggagalkan** run kalau dites dengan sengaja merusak data (mis. kosongkan `data/raw/` sementara) — coba ini sekali untuk membuktikan check-nya bekerja, bukan cuma "selalu hijau"
- [ ] Bisa menjelaskan ke orang lain: beda Spark vs Airflow (dari `00_overview.md`: "pekerja vs mandor"), kenapa pipeline ini pola ETL bukan ELT (`hari_3_etl_elt.md`), dan kenapa ini batch bukan streaming (`hari_4_batch_stream.md`)

Kalau semua tercentang, lanjut ke `minggu_4.md` — pipeline yang sama akan di-upgrade dengan Data Quality checks yang lebih matang, alerting, dan pengantar streaming.
