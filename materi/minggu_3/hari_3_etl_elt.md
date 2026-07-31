---
title: Hari 3 - ETL vs ELT
parent: Minggu 3 - Arsitektur Data & Big Data Pipeline
nav_order: 4
---

# Hari 3 — ETL vs ELT, Tools Overview

*Rabu, 2 jam. Konsep + pengenalan tools (belum hands-on — Airflow hands-on baru Sabtu–Minggu).*

## Tujuan Belajar

- [ ] Menjelaskan 3 tahap ETL (Extract, Transform, Load) dan ELT (Extract, Load, Transform)
- [ ] Menjelaskan **kenapa** urutan T dan L tertukar itu penting, bukan cuma soal penamaan
- [ ] Memetakan tools populer (Airflow, dbt, Fivetran/Airbyte) ke perannya masing-masing dalam pipeline
- [ ] Memilih ETL atau ELT untuk skenario yang diberikan, dengan alasan

## Untuk Instruktur: Mindset Shift

Ini bukan dua teknologi berbeda — **hurufnya sama** (Extract, Transform, Load), yang beda cuma **di mana dan kapan Transform terjadi**. Analogi yang efektif buat developer: bayangkan build pipeline CI/CD.

- **ETL** ≈ transform data di **staging server terpisah** sebelum deploy ke production — data yang masuk ke tujuan akhir sudah "matang".
- **ELT** ≈ deploy raw source langsung ke production dulu, transform terjadi **di tempat tujuan itu sendiri** (jalan di compute warehouse) setelah data sudah ada di sana.

Poin kunci yang harus ditekankan: ELT jadi populer bukan karena "lebih canggih", tapi karena **cloud data warehouse modern (Snowflake, BigQuery) punya compute yang sangat murah & elastis** — jadi masuk akal menaruh beban transform di sana, daripada di server ETL terpisah yang harus di-provision & di-maintain sendiri.

## Konsep & Sintaks

### ETL (Extract → Transform → Load)

```
Source (OLTP/API) --Extract--> Staging/Compute Terpisah --Transform--> Load --> Warehouse
```

Data ditransformasi **sebelum** masuk ke tempat penyimpanan akhir. Warehouse cuma menerima data yang sudah bersih & sudah dalam bentuk target (mis. sudah star schema).

**Kapan cocok:**
- Warehouse tujuan tidak punya compute kuat, atau biayanya mahal untuk beban transform berat
- Ada regulasi yang mengharuskan data sensitif dibersihkan/dimasking **sebelum** masuk sistem tertentu (mis. PII harus di-mask sebelum masuk warehouse yang bisa diakses banyak analyst — relevan dengan Data Governance Minggu 5)
- Pipeline batch tradisional, volume data medium, infrastruktur on-premise

### ELT (Extract → Load → Transform)

```
Source (OLTP/API) --Extract--> Load (raw) --> Warehouse/Lake --Transform (di dalam warehouse)--> Tabel Siap Pakai
```

Data mentah di-load **dulu** ke warehouse/lake apa adanya, transformasi terjadi **belakangan**, memakai compute power warehouse itu sendiri (biasanya lewat SQL).

**Kapan cocok:**
- Warehouse tujuan modern & elastis (Snowflake/BigQuery/Redshift) — compute murah, bisa scale sesuai kebutuhan
- Data mentah juga ingin disimpan (untuk audit, atau supaya transformasi bisa diulang/diubah tanpa extract ulang dari source)
- Tim lebih nyaman menulis transform dalam SQL (skill yang sudah dikuasai sejak Minggu 1) daripada kode pipeline terpisah

### Perbandingan

| | ETL | ELT |
|---|---|---|
| Urutan | Extract → Transform → Load | Extract → Load → Transform |
| Tempat transform | Sistem/server terpisah (compute engine ETL) | Di dalam warehouse/lake tujuan |
| Data mentah tersimpan? | Biasanya tidak (atau terpisah dari warehouse) | Ya, raw data ikut ter-load, jadi bisa transform ulang kapan saja |
| Skill utama transform | Kode (Python/Spark) atau tool ETL visual | SQL (dijalankan sebagai query di warehouse) |
| Fleksibilitas ubah logic | Perlu extract ulang / re-run pipeline dari source | Tinggal ubah query transform, replay dari data mentah yang sudah ada |
| Cocok untuk | Data sensitif yang harus diproses sebelum masuk sistem tertentu, infrastruktur legacy | Cloud warehouse modern, tim yang kuat di SQL, kebutuhan iterasi cepat |

**Penting untuk diluruskan**: ELT **tidak menghilangkan** kebutuhan transform — cuma memindahkan **di mana** dan **kapan** itu terjadi. Keduanya tetap butuh logic transformasi yang sama (cleaning, agregasi, star schema) — bedanya ELT menulis logic itu sebagai SQL yang jalan **setelah** data mendarat, ETL menjalankannya sebagai kode/job **sebelum** data mendarat.

### Peta Tools

| Tool | Peran | Analog di dunia software dev |
|---|---|---|
| **Airflow** | **Orkestrasi** — menjadwalkan & menjalankan urutan task (bisa Extract, Load, Transform, atau ketiganya), memantau kegagalan, retry | Seperti CI/CD orchestrator (GitHub Actions/Jenkins) — dia tidak menulis kodenya, dia yang menjalankan & mengurutkan |
| **dbt (data build tool)** | **Transform**, khusus pola ELT — kamu tulis SQL `SELECT`, dbt yang urus jadi `CREATE TABLE`/`VIEW`, tracking dependency antar model, testing | Seperti ORM/migration tool, tapi untuk transformasi data analitik, bukan skema aplikasi |
| **Fivetran / Airbyte** | **Extract + Load** terkelola (managed) — connector siap pakai ke ratusan sumber data (Postgres, Stripe, Salesforce, dst), otomatis sync ke warehouse | Seperti webhook/integration platform (Zapier) tapi khusus data pipeline |
| **Apache Spark** (dibahas penuh Hari 5) | **Transform** untuk volume besar — bisa dipakai gaya ETL (transform sebelum load) maupun sebagai bagian pipeline ELT | Seperti data processing engine — mirip menjalankan `.map()`/`.reduce()` tapi terdistribusi lintas mesin |

Airflow **bukan** tool Transform atau Extract itu sendiri — dia mengatur **kapan dan dalam urutan apa** task Extract/Transform/Load dijalankan (dan apa yang terjadi kalau salah satu gagal). Ini poin yang sering bikin bingung pemula: "kok Airflow tidak transform datanya sendiri?" — karena memang bukan itu tugasnya, sama seperti GitHub Actions tidak menulis kode aplikasinya sendiri, cuma menjalankan step yang kamu definisikan.

### Untuk Mini Project Minggu Ini: ETL atau ELT?

Pipeline yang akan dibangun di `latihan_pipeline_mini_project.md` memakai pola **ETL**: PySpark men-transform data **sebelum** di-load ke Postgres ("warehouse" lokal kita). Alasan pilih ETL di sini (bukan ELT) murni praktis untuk konteks belajar: Postgres lokal bukan cloud warehouse modern dengan compute elastis (beda dari Snowflake/BigQuery yang jadi alasan utama ELT populer), dan tujuan minggu ini memang untuk **berlatih Spark sebagai transformation engine**. Pola ELT (transform pakai dbt setelah data mendarat di warehouse) akan disinggung lagi saat masuk Cloud Data Warehouse di Minggu 6, saat toolingnya memang lebih relevan.

## Kesalahan Umum

1. **Mengira ELT berarti "tidak butuh transformasi sama sekali".** ELT tetap penuh transformasi — cuma dieksekusi belakangan, di dalam warehouse, biasanya sebagai SQL.
2. **Mengira Airflow adalah tool Transform.** Airflow itu orkestrator — dia menjalankan/menjadwalkan step, bukan menjalankan logic transformasinya sendiri (logic transform ada di script Python/Spark/SQL yang dipanggil Airflow sebagai task).
3. **Memilih ETL/ELT karena "yang lagi tren" tanpa lihat konteks infrastruktur.** Keputusan ini harus mempertimbangkan compute tujuan (elastis/tidak), kebutuhan regulasi (data harus dibersihkan sebelum masuk sistem tertentu?), dan skill tim (kuat SQL vs kuat kode pipeline).
4. **Menganggap harus pilih salah satu secara eksklusif untuk seluruh perusahaan.** Banyak organisasi pakai **keduanya** untuk pipeline berbeda — sumber data sensitif lewat ETL, sumber data umum lewat ELT ke warehouse modern. Ini bukan keputusan sekali untuk selamanya di semua kasus.

## Latihan

1. Sebuah rumah sakit perlu memindahkan data pasien dari sistem rekam medis ke data warehouse untuk analisis riset, tapi regulasi mewajibkan semua data identitas pasien (nama, NIK) di-mask/di-anonymize **sebelum** menyentuh warehouse yang bisa diakses tim riset lebih luas. ETL atau ELT? Jelaskan.
2. Startup e-commerce (skala kecil, mirip konteks portfolio kita) pakai BigQuery sebagai warehouse, dan timnya kuat di SQL tapi tidak banyak yang jago Python/Spark. Mereka ingin bisa cepat mengubah-ubah logic agregasi revenue tanpa harus re-run extract dari source tiap kali. ETL atau ELT? Tools apa yang relevan?
3. Jelaskan kenapa "ELT populer karena cloud warehouse murah" adalah alasan **infrastruktur**, bukan alasan bahwa ELT secara inheren "lebih baik" dari ETL.
4. Untuk pipeline mini project minggu ini (PySpark transform → load ke Postgres), gambarkan di mana posisi Extract, Transform, Load-nya, dan jelaskan kenapa ini pola ETL bukan ELT.

## Kunci Jawaban & Pembahasan

**1.** **ETL.** Regulasi mengharuskan transformasi (masking/anonymization) terjadi **sebelum** data masuk ke sistem yang diakses tim riset — ini persis definisi ETL (Transform sebelum Load). Kalau dipaksakan ELT (load data mentah termasuk NIK/nama dulu, baru mask belakangan di dalam warehouse), ada jendela waktu di mana data sensitif yang belum di-mask sudah ada di warehouse — berisiko secara compliance meski cuma sementara.

**2.** **ELT**, dengan **dbt** sebagai tool transform utama (tim kuat SQL, dbt memang dirancang supaya transformasi ditulis sebagai `SELECT` biasa) dan **Fivetran/Airbyte** untuk extract+load terkelola dari source ke BigQuery. Kebutuhan "cepat mengubah-ubah logic agregasi tanpa re-run extract" adalah keunggulan utama ELT — karena raw data sudah ada di warehouse, ubah logic tinggal ubah query dbt dan jalankan ulang transform-nya saja (compute BigQuery), tanpa perlu extract ulang dari source sama sekali.

**3.** Kalau warehouse tujuan **tidak** elastis/murah computenya (mis. Postgres on-premise dengan resource terbatas — persis kasus mini project kita), menjalankan transformasi berat **di dalam** warehouse itu (gaya ELT) justru bisa mengganggu beban kerja lain di sana, mirip masalah resource contention yang dibahas di Hari 1 untuk OLTP. Di konteks itu, ETL (transform di tempat terpisah, baru load hasil yang sudah matang) lebih masuk akal. Jadi keunggulan ELT itu **kondisional** — bergantung compute tujuan benar-benar murah & elastis, bukan keunggulan yang berlaku universal di semua infrastruktur.

**4.** **Extract**: baca file Online Retail II mentah (task Airflow `extract_raw_data`). **Transform**: PySpark job (`clean_transform.py` + `build_star_schema.py`) membersihkan & menyusun ulang data jadi bentuk star schema — ini terjadi **sebelum** data masuk Postgres, dijalankan sebagai job Spark terpisah, bukan sebagai query di dalam Postgres itu sendiri. **Load**: hasil Parquet yang sudah bersih & sudah berbentuk fact/dimension baru dimasukkan ke Postgres (task `load_to_warehouse`). Karena Transform terjadi sebelum Load dan di luar sistem tujuan (Postgres), ini pola **ETL**.
