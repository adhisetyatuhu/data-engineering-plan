---
title: EDA & Mini Project
parent: Minggu 2 - Python Fundamentals
nav_order: 7
---

# Sabtu–Minggu — EDA End-to-End & Mini Project: Analisis Penjualan + Segmentasi Pelanggan

*Sabtu 4 jam + Minggu 4 jam (+lanjutan). Ini kickoff resmi portfolio project `ecommerce-sales-analysis` yang disebut di `minggu_2.md` — akan terus dikembangkan sampai Minggu 8.*

> **Perbedaan penting dengan file `hari_1`–`hari_5`**: mulai file ini, dataset yang dipakai adalah data publik sungguhan (Sabtu) dan **Online Retail II** (Minggu — dataset asli, bukan versi mini yang dipakai Hari 1–5). Karena itu, kode di file ini adalah **template/panduan**, bukan hasil yang sudah diverifikasi angka pastinya seperti file-file sebelumnya — angka hasil analisis akan beda tergantung data yang benar-benar di-download peserta. Instruktur: jalankan sendiri dulu sebelum sesi untuk tahu angka aktualnya.

## Tujuan Belajar

- [ ] Menjalankan alur EDA end-to-end secara mandiri pada dataset publik yang belum pernah dilihat sebelumnya
- [ ] Membangun ulang alur SQL + Python yang sudah dipelajari 2 minggu ini pada dataset dunia nyata yang jauh lebih besar & lebih kotor
- [ ] Menghasilkan repo GitHub pertama untuk portfolio (`ecommerce-sales-analysis`) dengan struktur yang akan terus dipakai sampai Minggu 8

## Untuk Instruktur

Ini sesi paling "lepas tangan" — tujuannya peserta latihan bekerja **tanpa dituntun step-by-step** seperti Hari 1–5. Peran instruktur di sini lebih ke **fasilitator & code reviewer**, bukan pengajar materi baru. Kalau peserta stuck di step teknis (bukan step konseptual), itu sinyal balik ke modul hari terkait di Minggu 1/2 — jangan langsung kasih jawaban.

---

## Bagian 1 (Sabtu, 4 jam): EDA End-to-End pada Dataset Publik

### Kenapa Latihan Ini Dulu, Bukan Langsung Online Retail II

Online Retail II (dipakai mulai besok) itu dataset besar (500rb+ baris) dengan masalah data quality yang nyata dan kadang butuh riset tambahan untuk dipahami. Sabtu ini pemanasan dengan dataset yang **lebih kecil dan lebih rapi**, supaya peserta latihan alur EDA lengkap tanpa terbebani kompleksitas data quality dulu.

### Dataset Rekomendasi

**Sample Superstore** (Kaggle, sangat umum dipakai untuk latihan EDA) — kolom utamanya: `Order ID`, `Order Date`, `Ship Date`, `Customer Name`, `Segment`, `Country`, `City`, `State`, `Category`, `Sub-Category`, `Sales`, `Quantity`, `Discount`, `Profit`.

Alternatif: dataset publik lain dari Kaggle yang sudah disebut di `minggu_1.md`/`minggu_2.md` (Titanic, Northwind). Yang penting: **data tabular, minimal 1 kolom tanggal, minimal 1 kolom kategori, minimal 1 kolom numerik untuk dianalisis.**

### Checklist Alur EDA (Framework Umum — Bisa Dipakai untuk Dataset Apapun)

```
1. Load & Inspect     -> shape, dtypes, head(), info(), describe()
2. Data Quality Check  -> missing values, duplicate, outlier kasar (describe() + boxplot)
3. Univariate Analysis -> distribusi tiap kolom penting (histogram numerik, value_counts kategorikal)
4. Bivariate Analysis  -> hubungan 2 kolom (mis. Sales vs Discount, Sales per Category)
5. Time-based Analysis -> tren dari waktu ke waktu, kalau ada kolom tanggal
6. Ringkasan Temuan    -> 3-5 insight dalam bahasa manusia, bukan cuma angka
```

### Template Kode

```python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

df = pd.read_csv("superstore.csv", encoding="latin-1")   # dataset Kaggle sering perlu encoding non-default

# 1. Load & Inspect
print(df.shape)
df.info()
df.describe(include="all")

# 2. Data Quality Check
print(df.isna().sum())
print(df.duplicated().sum())

# 3. Univariate
df["Sales"].hist(bins=30)
df["Category"].value_counts().plot(kind="bar")

# 4. Bivariate
sns.boxplot(data=df, x="Category", y="Sales")
df.groupby("Category")["Sales"].sum().sort_values(ascending=False)

# 5. Time-based
df["Order Date"] = pd.to_datetime(df["Order Date"])
df.set_index("Order Date")["Sales"].resample("M").sum().plot()
```

### Deliverable Sabtu

Notebook (`eda_superstore.ipynb` atau nama dataset yang dipakai) berisi ke-6 langkah checklist di atas, dengan **minimal 3 insight tertulis** dalam bentuk teks (bukan cuma grafik tanpa penjelasan) di akhir notebook.

---

## Bagian 2 (Minggu + lanjutan): Mini Project — Online Retail II

> Mengikuti desain di `minggu_2.md` bagian "Mini Project: Analisis Performa Penjualan & Segmentasi Pelanggan E-Commerce". File ini menjabarkan tiap tahap dengan kode konkret.

### Tentang Dataset

**Online Retail II** — transaksi e-commerce UK, periode 2009–2011. Download dari Kaggle ("Online Retail II UCI") atau UCI ML Repository. Kolom asli (perhatikan — beda dari dataset "Online Retail" versi lama yang sering dipakai di tutorial lain):

| Kolom | Isi |
|---|---|
| `Invoice` | Nomor invoice. **Kalau diawali `'C'`, itu transaksi pembatalan/retur.** |
| `StockCode` | Kode produk |
| `Description` | Nama produk (ada baris dengan deskripsi aneh seperti `"POSTAGE"`, `"Manual"`, `"DOTCOM POSTAGE"` — itu bukan produk asli, tapi entri biaya/adjustment) |
| `Quantity` | Jumlah unit. **Bisa negatif** — artinya retur/pembatalan, sama seperti pola yang sudah dibahas di `hari_5_data_cleaning.md` |
| `InvoiceDate` | Tanggal & waktu transaksi |
| `Price` | Harga per unit (bukan `UnitPrice` — nama kolom di versi "II" ini beda dari dataset "Online Retail" versi lama) |
| `Customer ID` | ID pelanggan. **Banyak baris `NaN`** — transaksi tanpa akun/guest |
| `Country` | Negara pelanggan |

Dataset ini **datang sebagai 1 file flat** (1 baris = 1 baris item dalam 1 invoice) — bukan sudah dinormalisasi seperti dataset mini Minggu 1–2. Normalisasi penuh jadi star schema baru jadi fokus eksplisit Minggu 3 (`minggu_3.md`) — minggu ini cukup 1 tabel bersih hasil cleaning.

### Struktur Repo (sesuai `minggu_2.md`)

```
ecommerce-sales-analysis/
├── README.md
├── data/
├── sql/
│   ├── 01_data_cleaning.sql
│   ├── 02_sales_analysis.sql
│   └── 03_rfm_analysis.sql
├── notebooks/
│   └── analysis.ipynb
└── outputs/
    └── charts/
```

```bash
mkdir -p ecommerce-sales-analysis/{data,sql,notebooks,outputs/charts}
cd ecommerce-sales-analysis && git init
```

### Tahap 1: Setup & Data Cleaning (±2 jam)

**Load & tulis ke database** (pakai PostgreSQL yang sudah di-setup Minggu 1, `pg-belajar`):
```python
import pandas as pd
from sqlalchemy import create_engine

raw = pd.read_excel("data/online_retail_II.xlsx", sheet_name="Year 2010-2011")
# atau pd.read_csv(...) kalau versi Kaggle yang dipakai formatnya CSV

engine = create_engine("postgresql://postgres:belajar@localhost:5432/postgres")
raw.to_sql("retail_raw", engine, if_exists="replace", index=False)
```

**Cek & tangani data quality** (checklist sesuai `minggu_2.md`: missing values, duplicate invoice, quantity negatif, harga 0/negatif — pola yang sudah dilatih persis di `hari_5_data_cleaning.md`):
```python
print(raw.isna().sum())               # Customer ID biasanya paling banyak NaN
print(raw.duplicated().sum())
print((raw["Quantity"] < 0).sum())    # retur/pembatalan
print((raw["Price"] <= 0).sum())      # harga janggal — cek manual dulu, jangan langsung dibuang

# Dokumentasikan before/after row count (wajib, sesuai instruksi minggu_2.md)
n_before = len(raw)

clean = raw.dropna(subset=["Customer ID"]).copy()              # buang transaksi tanpa customer ID
clean = clean.drop_duplicates()
clean = clean[clean["Price"] > 0]                                # buang baris harga 0/negatif (adjustment/error entri)
clean["is_return"] = clean["Invoice"].astype(str).str.startswith("C")   # tandai retur, JANGAN dibuang — retur itu data valid
clean["revenue"] = clean["Quantity"] * clean["Price"]

n_after = len(clean)
print(f"Rows: {n_before} -> {n_after} ({n_before - n_after} dibuang, {100*(n_before-n_after)/n_before:.1f}%)")

clean.to_sql("retail_clean", engine, if_exists="replace", index=False)
```

Simpan query cleaning setara di `sql/01_data_cleaning.sql` (versi SQL dari langkah di atas, dijalankan di atas `retail_raw` untuk menghasilkan `retail_clean` — latihan bagus menerjemahkan balik logic Pandas ke SQL, kebalikan dari yang biasa dilakukan sepanjang Minggu 2 ini).

**Catatan desain**: dataset dimuat sebagai **1 tabel flat** (`retail_clean`), bukan dipecah jadi `customers`/`orders`/`order_items` seperti dataset mini Minggu 1–2 — itu keputusan sadar (lihat catatan skema di atas), bukan langkah yang terlewat.

### Tahap 2: Analisis dengan SQL (±3 jam)

Simpan di `sql/02_sales_analysis.sql` dan `sql/03_rfm_analysis.sql`. Semua query ini kelanjutan langsung teknik yang sudah dilatih di `materi/minggu_1/` — kalau ada yang lupa sintaksnya, itu sinyal balik ke file `hari_X` terkait.

**1. Top 10 produk terlaris (by revenue & by quantity)**
```sql
SELECT "StockCode", "Description",
       SUM(quantity) AS total_qty,
       SUM(revenue) AS total_revenue
FROM retail_clean
WHERE is_return = FALSE
GROUP BY "StockCode", "Description"
ORDER BY total_revenue DESC
LIMIT 10;
```

**2. Revenue per bulan/kuartal**
```sql
SELECT DATE_TRUNC('month', "InvoiceDate") AS month, SUM(revenue) AS revenue
FROM retail_clean
WHERE is_return = FALSE
GROUP BY 1
ORDER BY 1;
```

**3. Customer dengan total belanja tertinggi (CTE)** — pola identik `materi/minggu_1/hari_4_subquery_cte.md`
```sql
WITH customer_revenue AS (
    SELECT "Customer ID", SUM(revenue) AS total_revenue
    FROM retail_clean
    WHERE is_return = FALSE
    GROUP BY "Customer ID"
)
SELECT * FROM customer_revenue ORDER BY total_revenue DESC LIMIT 10;
```

**4. RFM Analysis (Recency, Frequency, Monetary) — window function**

Ini pemakaian window function paling langsung berguna secara bisnis dari semua yang dipelajari sejauh ini. `NTILE(n)` — fungsi window baru, satu keluarga dengan `RANK`/`ROW_NUMBER` yang sudah dipelajari di `materi/minggu_1/hari_5_window_function.md` — membagi baris jadi `n` kelompok berukuran setara berdasarkan urutan, dipakai di sini untuk bikin skor 1–5 per dimensi RFM:
```sql
WITH rfm_raw AS (
    SELECT "Customer ID",
           MAX("InvoiceDate") AS last_purchase,
           CURRENT_DATE - MAX("InvoiceDate")::date AS recency_days,
           COUNT(DISTINCT "Invoice") AS frequency,
           SUM(revenue) AS monetary
    FROM retail_clean
    WHERE is_return = FALSE
    GROUP BY "Customer ID"
),
rfm_scored AS (
    SELECT *,
           NTILE(5) OVER (ORDER BY recency_days DESC) AS r_score,  -- recency kecil = baru beli = skor tinggi
           NTILE(5) OVER (ORDER BY frequency ASC) AS f_score,
           NTILE(5) OVER (ORDER BY monetary ASC) AS m_score
    FROM rfm_raw
)
SELECT * FROM rfm_scored;
```

**5. Growth rate bulanan — `LAG`/`LEAD`**, pola identik contoh #3 `hari_5_window_function.md`
```sql
WITH monthly AS (
    SELECT DATE_TRUNC('month', "InvoiceDate") AS month, SUM(revenue) AS revenue
    FROM retail_clean
    WHERE is_return = FALSE
    GROUP BY 1
)
SELECT month, revenue,
       LAG(revenue) OVER (ORDER BY month) AS prev_month,
       ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month)) / LAG(revenue) OVER (ORDER BY month), 1) AS growth_pct
FROM monthly
ORDER BY month;
```

### Tahap 3: Analisis & Visualisasi dengan Python (±3 jam)

```python
rfm = pd.read_sql("SELECT * FROM rfm_scored", engine)   # atau jalankan ulang query #4 di atas lewat pandas.read_sql

rfm["rfm_total"] = rfm["r_score"] + rfm["f_score"] + rfm["m_score"]

def segment_customer(row):
    if row["r_score"] >= 4 and row["f_score"] >= 4 and row["m_score"] >= 4:
        return "Champions"
    elif row["f_score"] >= 4:
        return "Loyal"
    elif row["r_score"] <= 2:
        return "At Risk"
    elif row["rfm_total"] <= 6:
        return "Lost"
    else:
        return "Regular"

rfm["segment"] = rfm.apply(segment_customer, axis=1)
print(rfm["segment"].value_counts())
```

**Catatan penting**: aturan `segment_customer` di atas adalah **contoh titik awal**, bukan aturan baku yang harus diikuti persis — batas angka (`>= 4`, `<= 2`, dst.) adalah keputusan bisnis yang sebaiknya didiskusikan/disesuaikan, bukan angka ajaib. Bagian ini bagus untuk didiskusikan di code review sesama peserta: kenapa pilih threshold segitu?

**Visualisasi** (simpan ke `outputs/charts/`):
```python
import matplotlib.pyplot as plt

monthly_revenue = pd.read_sql("SELECT * FROM monthly", engine)  # atau dari query #2 Tahap 2
monthly_revenue.plot(x="month", y="revenue", kind="line", title="Trend Revenue Bulanan")
plt.savefig("outputs/charts/monthly_revenue_trend.png")

rfm["segment"].value_counts().plot(kind="bar", title="Distribusi Customer Segment")
plt.savefig("outputs/charts/customer_segment_distribution.png")
```

### Tahap 4: Business Insight & Recommendation (±2 jam)

Tulis di `README.md`, minimal 3–5 insight **konkret dan actionable** — bukan cuma deskripsi angka. Format yang berguna: **Temuan → Implikasi Bisnis → Rekomendasi**.

Contoh kerangka (isi angkanya sesuai hasil analisis kamu sendiri, karena datanya berbeda tiap peserta setelah proses cleaning):
> **Temuan**: Segmen "At Risk" menyumbang **X%** dari total revenue historis tapi tidak ada transaksi dalam **N** hari terakhir.
> **Implikasi**: Ada risiko kehilangan revenue signifikan kalau segmen ini benar-benar churn.
> **Rekomendasi**: Campaign retensi tertarget (mis. diskon khusus) untuk segmen ini, prioritas di atas akuisisi customer baru.

### Kriteria "Selesai" Minggu 2

- [ ] Repo `ecommerce-sales-analysis` ada di GitHub dengan struktur sesuai di atas
- [ ] `sql/01_data_cleaning.sql` mendokumentasikan keputusan cleaning (bukan cuma kode, tapi juga row count before/after)
- [ ] Semua 5 query di Tahap 2 jalan dan hasilnya masuk ke `notebooks/analysis.ipynb`
- [ ] Ada minimal 2 chart tersimpan di `outputs/charts/`
- [ ] `README.md` berisi 3–5 insight bisnis dalam format Temuan → Implikasi → Rekomendasi
- [ ] Bisa menjelaskan ke orang lain kenapa setiap keputusan cleaning diambil (bukan cuma "karena Pandas bisa")

Kalau semua tercentang, lanjut ke `minggu_3.md` — repo yang sama akan di-upgrade jadi pipeline otomatis dengan Spark + Airflow.
