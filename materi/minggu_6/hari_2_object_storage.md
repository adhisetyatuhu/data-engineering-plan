---
title: Hari 2 - Object Storage
parent: Minggu 6 - Cloud Platform Fundamentals
nav_order: 3
---

# Hari 2 — Object Storage: Bucket, Storage Class/Tiering, Lifecycle Policy

*Selasa, 2 jam. Konsep + contoh kode GCS (Google Cloud Storage) — hands-on setup bucket sungguhan baru Sabtu.*

> Sambungan langsung dari `materi/minggu_3/hari_1_arsitektur_data.md` — object storage adalah **implementasi konkret** dari konsep "data lake" yang dibahas minggu itu (S3/GCS disebut eksplisit di sana sebagai contoh tools data lake).

## Tujuan Belajar

- [ ] Menjelaskan object storage dan bedanya dari filesystem tradisional
- [ ] Menjelaskan konsep bucket, object, dan key/path
- [ ] Menjelaskan storage class/tiering dan trade-off biaya vs latency akses
- [ ] Merancang lifecycle policy untuk data pipeline `ecommerce-etl-pipeline`

## Untuk Instruktur: Mindset Shift

Developer sering menyamakan object storage dengan "folder di cloud" — mirip tapi **ada beda fundamental** yang penting diluruskan: object storage **tidak punya struktur direktori sungguhan**. Yang terlihat seperti folder (`raw/online_retail_II.csv`) sebenarnya cuma **1 string key panjang** dengan karakter `/` di dalamnya — bukan direktori nested sungguhan seperti di filesystem biasa.

Analogi yang pas: object storage itu seperti **key-value store raksasa** (mirip Redis/DynamoDB, tapi value-nya bisa berupa file berukuran besar) — `key = "raw/online_retail_II.csv"`, `value = isi file itu sendiri`. UI/CLI cloud provider **mensimulasikan** tampilan folder dari key yang punya prefix sama, tapi di baliknya tidak ada operasi "buat folder" atau "pindah folder" sungguhan seperti `mkdir`/`mv` di filesystem — yang ada cuma operasi terhadap **object individual** (put, get, delete, list-by-prefix).

## Konsep & Sintaks

### Bucket, Object, Key

| Istilah | Definisi | Analogi |
|---|---|---|
| **Bucket** | Namespace/container tingkat atas, nama harus **unik secara global** (GCS/S3 berbagi namespace global lintas semua pengguna) | Nama database/schema |
| **Object** | 1 file/blob data yang disimpan (bisa apapun: CSV, Parquet, gambar, model ML) | 1 baris di key-value store |
| **Key** | String path lengkap object di dalam bucket (mis. `processed/fact_sales/part-0001.parquet`) | Primary key di key-value store |

```
gs://ecommerce-data-lake/raw/online_retail_II.csv
    ^bucket^          ^--------- key ---------^
```

**Kenapa nama bucket harus unik global**: berbeda dari nama file di laptop (bisa sama-sama punya `data.csv` di folder berbeda), nama bucket di GCS/S3 **berbagi 1 namespace** lintas **semua** pengguna cloud provider itu di seluruh dunia — kalau nama `data-lake` sudah dipakai orang lain, kamu tidak bisa memakainya lagi. Ini sebabnya nama bucket biasanya diberi awalan/akhiran unik (mis. `ecommerce-data-lake-<nama-project-id>`).

### Kenapa Tidak Ada "Folder Sungguhan"

```python
from google.cloud import storage

client = storage.Client()
bucket = client.bucket("ecommerce-data-lake")

# Ini BUKAN "membuat folder raw/" -- ini cuma object dengan key yang mengandung "/"
blob = bucket.blob("raw/online_retail_II.csv")
blob.upload_from_filename("data/raw/online_retail_II.csv")

# "List folder raw/" sebenarnya query "cari semua object dengan prefix raw/"
for blob in bucket.list_blobs(prefix="raw/"):
    print(blob.name)
```

Konsekuensi praktis dari ini: **menghapus "folder"** (`raw/`) sebenarnya berarti menghapus **satu per satu** semua object yang key-nya berawalan `raw/` — tidak ada operasi tunggal "hapus folder ini beserta isinya" di level API dasar (walau UI Console biasanya menyediakan tombol yang **terlihat** seperti itu, di baliknya tetap melakukan banyak operasi delete individual).

### Storage Class / Tiering

Trade-off inti: **makin murah storage-nya, makin mahal/lambat mengaksesnya** (dan makin lama komitmen minimum penyimpanannya).

| Storage Class (GCS) | Padanan S3 | Biaya Simpan | Biaya Akses | Kapan Dipakai |
|---|---|---|---|---|
| **Standard** | S3 Standard | Paling mahal | Murah, cepat, tanpa minimum durasi | Data yang sering diakses (< 30 hari terakhir) |
| **Nearline** | S3 Standard-IA | Lebih murah | Ada biaya akses tambahan | Data diakses ~1x/bulan (backup, arsip jangka pendek) |
| **Coldline** | S3 Glacier | Lebih murah lagi | Biaya akses lebih tinggi | Data diakses ~1x/tahun (arsip jangka panjang) |
| **Archive** | S3 Glacier Deep Archive | Paling murah | Paling mahal, retrieval bisa butuh jam-an | Data yang wajib disimpan (compliance/regulasi) tapi hampir tidak pernah dibuka lagi |

Analogi yang membantu: ini persis seperti **caching tier** yang developer sudah kenal (memory cache vs disk cache vs cold storage) — data "panas" (sering diakses) ditaruh di tier cepat-mahal, data "dingin" (jarang diakses) ditaruh di tier lambat-murah, keputusannya berdasarkan **pola akses**, bukan jenis data itu sendiri.

Untuk `ecommerce-etl-pipeline`: `raw/online_retail_II.csv` yang sudah diproses (hasil transform sudah ada di `processed/`) jadi kandidat tepat untuk dipindah ke tier lebih murah setelah beberapa waktu — jarang dibuka lagi kecuali untuk audit/reprocessing, tapi tetap perlu disimpan (data governance, `materi/minggu_5/hari_5_compliance_privacy.md` soal retensi).

### Lifecycle Policy

Aturan **otomatis** yang memindahkan/menghapus object berdasarkan umur atau kondisi tertentu, tanpa perlu campur tangan manual.

```json
{
  "lifecycle": {
    "rule": [
      {
        "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
        "condition": {"age": 30, "matchesPrefix": ["raw/"]}
      },
      {
        "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
        "condition": {"age": 180, "matchesPrefix": ["raw/"]}
      },
      {
        "action": {"type": "Delete"},
        "condition": {"age": 730, "matchesPrefix": ["raw/"]}
      }
    ]
  }
}
```

Aturan ini: object di `raw/` pindah ke Nearline setelah 30 hari, ke Coldline setelah 180 hari, dihapus permanen setelah 2 tahun (730 hari) — persis kebijakan retensi hipotetis yang sudah ditulis di `GOVERNANCE.md` (`materi/minggu_5/latihan_catalog_lineage_mini_project.md`), sekarang **ditegakkan secara teknis**, bukan cuma tertulis di dokumen.

**Kenapa ini penting secara arsitektural, bukan cuma soal hemat biaya**: lifecycle policy adalah bukti konkret bagaimana **governance policy** (Minggu 5) diterjemahkan jadi **konfigurasi infrastruktur sungguhan** (Minggu 6) — dokumentasi kebijakan retensi yang cuma tertulis di README tanpa penegakan teknis gampang diabaikan/terlupakan; lifecycle policy membuatnya terjadi otomatis, tidak bergantung pada seseorang mengingat untuk menghapus data manual.

## Kesalahan Umum

1. **Mengira bisa "rename folder" seperti di filesystem.** Karena tidak ada folder sungguhan (cuma key dengan prefix sama), "memindahkan" object dari 1 prefix ke prefix lain sebenarnya berarti **copy ke key baru, lalu delete key lama** — bukan operasi rename atomik tunggal seperti `mv` di filesystem.
2. **Menaruh semua data di storage class Standard selamanya** tanpa mempertimbangkan lifecycle — untuk data yang jarang diakses setelah beberapa waktu (seperti raw data historis), ini berarti membayar lebih mahal dari yang seharusnya tanpa manfaat tambahan.
3. **Memindahkan data ke tier murah (Coldline/Archive) padahal masih sering diakses.** Biaya **akses** ke tier murah jauh lebih mahal per-operasi — kalau data itu ternyata masih rutin dibuka, biaya akses yang berulang bisa lebih mahal total dibanding tetap di Standard. Lifecycle policy harus berdasarkan **pola akses nyata**, bukan asumsi.
4. **Membuat bucket publik "sementara" untuk testing dan lupa mengembalikannya.** Sudah dibahas sebagai kesalahan umum #1 di `hari_1_konsep_cloud.md` — disebut lagi karena ini kesalahan paling sering terjadi justru di tahap belajar/eksperimen seperti mini project minggu ini.

## Latihan

1. Rancang struktur key (bukan "folder", tapi tetap boleh ditulis dengan notasi mirip path) untuk bucket `ecommerce-data-lake` yang menyimpan: raw CSV, hasil staging (`retail_clean`), dan hasil star schema (4 tabel Parquet) dari `materi/minggu_3/latihan_pipeline_mini_project.md`.
2. Jelaskan kenapa "menghapus folder `raw/` yang berisi 10.000 object" secara teknis berbeda dari "menghapus 1 file di laptop" — apa konsekuensi praktisnya kalau proses itu terputus di tengah jalan (mis. koneksi internet putus)?
3. Untuk data hasil `great_expectations/expectations/ecommerce_suite.json` (laporan validasi, Minggu 4) yang disimpan di bucket — apakah ini kandidat baik untuk lifecycle policy pindah ke tier murah? Jelaskan berdasarkan pola akses yang realistis untuk jenis data ini.
4. Sebuah tim menyimpan backup harian database (1 file besar per hari) langsung di storage class Standard, tidak pernah dipindah, tidak pernah dihapus, selama 3 tahun. Identifikasi 2 masalah dari pendekatan ini, dan usulkan lifecycle policy yang lebih baik.

## Kunci Jawaban & Pembahasan

**1.**
```
gs://ecommerce-data-lake/raw/online_retail_II.csv
gs://ecommerce-data-lake/staging/retail_clean/part-*.parquet
gs://ecommerce-data-lake/processed/dim_customer/part-*.parquet
gs://ecommerce-data-lake/processed/dim_product/part-*.parquet
gs://ecommerce-data-lake/processed/dim_date/part-*.parquet
gs://ecommerce-data-lake/processed/fact_sales/part-*.parquet
```
Struktur ini mencerminkan tahapan pipeline (`raw` → `staging` → `processed`) sebagai prefix tingkat atas — memudahkan lifecycle policy diterapkan berbeda per tahap (mis. `raw/` boleh dipindah ke tier murah lebih cepat karena sudah ada hasil olahannya di `processed/`, sementara `processed/` yang aktif dipakai warehouse tetap di Standard).

**2.** Menghapus 1 file di laptop adalah **1 operasi atomik** di level filesystem (biasanya cuma menghapus entry di direktori, sangat cepat, dan kalaupun gagal di tengah, hasilnya jelas: file itu tetap ada utuh atau sudah hilang). Menghapus "folder" `raw/` yang berisi 10.000 object berarti **10.000 operasi delete terpisah** (karena tidak ada folder sungguhan, cuma banyak object dengan prefix sama) — kalau proses ini terputus di tengah jalan (mis. baru 6.000 object terhapus), hasilnya **state yang tidak konsisten**: sebagian object di "folder" itu sudah hilang, sebagian belum, dan tidak ada cara otomatis tahu persis mana yang sudah/belum tanpa mengecek ulang list object yang tersisa. Ini beda fundamental yang wajib diketahui developer yang terbiasa berasumsi operasi filesystem selalu atomik.

**3.** **Kurang cocok** untuk lifecycle policy agresif ke tier murah — laporan validasi Great Expectations biasanya **sering diakses** dalam jangka pendek setelah dibuat (dicek segera setelah tiap pipeline run untuk memastikan data quality, terutama saat sedang men-debug kegagalan — lihat proses triase di `materi/minggu_5/hari_4_stewardship_quality_governance.md`), tapi jarang diakses lagi setelah beberapa minggu/bulan kecuali untuk audit historis. Kebijakan yang masuk akal: biarkan di Standard untuk beberapa minggu pertama (periode paling mungkin diakses untuk debugging), baru dipindah ke Nearline/Coldline setelahnya untuk keperluan arsip/audit jangka panjang — bukan dipindah agresif dalam hitungan hari seperti raw data mentah yang polanya beda.

**4.** Masalah 1: **biaya berlebih** — menyimpan backup harian di tier Standard (paling mahal) selama 3 tahun penuh, padahal backup yang lebih lama dari beberapa bulan hampir pasti sudah sangat jarang (atau tidak pernah) diakses lagi. Masalah 2: **tidak ada kebijakan retensi/penghapusan** — backup terus menumpuk tanpa batas, ini bukan cuma soal biaya storage yang terus bertambah, tapi juga berpotensi masalah governance (menyimpan data lebih lama dari kebutuhan/kebijakan retensi yang seharusnya, menyambung ke `materi/minggu_5/hari_5_compliance_privacy.md` prinsip data minimization). Lifecycle policy yang lebih baik: backup pindah ke Nearline setelah 30 hari, ke Coldline/Archive setelah 90 hari, dan dihapus setelah periode retensi yang disepakati (mis. 2-3 tahun sesuai kebutuhan bisnis/regulasi) — bukan disimpan selamanya tanpa batas waktu.
