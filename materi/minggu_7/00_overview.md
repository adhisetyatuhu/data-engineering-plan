---
title: Overview
parent: Minggu 7 - Containerization
nav_order: 1
---

# Modul Minggu 7 — Containerization (untuk Software Developer)

> Pendamping `minggu_7.md` (jadwal & outline). File ini konten pengajaran lengkap: penjelasan konsep, analogi kode, contoh, latihan, kunci jawaban. Lihat juga `materi/minggu_3/` sampai `materi/minggu_6/` — minggu ini tidak menambah tool baru, tapi membongkar tool yang **sudah dipakai sejak Minggu 1** (Docker) untuk dipahami jauh lebih dalam.

## Shift Minggu Ini: dari Pemakai Image ke Pembuat Image

Sejak Minggu 1, Docker sudah dipakai terus-menerus — `postgres:16` (Minggu 1), `apache/airflow:2.9.3` (Minggu 3), `bitnami/kafka:3.7` (Minggu 4) — tapi selalu sebagai **image resmi orang lain**, dipakai apa adanya atau di-extend sedikit lewat `Dockerfile` (menambah Java + `pyspark` ke image Airflow, Minggu 3). Minggu ini membalik posisinya: kode **sendiri** (`spark_jobs/`, `streaming-demo/`) dikemas jadi image sendiri, dari nol — bukan lagi cuma menambah lapisan tipis di atas image orang lain, tapi memahami penuh apa yang terjadi di dalam sebuah image.

Ini juga jawaban langsung untuk pertanyaan yang sengaja **belum dijawab** sejak Minggu 3: *"kenapa Airflow dijalankan lewat Docker, bukan install langsung?"* (`materi/minggu_3/00_overview.md` cuma menyebut singkat "jembatan pemanasan ke Minggu 7") — minggu ini jembatan itu dilewati penuh.

## Kenapa Bobot Materinya Begini

- **Senin–Selasa**: fondasi (container vs VM, Dockerfile) — wajib dipahami sebelum menyentuh Compose/K8s, karena keduanya cuma "cara menjalankan banyak container sekaligus" — kalau dasar 1 container saja belum jelas, Compose/K8s jadi hafalan perintah tanpa pemahaman.
- **Rabu**: Docker Compose — bukan tool baru (sudah dipakai sejak Minggu 3), tapi baru sekarang dibedah tiap barisnya.
- **Kamis**: networking & volume — pertanyaan "kenapa harus `host.docker.internal`?" (Minggu 3) dan "kenapa data hilang kalau container dihapus?" akhirnya dijawab tuntas di sini.
- **Jumat**: Kubernetes level konsep + hands-on ringan — sengaja **tidak** all-in ke K8s (deploy 1 komponen saja di mini project), karena tujuan minggu ini adalah paham **kenapa** K8s dibutuhkan, bukan mahir K8s (topik yang bisa jadi spesialisasi tersendiri).
- **Sabtu–Minggu**: hands-on penuh — containerize kode sendiri, lalu deploy 1 komponen ke Kubernetes lokal.

## Setup Environment

```bash
# Docker Desktop (Mac/Windows) sudah terpasang sejak Minggu 1 -- pastikan versi terbaru
docker --version
docker compose version

# kubectl (CLI untuk kontrol Kubernetes)
brew install kubectl

# minikube (Kubernetes 1-node lokal, paling umum dipakai untuk belajar)
brew install minikube
minikube start --driver=docker
```

- **minikube vs kind**: keduanya menjalankan Kubernetes lokal untuk belajar. `minikube` dipakai di modul ini karena punya `minikube dashboard` (UI visual, membantu peserta baru "melihat" apa yang terjadi) dan `driver=docker` (jalan di atas Docker yang sudah terpasang, tidak perlu VM terpisah). `kind` (Kubernetes IN Docker) valid juga kalau lebih familiar — semua perintah `kubectl` di modul ini sama persis, cuma cara start cluster-nya beda.
- Tidak ada dependency Python baru minggu ini — semua materi soal cara **mengemas & menjalankan** kode yang sudah ada, bukan soal menulis kode data baru.

## Dataset & Repo yang Dipakai Minggu Ini

Tidak ada dataset baru. Minggu ini membungkus ulang kode yang sudah ada di repo `ecommerce-etl-pipeline`: `spark_jobs/` (Minggu 3) dan `streaming-demo/` (Minggu 4) dikemas jadi image Docker sendiri, lalu 1 komponen (`streaming-demo/consumer.py`) di-deploy ke Kubernetes lokal.

## Struktur Modul

| File | Sesuai Jadwal `minggu_7.md` | Topik |
|---|---|---|
| [`hari_1_container_vs_vm.md`](hari_1_container_vs_vm.md) | Senin, 2 jam | Container vs VM, image vs container, Docker architecture |
| [`hari_2_dockerfile.md`](hari_2_dockerfile.md) | Selasa, 2 jam | Dockerfile: instruksi dasar, layer caching, best practice |
| [`hari_3_docker_compose.md`](hari_3_docker_compose.md) | Rabu, 2 jam | Docker Compose: membedah `docker-compose.yml` yang sudah dipakai sejak Minggu 3 |
| [`hari_4_networking_volume.md`](hari_4_networking_volume.md) | Kamis, 2 jam | Networking & volume: bind mount vs volume, komunikasi antar container |
| [`hari_5_kubernetes_intro.md`](hari_5_kubernetes_intro.md) | Jumat, 2 jam | Pengantar Kubernetes: pod, deployment, service, kenapa butuh orchestration |
| [`latihan_containerization_mini_project.md`](latihan_containerization_mini_project.md) | Sabtu (4 jam) + Minggu (4 jam) | Hands-on: custom Dockerfile, update Compose, deploy ke Kubernetes lokal |

Struktur tiap file `hari_X` sama dengan minggu-minggu sebelumnya: Tujuan Belajar → Untuk Instruktur → Konsep & Sintaks → Contoh → Kesalahan Umum → Latihan → Kunci Jawaban.

## Catatan Cara Mengajar

- **Pakai contoh nyata dari repo sendiri, bukan contoh generik.** Peserta sudah punya `Dockerfile` (Minggu 3) dan `docker-compose.yml` (Minggu 3-4) yang benar-benar mereka pakai — bedah **file itu sendiri** baris per baris, jauh lebih efektif daripada contoh `hello-world` abstrak.
- **Jangan buru-buru ke Kubernetes production-grade.** Minggu ini cuma level "paham konsep + bisa deploy 1 komponen sederhana lokal" — Helm, Ingress, StatefulSet, autoscaling adalah topik lanjutan di luar cakupan roadmap 8 minggu ini, sengaja tidak disinggung supaya tidak membingungkan peserta baru.
- **Tekankan bahwa Compose dan Kubernetes menjawab masalah berbeda.** Compose = "jalankan beberapa container di **1 mesin**", Kubernetes = "jalankan & kelola container di **banyak mesin**, dengan self-healing & scaling otomatis". Keduanya tidak saling menggantikan sepenuhnya untuk semua kasus — pilihan tergantung skala kebutuhan.
- Total waktu: 5 hari × 2 jam + Sabtu 4 jam + Minggu 4 jam = 18 jam, sesuai `minggu_7.md`.
