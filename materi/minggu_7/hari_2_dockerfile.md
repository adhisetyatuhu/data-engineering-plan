---
title: Hari 2 - Dockerfile & Layer Caching
parent: Minggu 7 - Containerization
nav_order: 3
---

# Hari 2 — Dockerfile: Instruksi Dasar & Best Practice Layer Caching

*Selasa, 2 jam. Membedah `Dockerfile` yang sudah ditulis (tapi belum dijelaskan detail) sejak `materi/minggu_3/latihan_pipeline_mini_project.md`.*

## Tujuan Belajar

- [ ] Menjelaskan fungsi instruksi Dockerfile dasar: `FROM`, `WORKDIR`, `COPY`, `RUN`, `ENV`, `CMD`, `ENTRYPOINT`
- [ ] Menjelaskan konsep layer & layer caching, dan menulis Dockerfile yang memanfaatkannya
- [ ] Menerapkan best practice: `.dockerignore`, pin versi base image, urutan instruksi yang cache-friendly
- [ ] Membedah `Dockerfile` Airflow (Minggu 3-4) baris per baris

## Untuk Instruktur

Peserta sudah **memakai** `Dockerfile` sejak Minggu 3 tanpa penjelasan detail tiap barisnya (materi saat itu fokus ke hasil, bukan mekanismenya). Sesi ini pas untuk kembali ke file itu dan menjelaskan **kenapa** ditulis seperti itu — pengalaman "oh, ternyata begini alasannya" biasanya lebih menempel dibanding dijelaskan dari nol tanpa konteks familiar.

## Konsep & Sintaks

### Instruksi Dasar Dockerfile

| Instruksi | Fungsi | Contoh |
|---|---|---|
| `FROM` | Base image — titik awal, harus jadi instruksi pertama | `FROM apache/airflow:2.9.3` |
| `WORKDIR` | Set direktori kerja di dalam image (seperti `cd`, tapi persisten untuk instruksi berikutnya) | `WORKDIR /app` |
| `COPY` | Salin file dari mesin host ke dalam image | `COPY requirements.txt .` |
| `RUN` | Jalankan perintah **saat image di-build** (hasilnya jadi bagian permanen image) | `RUN pip install -r requirements.txt` |
| `ENV` | Set environment variable, tersedia saat build **dan** saat container jalan | `ENV PYTHONUNBUFFERED=1` |
| `USER` | Ganti user yang menjalankan instruksi berikutnya (hindari jalan sebagai `root` kalau tidak perlu) | `USER airflow` |
| `CMD` | Perintah default saat container **dijalankan** (bisa di-override saat `docker run`) | `CMD ["python", "app.py"]` |
| `ENTRYPOINT` | Perintah utama saat container dijalankan (tidak gampang di-override, `CMD` jadi argumen tambahan untuknya) | `ENTRYPOINT ["python"]` |

**`RUN` vs `CMD`/`ENTRYPOINT` — beda paling sering membingungkan**: `RUN` dieksekusi **sekali, saat `docker build`**, hasilnya "dibekukan" jadi layer image (mis. package ter-install permanen di image). `CMD`/`ENTRYPOINT` dieksekusi **tiap kali `docker run`**, dan tidak mengubah image — cuma menentukan proses apa yang jalan saat container hidup.

**`CMD` vs `ENTRYPOINT`**: `CMD` gampang di-override (`docker run image echo "halo"` akan menjalankan `echo "halo"`, mengabaikan `CMD` di Dockerfile). `ENTRYPOINT` tidak gampang di-override — kalau ada `ENTRYPOINT ["python"]` dan `CMD ["app.py"]`, `docker run image other.py` akan menjalankan `python other.py` (argumen menggantikan `CMD`, bukan `ENTRYPOINT`-nya). Pola umum: pakai `ENTRYPOINT` untuk perintah yang **selalu** harus jalan, `CMD` untuk argumen default yang **boleh** diganti.

### Membedah Dockerfile Airflow yang Sudah Ada

```dockerfile
FROM apache/airflow:2.9.3
USER root
RUN apt-get update \
    && apt-get install -y --no-install-recommends openjdk-17-jre-headless \
    && apt-get clean
USER airflow
RUN pip install --no-cache-dir \
    pyspark==3.5.1 pandas sqlalchemy psycopg2-binary pyarrow \
    great_expectations==0.18.*
```

- `FROM apache/airflow:2.9.3` — mulai dari image resmi Airflow (bukan dari OS kosong), supaya tidak perlu install & konfigurasi Airflow dari nol.
- `USER root` — sementara ganti jadi `root` **karena** `apt-get install` butuh privilege admin (image resmi Airflow default jalan sebagai user `airflow`, bukan `root`, demi keamanan).
- `RUN apt-get update && apt-get install ... && apt-get clean` — **sengaja digabung jadi 1 instruksi `RUN`** (pakai `&&`), bukan 3 `RUN` terpisah — kalau dipisah, `apt-get clean` akan jadi layer **terpisah** dari `apt-get install`, dan cache file `.deb` yang sudah diunduh tetap "membekas" ukurannya di layer install sebelumnya walau dibersihkan setelahnya. Menggabungkan jadi 1 `RUN` memastikan hasil bersih itu ter-capture dalam **1 layer final** yang lebih kecil.
- `USER airflow` — kembali ke user non-root setelah selesai butuh privilege admin — praktik keamanan standar: container sebaiknya jalan dengan privilege **seminim mungkin**.
- `RUN pip install ...` — instal dependency Python **setelah** kembali ke user `airflow` (image resmi Airflow memang didesain supaya package Python diinstal sebagai user itu, bukan root).

### Layer & Layer Caching

Tiap instruksi (`FROM`, `RUN`, `COPY`, dst) di Dockerfile menghasilkan **1 layer** — potongan filesystem yang ditumpuk. Docker **cache** tiap layer: kalau instruksi (dan file yang di-`COPY`-nya) tidak berubah dari build sebelumnya, Docker **pakai ulang** layer lama tanpa mengeksekusi ulang — jauh lebih cepat.

**Konsekuensi penting: urutan instruksi menentukan efektivitas cache.** Taruh instruksi yang **jarang berubah** di **atas**, yang **sering berubah** (kode aplikasi) di **bawah**:

```dockerfile
# BURUK -- copy semua kode dulu, baru install dependency
FROM python:3.11-slim
WORKDIR /app
COPY . .                          # <- kode berubah tiap commit
RUN pip install -r requirements.txt   # <- cache invalid TIAP KALI ada perubahan kode apapun,
                                       #    walau requirements.txt tidak berubah sama sekali!
```

```dockerfile
# BAIK -- copy requirements.txt dulu, install, baru copy sisa kode
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt   # <- layer ini di-cache, TIDAK dieksekusi ulang
                                       #    selama requirements.txt tidak berubah
COPY . .                              # <- kode berubah tiap commit, tapi ini instruksi
                                       #    TERAKHIR, tidak merusak cache layer di atasnya
```

Versi "baik" ini bisa menghemat **menit** per build kalau `pip install`-nya berat (banyak dependency, seperti `pyspark`) — perubahan 1 baris kode tidak lagi memaksa seluruh dependency di-install ulang dari nol.

### `.dockerignore`

Sama seperti `.gitignore`, tapi untuk `docker build` — mencegah file yang tidak perlu ikut ter-`COPY` ke dalam **build context** (dan image final):

```
# .dockerignore
__pycache__/
*.pyc
.venv/
.git/
data/
*.log
```

**Dua manfaat sekaligus**: (1) image jadi lebih kecil (tidak ada file sampah/data mentah ikut terbawa), (2) `docker build` lebih cepat (build context yang dikirim ke Docker daemon lebih kecil — tanpa `.dockerignore`, folder `data/` berisi `online_retail_II.csv` yang besar bisa ikut ter-scan tiap build, walau tidak pernah benar-benar di-`COPY`).

### Multi-Stage Build (Sekilas)

Untuk image yang butuh tool build berat (compiler, dependency development) tapi ujungnya cuma butuh **hasil**-nya saja saat runtime:

```dockerfile
# Stage 1: build
FROM python:3.11 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

# Stage 2: runtime -- cuma copy HASIL install dari stage 1, base image lebih kecil
FROM python:3.11-slim
COPY --from=builder /root/.local /root/.local
COPY . /app
WORKDIR /app
CMD ["python", "main.py"]
```

Image final memakai base `-slim` (lebih kecil) dan cuma membawa **hasil** instalasi dari stage builder, bukan seluruh tool build yang dipakai sepanjang proses instalasi. Untuk mini project minggu ini (`spark_jobs/`, `streaming-demo/`) multi-stage **tidak wajib** (dependency-nya tidak butuh compiler berat) — disinggung di sini supaya dikenal konsepnya untuk kasus yang lebih kompleks nanti.

## Kesalahan Umum

1. **Pakai `FROM image:latest` alih-alih versi spesifik.** `latest` bisa berubah isinya kapan saja tanpa peringatan — build yang jalan hari ini bisa gagal/beda perilaku minggu depan tanpa ada perubahan di Dockerfile kamu sendiri. Selalu pin versi eksplisit (`python:3.11-slim`, bukan `python:latest`) — pola yang sudah konsisten dipakai sejak Minggu 3 (`apache/airflow:2.9.3`, bukan `apache/airflow:latest`).
2. **`COPY . .` sebelum `RUN pip install`.** Merusak layer caching seperti dibahas di atas — dependency ter-install ulang dari nol tiap kali **ada file apapun** yang berubah, bukan cuma saat `requirements.txt` berubah.
3. **Lupa `.dockerignore`.** Build context ikut membawa `.git/`, `__pycache__/`, atau (lebih parah) file kredensial (`.env`, JSON key GCP dari Minggu 6) yang tidak sengaja ikut ter-`COPY` ke dalam image — image yang di-push ke registry publik bisa membocorkan itu.
4. **Menjalankan semua instruksi sebagai `root` tanpa alasan.** Kalau image tidak butuh privilege admin di runtime, jalankan sebagai user non-root (seperti pola `USER airflow` di atas) — mengurangi dampak kalau ada kerentanan di dalam container.
5. **Terlalu banyak `RUN` terpisah untuk operasi yang seharusnya 1 unit logis.** Tiap `RUN` = 1 layer baru; buat operasi yang berhubungan (`apt-get update && apt-get install && apt-get clean`) jadi 1 `RUN` supaya cache & ukuran image lebih efisien, seperti dibahas di bagian pembedahan Dockerfile Airflow di atas.

## Latihan

1. Diberi 2 versi Dockerfile (versi "BURUK" dan "BAIK" di atas) — kalau kamu mengubah 1 baris di `main.py` (bukan `requirements.txt`) dan `docker build` ulang, versi mana yang lebih cepat? Jelaskan alasannya merujuk ke konsep layer caching.
2. Jelaskan kenapa `Dockerfile` Airflow di atas melakukan `USER root` lalu `USER airflow` lagi — apa yang terjadi kalau baris `USER airflow` (yang kedua) dihapus?
3. Tulis draf `.dockerignore` untuk folder `spark_jobs/` di repo `ecommerce-etl-pipeline` — pikirkan file/folder apa saja yang sebaiknya **tidak** ikut masuk ke image.
4. Kenapa pin versi base image (`python:3.11-slim`, bukan `python:latest`) penting khususnya untuk **reproducibility** pipeline data — kaitkan dengan pengalaman `write.mode("overwrite")` dan SCD Type 1 yang sudah dibahas di `materi/minggu_3/latihan_pipeline_mini_project.md` (pipeline yang harus menghasilkan hasil konsisten tiap kali dijalankan ulang).

## Kunci Jawaban & Pembahasan

**1.** Versi **"BAIK"** jauh lebih cepat. Karena `requirements.txt` tidak berubah, layer `RUN pip install -r requirements.txt` tetap **valid di cache** — Docker langsung pakai ulang layer itu tanpa menjalankan `pip install` lagi, cuma layer `COPY . .` (yang berisi `main.py` yang berubah) dan seterusnya yang dieksekusi ulang. Versi "BURUK" harus menjalankan ulang `pip install` **dari nol** setiap build, karena `COPY . .` (yang isinya berubah tiap kali `main.py` diedit) ada **sebelum** instruksi install — begitu 1 layer cache invalid, **semua layer setelahnya** ikut invalid juga (cache Docker bersifat berurutan dari atas ke bawah).

**2.** `USER root` dibutuhkan sementara karena `apt-get install openjdk-17-jre-headless` butuh privilege admin untuk memodifikasi sistem (instal package level OS) — user `airflow` default (non-root, demi keamanan) tidak punya izin ini. `USER airflow` di baris berikutnya **mengembalikan** privilege ke non-root untuk instruksi setelahnya (`pip install` dan seterusnya, termasuk saat container benar-benar berjalan nanti). Kalau baris `USER airflow` kedua **dihapus**, seluruh instruksi setelahnya (termasuk proses Airflow yang berjalan di container nanti) akan tetap berjalan sebagai **`root`** — bekerja secara fungsional, tapi melanggar prinsip least privilege (`materi/minggu_6/hari_4_iam.md`): kalau ada celah keamanan di aplikasi yang jalan di dalam container, dampaknya jadi lebih besar karena prosesnya punya privilege admin penuh di dalam container itu.

**3.** Contoh draf:
```
data/
__pycache__/
*.pyc
.venv/
venv/
.git/
*.md
tests/
```
`data/` dikecualikan karena data mentah/hasil (bisa besar, dan sebaiknya di-mount lewat volume saat runtime, bukan dibekukan permanen ke dalam image — dibahas `hari_4_networking_volume.md`); `__pycache__/`/`*.pyc` adalah artefak build Python yang tidak perlu ikut; `.git/` tidak relevan untuk menjalankan kode; `tests/` opsional dikecualikan kalau image production tidak perlu menjalankan test suite di dalamnya.

**4.** Pin versi base image itu perpanjangan langsung dari alasan yang sama kenapa `write.mode("overwrite")`/SCD Type 1 dipilih secara sadar di Minggu 3: **reproducibility** — pipeline data harus menghasilkan perilaku yang **bisa diprediksi & konsisten** tiap kali dijalankan ulang, baik hari ini maupun 6 bulan lagi. Kalau base image pakai `python:latest`, image yang di-build hari ini bisa membawa versi Python (dan library sistem) yang **berbeda** dari yang di-build 6 bulan lalu — kalau ada perubahan breaking di versi baru Python/dependency sistem, pipeline yang tadinya jalan normal bisa tiba-tiba gagal atau (lebih berbahaya) menghasilkan output **berbeda secara diam-diam** tanpa error jelas, tanpa ada satupun baris kode pipeline yang diubah. Pin versi eksplisit memastikan environment build-nya sendiri deterministik, sama pentingnya dengan memastikan logic transformasi datanya deterministik.
