# Minggu 7 — Containerization

## Breakdown Harian (±15-20 jam)

| Hari | Jam | Materi |
|---|---|---|
| Senin | 2 jam | Konsep dasar container: perbedaan container vs VM, image vs container, Docker architecture |
| Selasa | 2 jam | Dockerfile: instruksi dasar (FROM, COPY, RUN, CMD, ENTRYPOINT), best practice layer caching |
| Rabu | 2 jam | Docker Compose: menjalankan multi-container (sudah dipakai sejak Minggu 3-4, sekarang pahami lebih dalam) |
| Kamis | 2 jam | Networking & volume di Docker: bind mount vs volume, container networking antar service |
| Jumat | 2 jam | Pengantar Kubernetes: konsep pod, deployment, service, konsep orchestration |
| Sabtu | 4 jam | Hands-on: containerize Spark job & script Python sendiri (custom Dockerfile) |
| Minggu | 4 jam | Hands-on: setup Kubernetes lokal (minikube/kind), deploy 1 aplikasi sederhana |

## Detail Topik
1. **Container vs VM** — container share OS kernel host, image = blueprint, container = instance yang jalan
2. **Dockerfile** — multi-stage build, minimize image size, layer caching; latihan bikin Dockerfile untuk PySpark job sendiri
3. **Docker Compose** — review ulang `docker-compose.yml` yang sudah ada sejak Minggu 3-4, pahami tiap service, depends_on, environment variable
4. **Networking & Volume** — kenapa container butuh volume, bind mount (development) vs named volume (production)
5. **Pengantar Kubernetes** — level konsep + hands-on ringan, kenapa dibutuhkan orchestration selain Docker Compose (multi-node, self-healing, scaling)

## Sumber Belajar
- Docker official "Get Started" guide, atau TechWorld with Nana (YouTube)
- Docker Compose official docs
- Kubernetes official "Learn Kubernetes Basics" (interactive tutorial), atau minikube quickstart

## Target di Akhir Minggu 7
- Menjelaskan perbedaan container dan VM
- Menulis Dockerfile custom dengan best practice
- Memahami dan memodifikasi docker-compose.yml multi-service
- Menjelaskan konsep dasar Kubernetes dan mendeploy aplikasi sederhana di cluster lokal

---

## Mini Project: "Containerize & Orchestrate E-Commerce Pipeline dengan Docker & Kubernetes"

**Kenapa case ini?** Selama ini pakai `docker-compose.yml` dengan image orang lain (Airflow, Kafka). Sekarang bikin custom image untuk kode sendiri, dan coba deploy ke Kubernetes — menunjukkan pemahaman dari sisi builder, bukan cuma user.

> Lanjutkan repo `ecommerce-etl-pipeline`, tambahkan Dockerfile custom + folder Kubernetes baru.

### Tahap 1: Custom Dockerfile untuk Spark Job & Python Script
- Dockerfile untuk `spark_jobs/` (base image `python`/`bitnami/spark`, COPY script + RUN pip install)
- Dockerfile terpisah untuk `streaming-demo/` (producer/consumer Kafka Minggu 4)
- Best practice: multi-stage build (kalau relevan), `.dockerignore`, pin versi base image (jangan `latest`)
- Build & test image lokal sebelum masuk Compose

### Tahap 2: Update Docker Compose dengan Custom Images
- Ganti bagian yang tadinya jalankan script langsung (volume mount) jadi pakai custom image
- Pastikan semua service tetap saling terhubung (networking, environment variable)
- Dokumentasikan kenapa struktur ini lebih baik untuk production (image self-contained)

### Tahap 3: Deploy Sederhana ke Kubernetes (Local)
- Setup minikube atau kind
- Deploy 1 komponen paling sederhana (misal Kafka consumer dari streaming-demo):
  - 1 `Deployment` manifest
  - 1 `Service` manifest
- Tidak perlu deploy seluruh pipeline — cukup 1 komponen untuk membuktikan pemahaman konsep

### Struktur Repo (update dari Minggu 6)
```
ecommerce-etl-pipeline/
├── README.md
├── GOVERNANCE.md
├── dags/
├── spark_jobs/
│   ├── Dockerfile                    # baru
│   └── .dockerignore
├── streaming-demo/
│   ├── Dockerfile                    # baru
│   └── .dockerignore
├── great_expectations/
├── governance/
├── cloud/
├── k8s/
│   ├── consumer-deployment.yaml      # baru
│   └── consumer-service.yaml         # baru
├── docker-compose.yml                # update: pakai custom images
├── data/
└── diagrams/
    ├── ...
    └── containerization_architecture.png  # baru
```

### README yang Perlu Diupdate
- Section "Containerization" — kenapa custom image dibuat (bukan cuma pakai image publik)
- Screenshot: `docker build` sukses, `kubectl get pods` menunjukkan pod jalan
- Diagram: koneksi service di Compose vs 1 komponen di K8s

### Alokasi Waktu (±8 jam)
- Bikin custom Dockerfile (Spark job + streaming demo) + testing: 3 jam
- Update Docker Compose pakai custom images: 1.5 jam
- Setup minikube/kind + deploy consumer ke K8s: 2.5 jam
- Update README + diagram: 1 jam
