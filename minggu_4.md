# Minggu 4 — Lanjutan Pipeline: Data Quality, Orchestration Lanjutan, Streaming Intro

## Breakdown Harian (±18 jam)

| Hari | Jam | Materi |
|---|---|---|
| Senin | 2 jam | Data quality dalam pipeline: dimensi kualitas data (completeness, uniqueness, validity, consistency, timeliness) |
| Selasa | 2 jam | Tools data quality: pengantar Great Expectations — konsep expectation, validation |
| Rabu | 2 jam | Orchestration lanjutan: Airflow — sensors, task dependencies kompleks, retry & alerting, XComs |
| Kamis | 2 jam | Pengantar streaming: konsep Kafka (topic, producer, consumer, partition) — level konsep |
| Jumat | 2 jam | Incremental load vs full load, CDC (Change Data Capture) |
| Sabtu | 4 jam | Hands-on: implementasi Great Expectations pada pipeline Minggu 3 |
| Minggu | 4 jam | Hands-on: setup Kafka lokal (Docker), producer/consumer sederhana untuk simulasi streaming |

## Detail Topik
1. **Data Quality Dimensions** — 5 dimensi utama, fondasi untuk Data Governance Minggu 5
2. **Great Expectations** — buat "expectation suite" (kolom tidak boleh null, value dalam range, tipe data sesuai)
3. **Airflow Lanjutan** — sensors, branching, parallel tasks, alerting (email/Slack)
4. **Pengantar Kafka** — konsep event-driven architecture, kenapa dipakai untuk data real-time
5. **Incremental Load & CDC** — kenapa full load tidak efisien untuk data besar; tools seperti Debezium (pengenalan saja)

## Sumber Belajar
- Great Expectations official docs — "Getting Started"
- Airflow docs bagian "Concepts" (sensors, XComs, branching)
- Confluent "Kafka 101" (gratis, beginner-friendly), atau Kafka official quickstart
- Artikel/video overview untuk CDC

## Target di Akhir Minggu 4
- Mengimplementasikan data quality checks dengan Great Expectations
- Membuat Airflow DAG dengan dependency dan error handling production-ready
- Menjelaskan konsep dasar Kafka dan streaming architecture
- Menjelaskan perbedaan incremental vs full load serta konsep CDC

---

## Mini Project: "Production-Grade ETL Pipeline dengan Data Quality Checks & Real-Time Simulation"

**Kenapa case ini?** Upgrade langsung dari pipeline Minggu 3 — menunjukkan progresi dari "pipeline yang jalan" ke "pipeline yang reliable".

> Lanjutkan repo `ecommerce-etl-pipeline` yang sama (bukan repo baru).

### Tahap 1: Data Quality dengan Great Expectations
Validation layer sebelum data masuk warehouse:
- **Completeness**: `customer_id`, `invoice_id` tidak boleh null
- **Uniqueness**: kombinasi `invoice_id + product_id` tidak boleh duplikat
- **Validity**: `unit_price` dan `quantity` harus > 0, `country` harus ada di list valid
- **Consistency**: `total_amount` = `quantity × unit_price`

Integrasikan sebagai task baru di DAG:
```
extract → transform (spark) → data_quality_check (Great Expectations) → load → notify
```
Kalau validation gagal, task load tidak jalan (fail fast).

### Tahap 2: Airflow Lanjutan — Alerting & Robustness
- Email/Slack notification saat task gagal (`on_failure_callback`)
- Sensor: DAG jalan saat file source sudah "landing" di folder tertentu
- Retry logic dengan delay untuk task yang bergantung ke resource eksternal

### Tahap 3: Simulasi Real-Time dengan Kafka
Skenario terpisah dari batch pipeline utama, di folder `streaming-demo/`:
- **Producer**: script Python yang "stream" data transaksi satu per satu ke Kafka topic
- **Consumer**: script Python yang baca dari topic, agregasi ringan (running total revenue), print/log hasil

### Struktur Repo (update dari Minggu 3)
```
ecommerce-etl-pipeline/
├── README.md
├── dags/
│   └── ecommerce_etl_dag.py        # update: tambah task GE check, alerting, sensor
├── spark_jobs/
├── great_expectations/
│   └── expectations/
│       └── ecommerce_suite.json
├── streaming-demo/
│   ├── producer.py
│   └── consumer.py
├── data/
├── docker-compose.yml               # update: tambah service Kafka
└── diagrams/
    ├── star_schema.png
    └── pipeline_architecture_v2.png
```

### README yang Perlu Diupdate
- Section "What's New in v2": data quality checks, alerting, streaming demo
- Screenshot: DAG dengan task baru, output Great Expectations validation report, log consumer Kafka
- Penjelasan trade-off: kenapa fail-fast penting, kenapa streaming demo dipisah dari batch pipeline

### Alokasi Waktu (±8 jam)
- Great Expectations setup + integrasi ke DAG: 3 jam
- Airflow alerting + sensor: 2 jam
- Kafka producer/consumer demo: 2 jam
- Update README + diagram: 1 jam
