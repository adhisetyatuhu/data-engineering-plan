---
title: Overview
parent: Minggu 2 - Python Fundamentals
nav_order: 1
---

# Modul Minggu 2 — Python Fundamentals + Mini Project SQL & Python (untuk Software Developer)

> Pendamping `minggu_2.md` (jadwal & outline). File ini konten pengajaran lengkap: penjelasan konsep, analogi kode, contoh, latihan, kunci jawaban. Lihat juga `materi/minggu_1/` — minggu ini menyambung langsung dari SQL yang sudah dipelajari di sana.

## Untuk Siapa & Kenapa Bobot Materinya Timpang

Peserta sudah bisa programming, jadi **Python dasar (Hari 1–2) akan terasa cepat** — bukan konsep baru, cuma sintaks baru. Jangan habiskan waktu banyak di sana. **Bobot terberat ada di Pandas (Hari 3–4) dan data cleaning (Hari 5)** — ini yang benar-benar skill baru untuk data engineer, dan tidak otomatis dikuasai walau sudah jago programming umum. Instruktur: alokasikan energi mengajar sesuai ini, jangan merata.

## Filosofi Mengajar Minggu Ini: Sambungkan ke Minggu 1, Bukan Mulai dari Nol

Trik utama minggu ini: **Pandas paling cepat dipahami developer lewat pemetaan langsung ke SQL yang sudah mereka kuasai minggu lalu**, bukan diajarkan sebagai hal baru dari nol.

| SQL (Minggu 1) | Pandas (Minggu 2) |
|---|---|
| `SELECT col1, col2 FROM table` | `df[['col1', 'col2']]` |
| `WHERE condition` | `df[boolean_condition]` |
| `ORDER BY col` | `df.sort_values('col')` |
| `LIMIT n` | `df.head(n)` |
| `GROUP BY col` + agregasi | `df.groupby('col').agg(...)` |
| `JOIN ... ON` | `df.merge(...)` |
| `HAVING` | `.groupby(...).filter(...)` atau filter setelah agregasi |

Tabel ini akan diulang & diperluas di `hari_3_pandas_dasar.md` dan `hari_4_pandas_lanjutan.md`. Tekankan ke peserta di awal minggu: **kamu sudah tahu logikanya dari SQL, minggu ini cuma belajar notasi baru untuk logika yang sama** — dan kadang malah bisa cross-check jawaban Pandas dengan query SQL minggu lalu untuk validasi (dan memang beberapa contoh di modul ini sengaja meniru ulang query dari `materi/minggu_1/` dengan Pandas, hasilnya harus identik).

## Setup Environment

```bash
python3 -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate
pip install pandas matplotlib seaborn jupyter sqlalchemy psycopg2-binary
```

- **Jupyter Notebook/Lab** direkomendasikan untuk eksplorasi data (`jupyter lab`) — beda dari workflow developer biasa yang lebih sering nulis script/file `.py` linear, notebook mendukung eksplorasi iteratif per-cell yang cocok untuk data analysis.
- `sqlalchemy` + `psycopg2-binary` dipakai untuk konek ke PostgreSQL yang sudah di-setup Minggu 1 (`pg-belajar`) langsung dari Python — dipakai mulai `hari_4` dan terutama di mini project akhir minggu.

## Dataset yang Dipakai Minggu Ini

### Hari 1–4: Dataset Toko Elektronik & Furnitur Mini (lanjutan Minggu 1)

File CSV di folder `data/` — hasil export dari dataset SQL Minggu 1 (skema & isi identik dengan `materi/minggu_1/00_overview.md`):

```
data/
├── customers.csv      (7 baris)
├── products.csv       (7 baris)
├── orders.csv         (12 baris)
├── order_items.csv    (19 baris)
└── sales_raw_dirty.csv (20 baris, dipakai khusus Hari 5)
```

Dipakai terus sampai Hari 4 supaya peserta bisa membandingkan langsung hasil Pandas dengan hasil query SQL yang sudah mereka verifikasi minggu lalu.

### Hari 5 & Akhir Pekan: Data yang Lebih "Kotor" dan Data Publik Sungguhan

- **Hari 5** pakai `data/sales_raw_dirty.csv` — versi sengaja dikotori dari dataset yang sama (missing value, duplikat baris, harga dalam format campuran `$8.00`/`3,50`, negara dengan whitespace/casing tidak konsisten, format tanggal campuran). Setiap masalah di file ini **sengaja dan bisa diverifikasi** — instruktur tahu nilai "bersih" yang seharusnya karena berasal dari dataset yang sama di `data/order_items.csv`.
- **Sabtu** pakai dataset publik pilihan bebas dari Kaggle (lihat `latihan_eda_dan_mini_project.md`).
- **Minggu** mulai pakai **Online Retail II** — dataset yang akan dipakai terus sampai Minggu 8 sebagai portfolio project (`ecommerce-etl-pipeline` / `ecommerce-sales-analysis`).

## Struktur Modul

| File | Sesuai Jadwal `minggu_2.md` | Topik |
|---|---|---|
| [`hari_1_python_dasar.md`](hari_1_python_dasar.md) | Senin, 2 jam | Variabel, tipe data, control flow, function |
| [`hari_2_struktur_data.md`](hari_2_struktur_data.md) | Selasa, 2 jam | list, dict, set, tuple + comprehension |
| [`hari_3_pandas_dasar.md`](hari_3_pandas_dasar.md) | Rabu, 2 jam | DataFrame, read_csv, filtering, selection |
| [`hari_4_pandas_lanjutan.md`](hari_4_pandas_lanjutan.md) | Kamis, 2 jam | groupby, merge/join, pivot table |
| [`hari_5_data_cleaning.md`](hari_5_data_cleaning.md) | Jumat, 2 jam | Missing values, duplicate, dtype, string manipulation |
| [`latihan_eda_dan_mini_project.md`](latihan_eda_dan_mini_project.md) | Sabtu (4 jam) + Minggu (4 jam) | EDA end-to-end + Mini Project SQL+Python (Online Retail II) |

Struktur tiap file `hari_X` sama dengan Minggu 1: Tujuan Belajar → Untuk Instruktur → Konsep & Sintaks → Contoh Kode → Kesalahan Umum → Latihan → Kunci Jawaban.

## Catatan Cara Mengajar

- **Live coding di notebook**, bukan slide — sama seperti Minggu 1, tapi sekarang di Jupyter, bukan SQL client.
- **Jangan re-teach konsep pemrograman dasar** (apa itu variabel, apa itu function) — peserta sudah tahu ini dari bahasa lain. Fokus ke: apa yang **beda** di Python (indentation, dynamic typing, truthiness, list/dict comprehension) dan apa yang **baru sama sekali** (Pandas).
- **Setiap kali ngajarin method Pandas baru, tanya balik**: "ini setara SQL apa yang kita pelajari minggu lalu?" — kalau peserta bisa jawab sendiri, itu tanda tersambung baik.
- Total waktu: 5 hari × 2 jam + Sabtu 4 jam + Minggu 4 jam = 18 jam, sesuai `minggu_2.md`.
