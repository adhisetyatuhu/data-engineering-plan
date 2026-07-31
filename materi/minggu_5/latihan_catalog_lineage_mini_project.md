---
title: Hands-on Data Catalog & Lineage + Mini Project v3
parent: Minggu 5 - Data Governance
nav_order: 7
---

# Sabtu–Minggu — Hands-on Data Catalog, Lineage & Governance Policy

*Sabtu 4 jam + Minggu 4 jam. Kelanjutan langsung `minggu_5.md` bagian "Mini Project: Data Catalog, Lineage & Governance Layer untuk E-Commerce Pipeline" — upgrade dari `materi/minggu_4/latihan_dq_streaming_mini_project.md`.*

> **Repo yang sama**: tetap `ecommerce-etl-pipeline`. Tidak ada rename atau folder pipeline baru — minggu ini murni menambah **lapisan dokumentasi & tooling governance** di atas apa yang sudah dibangun.

## Tujuan Belajar

- [ ] Setup OpenMetadata via Docker dan meng-ingest metadata dari warehouse Postgres yang sudah ada
- [ ] Melengkapi metadata manual: deskripsi bisnis, tag PII, ownership di UI catalog
- [ ] Mendokumentasikan lineage end-to-end (raw CSV → Spark → star schema → warehouse)
- [ ] Menulis `GOVERNANCE.md` yang mencakup ownership, quality policy, PII handling, dan access policy

## Untuk Instruktur

Beda dari sesi hands-on minggu-minggu sebelumnya, sesi ini lebih berat ke **dokumentasi & konfigurasi UI** dibanding menulis kode — wajar dan sesuai sifat governance (`00_overview.md`). Kalau resource laptop peserta terbatas untuk menjalankan OpenMetadata (butuh ~6-8GB RAM untuk Docker), opsi fallback: fokuskan waktu ke Bagian 2 (lineage diagram manual + `GOVERNANCE.md`) yang tidak bergantung tool apapun, dan Bagian 1 cukup dipahami konsepnya + dicoba di environment dengan resource lebih besar kalau ada.

---

## Bagian 1 (Sabtu, 4 jam): Setup OpenMetadata & Ingest Metadata

### Setup (±1 jam)

```bash
mkdir -p governance && cd governance
curl -sL -o docker-compose-openmetadata.yml \
  https://github.com/open-metadata/OpenMetadata/releases/download/1.5.0-release/docker-compose.yml
docker compose -f docker-compose-openmetadata.yml up -d
```

Tunggu beberapa menit (banyak service internal yang harus siap), lalu buka `http://localhost:8585` — login default `admin@open-metadata.org` / `admin` (**ganti password ini** kalau dipakai di luar konteks belajar lokal).

### Ingest Metadata dari Postgres (±1.5 jam)

**Opsi A — lewat UI** (lebih visual, cocok untuk eksplorasi pertama kali):
1. **Settings → Services → Databases → Add New Service** → pilih `Postgres`
2. Isi koneksi: host `host.docker.internal`, port `5432`, database `postgres`, username/password sesuai `pg-belajar` (Minggu 1)
3. **Add Ingestion** → pilih `Metadata Ingestion` → jalankan sekali (manual trigger)
4. Setelah selesai, buka **Explore** → cari `fact_sales`/`dim_customer`/dst. — skema (technical metadata, `hari_2_data_catalog_metadata.md`) sudah otomatis terisi dari introspeksi database

**Opsi B — lewat CLI/YAML** (lebih reproducible, bisa disimpan sebagai kode di repo):

```yaml
# governance/postgres_ingestion.yaml
source:
  type: postgres
  serviceName: ecommerce-warehouse
  serviceConnection:
    config:
      type: Postgres
      username: postgres
      password: belajar
      hostPort: host.docker.internal:5432
      database: postgres
  sourceConfig:
    config:
      type: DatabaseMetadata
sink:
  type: metadata-rest
  config: {}
workflowConfig:
  openMetadataServerConfig:
    hostPort: http://localhost:8585/api
    authProvider: openmetadata
    securityConfig:
      jwtToken: "<ambil dari Settings > Bots > ingestion-bot di UI OpenMetadata>"
```

```bash
pip install "openmetadata-ingestion[postgres]"
metadata ingest -c governance/postgres_ingestion.yaml
```

Opsi B lebih disarankan untuk repo portfolio — file `postgres_ingestion.yaml` yang tersimpan di git **sendiri adalah bentuk dokumentasi** ("begini cara metadata di-refresh"), sesuatu yang tidak didapat kalau ingestion cuma dilakukan sekali lewat klik UI.

### Lengkapi Metadata Manual (±1.5 jam)

Untuk **tiap** tabel (`fact_sales`, `dim_customer`, `dim_product`, `dim_date`) di UI OpenMetadata, isi (business metadata, `hari_2_data_catalog_metadata.md`):

| Field | Isi untuk `fact_sales` (contoh) |
|---|---|
| **Description** | "1 baris = 1 item produk dalam 1 invoice (grain: order item level). `is_return=True` menandakan pembatalan/retur (invoice diawali 'C')." |
| **Owner** | Nama kamu (atau tim hipotetis "Data Engineering Team", sesuai `hari_4_stewardship_quality_governance.md`) |
| **Tags** | `PII` (karena mengandung `customer_id`) — lihat cara tag kolom spesifik di bawah |
| **Glossary Term** | Kalau OpenMetadata glossary sudah di-setup: "Net Revenue = revenue dari baris `is_return=False` saja" |

**Tag PII di level kolom** (bukan cuma level tabel) — buka kolom `customer_id` di `fact_sales`/`dim_customer`, tambahkan tag `PII.Sensitive` (tag bawaan OpenMetadata) — ini yang nanti dirujuk di `GOVERNANCE.md` Bagian 2 sebagai bukti kolom PII sudah teridentifikasi & terdokumentasi secara sistematis, bukan cuma disebutkan di teks bebas.

### Deliverable Sabtu

- [ ] OpenMetadata jalan, `fact_sales`/`dim_customer`/`dim_product`/`dim_date` sudah ter-ingest (technical metadata otomatis terisi)
- [ ] Business description terisi untuk keempat tabel
- [ ] `customer_id` (di `fact_sales` dan `dim_customer`) sudah ditag PII
- [ ] Owner terisi untuk keempat tabel
- [ ] Screenshot halaman salah satu tabel (untuk dipakai di README, sesuai `minggu_5.md`)

---

## Bagian 2 (Minggu, 4 jam): Dokumentasi Lineage & `GOVERNANCE.md`

### Tahap 2: Lineage Diagram (±1.5 jam)

Sesuai `hari_3_data_lineage.md`, buat **table-level lineage diagram** manual (draw.io/dbdiagram.io), simpan sebagai `governance/lineage_diagram.png` dan `diagrams/data_lineage.png` (boleh file yang sama, direferensikan dari 2 tempat):

```
online_retail_II.csv (raw)
        |
        v
clean_transform.py  --[dropna Customer ID, dedup, filter Price>0, hitung revenue & is_return]
        |
        v
retail_clean (staging parquet)
        |
        v
build_star_schema.py  --[split jadi fact + dimension]
        |
        +--> dim_customer (parquet) --+
        +--> dim_product (parquet)  --+
        +--> dim_date (parquet)     --+--> load_to_warehouse (Airflow) --> Postgres (dim_*, fact_sales)
        +--> fact_sales (parquet)   --+                                          |
                                                                                   v
                                                                    data_quality_check (Great Expectations)
                                                                                   |
                                                                                   v
                                                                    streaming-demo/producer.py (opsional,
                                                                    dari fact_sales -> Kafka -> consumer.py)
```

**Opsional (bonus, tidak wajib)**: kalau ingin coba lineage **otomatis**, integrasikan [OpenLineage](https://openlineage.io/) dengan Airflow (`apache-airflow-providers-openlineage`) — begitu terpasang, tiap run DAG otomatis memancarkan event lineage ke OpenMetadata/Marquez tanpa perlu digambar manual. Ini di luar cakupan wajib mini project (setup tambahan cukup signifikan), tapi baik diketahui ada sebagai opsi produksi yang lebih robust dibanding diagram statis — dibahas konsepnya di `hari_3_data_lineage.md`.

### Tahap 3: Tulis `GOVERNANCE.md` (±2 jam)

```markdown
# Data Governance — ecommerce-etl-pipeline

## Data Ownership

| Dataset | Owner | Steward | Custodian |
|---|---|---|---|
| fact_sales | Data Engineering Team | (kamu) | Platform Team (hipotetis) |
| dim_customer | Data Engineering Team | (kamu) | Platform Team (hipotetis) |
| dim_product | Data Engineering Team | (kamu) | Platform Team (hipotetis) |
| dim_date | Data Engineering Team | (kamu) | Platform Team (hipotetis) |

Lihat `materi/minggu_5/hari_4_stewardship_quality_governance.md` untuk definisi tiap peran.

## Data Quality Policy

Semua data yang masuk `fact_sales` harus lolos `great_expectations/expectations/ecommerce_suite.json`
sebelum di-load ke warehouse (fail-fast, lihat `dags/ecommerce_etl_dag.py`). Ringkasan aturan:

- **Completeness**: `customer_id`, `invoice` tidak boleh null
- **Uniqueness**: kombinasi `invoice` + `stock_code` tidak boleh duplikat
- **Validity**: `unit_price > 0`, `quantity != 0`, `country` sesuai allowlist
- **Consistency**: `revenue == quantity * unit_price`

Proses eskalasi kalau check gagal: lihat `materi/minggu_5/hari_4_stewardship_quality_governance.md`
bagian "Proses Eskalasi".

## PII Handling Policy

Kolom PII yang teridentifikasi (tag `PII` di OpenMetadata, lihat `governance/`):
- `customer_id` (direct identifier)
- `country` dikombinasikan dengan pola transaksi (quasi-identifier) -- lihat
  `materi/minggu_5/hari_5_compliance_privacy.md`

Aturan:
- `customer_id` di-**pseudonymize** (hash + salt tersimpan terpisah) untuk data yang diekspor
  ke tim di luar Data Engineering (mis. tim data science eksternal)
- Tidak ada ekspor data mentah (raw) ke pihak eksternal tanpa proses masking/anonymization
- Retensi: data mentah (`data/raw/`) disimpan maksimal 2 tahun (kebijakan hipotetis, sesuaikan
  dengan kebutuhan bisnis/regulasi sungguhan)

## Access Policy (Hipotetis)

| Role | fact_sales/dim_* (agregat) | customer_id asli | Raw data |
|---|---|---|---|
| Data Engineer | Full access | Full access | Full access |
| Analyst | SELECT only | Pseudonymized only | Tidak ada akses |
| Eksternal/Vendor | Agregat saja (via laporan) | Tidak ada akses | Tidak ada akses |
```

Sesuaikan isi tabel/kebijakan di atas dengan keputusan kamu sendiri — yang penting **format dan kelengkapan** (4 bagian: ownership, quality policy, PII handling, access policy) sesuai spesifikasi `minggu_5.md`, bukan menyalin persis contoh ini.

### Update README.md (±30 menit)

Tambahkan section baru **"Data Governance"**:
- Screenshot halaman tabel di OpenMetadata (metadata & tag PII terisi)
- Link ke `GOVERNANCE.md`
- Diagram lineage end-to-end (`diagrams/data_lineage.png`)

### Struktur Repo Final (Update dari Minggu 4)

```
ecommerce-etl-pipeline/
├── README.md
├── GOVERNANCE.md                       # baru
├── dags/
│   └── ecommerce_etl_dag.py
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
├── governance/                          # baru
│   ├── docker-compose-openmetadata.yml
│   ├── postgres_ingestion.yaml
│   └── lineage_diagram.png
├── data/
│   ├── raw/
│   └── warehouse/
├── Dockerfile
├── docker-compose.yml
└── diagrams/
    ├── star_schema.png
    ├── pipeline_architecture_v2.png
    └── data_lineage.png                # baru
```

### Kriteria "Selesai" untuk Minggu 5

- [ ] OpenMetadata jalan dan berhasil meng-ingest metadata dari warehouse Postgres
- [ ] Keempat tabel (`fact_sales`, `dim_customer`, `dim_product`, `dim_date`) punya business description, owner, dan tag PII di kolom yang relevan
- [ ] `diagrams/data_lineage.png` menunjukkan alur lengkap dari raw CSV sampai warehouse (dan opsional sampai streaming demo)
- [ ] `GOVERNANCE.md` berisi 4 bagian lengkap: ownership, data quality policy, PII handling policy, access policy
- [ ] Bisa menjelaskan ke orang lain: kenapa `customer_id` butuh pseudonymization (bukan anonymization) untuk kasus tim data science yang butuh grouping per customer (`hari_5_compliance_privacy.md`), dan siapa yang bertanggung jawab kalau `data_quality_check` gagal (`hari_4_stewardship_quality_governance.md`)

Kalau semua tercentang, lanjut ke `minggu_6.md` — repo yang sama akan dimigrasikan ke cloud platform (object storage, IAM, cloud data warehouse), di mana kebijakan akses & PII handling yang sudah didokumentasikan minggu ini akan diterapkan secara **teknis sungguhan** lewat IAM policy cloud provider.
