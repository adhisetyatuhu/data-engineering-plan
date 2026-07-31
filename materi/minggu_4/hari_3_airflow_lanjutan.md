---
title: Hari 3 - Airflow Lanjutan
parent: Minggu 4 - Data Quality, Orchestration & Streaming
nav_order: 4
---

# Hari 3 — Orchestration Lanjutan: Sensors, Dependency Kompleks, Retry & Alerting, XComs

*Rabu, 2 jam. Konsep + contoh kode — implementasi penuh ke DAG `ecommerce_etl_pipeline` ada di `latihan_dq_streaming_mini_project.md`.*

> Ini bukan tool baru — DAG dasar sudah dibangun di `materi/minggu_3/latihan_pipeline_mini_project.md`. Hari ini memperdalam **kemampuan** Airflow yang belum dipakai minggu lalu: bagaimana membuat pipeline yang tangguh (retry), transparan saat gagal (alerting), dan bisa menunggu kondisi eksternal (sensor) — bukan cuma jalan lurus dari task A ke B.

## Tujuan Belajar

- [ ] Menjelaskan XComs dan bagaimana return value TaskFlow API sebenarnya memakainya secara implisit
- [ ] Mengonfigurasi retry (`retries`, `retry_delay`) untuk task yang bergantung ke resource eksternal
- [ ] Membuat alerting sederhana lewat `on_failure_callback`
- [ ] Menjelaskan apa itu Sensor dan kapan dipakai (dibanding sekadar menjadwalkan DAG di jam tertentu)
- [ ] Membuat branching sederhana dengan `@task.branch`

## Untuk Instruktur: Mindset Shift

Developer sudah punya banyak intuisi yang relevan hari ini dari pengalaman membangun sistem backend:
- **Retry** ≈ retry logic untuk HTTP request ke API eksternal yang kadang timeout (exponential backoff, dsb) — konsepnya identik, cuma levelnya di task pipeline, bukan request HTTP.
- **Alerting (`on_failure_callback`)** ≈ error tracking (Sentry, PagerDuty) yang mengirim notifikasi saat exception terjadi di production.
- **Sensor** ≈ polling/long-polling — menunggu kondisi eksternal terpenuhi sebelum lanjut, mirip `while not file_exists(): sleep(n)` tapi dikelola Airflow (termasuk timeout, dan tidak memblokir resource terus-menerus kalau dikonfigurasi `mode="reschedule"`).
- **Branching** ≈ `if/else` biasa, tapi levelnya di graf task, bukan di dalam satu fungsi.

Tekankan: semua ini bukan fitur "canggih" yang eksotis, tapi jawaban langsung atas pertanyaan **"apa yang terjadi kalau ada yang salah?"** — pertanyaan yang belum banyak dijawab di DAG Minggu 3 (yang asumsinya semua berjalan mulus).

## Konsep & Sintaks

### XComs — Sudah Dipakai Sejak Minggu 3, Sekarang Dijelaskan Eksplisit

**XCom** (*cross-communication*) adalah mekanisme Airflow untuk mengirim data kecil **antar task**. Task Minggu 3 sudah memakainya tanpa disadari:

```python
@task
def extract_raw_data() -> str:
    return "/opt/airflow/data/raw/online_retail_II.csv"   # <- otomatis di-"push" sebagai XCom

@task
def transform_with_spark(raw_path: str) -> str:   # <- otomatis "pull" XCom dari task sebelumnya
    ...
```

Dengan **TaskFlow API** (`@task`, dipakai sejak `materi/minggu_3/latihan_pipeline_mini_project.md`), `return` value otomatis di-push sebagai XCom, dan meneruskannya ke task lain semudah memanggilnya sebagai argumen fungsi biasa — Airflow yang mengurus penyimpanan & pengambilannya di belakang layar. Dengan API klasik (gaya lama, `PythonOperator`), ini harus ditulis eksplisit:

```python
# Gaya klasik (untuk dikenali, bukan yang dipakai di mini project ini)
def extract(**context):
    context["ti"].xcom_push(key="raw_path", value="/opt/airflow/data/raw/online_retail_II.csv")

def transform(**context):
    raw_path = context["ti"].xcom_pull(key="raw_path", task_ids="extract")
```

**Batasan penting**: XCom didesain untuk data **kecil** (metadata: path file, row count, status), **bukan** untuk memindahkan dataset besar antar task — itu kenapa di DAG kita, task saling mengoper **path** ke file Parquet (string kecil), bukan DataFrame itu sendiri. Backend default Airflow menyimpan XCom di metadata database-nya sendiri, yang tidak dirancang menampung ratusan ribu baris data.

### Retry — Task yang Bergantung ke Resource Eksternal

```python
from datetime import timedelta

@task(
    retries=3,
    retry_delay=timedelta(minutes=2),
    retry_exponential_backoff=True,   # jeda makin lama tiap percobaan gagal (2 min, 4 min, 8 min, ...)
)
def load_to_warehouse(output_dir: str) -> None:
    ...
```

`load_to_warehouse` adalah kandidat tepat untuk retry karena bergantung ke **resource eksternal** (koneksi ke Postgres) yang bisa gagal sementara (network blip, database sedang restart) — kegagalan seperti ini sering **transient** (hilang sendiri kalau dicoba lagi), beda dari bug logic yang akan gagal terus-menerus berapa kali pun dicoba. Task seperti `data_quality_check` (Hari 1–2) sebaliknya **tidak** cocok diberi retry banyak — kalau datanya memang tidak valid, mencoba ulang 3x tidak akan mengubah hasilnya, cuma membuang waktu sebelum alert akhirnya terkirim.

**Aturan praktis**: retry untuk kegagalan yang **mungkin hilang sendiri** (koneksi, resource sementara sibuk), jangan untuk kegagalan yang **pasti konsisten** (data tidak valid, logic salah, file tidak ada sama sekali).

### Alerting — `on_failure_callback`

```python
def send_alert(context):
    task_id = context["task_instance"].task_id
    dag_id = context["task_instance"].dag_id
    exception = context.get("exception")
    message = f"[ALERT] Task `{task_id}` di DAG `{dag_id}` gagal.\nError: {exception}"
    print(message)   # placeholder — ganti dengan requests.post() ke Slack webhook, atau EmailOperator

@dag(
    dag_id="ecommerce_etl_pipeline",
    default_args={"on_failure_callback": send_alert},
    ...
)
def ecommerce_etl_pipeline():
    ...
```

`on_failure_callback` dipasang di `default_args` supaya berlaku untuk **semua** task di DAG (bisa juga dipasang per-task kalau butuh perilaku berbeda-beda). `context` yang diterima callback berisi banyak informasi tentang run yang gagal (task apa, DAG apa, exception apa, kapan) — cukup untuk menyusun pesan alert yang informatif tanpa peserta perlu menebak-nebak.

**Kenapa alerting penting justru sesudah ada retry**: tanpa alerting, kegagalan yang sudah melewati semua percobaan retry (`retries=3` habis, tetap gagal) akan tetap **diam-diam merah** di Airflow UI — tidak ada yang tahu kecuali ada yang cek dashboard secara manual. Alerting memastikan kegagalan itu **aktif memberi tahu**, bukan pasif menunggu ditemukan.

### Sensor — Menunggu Kondisi Eksternal

Bedanya dengan penjadwalan biasa: `schedule="@daily"` menentukan **kapan DAG mulai dicoba dijalankan**, tapi tidak menjamin data sumbernya **sudah siap** di jam itu. Sensor menunggu sampai kondisi tertentu benar-benar terpenuhi, baru melanjutkan.

```python
from airflow.sensors.filesystem import FileSensor

wait_for_file = FileSensor(
    task_id="wait_for_raw_file",
    filepath="/opt/airflow/data/raw/online_retail_II.csv",
    poke_interval=30,       # cek tiap 30 detik
    timeout=60 * 60,        # menyerah (gagal) kalau file belum ada juga setelah 1 jam
    mode="reschedule",      # lepaskan worker slot selagi menunggu (lihat catatan di bawah)
)
```

**`mode="poke"` vs `mode="reschedule"`**: `poke` (default) menahan **satu worker slot** terus-menerus selama menunggu (sensor terus "mengecek" tanpa melepas resource) — boros kalau waktu tunggunya bisa lama. `mode="reschedule"` melepas slot itu di antara tiap pengecekan (sensor "tidur" lalu dijadwalkan ulang), jauh lebih hemat resource untuk sensor dengan `poke_interval` besar/waktu tunggu berpotensi lama — analog dengan **event loop non-blocking** vs **thread yang menahan blocking call**, developer yang familiar `async`/`await` akan langsung paham bedanya.

### Branching — `@task.branch`

```python
@task.branch
def check_row_count(warehouse_dir: str) -> str:
    import pandas as pd
    df = pd.read_parquet(f"{warehouse_dir}/fact_sales")
    if len(df) == 0:
        return "skip_load_empty_data"
    return "load_to_warehouse"

@task
def skip_load_empty_data():
    print("Data kosong, load dilewati — tidak dianggap error, cuma tidak ada yang perlu di-load.")
```

`@task.branch` mengembalikan **task_id** (string, atau list of string) dari task mana yang harus dilanjutkan — task lain yang tidak dipilih otomatis berstatus `skipped` (bukan `failed`), penanda bahwa itu memang keputusan sadar, bukan kegagalan. Beda dari `data_quality_check` yang membuat pipeline **gagal** kalau data tidak valid, branching di sini untuk kasus di mana kondisi tertentu **bukan kesalahan** tapi butuh jalur berbeda (mis. tidak ada data baru hari itu — bukan bug, cuma tidak ada yang perlu dikerjakan).

## Kesalahan Umum

1. **Memakai XCom untuk memindahkan data besar** (mis. `return df.to_dict()` untuk DataFrame ratusan ribu baris) — akan memperlambat/membebani metadata database Airflow, bahkan bisa gagal karena batas ukuran. Aturan: XCom untuk path/metadata/angka kecil; data besar tetap lewat storage (Parquet di disk/object storage), task cuma saling mengoper **lokasinya**.
2. **Memberi retry ke semua task secara membabi buta** tanpa mikir jenis kegagalannya. Retry untuk `data_quality_check` yang gagal karena data memang tidak valid cuma menunda alert (dan membuang waktu compute), bukan menyelesaikan masalah.
3. **`on_failure_callback` yang gagal sendiri** (mis. exception di dalam fungsi callback-nya sendiri, atau bergantung ke service alerting yang juga sedang down) — bisa membuat kegagalan asli makin sulit terlihat. Callback alerting sebaiknya dibuat sesederhana & se-robust mungkin (kalau perlu, bungkus dengan `try/except` sendiri supaya kegagalan kirim alert tidak menutupi kegagalan task asli di log).
4. **Memakai `mode="poke"` untuk sensor dengan waktu tunggu berpotensi lama** (jam, bukan detik/menit) — menahan worker slot terlalu lama bisa membuat task lain di DAG (atau DAG lain) ikut tertahan karena kehabisan slot worker yang tersedia.

## Latihan

1. Task `load_to_warehouse` di DAG Minggu 3 belum punya retry sama sekali. Tuliskan konfigurasi `retries` dan `retry_delay` yang masuk akal untuk task ini, dan jelaskan alasannya (berapa kali coba, berapa lama jeda).
2. Kenapa `extract_raw_data` di DAG Minggu 3 (yang cuma mengecek `os.path.exists()`) sebenarnya kandidat yang lebih cocok diganti jadi **Sensor** dibanding tetap jadi task `@task` biasa? Apa bedanya perilaku keduanya kalau file belum ada saat DAG mulai jalan?
3. Rancang `on_failure_callback` yang mengirim pesan berbeda tergantung task mana yang gagal — misalnya pesan untuk kegagalan `data_quality_check` sebaiknya menyebutkan dimensi kualitas data apa yang gagal (dari `hari_1_data_quality_dimensions.md`/`hari_2_great_expectations.md`), sementara kegagalan `load_to_warehouse` cukup menyebut "koneksi database gagal, akan dicoba ulang". Bagaimana `context` yang diterima callback bisa dipakai membedakan ini?
4. Jelaskan kenapa branching (`@task.branch`) berbeda secara fundamental dari `data_quality_check` yang menghentikan pipeline — walau keduanya sama-sama "mengambil keputusan" berdasarkan kondisi data.

## Kunci Jawaban & Pembahasan

**1.**
```python
@task(retries=3, retry_delay=timedelta(minutes=1), retry_exponential_backoff=True)
def load_to_warehouse(output_dir: str) -> None:
    ...
```
3 kali percobaan cukup untuk mengatasi gangguan sementara (network blip, Postgres restart singkat) tanpa menunda pipeline terlalu lama kalau memang ada masalah yang lebih serius/persisten. `retry_exponential_backoff=True` supaya jeda antar percobaan makin lama (1 menit, 2 menit, 4 menit) — memberi waktu lebih untuk masalah sementara "sembuh sendiri" tanpa membebani database dengan percobaan koneksi bertubi-tubi dalam interval terlalu pendek.

**2.** `extract_raw_data` versi Minggu 3 memakai `raise FileNotFoundError` kalau file belum ada — ini membuat task langsung **gagal** (dan menghentikan pipeline run hari itu sepenuhnya) begitu dicek sekali dan filenya belum ada, walau mungkin file itu memang sedang dalam proses "mendarat" (misal proses upload dari sumbernya belum selesai tepat saat DAG mulai jalan). **Sensor** (`FileSensor`) akan **menunggu** dan mengecek ulang secara berkala (`poke_interval`) sampai file benar-benar ada atau sampai `timeout` tercapai — perilaku yang lebih sesuai untuk situasi "file mungkin belum sampai, tapi akan sampai sebentar lagi", dibanding "gagal sekali cek langsung menyerah". Ini beda mendasar: task biasa cocok untuk kondisi yang **seharusnya sudah pasti benar** saat dijalankan, sensor cocok untuk kondisi yang **mungkin masih dalam proses** terpenuhi.

**3.**
```python
def send_alert(context):
    task_id = context["task_instance"].task_id
    exception = context.get("exception")

    if task_id == "data_quality_check":
        message = f"[ALERT] Data quality check gagal: {exception}"
    elif task_id == "load_to_warehouse":
        message = "[ALERT] Load ke warehouse gagal (kemungkinan koneksi database), sedang dicoba ulang otomatis."
    else:
        message = f"[ALERT] Task `{task_id}` gagal: {exception}"

    print(message)
```
`context["task_instance"].task_id` memberi tahu task mana yang memicu callback (karena `on_failure_callback` yang sama dipasang untuk semua task lewat `default_args`, perlu dicek `task_id`-nya untuk menyesuaikan pesan), dan `context.get("exception")` memberi detail pesan error aslinya untuk disertakan di alert.

**4.** `data_quality_check` yang gagal berarti data **memang tidak memenuhi kontrak yang disepakati** (aturan completeness/uniqueness/validity/consistency dari Hari 1–2) — ini kondisi yang **seharusnya tidak terjadi**, jadi pipeline berhenti sebagai sinyal "ada yang salah, perlu ditangani manusia" (fail-fast, `hari_1_data_quality_dimensions.md`). Branching sebaliknya menangani kondisi yang **valid dan diantisipasi** sebagai bagian normal dari kemungkinan skenario (mis. "hari ini kebetulan tidak ada transaksi baru sama sekali" bukanlah data yang rusak, cuma kondisi yang memang bisa terjadi) — pipeline **tidak gagal**, cuma memilih jalur kerja yang berbeda dan lanjut sebagai `skipped`, bukan `failed`. Intinya: kegagalan (`failed`) untuk kondisi yang **tidak seharusnya terjadi**, branching untuk kondisi yang **sah tapi butuh penanganan berbeda**.
