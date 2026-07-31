---
title: Hari 4 - Networking & Volume di Docker
parent: Minggu 7 - Containerization
nav_order: 5
---

# Hari 4 — Networking & Volume: Kenapa Data Bertahan (atau Hilang)

*Kamis, 2 jam. Menjawab tuntas 2 pertanyaan yang sengaja dibiarkan menggantung sejak Minggu 1-3: kenapa data Postgres bisa hilang, dan kenapa `host.docker.internal` dibutuhkan.*

## Tujuan Belajar

- [ ] Menjelaskan kenapa data di dalam container hilang saat container dihapus, dan bagaimana volume mencegah itu
- [ ] Membedakan bind mount vs named volume, dan kapan masing-masing dipakai
- [ ] Menjelaskan mekanisme network default Docker (bridge network) untuk komunikasi antar container
- [ ] Menerapkan volume ke service yang benar-benar butuh data persisten di repo sendiri

## Untuk Instruktur

Materi ini idealnya dibuka dengan **demo langsung**: jalankan container Postgres baru, `INSERT` 1 baris, `docker rm -f` container-nya, jalankan container baru dari image yang sama, lalu tunjukkan datanya **hilang**. Efeknya jauh lebih kuat dilihat langsung daripada dijelaskan sebagai teori — banyak peserta baru pertama kali menyadari bind mount `./data:/opt/airflow/data` yang sudah mereka pakai sejak Minggu 3 sebenarnya adalah jawaban dari masalah ini.

## Konsep & Sintaks

### Kenapa Data di Container Hilang

Filesystem container itu **terikat ke siklus hidup container**-nya — dibuat saat container dibuat, dihapus saat container dihapus (`docker rm`). Ini konsekuensi langsung dari konsep image vs container (`hari_1_container_vs_vm.md`): image adalah **read-only template**, dan container menambahkan **1 layer writable** tipis di atasnya untuk perubahan selama berjalan — layer writable ini yang lenyap begitu container-nya dihapus.

```
docker run postgres:16 --name test-db
  -> INSERT data           (ditulis ke layer writable container "test-db")
docker rm -f test-db       (layer writable ikut terhapus bersama container)
docker run postgres:16 --name test-db-2
  -> data tadi TIDAK ADA   (container baru = layer writable baru, kosong)
```

**Volume** menyelesaikan ini dengan menyimpan data **di luar** siklus hidup container — data tetap ada meski container yang memakainya dihapus, dan bisa "dipasang ulang" ke container baru.

### Bind Mount vs Named Volume

Dua cara menghubungkan data eksternal ke dalam container, sudah dipakai (tanpa dijelaskan bedanya) sejak Minggu 3:

```yaml
volumes:
  - ./dags:/opt/airflow/dags        # <- BIND MOUNT: path spesifik di HOST
  - pgdata:/var/lib/postgresql/data # <- NAMED VOLUME: dikelola Docker, kamu tidak perlu tahu lokasinya
```

| | Bind Mount | Named Volume |
|---|---|---|
| Lokasi data | Path eksplisit di **host** yang kamu tentukan (`./dags`) | Dikelola Docker, tersimpan di area internal Docker (kamu tidak perlu tahu lokasi persisnya) |
| Cocok untuk | **Development** — kode di host langsung ter-reflect ke container tanpa rebuild image | **Production/data persisten** — database, state yang tidak perlu diedit manual dari host |
| Contoh di repo sendiri | `./dags:/opt/airflow/dags`, `./spark_jobs:/opt/airflow/spark_jobs` (Minggu 3) — edit file `.py` di host, langsung "terlihat" di container tanpa rebuild | Kalau `pg-belajar` dikonfigurasi dengan volume (bukan cuma `docker run` polos), data warehouse akan bertahan lintas restart container |
| Portabilitas | Terikat struktur folder host — kurang portable antar mesin | Lebih portable, dikelola sepenuhnya oleh Docker |

**Kenapa `dags`/`spark_jobs` dipasang sebagai bind mount, bukan `COPY` permanen ke image (Minggu 3)**: supaya perubahan kode selama development **langsung terlihat** tanpa perlu `docker build` ulang tiap kali edit 1 baris — trade-off kecepatan iterasi development. Untuk image yang benar-benar akan **di-deploy** (bukan cuma development lokal), kode biasanya di-`COPY` permanen ke dalam image saat build (pola yang akan dipakai di `latihan_containerization_mini_project.md` untuk `spark_jobs/Dockerfile` dan `streaming-demo/Dockerfile`) — image jadi self-contained, tidak bergantung folder host yang mungkin tidak ada di server production.

### Docker Networking: Bridge Network

Default, Docker membuat **bridge network** untuk container yang perlu saling terhubung dalam 1 host — ini yang sudah dibahas mekanismenya di `hari_3_docker_compose.md` (Compose otomatis membuat 1 bridge network per project, service saling terhubung lewat nama).

```
Tanpa network khusus (docker run polos)     Dengan Compose (1 bridge network otomatis)
┌──────────┐   ┌──────────┐                 ┌────────────────────────────┐
│ container │   │ container │                 │  bridge network            │
│    A      │   │    B      │                 │  ┌────────┐  ┌────────┐   │
└──────────┘   └──────────┘                 │  │ svc-A  │──│ svc-B  │   │
  Tidak saling kenal secara default            │  └────────┘  └────────┘   │
  (perlu docker network connect manual)        └────────────────────────────┘
                                                 Saling kenal otomatis lewat nama service
```

**Ini yang menjelaskan kenapa `pg-belajar` (Minggu 1, `docker run` biasa) butuh `host.docker.internal`**: dia tidak berada di bridge network manapun yang sama dengan container Airflow — keduanya "tidak saling kenal" secara default, beda dari service-service di dalam 1 file Compose yang sama yang otomatis satu network.

## Kesalahan Umum

1. **Menjalankan database (Postgres, dst) tanpa volume sama sekali untuk data yang **seharusnya** persisten.** `pg-belajar` sejak Minggu 1 sengaja tanpa volume (untuk kesederhanaan setup awal roadmap) — konsekuensinya: kalau container itu pernah dihapus (bukan cuma di-stop), **seluruh** data warehouse yang sudah di-load hilang dan harus dijalankan ulang dari `load_to_warehouse`. Ini trade-off yang layak disadari, bukan cuma kebetulan tidak pernah terjadi.
2. **Mengira bind mount dan named volume interchangeable begitu saja.** Bind mount butuh path host yang **konsisten** ada di semua mesin yang menjalankan compose file itu — kalau file Compose dipindah ke mesin lain dengan struktur folder beda, bind mount bisa gagal/salah lokasi. Named volume tidak punya masalah ini karena dikelola sepenuhnya oleh Docker.
3. **Mengira container yang berbeda network otomatis bisa saling `ping` pakai nama.** Cuma container dalam **network Docker yang sama** yang bisa saling resolve nama sebagai hostname — dua `docker compose up` terpisah (dari 2 file/folder Compose berbeda) menghasilkan **2 network berbeda**, meski sama-sama jalan di 1 mesin yang sama.
4. **Menghapus volume tanpa sadar lewat `docker compose down -v` atau `docker system prune --volumes`.** Flag `-v`/`--volumes` menghapus **juga** data yang tersimpan di named volume — beda dari `docker compose down` biasa (cuma hapus container & network, volume tetap ada). Kebiasaan baik: selalu baca ulang flag sebelum menjalankan perintah pembersihan.

## Latihan

1. Kalau kamu jalankan `docker compose down` (tanpa `-v`) di setup Airflow (Minggu 3), lalu `docker compose up` lagi — apakah DAG history & task log yang lama masih ada? Jelaskan berdasarkan apakah Postgres metadata Airflow dikonfigurasi pakai volume atau tidak (cek `docker-compose.yml` kamu sendiri).
2. Rancang: kalau kamu ingin `pg-belajar` (warehouse) **tetap** menyimpan data walau container-nya dihapus & dibuat ulang, volume seperti apa yang perlu ditambahkan ke perintah `docker run`-nya? Tulis contoh perintahnya.
3. Jelaskan ke rekan kerja yang baru belajar Docker: kenapa `./spark_jobs:/opt/airflow/spark_jobs` (bind mount) dipilih untuk development, tapi untuk image yang di-deploy ke server production nanti (`latihan_containerization_mini_project.md`), kode `spark_jobs/` sebaiknya di-`COPY` permanen ke dalam image, bukan tetap mengandalkan bind mount.
4. `kafka` dan `airflow-webserver` berada di file `docker-compose.yml` yang sama sejak Minggu 4. Tanpa menjalankan apapun, prediksi: apakah `airflow-webserver` bisa konek ke `kafka` cukup pakai hostname `kafka` (tanpa `host.docker.internal`)? Jelaskan alasannya berdasarkan konsep bridge network di atas.

## Kunci Jawaban & Pembahasan

**1.** Jawabannya tergantung isi `docker-compose.yml` masing-masing peserta (biasanya file Compose resmi Airflow **memang** sudah menyertakan named volume untuk Postgres metadata secara default) — kalau ada baris seperti `postgres-db-volume:/var/lib/postgresql/data` di `volumes:` service `postgres`, maka DAG history/task log **tetap ada** setelah `down` lalu `up` lagi (data tersimpan di named volume, terpisah dari siklus hidup container). Kalau **tidak** ada volume sama sekali untuk service `postgres`, semua history akan **hilang** — poin ini bagus dicek langsung di file masing-masing, bukan diasumsikan.

**2.**
```bash
docker run -d --name pg-belajar \
  -e POSTGRES_PASSWORD=belajar \
  -p 5432:5432 \
  -v pg-belajar-data:/var/lib/postgresql/data \
  postgres:16
```
`-v pg-belajar-data:/var/lib/postgresql/data` membuat/memakai **named volume** bernama `pg-belajar-data`, dipasang ke direktori tempat Postgres menyimpan data filesystem-nya di dalam container (`/var/lib/postgresql/data`, lokasi standar image resmi Postgres). Selama volume `pg-belajar-data` tidak dihapus eksplisit, container `pg-belajar` bisa dihapus & dibuat ulang dengan perintah ini kapan saja tanpa kehilangan data — cukup pasang ulang volume yang sama.

**3.** Bind mount cocok untuk **development** karena mengoptimalkan **kecepatan iterasi**: edit `clean_transform.py` di editor, langsung tersedia di container tanpa `docker build` ulang (yang bisa makan waktu tiap kali) — trade-off yang masuk akal saat kode masih sering berubah. Untuk image yang **di-deploy** (dijalankan di server lain, bukan laptop sendiri), bind mount jadi masalah: server production tidak (dan seharusnya tidak) punya struktur folder host yang identik dengan laptop development, dan kamu ingin image itu **self-contained** — bisa dijalankan di mesin manapun tanpa bergantung file eksternal yang harus disiapkan manual dulu. `COPY` kode ke dalam image saat build memastikan image itu portable: `docker run image-spark-jobs` jalan identik di laptop manapun atau server manapun, tanpa syarat tambahan.

**4.** **Ya**, bisa — karena `kafka` dan `airflow-webserver` didefinisikan di `docker-compose.yml` yang **sama**, keduanya otomatis masuk ke bridge network Compose yang sama, dan Docker menyediakan DNS internal yang meresolusi nama service (`kafka`) ke IP container yang benar di dalam network itu. Ini beda dari kasus `pg-belajar`, yang sengaja **tidak** didefinisikan di file Compose yang sama (`docker run` terpisah, Minggu 1) — makanya butuh `host.docker.internal` sebagai jalan pintas lewat host, bukan lewat DNS internal bridge network.
