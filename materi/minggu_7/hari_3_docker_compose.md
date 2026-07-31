---
title: Hari 3 - Docker Compose Mendalam
parent: Minggu 7 - Containerization
nav_order: 4
---

# Hari 3 — Docker Compose: Membedah `docker-compose.yml` yang Sudah Dipakai

*Rabu, 2 jam. Compose sudah dipakai sejak Minggu 3 (Airflow) dan Minggu 4 (+Kafka) — sesi ini membedah tiap bagiannya.*

## Tujuan Belajar

- [ ] Menjelaskan fungsi Docker Compose: mendefinisikan & menjalankan banyak container sebagai 1 kesatuan
- [ ] Membaca & menjelaskan tiap bagian `docker-compose.yml` yang sudah dipakai: `services`, `depends_on`, `environment`, `volumes`, `ports`, `build`
- [ ] Menjelaskan network default Compose dan bagaimana service saling terhubung lewat nama service sebagai hostname
- [ ] Menjelaskan kenapa `host.docker.internal` dibutuhkan untuk `pg-belajar` (Minggu 3) — sesuatu yang tidak akan dibutuhkan kalau `pg-belajar` ada di file Compose yang sama

## Untuk Instruktur

Ini kesempatan bagus untuk peserta membuka **file `docker-compose.yml` mereka sendiri** (bukan contoh instruktur) dan menjelaskan tiap barisnya sendiri, dipandu pertanyaan. Kalau ada baris yang tidak bisa dijelaskan, itu sinyal bagian yang perlu didalami — jauh lebih efektif daripada instruktur menjelaskan file yang tidak familiar.

## Konsep & Sintaks

### Kenapa Compose Dibutuhkan

`docker run` cocok untuk 1 container. Begitu kebutuhan naik jadi **banyak container yang saling terhubung** (Airflow scheduler + webserver + Postgres metadata + Redis + Kafka, seperti setup sejak Minggu 3-4), menjalankan & mengelola tiap `docker run` manual satu-satu jadi tidak praktis — perlu diingat urutan start, nama container, port, environment variable, network, dst tiap kali. **Docker Compose** menyelesaikan ini: 1 file YAML mendeklarasikan **semua** service dan hubungan antar mereka, dijalankan dengan 1 perintah (`docker compose up`).

### Anatomi `docker-compose.yml` yang Sudah Dipakai

Potongan relevan dari setup Airflow (Minggu 3) + Kafka (Minggu 4):

```yaml
services:
  airflow-webserver:
    build: .                              # <- pakai Dockerfile custom (hari_2_dockerfile.md), bukan image resmi apa adanya
    depends_on:
      - postgres
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
    volumes:
      - ./dags:/opt/airflow/dags
      - ./spark_jobs:/opt/airflow/spark_jobs
      - ./data:/opt/airflow/data
    ports:
      - "8080:8080"

  postgres:
    image: postgres:13                     # <- Postgres INTERNAL milik Airflow (metadata DB), BEDA dari pg-belajar
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow

  kafka:
    image: bitnami/kafka:3.7
    ports:
      - "9092:9092"
    environment:
      - KAFKA_CFG_NODE_ID=0
      - KAFKA_CFG_PROCESS_ROLES=controller,broker
```

| Field | Fungsi |
|---|---|
| `services` | Daftar semua container yang dikelola Compose ini — tiap key (`airflow-webserver`, `postgres`, `kafka`) jadi 1 service |
| `build: .` | Build image dari `Dockerfile` di direktori ini, bukan pakai image jadi dari registry |
| `image: postgres:13` | Alternatif `build` — langsung pakai image dari registry apa adanya |
| `depends_on` | Urutan **start** container (bukan urutan "siap dipakai" — lihat Kesalahan Umum #1) |
| `environment` | Environment variable yang di-inject ke dalam container saat jalan |
| `volumes` | Bind mount (kode/data dari host) atau named volume (dikelola Docker) — detail penuh di `hari_4_networking_volume.md` |
| `ports` | Format `"HOST:CONTAINER"` — memetakan port di container supaya bisa diakses dari mesin host (`8080:8080` artinya `localhost:8080` di laptop kamu terhubung ke port `8080` di dalam container) |

**Poin penting yang sering terlewat**: `postgres` di file Compose Airflow ini adalah database **metadata internal Airflow sendiri** (menyimpan status DAG run, task instance, dst) — **beda total** dari `pg-belajar` (Postgres warehouse tempat `fact_sales`/`dim_*` di-load, Minggu 1-3). Dua Postgres yang berbeda kepentingan, kebetulan sama-sama bernama mirip.

### Network: Kenapa Service Bisa Saling Panggil Pakai Nama

Compose otomatis membuat **1 network bersama** untuk semua service dalam file yang sama — di dalam network itu, tiap service bisa saling terhubung memakai **nama service-nya sebagai hostname** (Docker menyediakan DNS internal otomatis), tanpa perlu tahu IP container.

```
Network Compose "ecommerce-etl-pipeline_default"
┌─────────────────────────────────────────────────┐
│  airflow-webserver  <──────>  postgres            │
│         │                          │               │
│         └──────────>  kafka  <─────┘               │
└─────────────────────────────────────────────────┘
```

Ini sebabnya Airflow bisa terhubung ke Postgres **metadata**-nya lewat host `postgres` (bukan `localhost` atau IP) — nama service itu langsung "resolve" ke container yang benar, selama keduanya didefinisikan di file Compose yang **sama**.

### Kenapa `pg-belajar` Butuh `host.docker.internal` (Sambungan dari Minggu 3)

`pg-belajar` (warehouse) dijalankan terpisah lewat `docker run` biasa (Minggu 1) — **bukan** bagian dari `docker-compose.yml` Airflow. Karena itu, `pg-belajar` **tidak** ada di network Compose yang sama dengan `airflow-webserver`, sehingga tidak bisa dipanggil cukup pakai nama container-nya. `host.docker.internal` adalah hostname khusus yang disediakan Docker Desktop untuk mengakses **mesin host** dari dalam container — karena port `5432` `pg-belajar` sudah di-publish ke host, ini jadi jalan pintas untuk menjangkaunya.

**Kalau `pg-belajar` didefinisikan sebagai service di file Compose yang sama** (alternatif desain yang valid, tidak dilakukan sengaja di Minggu 3 untuk memisahkan konsep "warehouse yang dipakai sejak Minggu 1" dari "infrastruktur orkestrasi Minggu 3"), maka koneksinya bisa cukup pakai `postgresql://...@pg-belajar:5432/...` — tanpa `host.docker.internal` sama sekali, persis seperti `postgres` (metadata Airflow) di atas.

## Kesalahan Umum

1. **Mengira `depends_on` menunggu service "siap dipakai".** `depends_on` cuma menjamin urutan **container mulai berjalan (start)**, **bukan** menjamin aplikasi di dalamnya sudah **siap menerima koneksi**. Postgres container bisa "started" tapi proses database-nya masih inisialisasi beberapa detik — service lain yang `depends_on: [postgres]` bisa saja mencoba konek terlalu cepat dan gagal. Solusi produksi: `healthcheck` + `depends_on: condition: service_healthy` (di luar cakupan wajib minggu ini, tapi baik diketahui ada).
2. **Menaruh port yang sama di beberapa service.** `ports: "8080:8080"` di 2 service berbeda akan bentrok saat `docker compose up` — port di sisi **host** harus unik per service yang publish ke port yang sama itu.
3. **Lupa `docker compose down` setelah selesai, membiarkan resource menyala terus.** Beda dari `docker compose stop` (cuma stop, container & network masih ada), `docker compose down` benar-benar membersihkan container & network yang dibuat Compose — kebiasaan baik untuk laptop yang resource-nya terbatas.
4. **Mengira semua service di 1 file Compose otomatis "satu kesatuan" dengan container yang dijalankan `docker run` terpisah.** Seperti kasus `pg-belajar` di atas — container dari `docker run` biasa **tidak** otomatis masuk ke network Compose manapun, kecuali sengaja dihubungkan (`docker network connect`) atau diakses lewat `host.docker.internal`.

## Latihan

1. Buka `docker-compose.yml` di repo `ecommerce-etl-pipeline` kamu sendiri. Untuk tiap service di dalamnya, jelaskan: pakai `image` atau `build`? Port apa yang di-publish ke host, dan kenapa perlu?
2. Kalau kamu tambahkan service baru `redis` ke file Compose yang sama dengan `airflow-webserver`, host apa yang harus dipakai `airflow-webserver` untuk konek ke Redis itu — nama service, IP tertentu, atau `host.docker.internal`? Jelaskan alasannya.
3. Jelaskan kenapa `depends_on: [postgres]` di `airflow-webserver` **tidak cukup** untuk menjamin Airflow tidak akan pernah gagal konek ke Postgres saat startup — skenario konkret apa yang bisa membuatnya tetap gagal walau `depends_on` sudah dipasang?
4. `kafka` (Minggu 4) dan `pg-belajar` (Minggu 1) sama-sama dibutuhkan `streaming-demo/producer.py`/`consumer.py` (yang jalan dari **host**, bukan dari dalam container Compose manapun, sesuai `materi/minggu_4/latihan_dq_streaming_mini_project.md`). Kenapa `producer.py` bisa konek ke Kafka cukup pakai `localhost:9092` (bukan `host.docker.internal` atau nama service `kafka`)?

## Kunci Jawaban & Pembahasan

**1.** Jawaban bervariasi tergantung isi file masing-masing peserta, tapi pola yang diharapkan: `airflow-webserver`/`airflow-scheduler` pakai `build: .` (custom Dockerfile dengan Java+PySpark+GE), `postgres` (metadata) dan `kafka` pakai `image` langsung (tidak butuh modifikasi). Port yang di-publish ke host: `8080` (Airflow UI, diakses dari browser di mesin host) dan `9092` (Kafka broker, supaya `producer.py`/`consumer.py` yang jalan dari host venv bisa terhubung).

**2.** Pakai **nama service** (`redis`), bukan `host.docker.internal` atau IP — karena `redis` dan `airflow-webserver` didefinisikan di file Compose yang **sama**, keduanya otomatis berada di network Compose bersama, dan DNS internal Docker akan meresolusi `redis` ke container yang benar. `host.docker.internal` cuma dibutuhkan untuk kasus seperti `pg-belajar`, yang **berada di luar** file Compose ini (dijalankan `docker run` terpisah).

**3.** Skenario konkret: container `postgres` sudah berstatus "running" (proses container-nya jalan), tapi proses **PostgreSQL di dalamnya** masih dalam tahap inisialisasi database (beberapa detik pertama setelah start, terutama kalau ini pertama kali volume-nya dibuat). `depends_on` cuma menjamin Docker **memulai** container `postgres` lebih dulu sebelum `airflow-webserver` — begitu container `postgres` "started" (bukan berarti "siap menerima koneksi"), Docker langsung lanjut start `airflow-webserver`, yang mungkin langsung mencoba konek dalam hitungan milidetik, lebih cepat dari waktu Postgres benar-benar siap menerima koneksi. Ini kenapa `healthcheck` + `condition: service_healthy` (disinggung di Kesalahan Umum #1) jadi solusi yang lebih tepat untuk production, dibanding `depends_on` polos.

**4.** Karena `producer.py`/`consumer.py` dijalankan dari **host** (venv Python di laptop, bukan dari dalam container manapun) — dari sudut pandang host, port `9092` Kafka sudah di-publish langsung ke `localhost:9092` (lewat `ports: "9092:9092"` di Compose). Konsep hostname-service-sebagai-DNS (`kafka` sebagai nama) cuma berlaku **di dalam** network Docker, antar container — begitu kamu berada **di luar** Docker (di host), akses ke container manapun yang portnya di-publish selalu lewat `localhost:<port_host>`, sama sekali tidak relevan pakai `host.docker.internal` (itu untuk arah sebaliknya: dari **dalam** container mengakses **host**) atau nama service (itu untuk antar **container**).
