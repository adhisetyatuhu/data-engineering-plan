# Roadmap Belajar: Data Engineering Fundamentals (8 Minggu)

## Konteks
Roadmap ini disusun dengan asumsi kapasitas belajar:
- **Weekday**: 2 jam/hari x 5 hari = 10 jam
- **Weekend**: 4 jam/hari x 2 hari = 8 jam
- **Total**: 18 jam/minggu

Estimasi total materi: **100–130 jam**, dibagi menjadi **8 minggu** pembelajaran terstruktur, masing-masing dilengkapi mini project untuk portfolio.

## Cakupan Materi
1. SQL & Python overview
2. Arsitektur data & big data pipeline
3. Data governance
4. Cloud platform fundamentals
5. Containerization
6. NoSQL & storage strategy

## Struktur Mingguan

| Minggu | Topik Utama | File Detail |
|---|---|---|
| 1–2 | SQL & Python Overview | [minggu_1.md](minggu_1.md), [minggu_2.md](minggu_2.md) |
| 3 | Arsitektur Data & Big Data Pipeline (dasar) | [minggu_3.md](minggu_3.md) |
| 4 | Lanjutan Pipeline — Data Quality & Streaming Intro | [minggu_4.md](minggu_4.md) |
| 5 | Data Governance | [minggu_5.md](minggu_5.md) |
| 6 | Cloud Platform Fundamentals | [minggu_6.md](minggu_6.md) |
| 7 | Containerization | [minggu_7.md](minggu_7.md) |
| 8 | NoSQL & Storage Strategy | [minggu_8.md](minggu_8.md) |

## Konsep Portfolio: Continuous Project
Berbeda dari pendekatan "1 topik = 1 project terpisah", roadmap ini menggunakan **satu repository yang terus berkembang** setiap minggu: `ecommerce-etl-pipeline`. Dataset yang dipakai konsisten sepanjang roadmap: **Online Retail II** (e-commerce, Kaggle).

Progres arsitektur repo dari minggu ke minggu:

```
Minggu 1-2: Analisis data (SQL + Python)
     ↓
Minggu 3:   Pipeline otomatis (Spark + Airflow, star schema)
     ↓
Minggu 4:   Pipeline production-grade (Data Quality + Alerting + Streaming demo)
     ↓
Minggu 5:   Governance layer (Data Catalog, Lineage, Policy)
     ↓
Minggu 6:   Cloud migration (Object Storage, IAM, Cloud Data Warehouse)
     ↓
Minggu 7:   Containerization (Custom Docker images + Kubernetes deploy)
     ↓
Minggu 8:   Polyglot storage (MongoDB + Redis) — Capstone
```

Hasil akhir: satu portfolio project yang menunjukkan kemampuan **iterative system design** end-to-end, bukan sekadar kumpulan latihan terpisah.

## Cara Pakai File Ini
- Tiap file `minggu_X.md` berisi: breakdown jadwal harian + materi, sumber belajar, target capaian, dan desain mini project (kalau ada di minggu tersebut).
- Ikuti urutan minggu, tapi boleh disesuaikan kecepatannya sesuai progres masing-masing.
- Setiap mini project dirancang untuk langsung menambah isi repo GitHub yang sama, jadi commit history juga jadi nilai tambah portfolio.
