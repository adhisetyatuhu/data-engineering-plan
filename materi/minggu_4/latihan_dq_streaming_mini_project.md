---
title: Hands-on GE & Kafka + Mini Project v2
parent: Minggu 4 - Data Quality, Orchestration & Streaming
nav_order: 7
---

# Sabtu–Minggu — Hands-on Great Expectations & Kafka, Upgrade Pipeline Jadi Production-Grade

*Sabtu 4 jam + Minggu 4 jam. Kelanjutan langsung `minggu_4.md` bagian "Mini Project: Production-Grade ETL Pipeline dengan Data Quality Checks & Real-Time Simulation" — file ini menjabarkan tiap tahap dengan kode konkret, upgrade dari `materi/minggu_3/latihan_pipeline_mini_project.md`.*

> **Repo yang sama, bukan repo baru**: tetap `ecommerce-etl-pipeline`. Tidak ada rename minggu ini (beda dari Minggu 3) — cuma penambahan & perubahan struktur di dalam repo yang sama.

## Tujuan Belajar

- [ ] Mengintegrasikan Great Expectations sebagai task `data_quality_check` yang **fail-fast** (sebelum `load`, bukan sesudah seperti Minggu 3)
- [ ] Menambahkan retry & alerting (`on_failure_callback`) ke DAG yang sudah ada
- [ ] Membuktikan sendiri bahwa fail-fast bekerja: sengaja merusak data, lihat pipeline berhenti sebelum data buruk sampai warehouse
- [ ] Membangun demo Kafka producer/consumer terpisah yang mensimulasikan streaming dari data yang sama

## Untuk Instruktur

Sama seperti catatan di `materi/minggu_3/latihan_pipeline_mini_project.md`: ini sesi terberat minggu ini, sekarang ditambah lapisan integrasi baru (GE) di atas integrasi yang sudah kompleks (Spark+Airflow+Postgres). Dorong peserta untuk **benar-benar mencoba merusak data dengan sengaja** (bagian "Uji Fail-Fast" di bawah) — ini bagian paling penting secara pedagogis di seluruh minggu ini: kalau data quality check tidak pernah dilihat **gagal** sendiri oleh peserta, mereka tidak akan yakin check itu benar-benar bekerja, cuma percaya karena "kelihatannya hijau terus".

---

## Bagian 1 (Sabtu, 4 jam): Integrasi Great Expectations ke Pipeline

### Setup: Tambahkan GE ke Image Airflow (±15 menit)

Update `Dockerfile` dari Minggu 3 (`materi/minggu_3/latihan_pipeline_mini_project.md`):

```dockerfile
FROM apache/airflow:2.9.3
USER root
RUN apt-get update \
    && apt-get install -y --no-install-recommends openjdk-17-jre-headless \
    && apt-get clean
USER airflow
RUN pip install --no-cache-dir \
    pyspark==3.5.1 pandas sqlalchemy psycopg2-binary pyarrow \
    great_expectations==0.18.*
```

```bash
docker compose build
docker compose up -d
```

### Tahap 1: Bangun Expectation Suite (±1.5 jam)

Aturan yang diminta `minggu_4.md`, diterjemahkan ke expectation GE (lihat `hari_2_great_expectations.md` untuk penjelasan tiap method):

| Dimensi (`hari_1`) | Aturan `minggu_4.md` | Expectation GE |
|---|---|---|
| Completeness | `customer_id`, `invoice_id` tidak boleh null | `expect_column_values_to_not_be_null` |
| Uniqueness | `invoice_id + product_id` tidak boleh duplikat | `expect_compound_columns_to_be_unique` |
| Validity | `unit_price` dan `quantity` harus > 0; `country` harus di list valid | lihat catatan penyesuaian di bawah |
| Consistency | `total_amount` = `quantity × unit_price` | `expect_column_pair_values_to_be_equal` |

**Catatan penyesuaian penting**: aturan `minggu_4.md` bilang "`quantity` harus > 0" — tapi dari `materi/minggu_2/latihan_eda_dan_mini_project.md` dan `materi/minggu_3/latihan_pipeline_mini_project.md`, kita sudah sengaja mempertahankan `quantity` **negatif** sebagai penanda retur yang valid (kolom `is_return`), bukan data kotor. Menegakkan `quantity > 0` secara mentah akan membuat **semua transaksi retur** (yang sah) dianggap gagal validasi — ini contoh nyata kenapa aturan data quality harus dicek ulang terhadap keputusan desain yang sudah diambil sebelumnya, bukan diterapkan mentah-mentah dari spek. Diterapkan jadi 2 expectation terpisah yang lebih tepat: `quantity` tidak boleh **nol** (baik retur atau bukan, nol tidak masuk akal), dan **khusus baris non-retur**, `quantity` harus positif.

`great_expectations/build_suite.py` — dijalankan **sekali** secara manual untuk membangun & menyimpan suite (bukan bagian dari DAG — suite dibangun sekali, dipakai berulang):

```python
import great_expectations as gx
import pandas as pd

WAREHOUSE_DIR = "data/warehouse"
COUNTRY_ALLOWLIST = [
    "United Kingdom", "Germany", "France", "EIRE", "Spain", "Netherlands",
    "Belgium", "Switzerland", "Portugal", "Australia", "Norway", "Italy",
    "Channel Islands", "Finland", "Cyprus", "Sweden", "Austria", "Denmark",
    "Japan", "Poland", "USA", "Israel", "Hong Kong", "Singapore", "Iceland",
    "Canada", "Greece", "Malta", "United Arab Emirates", "European Community",
    "RSA", "Lebanon", "Lithuania", "Brazil", "Czech Republic", "Bahrain",
    "Saudi Arabia", "Unspecified",
]  # sesuaikan dengan negara yang benar-benar muncul di dataset yang kamu download


def build_enriched(warehouse_dir: str) -> pd.DataFrame:
    fact_sales = pd.read_parquet(f"{warehouse_dir}/fact_sales")
    dim_customer = pd.read_parquet(f"{warehouse_dir}/dim_customer")

    enriched = fact_sales.merge(dim_customer, on="customer_id", how="left")
    enriched["computed_total"] = enriched["quantity"] * enriched["unit_price"]
    return enriched


def main():
    context = gx.get_context(project_root_dir="great_expectations")
    enriched = build_enriched(WAREHOUSE_DIR)

    validator = context.sources.pandas_default.read_dataframe(
        enriched, asset_name="fact_sales_enriched"
    )

    # Completeness
    validator.expect_column_values_to_not_be_null("customer_id")
    validator.expect_column_values_to_not_be_null("invoice")

    # Uniqueness
    validator.expect_compound_columns_to_be_unique(column_list=["invoice", "stock_code"])

    # Validity
    validator.expect_column_values_to_not_be_in_set("quantity", [0])
    validator.expect_column_values_to_be_between("unit_price", min_value=0, strict_min=True)
    validator.expect_column_values_to_be_in_set("country", COUNTRY_ALLOWLIST)

    # Consistency
    validator.expect_column_pair_values_to_be_equal("revenue", "computed_total")
    # bonus: referential integrity -- tiap customer_id di fact_sales harus ADA di dim_customer
    validator.expect_column_values_to_not_be_null("country")  # NULL di sini = customer_id tidak ketemu saat JOIN

    validator.expectation_suite_name = "ecommerce_suite"
    validator.save_expectation_suite(discard_failed_expectations=False)
    print("Suite 'ecommerce_suite' tersimpan di great_expectations/expectations/ecommerce_suite.json")


if __name__ == "__main__":
    main()
```

**Baris non-retur (`quantity > 0`) divalidasi terpisah** — jalankan tambahan ini sekali (bisa di script yang sama, sebagai bagian eksplorasi, tidak perlu masuk suite utama karena butuh subset data):
```python
sales_only = enriched[~enriched["is_return"]]
print("Baris non-retur dengan quantity <= 0 (harus 0):", (sales_only["quantity"] <= 0).sum())
```

Jalankan: `python great_expectations/build_suite.py` — pastikan file `great_expectations/expectations/ecommerce_suite.json` benar-benar muncul sebelum lanjut ke Tahap 2.

### Tahap 2: Update DAG — Reorder + Task `data_quality_check` + `notify` (±1.5 jam)

Perubahan dari `materi/minggu_3/latihan_pipeline_mini_project.md`:
1. Urutan task berubah jadi **`extract → transform → data_quality_check → load → notify`** (data quality **sebelum** load — fail-fast, lihat `hari_1_data_quality_dimensions.md`).
2. Task `load_to_warehouse` diberi `retries` (lihat `hari_3_airflow_lanjutan.md`).
3. `on_failure_callback` dipasang untuk seluruh DAG (alerting).
4. Task baru `notify` di akhir, jalan kalau semua sukses.

`dags/ecommerce_etl_dag.py` (versi lengkap, gantikan file Minggu 3):

```python
from datetime import datetime, timedelta

from airflow.decorators import dag, task


def send_alert(context):
    task_id = context["task_instance"].task_id
    exception = context.get("exception")
    if task_id == "data_quality_check":
        message = f"[ALERT] Data quality check gagal, load DIBATALKAN: {exception}"
    else:
        message = f"[ALERT] Task `{task_id}` gagal: {exception}"
    print(message)  # ganti dengan requests.post() ke Slack webhook / EmailOperator untuk alerting sungguhan


@dag(
    dag_id="ecommerce_etl_pipeline",
    description="Extract -> Transform (Spark) -> Data Quality (GE) -> Load -> Notify",
    schedule="@daily",
    start_date=datetime(2026, 1, 1),
    catchup=False,
    default_args={"on_failure_callback": send_alert},
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
    def data_quality_check(warehouse_dir: str) -> str:
        import great_expectations as gx
        import pandas as pd

        context = gx.get_context(project_root_dir="/opt/airflow/great_expectations")

        fact_sales = pd.read_parquet(f"{warehouse_dir}/fact_sales")
        dim_customer = pd.read_parquet(f"{warehouse_dir}/dim_customer")
        enriched = fact_sales.merge(dim_customer, on="customer_id", how="left")
        enriched["computed_total"] = enriched["quantity"] * enriched["unit_price"]

        validator = context.sources.pandas_default.read_dataframe(
            enriched, asset_name="fact_sales_enriched"
        )
        suite = context.get_expectation_suite("ecommerce_suite")
        results = validator.validate(expectation_suite=suite)

        if not results.success:
            failed = [
                r.expectation_config.expectation_type
                for r in results.results if not r.success
            ]
            raise ValueError(f"Data quality check gagal. Expectation gagal: {failed}")

        print(f"Data quality check lolos untuk {len(enriched)} baris.")
        return warehouse_dir

    @task(retries=3, retry_delay=timedelta(minutes=1), retry_exponential_backoff=True)
    def load_to_warehouse(warehouse_dir: str) -> str:
        import pandas as pd
        from sqlalchemy import create_engine

        engine = create_engine("postgresql://postgres:belajar@host.docker.internal:5432/postgres")
        for table in ["dim_customer", "dim_product", "dim_date", "fact_sales"]:
            df = pd.read_parquet(f"{warehouse_dir}/{table}")
            df.to_sql(table, engine, if_exists="replace", index=False)
        return warehouse_dir

    @task
    def notify(warehouse_dir: str) -> None:
        import pandas as pd
        fact_sales = pd.read_parquet(f"{warehouse_dir}/fact_sales")
        total_revenue = fact_sales.loc[~fact_sales["is_return"], "revenue"].sum()
        print(
            f"[INFO] Pipeline sukses. fact_sales: {len(fact_sales)} baris, "
            f"total revenue: {total_revenue:.2f}"
        )

    raw_path = extract_raw_data()
    warehouse_dir = transform_with_spark(raw_path)
    checked_dir = data_quality_check(warehouse_dir)
    loaded_dir = load_to_warehouse(checked_dir)
    notify(loaded_dir)


ecommerce_etl_pipeline()
```

Perhatikan **tidak ada** `data_quality_check` versi SQL manual (dari Minggu 3) yang tersisa di sini — digantikan sepenuhnya oleh versi Great Expectations. Task lama itu memeriksa dimensi yang jauh lebih sempit (cuma row count > 0 dan duplikat sederhana); versi baru memeriksa completeness, uniqueness, validity, **dan** consistency sekaligus, terdokumentasi sebagai suite yang bisa dibaca ulang siapa saja tanpa buka kode Python.

### Uji Fail-Fast (±30 menit) — Wajib Dicoba, Bukan Cuma Dibaca

Bukti langsung bahwa `data_quality_check` benar-benar berfungsi sebagai gerbang, bukan formalitas:

1. Buat **salinan** `data/raw/online_retail_II.csv` (jangan timpa file asli), rusak beberapa baris dengan sengaja — misalnya set `Price` jadi negatif di 5 baris pertama pakai script kecil Pandas.
2. Ganti sementara path yang dibaca `extract_raw_data` ke file rusak itu, trigger run manual di Airflow UI.
3. **Amati**: `data_quality_check` harus **gagal merah**, dan `load_to_warehouse`/`notify` berstatus **`upstream_failed`/skipped** — bukti data yang rusak **tidak pernah** sampai menyentuh Postgres.
4. Cek log task `data_quality_check` — pesan error harus menyebutkan expectation mana yang gagal (`expect_column_values_to_be_between` untuk `unit_price`).
5. Kembalikan path ke file asli, trigger ulang, pastikan kembali hijau semua.

### Deliverable Sabtu

- [ ] `great_expectations/expectations/ecommerce_suite.json` tersimpan dan berisi minimal 6 expectation
- [ ] DAG update jalan sukses end-to-end dengan urutan baru (`extract → transform → data_quality_check → load → notify`)
- [ ] Screenshot/log hasil "Uji Fail-Fast" — bukti `data_quality_check` benar-benar bisa gagal dan menghentikan `load`

---

## Bagian 2 (Minggu, 4 jam): Demo Streaming dengan Kafka

### Kenapa Terpisah dari Pipeline Batch Utama

`streaming-demo/` sengaja **dipisah total** dari `dags/ecommerce_etl_dag.py` — bukan kelalaian, tapi keputusan desain yang konsisten dengan `materi/minggu_3/hari_4_batch_stream.md`: pipeline utama (`ecommerce_etl_pipeline`) tetap **batch**, karena kebutuhan bisnisnya (analisis historis) memang tidak butuh real-time. Demo Kafka ini murni untuk **latihan konsep streaming**, mensimulasikan skenario hipotetis "kalau saja butuh proses transaksi real-time" — dijalankan manual/terpisah, bukan bagian dari DAG terjadwal.

### Setup Kafka (±45 menit)

Tambahkan service ke `docker-compose.yml` (KRaft mode — tanpa Zookeeper, image [Bitnami Kafka](https://hub.docker.com/r/bitnami/kafka)):

```yaml
# tambahan service di docker-compose.yml
services:
  kafka:
    image: bitnami/kafka:3.7
    ports:
      - "9092:9092"
    environment:
      - KAFKA_CFG_NODE_ID=0
      - KAFKA_CFG_PROCESS_ROLES=controller,broker
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
```

```bash
docker compose up -d kafka
pip install kafka-python==2.0.*   # di venv lokal -- producer/consumer dijalankan dari host, bukan dari container Airflow
```

### `streaming-demo/producer.py` (±1 jam)

Mensimulasikan transaksi "mengalir masuk satu per satu", diambil dari `fact_sales` hasil pipeline batch (data yang sama, cuma **cara memprosesnya** yang disimulasikan berbeda — batch vs stream, persis perbandingan konsep di `materi/minggu_3/hari_4_batch_stream.md`):

```python
import json
import time

import pandas as pd
from kafka import KafkaProducer

TOPIC = "ecommerce-transactions"

producer = KafkaProducer(
    bootstrap_servers="localhost:9092",
    value_serializer=lambda v: json.dumps(v, default=str).encode("utf-8"),
    key_serializer=lambda k: str(k).encode("utf-8"),
)

fact_sales = pd.read_parquet("data/warehouse/fact_sales").head(200)  # subset supaya demo tidak kelamaan

for _, row in fact_sales.iterrows():
    event = row.to_dict()
    # key = customer_id -> pesan 1 customer yang sama selalu ke partition yang sama (urutan terjaga)
    producer.send(TOPIC, key=event["customer_id"], value=event)
    print(f"Sent: invoice={event['invoice']} customer={event['customer_id']} revenue={event['revenue']}")
    time.sleep(0.3)   # simulasi jeda antar transaksi, seolah real-time

producer.flush()
print("Selesai mengirim semua event.")
```

### `streaming-demo/consumer.py` (±1 jam)

```python
import json

from kafka import KafkaConsumer

TOPIC = "ecommerce-transactions"

consumer = KafkaConsumer(
    TOPIC,
    bootstrap_servers="localhost:9092",
    value_deserializer=lambda v: json.loads(v.decode("utf-8")),
    auto_offset_reset="earliest",   # kalau consumer baru, mulai baca dari awal topic
    group_id="revenue-aggregator",  # consumer group -- lihat hari_4_pengantar_kafka.md
)

running_total = 0.0
event_count = 0

print("Menunggu event... (jalankan producer.py di terminal lain)")
for message in consumer:
    event = message.value
    running_total += event.get("revenue", 0)
    event_count += 1
    print(
        f"[{event_count}] invoice={event['invoice']} revenue={event['revenue']:.2f} "
        f"| running_total={running_total:.2f}"
    )
```

**Cara jalankan** (2 terminal terpisah):
```bash
# Terminal 1
python streaming-demo/consumer.py

# Terminal 2 (setelah consumer siap "Menunggu event...")
python streaming-demo/producer.py
```

Amati: consumer menerima & mengagregasi tiap event **saat itu juga**, satu-per-satu — beda total dari `notify` task di DAG batch yang baru menghitung total **setelah semua data selesai diproses sekaligus**. Ini perbedaan nyata batch vs stream yang sudah dibahas konsepnya di `materi/minggu_3/hari_4_batch_stream.md`, sekarang terlihat langsung sebagai perilaku program.

**Latihan tambahan** (opsional, kalau waktu cukup): jalankan **2 instance consumer** dengan `group_id` yang **sama**, perhatikan pesan terbagi di antara keduanya (load balancing — sesuai `hari_4_pengantar_kafka.md`). Lalu jalankan 1 consumer lagi dengan `group_id` **berbeda**, perhatikan dia menerima **semua** pesan dari awal lagi (broadcast ke consumer group baru).

### Struktur Repo Final (Update dari Minggu 3)

```
ecommerce-etl-pipeline/
├── README.md
├── dags/
│   └── ecommerce_etl_dag.py          # update: reorder task, GE check, retry, alerting
├── spark_jobs/
│   ├── clean_transform.py
│   └── build_star_schema.py
├── great_expectations/
│   ├── build_suite.py
│   └── expectations/
│       └── ecommerce_suite.json
├── streaming-demo/
│   ├── producer.py
│   └── consumer.py
├── data/
│   ├── raw/
│   └── warehouse/
├── Dockerfile                          # update: tambah great_expectations
├── docker-compose.yml                  # update: tambah service kafka
└── diagrams/
    ├── star_schema.png
    └── pipeline_architecture_v2.png    # baru: diagram alur dengan GE check + notify
```

### README yang Perlu Diupdate (sesuai `minggu_4.md`)

- Section **"What's New in v2"**: ringkas 3 penambahan (data quality gate, alerting, streaming demo)
- Screenshot: DAG graph view dengan task baru (5 task: extract, transform, data_quality_check, load, notify)
- Screenshot/output hasil validasi Great Expectations (sukses **dan** gagal — dari "Uji Fail-Fast")
- Cuplikan log consumer Kafka (running total revenue bertambah tiap event)
- Penjelasan trade-off (2-3 kalimat masing-masing): kenapa fail-fast penting (kaitkan ke `hari_1_data_quality_dimensions.md`), kenapa streaming demo dipisah dari batch pipeline (kaitkan ke `hari_4_batch_stream.md` Minggu 3)

### Kriteria "Selesai" untuk Minggu 4

- [ ] `ecommerce_suite.json` berisi expectation completeness, uniqueness, validity, consistency — dan bisa dijelaskan kenapa aturan `quantity` disesuaikan dari spek awal (bagian "Catatan penyesuaian" di atas)
- [ ] DAG jalan sukses dengan urutan `extract → transform → data_quality_check → load → notify`
- [ ] Fail-fast terbukti bekerja (sudah dicoba merusak data dengan sengaja, `load` benar-benar tidak jalan)
- [ ] `load_to_warehouse` punya retry, DAG punya `on_failure_callback`
- [ ] Producer & consumer Kafka jalan berpasangan, consumer menampilkan running total yang bertambah tiap event diterima
- [ ] Bisa menjelaskan ke orang lain: beda consumer group untuk load balancing vs broadcast (`hari_4_pengantar_kafka.md`), dan kenapa pipeline ini masih pakai full load bukan incremental (`hari_5_incremental_cdc.md`) — sebagai limitasi yang disadari, bukan terlewat

Kalau semua tercentang, lanjut ke `minggu_5.md` — dimensi data quality yang sudah ditegakkan minggu ini jadi fondasi langsung untuk Data Governance (data catalog, lineage, policy).
