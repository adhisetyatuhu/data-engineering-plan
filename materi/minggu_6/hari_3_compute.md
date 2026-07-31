---
title: Hari 3 - Compute (VM & Serverless)
parent: Minggu 6 - Cloud Platform Fundamentals
nav_order: 4
---

# Hari 3 — Compute: Virtual Machine Dasar & Serverless Functions

*Rabu, 2 jam. Konsep + contoh kode — VM dibahas ringkas (jarang dipakai langsung oleh data engineer sehari-hari), serverless functions lebih dalam karena lebih relevan.*

## Tujuan Belajar

- [ ] Menjelaskan apa itu Virtual Machine dan kapan data engineer benar-benar perlu mengurusnya langsung
- [ ] Menjelaskan model serverless functions dan bedanya dari VM/container yang selalu menyala
- [ ] Merancang trigger event-driven sederhana (Cloud Function dipicu upload file ke bucket)
- [ ] Menghubungkan konsep serverless ke event-driven architecture yang sudah dipelajari (Kafka, Minggu 4)

## Untuk Instruktur: Mindset Shift

Poin penting yang perlu ditekankan **di awal**: dibanding software engineer backend pada umumnya, **data engineer modern jarang provisioning VM secara manual**. Sebagian besar workload data engineering (Spark, Airflow, warehouse) sekarang dijalankan lewat layanan **managed** (Cloud Composer untuk Airflow, Dataproc/serverless Spark, BigQuery) yang beroperasi di atas VM **di balik layar**, tanpa data engineer perlu menyentuhnya langsung. VM tetap penting **dipahami** (karena masih jadi fondasi banyak layanan lain, dan kadang tetap dibutuhkan untuk kasus khusus), tapi bobotnya di materi ini sengaja lebih ringan dibanding serverless functions, yang jauh lebih sering dipakai langsung oleh data engineer untuk automasi ringan (trigger pipeline, notifikasi, transformasi kecil).

Analogi serverless yang efektif: VM/container yang selalu menyala itu seperti **karyawan tetap** (digaji terus walau sedang tidak ada kerjaan). Serverless function itu seperti **freelancer yang dipanggil per-tugas** — cuma "dibayar" (dan cuma "ada") saat benar-benar ada event yang memicunya, langsung "pulang" begitu tugasnya selesai.

## Konsep & Sintaks

### Virtual Machine (Compute Engine/EC2) — Ringkas

VM adalah komputer virtual yang kamu sewa — kamu pilih spesifikasi (CPU, RAM, disk), pilih image OS (Ubuntu, dsb), lalu **kamu** yang bertanggung jawab atas semua yang berjalan di dalamnya (persis level tanggung jawab IaaS di `hari_1_konsep_cloud.md`).

```bash
gcloud compute instances create airflow-vm \
  --zone=asia-southeast1-a \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud
```

**Kapan data engineer tetap butuh VM langsung**: menjalankan tool yang belum ada versi managed-nya di provider yang dipakai, kebutuhan kontrol penuh atas environment (versi library/OS spesifik), atau alasan biaya untuk beban kerja yang benar-benar konstan 24/7 dalam volume besar (kadang VM dengan committed use discount lebih murah dari layanan managed setara — kembali ke pembahasan elastisitas di `hari_1_konsep_cloud.md` Latihan #4).

### Serverless Functions (Cloud Functions/Lambda)

Kode yang dijalankan **cuma saat dipicu event tertentu** — tidak ada server yang "menyala" menunggu, provider yang urus penyediaan compute saat event terjadi, lalu melepaskannya lagi setelah selesai.

```python
# main.py -- Cloud Function dipicu event upload ke Cloud Storage
import functions_framework

@functions_framework.cloud_event
def on_new_raw_file(cloud_event):
    data = cloud_event.data
    bucket_name = data["bucket"]
    file_name = data["name"]

    if not file_name.startswith("raw/"):
        return   # bukan file yang relevan, abaikan

    print(f"File baru terdeteksi: gs://{bucket_name}/{file_name}")
    # contoh aksi: trigger DAG Airflow lewat REST API, atau validasi cepat sebelum pipeline utama jalan
```

```bash
gcloud functions deploy on_new_raw_file \
  --runtime=python312 \
  --trigger-bucket=ecommerce-data-lake \
  --entry-point=on_new_raw_file
```

### Karakteristik Serverless

| Karakteristik | Penjelasan |
|---|---|
| **Event-driven** | Dijalankan sebagai respons terhadap event (file baru di bucket, pesan masuk ke topic, HTTP request) — bukan berjalan terus menunggu |
| **Stateless** | Tiap invocation dianggap independen, tidak ada state yang bertahan otomatis antar pemanggilan (state harus disimpan eksternal — database, bucket — kalau dibutuhkan) |
| **Auto-scaling** | Kalau ada 100 event bersamaan, provider otomatis menjalankan banyak instance function secara paralel — tidak perlu dikonfigurasi manual |
| **Pay-per-invocation** | Biaya berdasarkan **jumlah pemanggilan** dan waktu eksekusi, bukan waktu resource menyala — kalau tidak ada event, biayanya nol |
| **Cold start** | Invocation **pertama** setelah idle lama butuh waktu ekstra untuk "menyalakan" environment-nya — trade-off dari sifat "tidak selalu menyala" |

### Menghubungkan ke Event-Driven Architecture (Kafka, Minggu 4)

Serverless functions dan Kafka consumer (`materi/minggu_4/hari_4_pengantar_kafka.md`) sama-sama **event-driven**, tapi levelnya beda:

| | Kafka Consumer | Serverless Function |
|---|---|---|
| Selalu "mendengarkan"? | Ya — proses consumer tetap berjalan terus, menunggu pesan baru | Tidak — cuma "hidup" saat event benar-benar terjadi, provider yang memicunya |
| Skala kontrol | Kamu atur sendiri berapa banyak consumer/partition | Provider yang otomatis scale, kamu tidak perlu pikirkan |
| Cocok untuk | Volume event tinggi & terus-menerus, butuh kontrol detail (consumer group, offset) | Event jarang/tidak terduga, logic ringan, tidak mau mengurus infrastruktur listener sama sekali |

Contoh konkret relevansi ke pipeline `ecommerce-etl-pipeline`: kalau file raw baru **tidak** datang di jadwal tetap (`@daily`) tapi **kapan saja** tidak terduga (mis. sistem sumber upload file begitu ada batch baru selesai diproses di sisi mereka), Cloud Function yang dipicu event upload bucket adalah cara yang jauh lebih natural untuk memulai pipeline dibanding Airflow `FileSensor` yang terus-menerus polling (`materi/minggu_4/hari_3_airflow_lanjutan.md`) — trigger langsung dari event, bukan mengecek berkala.

## Kesalahan Umum

1. **Mengira serverless function bisa menyimpan state antar pemanggilan** (mis. variabel global yang diasumsikan tetap ada di invocation berikutnya). Karena stateless & bisa dijalankan di instance/mesin fisik berbeda tiap kali, apapun yang perlu "diingat" antar pemanggilan harus disimpan eksternal (database, bucket), bukan di memory function itu sendiri.
2. **Memakai serverless function untuk proses yang butuh berjalan lama/berat** (mis. transformasi Spark skala besar). Serverless function biasanya punya **batas waktu eksekusi maksimum** (beberapa menit) dan resource terbatas — cocok untuk logic ringan/singkat (validasi cepat, trigger pipeline lain, notifikasi), bukan pengganti Spark job yang berjalan lama.
3. **Mengabaikan cold start untuk kasus yang sensitif latency.** Kalau function jarang dipanggil (idle lama antar invocation), invocation pertama bisa terasa lambat — untuk kasus yang butuh respons cepat konsisten, ini perlu dipertimbangkan (beberapa provider punya opsi "minimum instance" yang tetap menyala untuk menghindari cold start, dengan trade-off biaya tambahan).
4. **Menjalankan VM untuk tugas yang sebenarnya cocok serverless** (mis. VM yang cuma dipakai untuk menjalankan 1 script kecil sekali sehari, tapi dibiarkan menyala 24/7). Ini pemborosan biaya nyata — kembali ke prinsip elastisitas yang dibahas `hari_1_konsep_cloud.md`.

## Latihan

1. Untuk tiap skenario berikut, tentukan apakah lebih cocok VM, serverless function, atau tidak keduanya (mungkin managed service lain): (a) menjalankan Spark job transformasi yang makan waktu 45 menit tiap hari, (b) mengirim notifikasi Slack setiap kali `data_quality_check` gagal, (c) menjalankan warehouse query analitik ad-hoc.
2. Jelaskan kenapa "cold start" adalah trade-off yang **sepadan** untuk kebanyakan use case serverless, meski terdengar seperti kekurangan.
3. Rancang skenario konkret di `ecommerce-etl-pipeline` di mana Cloud Function yang dipicu event upload bucket lebih masuk akal dibanding menjadwalkan DAG Airflow `@daily`.
4. Seorang rekan mengusulkan menaruh **seluruh** logic `clean_transform.py` (PySpark, `materi/minggu_3/latihan_pipeline_mini_project.md`) ke dalam 1 Cloud Function. Jelaskan kenapa ini kemungkinan besar **bukan** desain yang tepat.

## Kunci Jawaban & Pembahasan

**1.** (a) **Tidak keduanya secara langsung** — 45 menit kemungkinan melebihi batas waktu eksekusi serverless function, dan menjalankan VM manual untuk 1 job terjadwal juga bukan pilihan paling efisien; yang paling tepat adalah **managed data processing service** (mis. Dataproc Serverless untuk Spark) yang dirancang khusus untuk beban kerja seperti ini — kompute disediakan otomatis cuma selama job berjalan, tanpa kamu perlu mengurus VM sendiri. (b) **Serverless function** — logic ringan (kirim pesan ke Slack API), dipicu event (kegagalan task), durasi singkat, jarang terjadi (cuma saat ada kegagalan) — kandidat serverless yang ideal. (c) **Tidak keduanya** — ini pekerjaan **cloud data warehouse** (BigQuery, `hari_5_cloud_data_warehouse.md`) yang memang dirancang khusus untuk query analitik, bukan compute generik seperti VM/serverless function.

**2.** Cold start adalah konsekuensi **langsung** dari sifat yang membuat serverless murah & efisien: instance **tidak menyala** saat tidak ada event, jadi wajar butuh waktu "menyalakan" saat event pertama tiba setelah idle. Trade-off ini sepadan untuk **mayoritas** use case (trigger pipeline, notifikasi, validasi ringan) karena delay beberapa detik ekstra sesekali jauh lebih murah/simpel dibanding membayar terus-menerus untuk instance yang menyala 24/7 hanya demi menghindari delay itu — terutama kalau frekuensi event memang jarang (kalau sering dipanggil, instance cenderung tetap "hangat" dan cold start jarang terjadi). Trade-off ini **tidak sepadan** untuk use case yang butuh latency rendah **konsisten** setiap saat (mis. API yang melayani user langsung) — di situ opsi "minimum instance" (bayar lebih untuk menghindari cold start) atau pindah ke layanan yang selalu menyala baru masuk akal.

**3.** Skenario: sumber data eksternal (mis. vendor pihak ketiga) mengirim file transaksi ke bucket **kapan saja mereka selesai memprosesnya di sisi mereka** — tidak ada jadwal tetap yang bisa diandalkan (kadang jam 3 pagi, kadang jam 11 siang, kadang terlambat 1 hari). Menjadwalkan DAG `@daily` di jam tetap berisiko **kepagian** (file belum ada, mirip masalah yang dibahas `materi/minggu_4/hari_3_airflow_lanjutan.md` soal `FileSensor`) atau **kesorean** (file sudah ada dari beberapa jam lalu, tapi pipeline baru jalan sesuai jadwal, menambah latency yang tidak perlu). Cloud Function yang dipicu **langsung** oleh event upload bucket bisa memulai pipeline **segera** setelah file benar-benar tersedia, tanpa perlu polling terjadwal maupun menebak jadwal yang tepat.

**4.** Alasan utama: `clean_transform.py` menjalankan **PySpark** (`materi/minggu_3/hari_5_spark.md`) yang butuh JVM, resource memori signifikan untuk memproses ratusan ribu baris, dan waktu eksekusi yang kemungkinan melebihi batas serverless function (dibahas di poin Kesalahan Umum #2). Cloud Function dirancang untuk logic **ringan & singkat**, bukan job pemrosesan data terdistribusi — memaksakan Spark ke dalamnya kemungkinan besar akan gagal karena batas resource/waktu, atau kalaupun berhasil untuk data kecil, tidak akan scale untuk volume data yang lebih besar. Desain yang lebih tepat: biarkan transformasi berat tetap di layanan yang memang dirancang untuk itu (Dataproc Serverless, atau tetap dijalankan lewat Airflow seperti sekarang), dan pakai Cloud Function murni untuk bagian **event-driven ringan** di sekitarnya (mis. memicu DAG begitu file baru terdeteksi, seperti Latihan #3) — bukan menggantikan seluruh pipeline transformasinya.
