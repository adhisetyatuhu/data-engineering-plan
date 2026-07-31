---
title: Hari 4 - Batch vs Stream Processing
parent: Minggu 3 - Arsitektur Data & Big Data Pipeline
nav_order: 5
---

# Hari 4 — Batch Processing vs Stream Processing

*Kamis, 2 jam. Konsep level pengantar — implementasi streaming sungguhan (Kafka) baru masuk Minggu 4.*

## Tujuan Belajar

- [ ] Menjelaskan beda fundamental batch dan stream processing
- [ ] Menjelaskan trade-off latency vs throughput vs kompleksitas antara keduanya
- [ ] Mengenal istilah **micro-batch** sebagai titik tengah
- [ ] Memilih pendekatan yang tepat untuk skenario bisnis yang diberikan

## Untuk Instruktur: Mindset Shift

Analogi paling langsung buat developer: **batch processing itu seperti cron job**, **stream processing itu seperti event listener/webhook**.

```js
// Batch  ≈ cron job: jalan terjadwal, proses SEMUA data yang terkumpul sejak run terakhir
cron.schedule('0 1 * * *', () => processAllOrdersFromYesterday());

// Stream ≈ event listener: proses SATU event, saat itu juga, begitu terjadi
orderEmitter.on('order_placed', (order) => processOrderImmediately(order));
```

Developer yang sudah pernah pakai message queue (RabbitMQ/SQS) atau webhook akan langsung punya intuisi soal stream processing — istilah dan tools-nya beda (Kafka, Flink), tapi pola mentalnya sama: **reaksi terhadap event individual, bukan menunggu kumpulan data lalu diproses sekaligus**.

## Konsep & Sintaks

### Batch Processing

Data dikumpulkan dulu dalam periode waktu tertentu (per jam/per hari), baru diproses **sekaligus** dalam satu job.

```
[data jam 00:00-01:00] ---.
[data jam 01:00-02:00] ----+--> kumpul --> job batch (jalan 1x, jam 02:00) --> hasil
[data jam 02:00-03:00] ---'
```

- **Karakteristik**: latency tinggi (hasil baru "segar" setelah job jalan, bisa berjam-jam tertunda dari kejadian aslinya), tapi throughput tinggi & lebih murah dieksekusi (bisa proses jutaan baris sekaligus, optimasi lebih mudah karena tahu semua data yang harus diproses di depan).
- **Contoh use case**: laporan revenue harian, rekonsiliasi keuangan bulanan, training model ML dari data historis, pipeline `ecommerce-etl-pipeline` yang sedang dibangun minggu ini (DAG Airflow yang jalan **daily**, sesuai `minggu_3.md`).
- **Tools**: Apache Spark (mode batch), Airflow (menjadwalkan job batch).

### Stream Processing

Tiap **kejadian individual** (event) diproses segera setelah terjadi, tanpa menunggu dikumpulkan.

```
event 1 --> proses segera --> hasil 1
event 2 --> proses segera --> hasil 2
event 3 --> proses segera --> hasil 3
```

- **Karakteristik**: latency sangat rendah (hasil tersedia dalam detik/milidetik dari kejadian aslinya), tapi lebih kompleks dibangun & dioperasikan (harus menangani data yang datang terus-menerus, out-of-order events, sistem harus selalu "hidup" — beda dari batch job yang cuma jalan lalu selesai).
- **Contoh use case**: deteksi fraud transaksi kartu kredit (harus tahu **saat itu juga**, bukan besok), dashboard monitoring real-time, rekomendasi produk yang bereaksi terhadap klik user detik itu juga.
- **Tools**: Apache Kafka (message broker/event log), Apache Flink/Spark Structured Streaming (processing engine) — semua ini topik penuh di **Minggu 4**, hari ini baru pengantar konsepnya.

### Micro-batch — Titik Tengah

Beberapa sistem (termasuk **Spark Structured Streaming**) memakai pendekatan **micro-batch**: bukan benar-benar memproses 1 event pada satu waktu (itu disebut *native streaming*, dipakai Flink), tapi mengumpulkan event dalam jendela waktu yang **sangat pendek** (mis. tiap 1–10 detik), lalu diproses sebagai batch kecil berulang-ulang.

```
[event masuk 0-1 detik] --> proses sebagai batch kecil --> hasil (delay ~1 detik)
[event masuk 1-2 detik] --> proses sebagai batch kecil --> hasil (delay ~1 detik)
```

Ini kompromi praktis: latency jauh lebih rendah dari batch tradisional (menit/jam → detik), tapi lebih sederhana diimplementasikan & dioperasikan dibanding native streaming murni. Banyak kasus "real-time" di industri sebenarnya cukup dilayani micro-batch — native streaming murni biasanya baru dibutuhkan untuk kasus latency sangat ketat (sub-detik, seperti fraud detection finansial).

### Perbandingan

| | Batch | Micro-batch | Stream (native) |
|---|---|---|---|
| Latency | Menit–jam (bahkan hari) | Detik | Milidetik–detik |
| Throughput | Sangat tinggi | Tinggi | Bervariasi, umumnya lebih rendah per-event overhead |
| Kompleksitas bangun & operasikan | Rendah–sedang | Sedang | Tinggi (state management, out-of-order events, sistem harus selalu hidup) |
| Biaya infrastruktur | Bisa hemat (compute cuma jalan saat job aktif) | Sedang | Cenderung lebih mahal (compute harus selalu siap menerima event) |
| Contoh tools | Spark batch, Airflow scheduled DAG | Spark Structured Streaming | Apache Flink, Kafka Streams |

### Bagaimana Memilih

Pertanyaan kunci: **"seberapa segar hasilnya harus tersedia setelah kejadian aslinya terjadi, dan apa konsekuensi bisnis kalau terlambat?"**

- Kalau keterlambatan beberapa jam **tidak apa-apa** secara bisnis (laporan revenue harian tetap berguna walau baru "jadi" jam 2 pagi) → **batch**, karena lebih murah & lebih sederhana dioperasikan, tanpa kehilangan apapun secara bisnis.
- Kalau keterlambatan beberapa detik **masih bisa diterima**, tapi menit/jam sudah terlalu lambat (dashboard monitoring operasional) → **micro-batch**.
- Kalau keterlambatan bahkan beberapa detik **berdampak nyata** (deteksi fraud, sistem trading) → **stream (native)**, walau biaya kompleksitas & infrastrukturnya jauh lebih tinggi — dan biaya itu harus sepadan dengan nilai bisnisnya.

**Jangan pilih streaming "karena keren"** — ini kesalahan umum yang wajib ditekankan ke peserta (lihat "Kesalahan Umum" di bawah).

## Kesalahan Umum

1. **Memilih streaming padahal kebutuhan bisnisnya sebenarnya bisa dilayani batch.** Streaming menambah kompleksitas signifikan (infrastruktur harus selalu hidup, penanganan failure lebih rumit, state management, biaya operasional lebih tinggi) — kalau laporan yang telat beberapa jam saja sudah cukup, biaya kompleksitas ini tidak sepadan. Aturan praktis: **default ke batch**, baru pertimbangkan streaming kalau ada kebutuhan latency yang jelas dan terukur yang tidak bisa dipenuhi batch/micro-batch.
2. **Mengira micro-batch = stream murni.** Micro-batch tetap memproses **kumpulan** event dalam jendela waktu, bukan 1 event satu-satu — bedanya jendela waktunya sangat pendek. Untuk kebutuhan latency sub-detik yang ketat, ini masih belum cukup.
3. **Tidak mempertimbangkan biaya operasional streaming jangka panjang.** Sistem streaming harus terus berjalan (dan dipantau) 24/7, beda dari batch job yang cuma "hidup" saat dijalankan lalu selesai — ini biaya infrastruktur & operasional yang berkelanjutan, bukan biaya sekali bangun saja.
4. **Menganggap semua data di perusahaan harus diproses dengan cara yang sama.** Sama seperti ETL/ELT di Hari 3, kebanyakan perusahaan memakai **kombinasi**: mayoritas pipeline tetap batch, hanya use case spesifik yang benar-benar butuh latency rendah yang dijadikan stream.

## Latihan

1. Untuk tiap skenario berikut, tentukan batch, micro-batch, atau stream, dan jelaskan alasannya:
   a. Laporan penjualan mingguan untuk rapat manajemen tiap Senin pagi.
   b. Sistem yang memblokir transaksi kartu kredit mencurigakan **sebelum** transaksi selesai diproses.
   c. Dashboard jumlah pengunjung aktif di website saat ini (update tiap beberapa detik dianggap cukup).
   d. Menghitung ulang skor RFM customer (seperti di mini project Minggu 2) tiap malam.
2. Kenapa "biaya operasional yang lebih tinggi" jadi kelemahan nyata streaming, bukan cuma soal kompleksitas teknis saat membangunnya? Jelaskan dari sisi infrastruktur yang harus terus berjalan.
3. Jelaskan kenapa micro-batch bisa dianggap "kompromi" antara batch dan stream — sebutkan apa yang didapat dan apa yang dikorbankan dibanding masing-masing pendekatan murni.
4. Pipeline `ecommerce-etl-pipeline` yang sedang dibangun minggu ini (Airflow DAG dijadwalkan **daily**) itu contoh pendekatan apa? Apakah pilihan ini masuk akal untuk kasus analisis data e-commerce historis? Jelaskan.

## Kunci Jawaban & Pembahasan

**1.**
   a. **Batch.** Kebutuhan bisnisnya cuma butuh data "cukup segar" sampai Senin pagi — keterlambatan beberapa jam/hari tidak berdampak, dan batch jauh lebih murah & sederhana untuk kasus ini.
   b. **Stream (native).** Keputusan blokir harus terjadi **sebelum** transaksi selesai — keterlambatan bahkan beberapa detik sudah berarti transaksi fraud lolos. Ini kasus klasik yang benar-benar butuh latency sangat rendah, di mana biaya kompleksitas streaming sepadan dengan risiko bisnisnya.
   c. **Micro-batch.** "Beberapa detik dianggap cukup" adalah sinyal jelas kasus ini, bukan butuh sub-detik (stream native) dan bukan pula bisa menunggu batch berjam-jam.
   d. **Batch.** Skor RFM dihitung dari data historis yang terus terakumulasi (Recency/Frequency/Monetary) — tidak ada kebutuhan bisnis untuk update dalam hitungan detik, dan pola "dijalankan tiap malam" persis definisi batch job terjadwal.

**2.** Sistem batch cuma "menghidupkan" compute-nya saat job dijalankan (mis. 10 menit tiap malam) — di luar itu, tidak ada resource yang terpakai/dibayar. Sistem streaming sebaliknya harus **selalu siap** menerima & memproses event kapan saja (event bisa datang jam berapa saja, tidak terjadwal) — artinya compute (cluster Kafka/Flink) harus terus berjalan 24/7, bahkan di saat traffic sepi, dan perlu dipantau terus-menerus (alerting kalau consumer lag menumpuk, kalau salah satu node down, dst). Biaya "selalu hidup" ini konsisten ada setiap bulan, beda dari batch yang biayanya proporsional cuma saat job benar-benar jalan.

**3.** Yang **didapat** dibanding batch murni: latency jauh lebih rendah (detik, bukan jam), karena data diproses dalam jendela waktu pendek berulang-ulang, bukan menunggu satu periode besar terkumpul. Yang **dikorbankan** dibanding stream native: masih ada delay (walau kecil) karena tetap menunggu jendela waktu (bukan reaksi instan per-event), dan untuk kebutuhan latency benar-benar sub-detik, ini masih belum cukup. Dibanding stream native, yang **didapat**: kompleksitas implementasi & operasional lebih rendah (tidak perlu menangani state per-event yang rumit, lebih mudah di-debug karena tetap berbasis "batch kecil" yang lebih mudah dinalar). Yang **dikorbankan**: latency tetap lebih tinggi dari stream native murni.

**4.** Ini **batch processing** — DAG dijadwalkan jalan sekali sehari (`daily`), memproses semua data yang terkumpul sejak run sebelumnya, bukan bereaksi ke tiap transaksi individual saat terjadi. Ini **masuk akal** untuk konteks mini project: analisis e-commerce historis (revenue trend, customer segmentation, top produk) tidak butuh update real-time — keterlambatan sampai hari berikutnya tidak mengubah kualitas insight bisnisnya, dan batch jauh lebih sederhana dibangun untuk tujuan belajar minggu ini. Kalau nanti ada kebutuhan spesifik seperti "notifikasi instan saat stok produk hampir habis", itu baru kandidat use case streaming — dan memang jadi topik pengantar (demo, bukan implementasi penuh) di Minggu 4.
