---
title: Overview
parent: Minggu 4 - Data Quality, Orchestration & Streaming
nav_order: 1
---

# Modul Minggu 4 — Data Quality, Orchestration Lanjutan, Streaming Intro (untuk Software Developer)

> Pendamping `minggu_4.md` (jadwal & outline). File ini konten pengajaran lengkap: penjelasan konsep, analogi kode, contoh, latihan, kunci jawaban. Lihat juga `materi/minggu_3/` — minggu ini adalah **upgrade langsung** dari pipeline yang dibangun di sana, bukan project baru.

## Shift Minggu Ini: dari "Jalan" ke "Bisa Dipercaya"

Pipeline Minggu 3 sudah **jalan** — extract, transform, load, terjadwal otomatis. Tapi "jalan" dan "bisa dipercaya" itu dua hal berbeda. Pipeline Minggu 3 akan tetap "sukses" (task hijau semua) walaupun datanya diam-diam rusak — misalnya kalau suatu hari file sumber punya jutaan baris `Customer ID` kosong, atau harga negatif menyelinap masuk. Task Airflow tidak tahu bedanya "data valid" dan "data yang kebetulan tidak bikin script error", karena minggu lalu belum ada lapisan yang benar-benar memeriksa **kualitas isi datanya**.

Itulah fokus utama minggu ini: menambahkan lapisan yang secara eksplisit mengecek "apakah data ini masuk akal?" **sebelum** dipakai orang lain, dan membuat pipeline **memberi tahu** kalau ada yang salah, bukan diam-diam lanjut jalan. Tiga potongan besar minggu ini:

1. **Data Quality (Senin–Selasa, Great Expectations)** — mendefinisikan & menegakkan "apa artinya data ini valid", terintegrasi jadi gerbang **sebelum** load ke warehouse (*fail-fast*).
2. **Airflow Lanjutan (Rabu)** — membuat pipeline yang gagal itu **kelihatan** (alerting) dan **tangguh** (retry, sensor), bukan cuma "jalan atau tidak jalan" secara diam-diam.
3. **Streaming Intro (Kamis–Minggu, Kafka)** — pengantar dunia di luar batch: bagaimana kalau data harus diproses **saat itu juga**, bukan menunggu jadwal harian (sambungan langsung dari `materi/minggu_3/hari_4_batch_stream.md`).

## Kenapa Bobot Materinya Begini

- **Senin–Selasa** dalamnya di Data Quality — ini yang paling langsung dipakai di mini project (Tahap 1, bobot waktu terbesar: 3 dari 8 jam) dan jadi fondasi literal untuk Data Governance Minggu 5.
- **Rabu** satu hari penuh untuk Airflow lanjutan — bukan tool baru, tapi kedalaman baru dari tool yang sudah dikenal.
- **Kamis–Jumat** Kafka & CDC level **konsep**, sengaja belum implementasi penuh — mengikuti pola yang sama dengan Spark/Airflow di Minggu 3 (konsep dulu Senin–Kamis, hands-on di akhir pekan).
- **Sabtu–Minggu** hands-on: Sabtu upgrade pipeline batch yang sudah ada, Minggu bikin demo streaming **terpisah** (folder `streaming-demo/`, sengaja tidak dicampur ke pipeline utama — dibahas alasannya di `latihan_dq_streaming_mini_project.md`).

## Setup Environment

```bash
# tambahan dependency di venv yang sama dari Minggu 1-3
pip install great_expectations==0.18.* kafka-python==2.0.*

# Kafka lokal via docker-compose (ditambahkan ke docker-compose.yml Minggu 3, KRaft mode — tanpa Zookeeper)
# detail service ada di latihan_dq_streaming_mini_project.md
```

- **Great Expectations (GE)** dipasang di **venv yang sama** yang juga dipakai membangun image Airflow kustom (`Dockerfile` dari `materi/minggu_3/latihan_pipeline_mini_project.md`) — task GE nanti jalan **di dalam** container Airflow, jadi `great_expectations` perlu ditambahkan ke `Dockerfile` itu juga (detail di `latihan_dq_streaming_mini_project.md`).
- **kafka-python** dipilih (bukan `confluent-kafka`) karena pure-Python, tidak butuh library native (`librdkafka`) yang bisa merepotkan instalasi lintas OS — cukup untuk tujuan belajar konsep producer/consumer, walau `confluent-kafka` lebih umum dipakai di production karena performanya lebih baik.

## Dataset & Repo yang Dipakai Minggu Ini

Tidak ada dataset baru — minggu ini **seluruhnya** bekerja di atas output pipeline Minggu 3: tabel `fact_sales`/`dim_customer`/`dim_product`/`dim_date` (hasil `spark_jobs/build_star_schema.py`) dan repo `ecommerce-etl-pipeline` yang sama. Kalau pipeline Minggu 3 belum jalan sukses minimal 1 kali, selesaikan itu dulu sebelum lanjut — semua yang dibahas minggu ini menumpuk di atasnya, bukan berdiri sendiri.

## Struktur Modul

| File | Sesuai Jadwal `minggu_4.md` | Topik |
|---|---|---|
| [`hari_1_data_quality_dimensions.md`](hari_1_data_quality_dimensions.md) | Senin, 2 jam | 5 dimensi data quality: completeness, uniqueness, validity, consistency, timeliness |
| [`hari_2_great_expectations.md`](hari_2_great_expectations.md) | Selasa, 2 jam | Pengantar Great Expectations: Expectation, Suite, Validator, Checkpoint |
| [`hari_3_airflow_lanjutan.md`](hari_3_airflow_lanjutan.md) | Rabu, 2 jam | Sensors, dependency kompleks/branching, retry, alerting, XComs |
| [`hari_4_pengantar_kafka.md`](hari_4_pengantar_kafka.md) | Kamis, 2 jam | Konsep Kafka: topic, broker, producer, consumer, partition, offset |
| [`hari_5_incremental_cdc.md`](hari_5_incremental_cdc.md) | Jumat, 2 jam | Incremental load vs full load, Change Data Capture (CDC) |
| [`latihan_dq_streaming_mini_project.md`](latihan_dq_streaming_mini_project.md) | Sabtu (4 jam) + Minggu (4 jam) | Hands-on: integrasi Great Expectations ke DAG, alerting, demo Kafka producer/consumer |

Struktur tiap file `hari_X` sama dengan minggu-minggu sebelumnya: Tujuan Belajar → Untuk Instruktur → Konsep & Sintaks → Contoh → Kesalahan Umum → Latihan → Kunci Jawaban.

## Catatan Cara Mengajar

- **Tekankan bedanya "task sukses" dan "data benar"** sejak kalimat pertama Senin — ini kesalahpahaman paling umum yang jadi motivasi seluruh minggu ini. Pipeline yang tidak pernah gagal itu **bukan** berarti data selalu benar — bisa jadi cuma berarti belum ada yang mengecek.
- **Bahas `fail-fast` sebagai keputusan desain sadar**, bukan default: DAG Minggu 4 sengaja meletakkan `data_quality_check` **sebelum** `load` (beda urutan dari Minggu 3 yang taruh check setelah load) — supaya data buruk tidak pernah sampai menyentuh warehouse sama sekali, bukan sekadar terdeteksi setelah telanjur masuk.
- **Kafka boleh terasa "berat" secara konsep** dibanding Spark/Airflow — wajar, karena ini pengantar ke paradigma yang sepenuhnya beda (event-driven vs terjadwal). Jangan buru-buru ke implementasi sebelum konsep dasar (topic/partition/offset/consumer group) benar-benar dipahami secara mental, karena tanpa itu error producer/consumer di akhir pekan akan sangat membingungkan.
- Total waktu: 5 hari × 2 jam + Sabtu 4 jam + Minggu 4 jam = 18 jam, sesuai `minggu_4.md`.
