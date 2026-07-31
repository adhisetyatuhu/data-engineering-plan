---
title: Hari 1 - OLTP vs OLAP, Warehouse vs Lake vs Lakehouse
parent: Minggu 3 - Arsitektur Data & Big Data Pipeline
nav_order: 2
---

# Hari 1 — OLTP vs OLAP, Data Warehouse vs Data Lake vs Lakehouse

*Senin, 2 jam. Konsep murni — tidak ada setup tool baru hari ini.*

> Sesi ini fondasi untuk seluruh sisa roadmap. Kalau peserta paham betul kenapa data "dipindah keluar" dari database aplikasi ke sistem terpisah, semua keputusan desain minggu-minggu berikutnya (Minggu 5 governance, Minggu 6 cloud warehouse) akan terasa masuk akal, bukan sekadar hafalan istilah.

## Tujuan Belajar

- [ ] Menjelaskan beda OLTP dan OLAP dari sisi **tujuan penggunaan**, bukan cuma definisi
- [ ] Menjelaskan kenapa menjalankan query analitik berat langsung di database production itu berbahaya
- [ ] Membedakan Data Warehouse, Data Lake, dan Data Lakehouse: struktur data, jenis query, tipe user, contoh tools
- [ ] Memilih arsitektur yang tepat untuk skenario yang diberikan, dengan alasan

## Untuk Instruktur: Mindset Shift

Developer sudah sangat familiar dengan **satu** database: yang menopang aplikasi mereka (Postgres/MySQL/dst — persis `pg-belajar` dari Minggu 1). Itu adalah **OLTP** (Online Transaction Processing). Materi hari ini menjelaskan **kenapa** data dari sana biasanya perlu "dipindah" ke sistem lain untuk dianalisis, bukan langsung di-query di tempat.

Analogi yang biasanya langsung klik: database OLTP itu seperti **meja kasir toko** — dirancang untuk transaksi cepat, satu-per-satu, akurat (checkout pelanggan A, lalu B, lalu C). Kalau tiba-tiba manajer datang minta "hitung total penjualan 3 tahun terakhir per kategori produk" langsung di meja kasir itu, antrian pelanggan macet total. Data warehouse adalah **ruang belakang khusus** tempat semua data transaksi disalin dan disusun ulang supaya pertanyaan seperti itu bisa dijawab tanpa mengganggu kasir yang sedang melayani pelanggan.

## Konsep & Sintaks

### OLTP vs OLAP

| | OLTP (Online Transaction Processing) | OLAP (Online Analytical Processing) |
|---|---|---|
| Tujuan | Menjalankan operasi bisnis sehari-hari | Menganalisis data historis untuk keputusan |
| Contoh query | `INSERT` 1 order baru, `UPDATE` status order | `SUM(revenue)` jutaan baris, `GROUP BY` bertingkat |
| Bentuk query | Banyak, kecil, cepat (milidetik), sering `INSERT`/`UPDATE` | Sedikit, besar, kompleks (detik–menit), mayoritas `SELECT` |
| Desain skema | **Normalized** (minim duplikasi, banyak tabel kecil) — persis dataset `customers`/`orders`/`order_items` Minggu 1 | **Denormalized** (star schema, sengaja ada duplikasi demi kecepatan baca) — dibahas detail Hari 2 |
| User tipikal | Aplikasi/API (banyak user bersamaan) | Analyst, data scientist, dashboard BI (lebih sedikit user, tiap query lebih berat) |
| Contoh sistem | PostgreSQL/MySQL yang menopang aplikasi | Snowflake, BigQuery, Redshift |

Database yang sudah dipakai sepanjang Minggu 1–2 (`pg-belajar`) adalah contoh **OLTP** — skemanya normalized (`customers`, `orders`, `order_items` terpisah) supaya insert/update konsisten dan cepat. `retail_clean` yang dibuat di Minggu 2 (1 tabel flat, isinya hasil `GROUP BY`/agregasi) mulai condong ke pola **OLAP** — struktur diubah supaya query analitik lebih gampang, bukan supaya insert satu baris paling efisien.

### Kenapa Tidak Query Analitik Langsung di OLTP?

1. **Resource contention** — query analitik berat (scan jutaan baris, agregasi) bisa mengunci/melambatkan tabel yang sedang dipakai transaksi live. Analog: menjalankan `SELECT * FROM huge_table` tanpa index di production API yang sedang melayani traffic — bisa bikin API lambat/timeout untuk user asli.
2. **Skema tidak cocok** — skema normalized OLTP butuh banyak `JOIN` untuk pertanyaan analitik sederhana (mis. "revenue per bulan" butuh join `orders` + `order_items` + `products`). Untuk analisis berulang, ini mahal dan lambat dibanding skema yang sudah "dipratangkan" bentuknya.
3. **Data historis vs data terkini** — OLTP biasanya cuma butuh data terkini (status order sekarang), analitik butuh histori panjang (trend 3 tahun) yang kalau disimpan semua di OLTP akan membengkak dan memperlambat operasi transaksi sehari-hari.
4. **Isolasi kegagalan** — kalau query analitik "nakal" bikin sistem analitik down, aplikasi/transaksi bisnis tetap jalan normal karena keduanya sistem terpisah.

### Data Warehouse vs Data Lake vs Data Lakehouse

| | Data Warehouse | Data Lake | Data Lakehouse |
|---|---|---|---|
| Jenis data | **Structured** saja (tabel, skema ketat, ditentukan sebelum data masuk / *schema-on-write*) | Structured + semi-structured + unstructured (CSV, JSON, gambar, log, apa saja) | Sama fleksibelnya dengan lake, tapi menambahkan lapisan skema & transaksi |
| Skema | *Schema-on-write* — divalidasi saat data ditulis | *Schema-on-read* — divalidasi saat data dibaca, boleh berubah-ubah | Schema-on-read + *schema enforcement* opsional |
| Storage | Biasanya proprietary, terintegrasi dengan compute | Object storage murah (S3, GCS, ADLS), compute terpisah | Object storage murah + format tabel terbuka (Delta Lake, Apache Iceberg, Apache Hudi) |
| Kekuatan | Query cepat, konsisten, cocok untuk BI/dashboard | Murah, fleksibel, cocok untuk data mentah/ML | Coba ambil kekuatan warehouse (query cepat, ACID transaction) + lake (murah, fleksibel) sekaligus |
| Kelemahan | Mahal untuk data mentah/tidak terstruktur dalam volume besar | Gampang jadi "data swamp" — data numpuk tanpa tata kelola, susah dipercaya kualitasnya | Ekosistem lebih baru, sebagian tooling belum sematang warehouse murni |
| Contoh tools | Snowflake, BigQuery, Redshift | S3/GCS/ADLS (mentah) + Athena/Presto untuk query | Databricks (Delta Lake), Snowflake (Iceberg support), BigQuery (BigLake) |

**Data swamp** — istilah penting untuk disebut ke peserta: data lake yang tidak punya tata kelola (tidak ada katalog, tidak ada standar penamaan/format, tidak ada yang tahu data mana yang valid) berubah jadi "rawa data" — secara teknis semua data ada di sana, tapi tidak ada yang bisa/berani pakai karena tidak percaya kualitasnya. Ini alasan **Data Governance** jadi topik penuh 1 minggu sendiri (Minggu 5) — lake tanpa governance adalah risiko nyata, bukan ketakutan teoretis.

**Kenapa lakehouse muncul**: dulu tim data sering harus punya **dua** sistem terpisah — data lake untuk data mentah/ML, data warehouse untuk BI/dashboard — dan harus terus menyalin data antara keduanya (duplikasi, biaya, dan risiko data tidak sinkron). Lakehouse mencoba menggabungkan keduanya jadi satu lapisan storage, dengan format tabel modern (Delta Lake/Iceberg/Hudi) yang menambahkan fitur ala-warehouse (ACID transaction, `UPDATE`/`DELETE` per baris, time travel) di atas object storage murah ala-lake.

## Contoh Kasus

**Skenario**: startup e-commerce (persis konteks portfolio `ecommerce-etl-pipeline`) punya database Postgres yang menopang website (order, cart, user login). Tim data analytics minta akses untuk bikin dashboard revenue harian, dan tim data science mau mulai eksperimen model rekomendasi produk (butuh data mentah + log clickstream user, formatnya JSON).

- **Kalau langsung kasih akses ke Postgres production**: berisiko (poin 1–4 di atas). Query dashboard yang berat bisa mengganggu checkout pelanggan asli.
- **Data warehouse saja**: bagus untuk dashboard revenue (data terstruktur, query cepat), tapi kurang cocok untuk log clickstream JSON mentah yang skemanya belum jelas/berubah-ubah, dan mahal kalau menyimpan semua data mentah "kalau-kalau dibutuhkan nanti".
- **Data lake saja**: bagus untuk simpan semua data mentah (termasuk clickstream JSON) dengan murah, tapi dashboard BI jadi lebih lambat/rumit karena tidak ada struktur siap pakai, dan tanpa governance berisiko jadi data swamp.
- **Pendekatan realistis** (dan ini yang akan dibangun bertahap di roadmap ini): pipeline salin data dari Postgres OLTP → data lake (data mentah, murah, semua jenis data) → diproses & disusun ulang → data warehouse (untuk dashboard/BI yang butuh cepat & terstruktur). Inilah pola **ETL/ELT** yang jadi topik Hari 3, dan alasan Minggu 3 ini ada.

## Kesalahan Umum

1. **Mengira "data warehouse" = "database besar".** Bedanya bukan ukuran, tapi **tujuan desain**: warehouse dioptimalkan untuk baca/agregasi data besar (kolom-oriented, biasanya), database aplikasi dioptimalkan untuk baca-tulis baris individual cepat (baris-oriented). Database aplikasi kecil pun tetap OLTP; warehouse kecil pun tetap OLAP.
2. **Mengira data lake = "tempat buang semua file".** Tanpa struktur folder/katalog/standar minimal, itu justru definisi data swamp, bukan data lake yang berfungsi.
3. **Mengira lakehouse selalu pilihan terbaik.** Untuk kebutuhan BI sederhana dengan data terstruktur yang sudah jelas skemanya, warehouse murni bisa lebih simpel dan matang toolingnya. Lakehouse unggul justru saat kebutuhannya campuran (data terstruktur + tidak terstruktur, atau butuh warehouse + ML platform sekaligus).
4. **Menganggap OLTP dan OLAP saling menggantikan.** Keduanya biasanya **hidup berdampingan** — OLTP menopang aplikasi, OLAP menopang analisis, dan ada pipeline yang menjembatani (topik Hari 3–5).

## Latihan

1. Sebuah aplikasi e-commerce production mulai lemot setiap jam 9 pagi — ternyata karena tim finance menjalankan laporan bulanan langsung query ke database yang sama dengan aplikasi. Jelaskan dengan istilah OLTP/OLAP kenapa ini terjadi, dan usulkan solusi arsitektur.
2. Perusahaan logistik ingin menyimpan: (a) data pengiriman terstruktur (tabel `shipments`, `drivers`, `routes`) untuk dashboard operasional harian, dan (b) rekaman GPS mentah tiap kendaraan dalam format JSON, volumenya sangat besar, sebagian besar tidak pernah dibuka lagi. Rancang arsitektur (warehouse/lake/lakehouse/kombinasi) beserta alasan untuk masing-masing jenis data.
3. Kenapa skema OLTP biasanya normalized (banyak tabel kecil), sementara skema OLAP biasanya denormalized (sedikit tabel besar, sengaja ada duplikasi)? Kaitkan dengan trade-off kecepatan tulis vs kecepatan baca.
4. Jelaskan ke rekan kerja yang belum paham istilah "data swamp" — apa itu, kenapa bisa terjadi, dan bagaimana mencegahnya.

## Kunci Jawaban & Pembahasan

**1.** Database aplikasi adalah sistem OLTP, dioptimalkan untuk transaksi kecil-cepat-banyak (checkout, update cart). Laporan bulanan finance adalah beban kerja OLAP (scan besar, agregasi berat) yang dijalankan langsung di sistem OLTP — inilah **resource contention** yang dibahas di atas: query berat itu mengunci/memperlambat tabel yang sedang dipakai aplikasi live, persis di jam sibuk (9 pagi). Solusi: pisahkan beban kerja — replikasi/salin data ke data warehouse terpisah (via pipeline ETL/ELT), lalu arahkan laporan finance ke warehouse itu, bukan ke database production.

**2.** Data (a) — terstruktur, dipakai rutin untuk dashboard operasional yang butuh query cepat & konsisten — cocok masuk **data warehouse**. Data (b) — volume sangat besar, format semi-structured (JSON), jarang diakses, skema GPS record bisa berubah-ubah antar versi device — cocok masuk **data lake** (object storage murah, schema-on-read, tidak perlu didefinisikan strukturnya di depan). Kombinasi keduanya masuk akal di sini: pipeline bisa mengekstrak ringkasan/agregat dari data lake (b) untuk diisi ke warehouse kalau suatu saat memang dibutuhkan untuk dashboard. Ini juga skenario tipikal yang mengarah ke pilihan **lakehouse** kalau perusahaan mau satu platform saja untuk keduanya.

**3.** Normalized (OLTP): tiap fakta cuma disimpan **sekali** (mis. nama produk cuma ada di tabel `products`, bukan diulang di tiap baris order) — ini bikin `INSERT`/`UPDATE` cepat & konsisten (update nama produk cukup 1 baris, tidak perlu update ribuan baris order yang mengandung produk itu). Trade-off-nya: baca butuh `JOIN` banyak tabel. Denormalized (OLAP): data sengaja "diratakan"/diduplikasi (mis. nama produk ikut disalin ke tiap baris fact) supaya `SELECT`/agregasi tidak perlu `JOIN` berat — trade-off-nya: kalau ada update, harus diupdate di banyak tempat, dan storage lebih boros. Karena OLAP dominan baca (jarang update baris individual — data historis biasanya tidak berubah lagi), trade-off ini masuk akal; karena OLTP dominan tulis, trade-off sebaliknya yang masuk akal.

**4.** Data swamp adalah data lake yang kehilangan kegunaannya karena tidak punya tata kelola — data terus ditumpuk (log, file, hasil ekspor) tanpa standar penamaan, tanpa katalog yang menjelaskan isi/kualitas tiap dataset, dan tanpa ada yang tahu mana data yang masih valid/dipakai. Akibatnya, biarpun secara teknis semua data "ada", tidak ada yang percaya diri memakainya — sama seperti gudang barang yang penuh sesak tapi tidak ada label, jadi lebih cepat beli baru daripada cari barang yang sudah ada. Pencegahan: definisikan struktur folder/penamaan sejak awal, dokumentasikan tiap dataset (skema, sumber, kapan terakhir update — ini yang disebut **data catalog**, dibahas penuh di Minggu 5), dan terapkan proses onboarding data yang jelas (bukan sekadar "taruh file di sini").
