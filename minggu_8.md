# Minggu 8 — NoSQL & Storage Strategy (Capstone)

## Breakdown Harian (±15-20 jam)

| Hari | Jam | Materi |
|---|---|---|
| Senin | 2 jam | Konsep dasar NoSQL: kenapa muncul, perbedaan dengan RDBMS, CAP theorem |
| Selasa | 2 jam | Tipe-tipe NoSQL: key-value (Redis), document (MongoDB), column-family (Cassandra), graph (Neo4j) |
| Rabu | 2 jam | Document database mendalam: MongoDB — schema design, embedding vs referencing |
| Kamis | 2 jam | Key-value store: Redis — use case (caching, session store, rate limiting), struktur data |
| Jumat | 2 jam | Storage strategy: kapan pakai SQL vs NoSQL vs data lake vs data warehouse — decision framework |
| Sabtu | 4 jam | Hands-on: setup MongoDB lokal, migrasikan `dim_product` ke bentuk document |
| Minggu | 4 jam | Hands-on: setup Redis, implementasikan caching sederhana untuk query yang sering diakses |

## Detail Topik
1. **Kenapa NoSQL Muncul** — masalah scalability RDBMS, CAP theorem (implikasi trade-off)
2. **Tipe-Tipe NoSQL** — key-value (cepat, caching), document (schema fleksibel), column-family (write-heavy), graph (relasi kompleks)
3. **MongoDB Mendalam** — kapan embed vs reference; latihan pakai data e-commerce (`dim_product` dengan atribut bervariasi per kategori)
4. **Redis** — fokus use case caching hasil query yang sering diakses
5. **Decision Framework** — cheat sheet pribadi: kapan pilih SQL vs NoSQL vs data lake berdasarkan struktur data, skala, konsistensi, pola akses

## Sumber Belajar
- "NoSQL Distilled" (Fowler) — ringkasan bab awal, atau Hussein Nasser (YouTube)
- MongoDB University — course "M001: Basics" (gratis)
- Redis official docs "Getting Started", atau Redis University "RU101"

## Target di Akhir Minggu 8
- Menjelaskan kapan menggunakan NoSQL vs RDBMS, dan tipe NoSQL yang cocok untuk use case tertentu
- Merancang schema document di MongoDB (embed vs reference)
- Mengimplementasikan caching sederhana dengan Redis
- Membuat keputusan storage strategy berdasarkan karakteristik data & workload

---

## Mini Project (Capstone): "Polyglot Storage Strategy — Integrasi NoSQL ke E-Commerce Data Platform"

**Kenapa case ini?** Menunjukkan pemahaman *polyglot persistence* — sistem data modern tidak pakai satu jenis database saja, tapi kombinasi tepat sesuai use case. Ini penutup yang menyatukan semua komponen Minggu 1-7.

> Lanjutkan repo `ecommerce-etl-pipeline` — commit terakhir yang melengkapi seluruh arsitektur.

### Tahap 1: MongoDB untuk Data Semi-Structured
- Migrasikan `dim_product` ke MongoDB, manfaatkan fleksibilitas document model: tambahkan atribut bervariasi per kategori (misal elektronik punya `warranty_period`, fashion punya `size_variant`)
- Script Python (`pymongo`) untuk load data dari star schema (warehouse) ke MongoDB
- Buat 2-3 query MongoDB yang menunjukkan keunggulan document model (filter berdasarkan atribut nested)

### Tahap 2: Redis untuk Caching Layer
- Identifikasi query "mahal" & sering diakses dari Minggu 1 (top 10 produk terlaris, RFM summary)
- Implementasikan caching: cek Redis dulu → cache miss → query ke warehouse → simpan ke Redis dengan TTL (misal 1 jam)
- Ukur & bandingkan response time: query langsung ke warehouse vs lewat cache

### Tahap 3: Storage Strategy Decision Document
Buat `STORAGE_STRATEGY.md`:
- Diagram polyglot architecture: data mengalir ke mana untuk kebutuhan apa
  - **BigQuery/Redshift**: analytical queries, historical reporting
  - **MongoDB**: catalog produk dengan atribut fleksibel
  - **Redis**: caching untuk query sering diakses
  - **GCS/S3**: raw data lake, backup Parquet
- Tabel keputusan: "kenapa data X disimpan di Y, bukan di Z" — bagian paling berharga untuk interview

### Struktur Repo (Final, update dari Minggu 7)
```
ecommerce-etl-pipeline/
├── README.md                        # update final: overview keseluruhan project 8 minggu
├── STORAGE_STRATEGY.md               # baru
├── GOVERNANCE.md
├── dags/
├── spark_jobs/
├── great_expectations/
├── streaming-demo/
├── governance/
├── cloud/
├── k8s/
├── nosql/
│   ├── mongo_migration.py            # baru
│   ├── redis_cache_layer.py          # baru
│   └── benchmark_results.md          # baru
├── docker-compose.yml                # update: tambah service MongoDB & Redis
├── data/
└── diagrams/
    └── polyglot_architecture.png     # baru
```

### README Final yang Perlu Ditulis
- Section "Project Journey": timeline singkat 8 minggu (analisis → pipeline → governance → cloud → containerization → polyglot storage)
- Diagram arsitektur final yang menunjukkan SEMUA komponen terhubung
- Link ke tiap sub-project/dokumen relevan

### Alokasi Waktu (±8 jam)
- Setup MongoDB + migrasi & query produk: 2.5 jam
- Setup Redis + implementasi caching + benchmark: 2.5 jam
- Tulis `STORAGE_STRATEGY.md`: 1.5 jam
- Update README final + diagram arsitektur keseluruhan: 1.5 jam

---

## 🎉 Hasil Akhir Portfolio
Setelah 8 minggu, repo `ecommerce-etl-pipeline` mencakup:

- ✅ Data analysis (SQL + Python)
- ✅ Data pipeline & orchestration (Spark + Airflow)
- ✅ Data quality & streaming (Great Expectations + Kafka)
- ✅ Data governance (catalog, lineage, policy)
- ✅ Cloud migration (GCP/AWS, BigQuery/Redshift)
- ✅ Containerization & orchestration (Docker + Kubernetes)
- ✅ Polyglot storage strategy (MongoDB + Redis)

Satu project yang berkembang koheren, menunjukkan kemampuan **iterative system design** end-to-end — jauh lebih kuat dibanding kumpulan project terpisah.
