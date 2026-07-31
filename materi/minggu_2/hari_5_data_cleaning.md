---
title: Hari 5 - Data Cleaning
parent: Minggu 2 - Python Fundamentals
nav_order: 6
---

# Hari 5 — Data Cleaning: Missing Values, Duplicate, Dtype, String Manipulation

*Jumat, 2 jam. Setup & dataset: lihat `00_overview.md`. Dataset khusus: `data/sales_raw_dirty.csv`.*

## Tujuan Belajar

- [ ] Mendeteksi missing value dan memilih strategi penanganan yang tepat (bukan asal `dropna()`/`fillna()`)
- [ ] Mendeteksi & menangani baris duplikat
- [ ] Membersihkan kolom numerik/tanggal yang formatnya tidak konsisten, lalu konversi tipe data dengan benar
- [ ] Membersihkan teks (`whitespace`, casing) dengan `.str` accessor
- [ ] Memahami kenapa **parsing tanggal ambigu itu berbahaya**, dan cara menghindarinya

## Untuk Instruktur: Mindset Shift Paling Penting Minggu Ini

Ini beda secara filosofis dari 4 hari sebelumnya. Hari 1–4 soal *"bagaimana cara menulis kode yang benar"*. Hari ini soal *"apa keputusan yang benar saat data-nya sendiri tidak lengkap/tidak konsisten"* — dan itu sering kali **bukan keputusan teknis**, tapi keputusan bisnis yang harus dikonfirmasi.

Developer yang terbiasa membangun aplikasi biasanya punya refleks: **input tidak valid → tolak / lempar exception**. Di data engineering, refleks itu **tidak selalu tepat** — data yang "kotor" adalah kondisi **normal**, bukan kondisi eksepsional, karena datanya berasal dari sistem lain (kadang legacy, kadang input manual manusia) yang tidak bisa kamu kontrol. Tugas data engineer bukan menolak data kotor, tapi **punya strategi eksplisit dan terdokumentasi** untuk tiap jenis masalah: dibuang? diperbaiki (dan dengan asumsi apa)? dibiarkan tapi ditandai untuk dicek manual? Ini akan jadi pola pikir utama di mini project akhir pekan ini juga.

## Konsep & Sintaks

### 1. Deteksi & Strategi Missing Value

```python
df.isna().sum()          # jumlah nilai NaN per kolom — langkah pertama selalu
df.info()                 # cara cepat lain lihat non-null count per kolom
```

Strategi (pilih sesuai konteks, **bukan** template baku):
```python
df.dropna()                          # buang baris yang ADA NaN di kolom manapun — agresif
df.dropna(subset=["customer_name"])  # buang baris HANYA kalau customer_name kosong — lebih presisi
df.fillna({"category": "Unknown"})   # isi dengan nilai default eksplisit
df["quantity"].fillna(df["quantity"].median())  # isi dengan statistik (median lebih tahan outlier dari mean)
```

### 2. Deteksi & Penanganan Duplicate

```python
df.duplicated()              # boolean Series, True untuk baris yang identik dengan baris SEBELUMNYA
df.duplicated().sum()        # jumlah baris duplikat
df[df.duplicated()]          # lihat baris mana saja yang duplikat, sebelum dibuang
df.drop_duplicates()         # buang duplikat (simpan kemunculan pertama by default)
```

Selalu **lihat dulu** baris yang keteridentifikasi duplikat sebelum `drop_duplicates()` — supaya yakin itu betul duplikat data (bukan 2 transaksi berbeda yang kebetulan datanya identik).

### 3. Membersihkan & Konversi Tipe Data

```python
pd.to_numeric(series, errors="coerce")     # ubah ke angka, yang gagal parse jadi NaN (bukan error)
pd.to_datetime(series, errors="coerce")    # ubah ke datetime, yang gagal parse jadi NaT
```

`errors="coerce"` itu pilihan sadar: kamu **memilih** kehilangan baris yang gagal parse (jadi `NaN`/`NaT`) daripada program crash. Setelah itu, **wajib cek** berapa banyak yang jadi `NaN`/`NaT` — itu petunjuk seberapa "kotor" data sumbernya.

### 4. String Cleaning dengan `.str`

```python
df["country"].str.strip()           # buang whitespace di awal/akhir
df["country"].str.upper()           # normalisasi casing (lihat catatan di Kesalahan Umum soal .title())
df["unit_price"].str.replace("$", "", regex=False)
```

## Contoh Kode — Membersihkan `sales_raw_dirty.csv` Langkah demi Langkah

```python
import pandas as pd

df = pd.read_csv("data/sales_raw_dirty.csv")
df.info()
```
Output `.info()` akan menunjukkan: 20 baris, tapi `customer_name` cuma 19 non-null, `category` cuma 18 non-null, `quantity` cuma 19 non-null (dan **sudah** ke-cast jadi `float64`, bukan `int64`, karena ada `NaN` — Pandas otomatis "menaikkan" tipe int ke float begitu ada nilai kosong di kolom itu). `unit_price` bertipe `object` (string), bukan angka — karena isinya campuran `"15.00"`, `"$8.00"`, `"3,50"`.

**Langkah 1 — cek & tangani missing value**
```python
print(df.isna().sum())
# customer_name: 1, category: 2, quantity: 1

df["category"] = df["category"].fillna("Unknown")
df["quantity"] = df["quantity"].fillna(df["quantity"].median())
df = df.dropna(subset=["customer_name"])   # baris tanpa nama customer tidak berguna untuk analisis per-customer
```

**Langkah 2 — cek & buang duplikat**
```python
print(df.duplicated().sum())   # 1
df = df.drop_duplicates()
```

**Langkah 3 — bersihkan `country` (whitespace + casing tidak konsisten)**
```python
print(df["country"].unique())
# [' indonesia', 'Indonesia', 'UK ', 'UK', 'Germany', 'Singapore', 'INDONESIA']

df["country"] = df["country"].str.strip().str.upper()
print(df["country"].unique())
# ['INDONESIA', 'UK', 'GERMANY', 'SINGAPORE']
```

**Langkah 4 — bersihkan `unit_price` (simbol mata uang & pemisah desimal campuran)**
```python
df["unit_price"] = (
    df["unit_price"].astype(str)
    .str.replace("$", "", regex=False)
    .str.replace(",", ".", regex=False)
)
df["unit_price"] = pd.to_numeric(df["unit_price"], errors="coerce")
```

**Langkah 5 — parsing tanggal, dengan penuh kewaspadaan (lihat Kesalahan Umum #2 sebelum coba shortcut apapun)**
```python
df["order_date_parsed"] = pd.to_datetime(df["order_date"], format="%Y-%m-%d", errors="coerce")
gagal = df[df["order_date_parsed"].isna()]
print(gagal[["order_id", "order_date"]])
# order_id 10, order_date '12/03/2024' — satu-satunya yang gagal parse format standar
```

## Kesalahan Umum

1. **Percaya begitu saja tipe data hasil `read_csv()`.** Kolom `unit_price` di dataset ini **terbaca sebagai `object` (string)**, bukan angka — karena campuran `"$8.00"` dan `"3,50"`. Kalau langsung dipakai untuk operasi matematika tanpa cek `dtypes` dulu, hasilnya bisa error (`TypeError`) atau, lebih berbahaya, **operasi string yang "kebetulan jalan"** tapi hasilnya salah (mis. `"15.00" * 2` menghasilkan `"15.0015.00"`, bukan `30.0`, karena Python mengalikan **string**, bukan angka). Selalu `df.dtypes` atau `df.info()` di awal, jangan asumsi.

2. **Mempercayai `pd.to_datetime()` menebak format tanggal dengan benar untuk tanggal ambigu.** Ini bug paling berbahaya di sesi ini karena **tidak menghasilkan error** — hasilnya salah secara diam-diam.
   ```python
   # BAHAYA:
   pd.to_datetime(df["order_date"], format="mixed")
   # '12/03/2024' ditebak sebagai 3 Desember 2024, PADAHAL baris lain untuk order_id yang sama
   # menunjukkan '2024-03-12' (12 Maret) — keduanya order_id 10, tanggalnya harus sama!
   ```
   `12/03/2024` bisa berarti **12 Maret** (format `DD/MM/YYYY`, umum di Indonesia/Eropa) atau **3 Desember** (format `MM/DD/YYYY`, umum di Amerika) — keduanya valid secara sintaks, dan pandas akan menebak salah satu tanpa memberitahu kamu bahwa itu tebakan. Pendekatan yang aman: **jangan pernah andalkan inference otomatis untuk tanggal dari sumber yang formatnya tidak kamu ketahui pasti.** Parsing eksplisit dengan format yang diketahui benar, isolasi baris yang gagal parse (`errors="coerce"` lalu cek `.isna()`), lalu investigasi baris-baris itu satu per satu — seperti dicontohkan di Langkah 5 di atas.

3. **`.str.title()` merusak akronim.** `"UK".str.title()` menghasilkan `"Uk"`, bukan `"UK"` — title-case mengasumsikan setiap kata adalah kata biasa, bukan singkatan. Untuk normalisasi kategori/kode (negara, status, dsb.), `.str.upper()` atau `.str.lower()` jauh lebih aman daripada `.str.title()`. `.str.title()` baru masuk akal untuk data yang memang nama orang/tempat biasa (`"andi wijaya"` → `"Andi Wijaya"`), bukan kode/singkatan.

4. **`dropna()` tanpa `subset=` terlalu agresif.** `df.dropna()` polos akan membuang baris kalau **kolom manapun** ada yang `NaN` — di dataset ini efeknya membuang baris yang cuma `category`-nya kosong padahal datanya sendiri (customer, produk, harga) lengkap dan berguna. Selalu spesifik: `dropna(subset=[...])` untuk kolom yang benar-benar wajib ada isinya.

5. **Asumsi `quantity` negatif itu data error, padahal itu sinyal bisnis.** Baris `order_id=11` (Emma Watson, refunded) punya `quantity = -1` — ini **bukan** data kotor yang perlu "diperbaiki" jadi positif, tapi **konvensi umum** di data retail: quantity negatif = barang dikembalikan/retur. Ini pola yang sama persis dengan yang akan ditemui di dataset **Online Retail II** minggu ini (lihat `latihan_eda_dan_mini_project.md`) — jangan "bersihkan" nilai yang sebetulnya bermakna.

## Latihan

Pakai `data/sales_raw_dirty.csv`.

1. Hitung jumlah `NaN` per kolom, lalu tentukan strategi untuk masing-masing (drop baris / isi default / isi statistik) — tulis alasan tiap keputusan dalam 1 kalimat.
2. Bersihkan kolom `country` (whitespace + casing), lalu tampilkan `value_counts()`-nya setelah dibersihkan.
3. Bersihkan kolom `unit_price` jadi numerik sepenuhnya (tangani `$` dan koma desimal), lalu hitung `subtotal = quantity * unit_price` untuk tiap baris.
4. Deteksi baris yang gagal di-parse `pd.to_datetime(..., format="%Y-%m-%d", errors="coerce")`, investigasi manual (cross-check ke baris lain dengan `order_id` yang sama kalau ada), lalu perbaiki.
5. Setelah semua langkah cleaning selesai (gabungkan #1–#4), hitung total revenue (`sum(subtotal)` untuk `status == 'completed'`) dari data yang sudah bersih — bandingkan dengan angka `807.00` yang sudah diverifikasi berkali-kali di `materi/minggu_1/` untuk dataset yang sama dalam kondisi bersih. Kalau angkanya beda, itu tandanya ada langkah cleaning yang belum tepat — cari selisihnya.

## Kunci Jawaban & Pembahasan

**1.**
```python
df.isna().sum()
```
- `customer_name` (1 baris kosong): **drop baris ini** — tanpa identitas customer, baris ini tidak berguna untuk analisis per-customer yang jadi fokus mini project.
- `category` (2 baris kosong): **isi `"Unknown"`** — baris tetap berguna untuk analisis revenue total, cuma tidak bisa masuk breakdown per kategori; membuang baris ini artinya kehilangan data penjualan asli tanpa alasan kuat.
- `quantity` (1 baris kosong): **isi dengan median** — dianggap lebih aman daripada `0` (yang akan salah membuat subtotal jadi 0, padahal transaksinya nyata) atau mean (lebih rentan bias dari outlier seperti order dengan quantity 10).

**2.**
```python
df["country"] = df["country"].str.strip().str.upper()
df["country"].value_counts()
```
Hasil: `INDONESIA` (paling banyak), `UK`, `GERMANY`, `SINGAPORE` — total distinct values turun dari 7 varian kotor jadi 4 nilai bersih.

**3.**
```python
df["unit_price"] = (
    df["unit_price"].astype(str)
    .str.replace("$", "", regex=False)
    .str.replace(",", ".", regex=False)
)
df["unit_price"] = pd.to_numeric(df["unit_price"], errors="coerce")
df["subtotal"] = df["quantity"] * df["unit_price"]
```

**4.**
```python
df["order_date_parsed"] = pd.to_datetime(df["order_date"], format="%Y-%m-%d", errors="coerce")
gagal = df[df["order_date_parsed"].isna()]
print(gagal)   # order_id 10, '12/03/2024'
```
Cross-check: baris lain dengan `order_id == 10` punya `order_date = '2024-03-12'` (12 Maret) — jadi `'12/03/2024'` seharusnya juga 12 Maret, **bukan** 3 Desember. Perbaikan manual:
```python
df.loc[df["order_date"] == "12/03/2024", "order_date_parsed"] = pd.Timestamp("2024-03-12")
```
Ini contoh nyata kenapa Langkah 5 di atas (isolasi baris gagal, investigasi manual) jauh lebih aman daripada mempercayai `pd.to_datetime(..., format="mixed")` begitu saja — kalau dibiarkan, tanggal order ini akan salah 3 bulan lebih tanpa ada error apapun yang muncul.

**5.**
Setelah semua langkah #1–#4 diterapkan dengan benar (termasuk membuang 1 baris duplikat, dan `dropna(subset=["customer_name"])`), filter `status == 'completed'` lalu `sum(subtotal)` menghasilkan **807.00** — cocok persis dengan angka yang sudah diverifikasi berkali-kali di `materi/minggu_1/` untuk dataset bersihnya.

Menariknya, kecocokan ini sedikit "kebetulan": baris `order_id=2` (Mechanical Keyboard) dibuang karena `customer_name`-nya kosong (kehilangan revenue 45.00), sementara baris `order_id=5` (Mechanical Keyboard juga) yang `quantity`-nya kosong diisi median (`2.0`) — padahal nilai aslinya (bisa dicek di `data/order_items.csv`) adalah `1`, jadi revenue baris itu jadi *kelebihan* 45.00 (90.00 alih-alih 45.00 yang seharusnya). Dua penyimpangan ini (−45.00 dan +45.00) kebetulan saling menutupi. **Poin pentingnya**: strategi cleaning yang berbeda (mis. `dropna()` untuk `quantity` alih-alih diisi median) akan menghasilkan total akhir yang **berbeda** dari 807.00 — kecocokan sempurna di atas bukan jaminan, cuma kebetulan dari data spesifik ini. Yang harus dipegang: keputusan cleaning didokumentasikan & masuk akal, bukan dikejar supaya "pas" dengan angka tertentu.
