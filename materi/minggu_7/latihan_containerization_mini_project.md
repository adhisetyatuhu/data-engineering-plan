---
title: Hands-on Containerization & Kubernetes + Mini Project v5
parent: Minggu 7 - Containerization
nav_order: 7
---

# Sabtu–Minggu — Hands-on: Custom Dockerfile, Update Compose, Deploy ke Kubernetes

*Sabtu 4 jam + Minggu 4 jam. Kelanjutan langsung `minggu_7.md` bagian "Mini Project: Containerize & Orchestrate E-Commerce Pipeline dengan Docker & Kubernetes" — upgrade dari `materi/minggu_6/latihan_cloud_migration_mini_project.md`.*

> **Repo yang sama**: tetap `ecommerce-etl-pipeline`. Tidak ada resource cloud baru minggu ini (beda dari Minggu 6) — semua hands-on 100% lokal, tidak ada risiko biaya.

## Tujuan Belajar

- [ ] Membuat custom Dockerfile untuk `spark_jobs/` dan `streaming-demo/`, dan membuktikan keduanya bisa jalan sebagai image **berdiri sendiri**
- [ ] Meng-containerize `streaming-demo/producer.py`/`consumer.py` ke dalam `docker-compose.yml`, menggantikan cara lama (jalan dari host venv)
- [ ] Deploy `streaming-demo/consumer.py` ke Kubernetes lokal (minikube) sebagai 1 Deployment + 1 Service
- [ ] Menjelaskan kapan pola image self-contained lebih baik daripada bind mount + `subprocess`, dan kapan Kubernetes benar-benar dibutuhkan dibanding Compose

## Untuk Instruktur

Berbeda dari kompleksitas Minggu 6 (risiko biaya cloud), sesi ini murni belajar **skill mengemas kode**, tidak ada risiko finansial — tapi tetap ada risiko frustrasi teknis (image build gagal, minikube tidak bisa diakses, dst). Dorong peserta membuktikan tiap tahap **berjalan berdiri sendiri** sebelum lanjut ke tahap berikutnya (build image → test lokal dengan `docker run` → baru masuk Compose → baru masuk Kubernetes) — pola debugging bertahap yang sudah terbukti efektif sejak `materi/minggu_3/latihan_pipeline_mini_project.md` ("jalankan manual dulu sebelum masuk Airflow").

---

## Bagian 1 (Sabtu, 4 jam): Custom Dockerfile untuk Spark Job & Streaming Demo

### Tahap 1: Dockerfile untuk `spark_jobs/` (±1.5 jam)

**Kenapa `spark_jobs/` butuh Dockerfile sendiri, padahal sudah bisa jalan lewat image Airflow (Minggu 3)?** Image Airflow (`Dockerfile` di root repo) memang sudah bisa menjalankan `clean_transform.py`/`build_star_schema.py` lewat `subprocess` — tapi itu membuat script Spark **terikat** ke image Airflow yang jauh lebih besar (berisi seluruh Airflow, bukan cuma yang dibutuhkan job Spark). Dockerfile terpisah ini membuktikan `spark_jobs/` bisa jalan **independen**, portable ke konteks manapun (server Spark cluster sungguhan, CI/CD job terpisah, dst) — bukan mengganti cara Airflow menjalankannya (yang tetap dipertahankan), tapi menunjukkan pemahaman package image self-contained.

```dockerfile
# spark_jobs/Dockerfile
FROM python:3.11-slim

RUN apt-get update \
    && apt-get install -y --no-install-recommends openjdk-17-jre-headless \
    && apt-get clean

WORKDIR /app
COPY spark_jobs/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY spark_jobs/*.py .

ENTRYPOINT ["python"]
```

```
# spark_jobs/requirements.txt
pyspark==3.5.1
pandas
pyarrow
```

```
# spark_jobs/.dockerignore
__pycache__/
*.pyc
```

Perhatikan urutan instruksi mengikuti best practice layer caching dari `hari_2_dockerfile.md`: `requirements.txt` di-`COPY` & di-`RUN pip install` **sebelum** kode `.py` — perubahan kode nanti tidak memaksa `pip install` (yang berat, karena `pyspark`) dijalankan ulang.

**Build & test lokal** (buktikan image ini berjalan **sendiri**, tanpa Airflow sama sekali):

```bash
docker build -t ecommerce-spark-jobs:v1 -f spark_jobs/Dockerfile .

docker run --rm \
  -v $(pwd)/data:/app/data \
  ecommerce-spark-jobs:v1 \
  clean_transform.py /app/data/raw/online_retail_II.csv /app/data/warehouse/_staging/retail_clean

docker run --rm \
  -v $(pwd)/data:/app/data \
  ecommerce-spark-jobs:v1 \
  build_star_schema.py /app/data/warehouse/_staging/retail_clean /app/data/warehouse
```

Data (`-v $(pwd)/data:/app/data`) sengaja tetap di-bind mount, bukan di-`COPY` ke image — data mentah berukuran besar dan berubah-ubah, beda dari kode yang tetap (`hari_4_networking_volume.md`: image self-contained untuk **kode**, volume untuk **data**).

**Verifikasi**: `data/warehouse/` di host berisi 4 folder Parquet baru, identik hasilnya dengan menjalankan script yang sama lewat Airflow (Minggu 3) — membuktikan logic-nya benar-benar portable, tidak bergantung environment Airflow spesifik.

### Tahap 2: Dockerfile untuk `streaming-demo/` (±1.5 jam)

Sebelumnya (`materi/minggu_4/latihan_dq_streaming_mini_project.md`), `producer.py`/`consumer.py` dijalankan dari **host venv** — bukan container sama sekali. Sekarang keduanya di-containerize:

```dockerfile
# streaming-demo/Dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY streaming-demo/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY streaming-demo/*.py .

CMD ["python", "consumer.py"]
```

```
# streaming-demo/requirements.txt
kafka-python==2.0.*
pandas
pyarrow
```

**Modifikasi kecil di `consumer.py` dan `producer.py`**: sebelumnya `bootstrap_servers` di-hardcode `"localhost:9092"` — itu cuma benar kalau dijalankan dari **host**. Sekarang berjalan **di dalam** container, jadi harus bisa dikonfigurasi lewat environment variable (host `kafka` kalau di Compose network yang sama, host lain lagi kalau nanti di Kubernetes — lihat Bagian 3):

```python
# streaming-demo/consumer.py -- ganti baris bootstrap_servers
import os

KAFKA_BOOTSTRAP = os.environ.get("KAFKA_BOOTSTRAP_SERVERS", "localhost:9092")

consumer = KafkaConsumer(
    TOPIC,
    bootstrap_servers=KAFKA_BOOTSTRAP,   # <- sebelumnya hardcode "localhost:9092"
    value_deserializer=lambda v: json.loads(v.decode("utf-8")),
    auto_offset_reset="earliest",
    group_id="revenue-aggregator",
)
```

Terapkan perubahan yang sama (`os.environ.get("KAFKA_BOOTSTRAP_SERVERS", "localhost:9092")`) di `producer.py`. Default tetap `"localhost:9092"` supaya cara lama (jalan dari host venv, Minggu 4) **tetap berfungsi** kalau dibutuhkan untuk debugging cepat — environment variable cuma dipakai kalau eksplisit di-set.

**Build & test lokal**:

```bash
docker build -t ecommerce-streaming-demo:v1 -f streaming-demo/Dockerfile .
docker run --rm ecommerce-streaming-demo:v1 python producer.py --help  # sekadar cek image jalan, belum ada Kafka
```

### Deliverable Sabtu

- [ ] `spark_jobs/Dockerfile` build sukses, dan `clean_transform.py`/`build_star_schema.py` terbukti jalan **berdiri sendiri** lewat `docker run` (tanpa Airflow), menghasilkan output identik
- [ ] `streaming-demo/Dockerfile` build sukses
- [ ] `producer.py`/`consumer.py` sudah pakai `KAFKA_BOOTSTRAP_SERVERS` dari environment variable, bukan hardcode

---

## Bagian 2 (Minggu, 4 jam): Update Compose + Deploy ke Kubernetes

### Tahap 3: Containerize Producer/Consumer di `docker-compose.yml` (±1.5 jam)

Tambahkan service baru ke `docker-compose.yml` (yang sudah berisi `airflow-webserver`, `postgres`, `kafka` sejak Minggu 3-4), memakai custom image dari Tahap 2:

```yaml
# tambahan service di docker-compose.yml
services:
  streaming-consumer:
    build:
      context: .
      dockerfile: streaming-demo/Dockerfile
    command: ["python", "consumer.py"]
    environment:
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092    # <- nama service, bukan localhost (hari_3_docker_compose.md)
    depends_on:
      - kafka
```

**Kenapa `KAFKA_BOOTSTRAP_SERVERS: kafka:9092` sekarang, bukan `localhost:9092`**: `streaming-consumer` dan `kafka` sekarang **sama-sama** service di file Compose yang sama — sesuai `hari_3_docker_compose.md`, itu artinya keduanya otomatis berada di bridge network Compose yang sama, dan bisa saling terhubung lewat **nama service** sebagai hostname. Ini upgrade nyata dari Minggu 4: dulu `consumer.py` jalan dari host (butuh `localhost:9092` karena port sudah di-publish ke host), sekarang jalan **di dalam** network Compose (butuh nama service, bukan `localhost`).

`producer.py` **sengaja tidak** dijadikan service Compose permanen (dia cuma perlu dijalankan sesekali untuk demo, beda dari `consumer` yang lebih masuk akal terus "mendengarkan") — tetap dijalankan manual saat demo:

```bash
docker compose build streaming-consumer
docker compose up -d kafka streaming-consumer

# jalankan producer manual, sekali, saat mau demo (image sudah ada dari Tahap 2)
docker run --rm --network ecommerce-etl-pipeline_default \
  -e KAFKA_BOOTSTRAP_SERVERS=kafka:9092 \
  ecommerce-streaming-demo:v1 python producer.py
```

`--network ecommerce-etl-pipeline_default` menyambungkan container `producer` (dijalankan `docker run` biasa, di luar `docker compose up`) ke network Compose yang sudah ada — nama network persisnya bisa dicek dengan `docker network ls` (formatnya `<nama_folder_project>_default`).

**Verifikasi**: `docker compose logs -f streaming-consumer` menunjukkan event masuk & running total bertambah — perilaku identik dengan Minggu 4, tapi sekarang berjalan sepenuhnya di dalam container terisolasi, bukan bergantung venv Python di host.

### Tahap 4: Deploy Consumer ke Kubernetes Lokal (±2 jam)

```bash
minikube start --driver=docker
minikube dashboard &   # opsional, buka UI visual di browser
```

**Kafka tetap jalan di Docker Compose** (di host, lewat `docker compose up -d kafka`) — mini project ini **tidak** memindahkan Kafka ke dalam Kubernetes (sesuai `minggu_7.md`: "tidak perlu deploy seluruh pipeline, cukup 1 komponen"). Pod consumer di minikube perlu menjangkau Kafka yang jalan di host — pola yang **persis sama** dengan kebutuhan `host.docker.internal` di Minggu 3, tapi versi Kubernetes-nya:

```bash
minikube ssh -- getent hosts host.minikube.internal
# atau langsung dipakai di manifest tanpa perlu dicek dulu -- minikube menyediakannya otomatis
```

`k8s/consumer-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: streaming-consumer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: streaming-consumer
  template:
    metadata:
      labels:
        app: streaming-consumer
    spec:
      containers:
        - name: consumer
          image: ecommerce-streaming-demo:v1
          imagePullPolicy: Never   # <- pakai image lokal, jangan coba pull dari registry (image belum di-push)
          env:
            - name: KAFKA_BOOTSTRAP_SERVERS
              value: "host.minikube.internal:9092"   # <- Kafka jalan di host (Docker Compose), bukan di cluster ini
          command: ["python", "consumer.py"]
```

`k8s/consumer-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: consumer-service
spec:
  selector:
    app: streaming-consumer
  ports:
    - port: 8000
      targetPort: 8000
```

**Catatan jujur soal Service di sini**: `consumer.py` cuma **membaca** dari Kafka (koneksi keluar/outbound), tidak menerima traffic masuk apapun — secara fungsional, Service ini **tidak benar-benar dibutuhkan** supaya consumer bisa bekerja (beda dari kasus web app yang butuh dijangkau dari luar). Dibuat di sini murni untuk **latihan** menulis manifest Service sesuai permintaan `minggu_7.md`, dan supaya konsepnya nyambung ke `hari_5_kubernetes_intro.md` — kejujuran ini penting: jangan sampai peserta mengira Service selalu wajib untuk **semua** jenis Pod, padahal manfaat nyatanya baru terasa untuk Pod yang **menerima** traffic (mis. web server, API).

**Image harus tersedia untuk minikube** (karena `imagePullPolicy: Never`, image lokal harus "dipindahkan" ke Docker daemon milik minikube dulu):

```bash
minikube image load ecommerce-streaming-demo:v1
```

**Deploy & verifikasi**:

```bash
kubectl apply -f k8s/consumer-deployment.yaml
kubectl apply -f k8s/consumer-service.yaml

kubectl get pods                        # status "Running" untuk pod streaming-consumer
kubectl logs -f deployment/streaming-consumer   # lihat event masuk dari Kafka, sama seperti log Compose

kubectl get services                     # consumer-service muncul dengan ClusterIP
```

**Buktikan self-healing** (poin pedagogis penting, sejalan pola "Uji Fail-Fast" Minggu 4 — jangan cuma percaya, buktikan sendiri):

```bash
kubectl delete pod -l app=streaming-consumer   # hapus paksa pod yang sedang jalan
kubectl get pods -w                            # amati: Kubernetes otomatis membuat pod BARU menggantikannya
```

Jalankan `producer.py` lagi (dari Tahap 3) sambil `kubectl logs -f` aktif — verifikasi Pod di Kubernetes benar-benar menerima event Kafka yang sama seperti versi Compose.

### Deliverable Minggu

- [ ] `docker compose logs streaming-consumer` menunjukkan event Kafka diterima & running total bertambah
- [ ] `kubectl get pods` menunjukkan `streaming-consumer` berstatus `Running`
- [ ] Screenshot/log hasil uji self-healing (`kubectl delete pod` diikuti pod baru otomatis muncul)

### Struktur Repo (Final, Update dari Minggu 6)

```
ecommerce-etl-pipeline/
├── README.md
├── GOVERNANCE.md
├── dags/
│   └── ecommerce_etl_dag.py
├── spark_jobs/
│   ├── clean_transform.py
│   ├── build_star_schema.py
│   ├── requirements.txt              # baru
│   ├── Dockerfile                    # baru
│   └── .dockerignore                 # baru
├── streaming-demo/
│   ├── producer.py                   # update: KAFKA_BOOTSTRAP_SERVERS dari env
│   ├── consumer.py                   # update: KAFKA_BOOTSTRAP_SERVERS dari env
│   ├── requirements.txt              # baru
│   ├── Dockerfile                    # baru
│   └── .dockerignore                 # baru
├── great_expectations/
├── governance/
├── cloud/
├── k8s/                                # baru
│   ├── consumer-deployment.yaml      # baru
│   └── consumer-service.yaml         # baru
├── docker-compose.yml                # update: service streaming-consumer pakai custom image
├── Dockerfile                          # tetap (image Airflow, tidak berubah)
├── data/
└── diagrams/
    ├── star_schema.png
    ├── pipeline_architecture_v2.png
    ├── data_lineage.png
    ├── cloud_architecture.png
    └── containerization_architecture.png  # baru
```

### README yang Perlu Diupdate

- Section **"Containerization"** — jelaskan kenapa `spark_jobs/` dan `streaming-demo/` dapat Dockerfile sendiri (image self-contained, portable), bukan cuma mengandalkan bind mount ke image Airflow
- Screenshot: `docker build` sukses untuk kedua Dockerfile baru
- Screenshot: `kubectl get pods` menunjukkan `streaming-consumer` `Running`, dan hasil uji self-healing
- Diagram (`diagrams/containerization_architecture.png`): perbandingan visual — service mana yang jalan di Docker Compose vs 1 komponen yang juga bisa di-deploy ke Kubernetes

### Kriteria "Selesai" untuk Minggu 7

- [ ] `spark_jobs/Dockerfile` dan `streaming-demo/Dockerfile` build sukses dan bisa dijalankan **berdiri sendiri** lewat `docker run`, tanpa bergantung image Airflow
- [ ] `docker-compose.yml` ter-update: `streaming-consumer` jalan sebagai service dengan custom image, terhubung ke `kafka` lewat nama service (bukan `localhost`)
- [ ] `streaming-demo/consumer.py` berhasil di-deploy ke Kubernetes lokal (1 Deployment + 1 Service), terbukti menerima event Kafka yang jalan di Docker Compose (lewat `host.minikube.internal`)
- [ ] Self-healing terbukti bekerja: `kubectl delete pod` diikuti Kubernetes otomatis membuat Pod baru
- [ ] Bisa menjelaskan ke orang lain: beda container vs VM (`hari_1_container_vs_vm.md`), kenapa urutan instruksi Dockerfile mempengaruhi layer caching (`hari_2_dockerfile.md`), dan kenapa Kubernetes dianggap "overkill" untuk skala mini project ini tapi penting untuk skala production yang lebih besar (`hari_5_kubernetes_intro.md`)

Kalau semua tercentang, lanjut ke `minggu_8.md` — minggu penutup (Capstone): menambahkan MongoDB & Redis ke arsitektur yang sudah dibangun, menyatukan seluruh pekerjaan 8 minggu jadi 1 cerita *polyglot persistence* yang koheren di `STORAGE_STRATEGY.md`.
