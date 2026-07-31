---
title: Hari 5 - Incremental Load & CDC
parent: Minggu 4 - Data Quality, Orchestration & Streaming
nav_order: 6
---

# Hari 5 — Incremental Load vs Full Load, Change Data Capture (CDC)

*Jumat, 2 jam. Konsep — menutup materi konsep minggu ini sebelum hands-on Sabtu–Minggu.*

> Hari ini menjawab langsung catatan yang sengaja digantung di `materi/minggu_3/latihan_pipeline_mini_project.md`: *"`write.mode('overwrite')`... bukan pola yang scalable untuk data yang sangat besar... poin bagus untuk didiskusikan, meski di luar cakupan implementasi minggu ini."* Sekarang saatnya dibahas.

## Tujuan Belajar

- [ ] Menjelaskan kenapa full load berhenti masuk akal ketika volume data bertambah besar
- [ ] Menjelaskan konsep incremental load dan **watermark**/kolom penentu "data baru"
- [ ] Menjelaskan apa itu Change Data Capture (CDC) dan bedanya dengan incremental load berbasis kolom timestamp
- [ ] Memilih strategi load (full/incremental/CDC) yang tepat untuk skenario yang diberikan

## Untuk Instruktur: Mindset Shift

Analogi paling relevan buat developer: ini persis perbedaan antara **rebuild penuh** vs **incremental build** di sistem CI/CD. Full load itu seperti `rm -rf build/ && build_everything_from_scratch()` tiap kali — selalu benar hasilnya, tapi makin lama makin mahal seiring project membesar. Incremental load itu seperti build tool modern (Webpack/Turborepo) yang cuma memproses ulang **bagian yang berubah** sejak build terakhir — jauh lebih cepat, tapi butuh cara yang andal untuk tahu **apa saja yang berubah**.

## Konsep & Sintaks

### Full Load — Yang Sudah Dipraktikkan Minggu 3

```python
fact_sales.write.mode("overwrite").parquet(f"{output_dir}/fact_sales")
```

Tiap kali pipeline jalan, **seluruh** data sumber dibaca ulang dan **seluruh** tabel tujuan ditulis ulang dari nol — inilah yang dilakukan pipeline Minggu 3.

**Kenapa itu keputusan yang wajar saat itu**: sederhana untuk diimplementasikan & dinalar (tidak perlu melacak "apa yang sudah diproses sebelumnya"), dan untuk volume data skala latihan (ratusan ribu baris Online Retail II), full load tetap selesai dalam hitungan detik–menit — biaya kesederhanaannya belum terasa.

**Kenapa itu berhenti masuk akal di skala lebih besar**: kalau sumber data punya miliaran baris (atau terus bertambah tiap hari dalam volume besar), membaca ulang **semuanya** tiap hari cuma untuk menangkap perubahan yang sebenarnya kecil (misalnya cuma 10 ribu transaksi baru hari itu) adalah pemborosan besar — waktu proses dan biaya compute tumbuh proporsional dengan **total data historis**, bukan dengan **data yang benar-benar baru/berubah**.

### Incremental Load

Cuma memproses data yang **baru** atau **berubah** sejak run terakhir, memakai kolom penanda waktu/urutan — disebut **watermark**.

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine("postgresql://postgres:belajar@host.docker.internal:5432/postgres")

# Ambil watermark terakhir (timestamp maksimum yang SUDAH diproses sebelumnya)
last_watermark = pd.read_sql(
    "SELECT COALESCE(MAX(invoice_date), '1900-01-01') AS wm FROM fact_sales", engine
)["wm"][0]

# Baca dari SUMBER hanya baris yang lebih baru dari watermark itu
new_rows = spark.read.csv(raw_path, header=True, inferSchema=True) \
    .filter(F.col("InvoiceDate") > last_watermark)

# Append, BUKAN overwrite -- tambahkan ke data yang sudah ada
new_rows_transformed.write.mode("append").parquet(f"{output_dir}/fact_sales")
```

**Watermark** di sini adalah `InvoiceDate` — kolom yang **selalu bertambah** (data transaksi baru selalu punya tanggal lebih baru dari transaksi lama) dan dijadikan penanda "sudah diproses sampai mana". Kolom watermark yang baik harus **monoton** (selalu naik, tidak pernah mundur) — kolom seperti `updated_at`/`created_at`/`InvoiceDate` cocok, kolom seperti `status` (bisa berubah bolak-balik) tidak cocok jadi watermark.

### Perbandingan Full vs Incremental

| | Full Load | Incremental Load |
|---|---|---|
| Data yang diproses tiap run | **Semua** data sumber | Cuma yang baru/berubah sejak watermark terakhir |
| Kompleksitas implementasi | Rendah — tidak perlu melacak apapun | Sedang — perlu simpan & kelola watermark, tangani edge case (data telat masuk, dsb) |
| Biaya compute | Tumbuh sebanding total data historis | Tumbuh sebanding **data baru** saja — jauh lebih murah untuk data besar |
| Risiko | Rendah — kalau ada bug, cukup jalankan ulang dari sumber, hasil selalu konsisten | Lebih tinggi — watermark yang salah/hilang bisa bikin data terlewat atau terduplikasi |
| Cocok untuk | Data kecil–sedang, atau tujuan kesederhanaan (Minggu 3) | Data besar yang terus bertambah, di mana full load sudah tidak praktis |

### Change Data Capture (CDC)

Incremental load berbasis watermark (di atas) mengasumsikan sumber data punya kolom timestamp yang bisa dipercaya, dan **cuma menangkap `INSERT` baru** — tidak otomatis menangkap `UPDATE`/`DELETE` yang terjadi di sumber (kecuali sumbernya juga mengupdate kolom timestamp itu saat baris diubah, yang tidak selalu terjadi).

**CDC** mengatasi ini dengan menangkap **setiap perubahan** (`INSERT`/`UPDATE`/`DELETE`) langsung dari log transaksi internal database sumber (mis. **WAL** — Write-Ahead Log di PostgreSQL, atau **binlog** di MySQL) — mekanisme internal yang **sama** yang dipakai database itu sendiri untuk replikasi. Setiap perubahan di sumber otomatis "ditangkap" sebagai event, hampir real-time, tanpa perlu men-query ulang tabel sumber sama sekali.

```
Database Sumber (Postgres)          CDC Tool (mis. Debezium)         Tujuan
  WAL: INSERT row X       ------>    tangkap event  ------>    kirim ke Kafka topic
  WAL: UPDATE row Y       ------>    tangkap event  ------>    ("order-events") atau
  WAL: DELETE row Z       ------>    tangkap event  ------>    langsung ke warehouse
```

**Debezium** (disebut di `minggu_4.md`) adalah tool CDC paling umum dipakai — biasanya dijalankan sebagai *Kafka Connect connector*, menyambung ke sumber (Postgres/MySQL/dst), membaca WAL/binlog-nya, dan mempublikasikan tiap perubahan sebagai event ke topic Kafka — inilah titik pertemuan CDC dengan konsep streaming (`hari_4_pengantar_kafka.md`): CDC sering jadi **sumber event** yang mengalir ke Kafka, bukan tool yang berdiri sendiri.

### Perbandingan Incremental (Watermark) vs CDC

| | Incremental (Watermark) | CDC |
|---|---|---|
| Menangkap `INSERT` baru | Ya | Ya |
| Menangkap `UPDATE` | Cuma kalau kolom watermark ikut ter-update | Ya, selalu (semua perubahan tercatat di WAL/binlog) |
| Menangkap `DELETE` | **Tidak** (baris yang dihapus dari sumber tidak "muncul" untuk dideteksi lewat query biasa) | Ya |
| Beban ke database sumber | Query `SELECT ... WHERE updated_at > X` berkala | Minimal — CDC membaca log internal, bukan menjalankan query berat berulang ke tabel |
| Kompleksitas setup | Rendah–sedang | Lebih tinggi (butuh akses ke WAL/binlog, tool tambahan seperti Debezium, biasanya + Kafka) |
| Latency | Tergantung jadwal (biasanya batch/micro-batch) | Bisa hampir real-time |

## Kesalahan Umum

1. **Memakai kolom yang tidak monoton sebagai watermark.** Kalau kolom yang dipakai bisa "mundur" atau di-update tidak konsisten (mis. kolom `status` yang berubah bolak-balik `pending → completed → refunded`), logic `WHERE kolom > last_watermark` bisa melewatkan baris yang seharusnya diproses ulang.
2. **Incremental load yang tidak menangani `UPDATE`/`DELETE` di sumber**, padahal use case-nya butuh itu. Kalau bisnisnya butuh tahu transaksi yang **dibatalkan/dihapus** di sumber, watermark biasa tidak cukup — inilah pemicu paling umum kenapa tim akhirnya butuh CDC.
3. **Langsung lompat ke CDC/Debezium untuk kasus yang sebenarnya cukup incremental load sederhana.** CDC menambah kompleksitas infrastruktur signifikan (akses WAL, Kafka Connect, monitoring tambahan) — kalau kebutuhannya cuma "tangkap transaksi baru, `UPDATE`/`DELETE` jarang terjadi dan tidak kritis", incremental load berbasis watermark sudah cukup dan jauh lebih sederhana dioperasikan.
4. **Tidak menyimpan watermark secara andal.** Kalau watermark cuma disimpan di variabel lokal/memory (bukan di database/file persisten), restart pipeline bisa kehilangan jejak "sudah sampai mana", berisiko data terlewat (watermark hilang, dianggap mulai dari awal lagi tapi cuma proses N hari terakhir) atau terduplikasi (watermark ke-reset ke titik lebih awal dari yang seharusnya).

## Latihan

1. Pipeline `ecommerce_etl_pipeline` (Minggu 3) sekarang memakai full load (`overwrite`). Kalau volume Online Retail II tiba-tiba naik 1000x (skenario hipotetis), apa dampak konkretnya ke waktu eksekusi pipeline, dan bagaimana incremental load mengatasinya?
2. Rancang strategi incremental load untuk `fact_sales`: kolom apa yang dipakai sebagai watermark, dan di mana watermark itu sebaiknya disimpan supaya bisa diandalkan antar run pipeline (petunjuk: pikirkan tempat yang tidak hilang kalau container Airflow di-restart)?
3. Tim customer service butuh laporan akurat kalau ada customer yang **menghapus akun** mereka (dan datanya harus ikut dihapus dari warehouse sesuai kebijakan privasi). Apakah incremental load berbasis watermark `InvoiceDate` cukup untuk kasus ini? Jelaskan, dan apa alternatif yang lebih tepat.
4. Jelaskan kenapa CDC disebut membebani database sumber **lebih sedikit** dibanding incremental load berbasis query watermark, padahal CDC terdengar lebih "canggih"/kompleks.

## Kunci Jawaban & Pembahasan

**1.** Dengan full load, waktu eksekusi (baca ulang semua data + tulis ulang semua tabel) akan **ikut naik proporsional** — kalau tadinya proses 500 ribu baris makan waktu beberapa menit, memproses 500 juta baris bisa makan waktu berjam-jam, kemungkinan besar melebihi jendela waktu yang wajar untuk pipeline `daily` (bisa tumpang tindih dengan jadwal run berikutnya). Incremental load mengatasi ini karena waktu eksekusi cuma proporsional dengan **data baru sejak run terakhir** (mis. transaksi 1 hari terakhir) — walau total data historis sudah 500 juta baris, kalau transaksi baru per hari "cuma" puluhan ribu baris, waktu prosesnya tetap kecil dan stabil, tidak ikut membengkak seiring bertambahnya histori.

**2.** Watermark: `InvoiceDate` (atau `date_id` dari `dim_date`) — kolom yang monoton bertambah seiring transaksi baru masuk. Tempat penyimpanan: **jangan** di variabel Python biasa (hilang tiap restart) — simpan di Postgres sendiri (mis. tabel kecil `pipeline_metadata(pipeline_name, last_watermark)` yang di-`UPDATE` di akhir tiap run sukses), atau baca langsung dari `MAX(invoice_date)` di tabel `fact_sales` yang sudah ada (pendekatan yang dipakai di contoh kode di atas) — keduanya persisten karena disimpan di database, bukan di memory proses yang hilang saat container di-restart.

**3.** **Tidak cukup.** Incremental load berbasis watermark `InvoiceDate` cuma menangkap **transaksi baru** — penghapusan akun customer di sistem sumber **tidak muncul** sebagai baris baru dengan `InvoiceDate` terbaru, jadi tidak pernah terdeteksi lewat query `WHERE InvoiceDate > last_watermark`. Alternatif yang lebih tepat: **CDC**, karena CDC menangkap event `DELETE` langsung dari WAL/binlog sumber begitu terjadi — inilah kasus konkret yang jadi alasan utama kenapa CDC dibutuhkan, bukan cuma "karena lebih canggih", tapi karena incremental load sederhana **secara struktural tidak bisa** menangkap penghapusan data.

**4.** Incremental load berbasis watermark harus **menjalankan query** ke tabel sumber tiap kali pipeline berjalan (`SELECT ... WHERE updated_at > X`) — ini tetap beban baca tambahan ke database sumber tiap run, dan kalau tabelnya besar, query ini sendiri bisa lumayan berat (butuh index yang tepat di kolom watermark, dsb). CDC **tidak** menjalankan query sama sekali ke tabel sumber — dia membaca WAL/binlog, yaitu log internal yang **sudah** ditulis database itu sendiri untuk keperluan lain (recovery, replikasi), jadi CDC "menumpang" di mekanisme yang sudah berjalan, bukan menambah beban query baru. Kompleksitasnya lebih tinggi di sisi **infrastruktur & setup** (butuh akses WAL, tool tambahan, biasanya + Kafka) — bukan di sisi beban terhadap database sumber, yang justru lebih ringan.
