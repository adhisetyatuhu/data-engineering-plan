---
title: Minggu 2 - Python Fundamentals
nav_order: 3
has_children: true
---

# Minggu 2 — Python Fundamentals + Mini Project SQL & Python

## Breakdown Harian (±18 jam)

| Hari | Jam | Materi |
|---|---|---|
| Senin | 2 jam | Python dasar: variabel, tipe data, control flow, function |
| Selasa | 2 jam | Struktur data: list, dict, set, tuple + comprehension |
| Rabu | 2 jam | Pandas dasar: DataFrame, read_csv, filtering, selection |
| Kamis | 2 jam | Pandas lanjutan: groupby, merge/join, pivot table |
| Jumat | 2 jam | Data cleaning: handle missing values, duplicate, tipe data, string manipulation |
| Sabtu | 4 jam | Latihan: exploratory data analysis (EDA) end-to-end pada 1 dataset publik (Kaggle) |
| Minggu | 4 jam | Mini project: gabungkan SQL + Python |

## Sumber Belajar
- Python.org tutorial atau Corey Schafer (YouTube)
- Pandas official docs / "10 Minutes to Pandas"
- Dataset latihan: Kaggle

## Target di Akhir Minggu 2
- Bisa melakukan data cleaning & EDA dasar dengan Pandas
- Bisa menghubungkan alur kerja SQL → Python dalam satu project kecil

---

## Mini Project: "Analisis Performa Penjualan & Segmentasi Pelanggan E-Commerce"

**Kenapa case ini?** Real-world, mudah dipahami recruiter/interviewer, dan memaksa penggunaan semua skill: query kompleks, data cleaning, agregasi, sampai insight bisnis.

### Dataset
- **"Online Retail II"** (UK e-commerce, ada invoice, customer, produk, tanggal) — rekomendasi utama
- Alternatif: "Superstore Sales" atau "Brazilian E-Commerce (Olist)"
- Import ke SQLite/PostgreSQL supaya benar-benar kerja dengan SQL, bukan cuma baca CSV di Pandas

### Tahap 1: Setup & Data Cleaning (SQL + Python)
- Load raw data ke database (Python script, `sqlite3`/`sqlalchemy`)
- Cek & handle: missing values, duplicate invoice, quantity negatif (return), harga 0/negatif
- Dokumentasikan proses cleaning (before/after row count, alasan keputusan)

### Tahap 2: Analisis dengan SQL
1. Top 10 produk terlaris (by revenue & by quantity) — JOIN + GROUP BY
2. Revenue per bulan/kuartal — date functions
3. Customer dengan total belanja tertinggi (CTE)
4. **RFM Analysis** (Recency, Frequency, Monetary) — window functions
5. Growth rate bulanan — LAG/LEAD

### Tahap 3: Analisis & Visualisasi dengan Python
- Tarik hasil query SQL ke Pandas
- Customer segmentation berdasarkan RFM score (Champions, Loyal, At Risk, Lost)
- Visualisasi (matplotlib/seaborn): trend revenue bulanan, distribusi customer segment, top produk & kategori

### Tahap 4: Business Insight & Recommendation
- Tulis 3–5 insight konkret dan actionable (contoh: segmen "At Risk" menyumbang X% revenue → perlu campaign retensi)

### Struktur Repo
```
ecommerce-sales-analysis/
├── README.md              # ringkasan project, insight utama, screenshot chart
├── data/                  # raw & cleaned data (atau link download)
├── sql/
│   ├── 01_data_cleaning.sql
│   ├── 02_sales_analysis.sql
│   └── 03_rfm_analysis.sql
├── notebooks/
│   └── analysis.ipynb     # python analysis + visualisasi
└── outputs/
    └── charts/
```

### Alokasi Waktu (±10 jam)
- Data cleaning: 2 jam
- SQL analysis queries: 3 jam
- Python analysis + visualisasi: 3 jam
- Insight writing + README: 2 jam
