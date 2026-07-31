---
title: Overview
parent: Minggu 8 - NoSQL & Storage Strategy (Capstone)
nav_order: 1
---

# Modul Minggu 8 — NoSQL & Storage Strategy (Capstone) untuk Software Developer

> Pendamping `minggu_8.md` (jadwal & outline). File ini konten pengajaran lengkap: penjelasan konsep, analogi kode, contoh, latihan, kunci jawaban. Ini **modul terakhir** roadmap 8 minggu — lihat `materi/minggu_1/` sampai `materi/minggu_7/` untuk seluruh perjalanan yang dirangkum di sini.

## Shift Minggu Ini: dari "1 Database untuk Semua" ke Polyglot Persistence

Sejak Minggu 1, **satu** database (`pg-belajar`, PostgreSQL) dipakai untuk hampir semua kebutuhan — dari tabel mentah, star schema, sampai (opsional) target BigQuery di Minggu 6. Ini bukan kebetulan: PostgreSQL memang **cukup baik di banyak hal sekaligus** (general purpose), dan untuk skala roadmap ini, memisahkan storage lebih awal cuma akan menambah kompleksitas tanpa manfaat nyata. Minggu ini akhirnya memperkenalkan alasan **kapan** satu database generik tidak lagi cukup — bukan karena PostgreSQL "kurang bagus", tapi karena beberapa kebutuhan (skema fleksibel per kategori produk, caching super cepat untuk query yang sama berulang-ulang) punya tool yang **secara spesifik dirancang** untuk itu, dan akan mengungguli general-purpose database untuk kasus itu.

Prinsip yang dipelajari minggu ini disebut **polyglot persistence** — sistem data modern jarang cuma pakai 1 jenis database, tapi kombinasi yang **masing-masing dipilih sesuai karakteristik data & pola aksesnya**. Ini bukan soal mengoleksi tool sebanyak mungkin, tapi soal membuat keputusan sadar: data X cocok di Y karena alasan Z — kerangka berpikir ini yang jadi output paling berharga minggu ini (`STORAGE_STRATEGY.md`), lebih penting daripada hafal syntax MongoDB/Redis itu sendiri.

## Kenapa Bobot Materinya Begini

- **Senin–Selasa**: fondasi konsep (kenapa NoSQL muncul, CAP theorem, tipe-tipe NoSQL) — supaya peserta paham **kategori masalah** yang dijawab NoSQL sebelum masuk ke tool spesifik.
- **Rabu**: MongoDB mendalam — document model, embed vs reference, langsung dipakai Sabtu untuk migrasi `dim_product`.
- **Kamis**: Redis — key-value & caching, langsung dipakai Minggu untuk caching layer.
- **Jumat**: decision framework — sintesis dari seluruh minggu (dan sebenarnya seluruh roadmap 8 minggu), jawaban atas pertanyaan "kapan pakai apa" yang sudah implisit muncul sejak Minggu 3 (data warehouse vs data lake) dan Minggu 6 (cloud data warehouse vs Postgres lokal).
- **Sabtu–Minggu**: hands-on penuh + penutup portfolio — `STORAGE_STRATEGY.md` dan `README.md` final yang merangkum seluruh perjalanan 8 minggu.

## Setup Environment

```bash
docker run -d --name mongo-belajar -p 27017:27017 mongo:7
docker run -d --name redis-belajar -p 6379:6379 redis:7

pip install pymongo redis
```

- **MongoDB & Redis via Docker**, konsisten dengan pola sejak Minggu 1 (Postgres) dan Minggu 4 (Kafka) — hindari install manual yang rawan konflik versi sistem.
- Kedua container ini **terpisah** dari `docker-compose.yml` yang sudah dipakai sejak Minggu 3 (`docker run` biasa, seperti `pg-belajar`) — kalau ingin diintegrasikan penuh ke pipeline Airflow nanti, ingat kembali `hari_3_docker_compose.md`/`hari_4_networking_volume.md` (Minggu 7): tambahkan sebagai service di file Compose yang sama supaya bisa saling terhubung lewat nama service, bukan `host.docker.internal`.

## Dataset & Repo yang Dipakai Minggu Ini

Tidak ada dataset baru. `dim_product` (hasil `build_star_schema.py`, Minggu 3) dimigrasikan ke MongoDB sebagai document; query "mahal" yang sudah dikenal sejak Minggu 1-2 (top produk, RFM) dijadikan target caching Redis. Repo tetap `ecommerce-etl-pipeline` — ini adalah **commit terakhir** yang melengkapi seluruh arsitektur 8 minggu.

## Struktur Modul

| File | Sesuai Jadwal `minggu_8.md` | Topik |
|---|---|---|
| [`hari_1_konsep_nosql.md`](hari_1_konsep_nosql.md) | Senin, 2 jam | Kenapa NoSQL muncul, RDBMS vs NoSQL, CAP theorem |
| [`hari_2_tipe_nosql.md`](hari_2_tipe_nosql.md) | Selasa, 2 jam | Key-value, document, column-family, graph |
| [`hari_3_mongodb.md`](hari_3_mongodb.md) | Rabu, 2 jam | MongoDB: schema design, embedding vs referencing |
| [`hari_4_redis.md`](hari_4_redis.md) | Kamis, 2 jam | Redis: caching, session store, rate limiting |
| [`hari_5_storage_strategy.md`](hari_5_storage_strategy.md) | Jumat, 2 jam | Decision framework: SQL vs NoSQL vs data lake vs data warehouse |
| [`latihan_polyglot_storage_mini_project.md`](latihan_polyglot_storage_mini_project.md) | Sabtu (4 jam) + Minggu (4 jam) | Hands-on: migrasi ke MongoDB, caching Redis, `STORAGE_STRATEGY.md`, README final |

Struktur tiap file `hari_X` sama dengan minggu-minggu sebelumnya: Tujuan Belajar → Untuk Instruktur → Konsep & Sintaks → Contoh → Kesalahan Umum → Latihan → Kunci Jawaban.

## Catatan Cara Mengajar

- **Jangan biarkan minggu ini terasa seperti "tool baru untuk tool baru".** Tekankan terus: setiap tool baru (MongoDB, Redis) dipilih untuk menjawab **keterbatasan spesifik** yang sudah dialami sendiri sejak Minggu 1-2 (skema `dim_product` yang kaku untuk atribut bervariasi, query RFM yang mahal dijalankan berulang) — bukan dipelajari karena "harus tahu semua database".
- **Minggu 8 adalah sintesis, bukan sekadar penutup.** Dorong peserta menoleh balik ke keputusan-keputusan yang sudah diambil sejak Minggu 1 (kenapa Postgres, kenapa star schema, kenapa Parquet, kenapa BigQuery) dan menjelaskan semuanya dalam **1 kerangka berpikir polyglot** yang koheren — ini yang membedakan portfolio yang "menunjukkan proses berpikir" dari sekadar "kumpulan tool yang pernah dipakai".
- **Jujur soal skala.** Untuk data seukuran Online Retail II, manfaat performa MongoDB/Redis dibanding Postgres kemungkinan **tidak dramatis** — sama seperti catatan jujur PySpark vs Pandas (Minggu 3) dan BigQuery vs Postgres lokal (Minggu 6). Nilai pembelajarannya ada di **pemahaman kapan** tool ini unggul, bukan di angka benchmark mini project itu sendiri.
- Total waktu: 5 hari × 2 jam + Sabtu 4 jam + Minggu 4 jam = 18 jam, sesuai `minggu_8.md`.
