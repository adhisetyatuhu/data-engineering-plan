---
title: Hari 4 - Pengantar Kafka
parent: Minggu 4 - Data Quality, Orchestration & Streaming
nav_order: 5
---

# Hari 4 — Pengantar Streaming: Konsep Kafka (Topic, Producer, Consumer, Partition)

*Kamis, 2 jam. Level konsep — implementasi producer/consumer sungguhan baru Minggu (hands-on), lihat `latihan_dq_streaming_mini_project.md`.*

> Sambungan langsung dari `materi/minggu_3/hari_4_batch_stream.md` — hari itu membahas **kapan** memilih streaming, hari ini membahas **bagaimana** Kafka (tool streaming paling umum dipakai industri) bekerja secara konsep.

## Tujuan Belajar

- [ ] Menjelaskan Kafka sebagai **event log terdistribusi**, bukan sekadar message queue biasa
- [ ] Menjelaskan istilah: broker, topic, partition, producer, consumer, consumer group, offset
- [ ] Menjelaskan beda Kafka dengan message queue tradisional (RabbitMQ/SQS) dari sisi model konsumsi pesan
- [ ] Menjelaskan kenapa partition penting untuk skalabilitas & urutan pesan

## Untuk Instruktur: Mindset Shift

Developer yang pernah pakai message queue (RabbitMQ, SQS, atau bahkan Redis pub/sub) akan punya intuisi **sebagian** benar tentang Kafka — tapi ada satu perbedaan fundamental yang wajib diluruskan di awal:

**Message queue tradisional**: pesan **dihapus/diambil** begitu dikonsumsi (consume-once semantics — mirip antrian fisik, begitu diambil, hilang dari antrian).

**Kafka**: pesan **tetap tersimpan** di topic (sebagai log, dengan retention period yang bisa dikonfigurasi — bisa hari, minggu, bahkan permanen), consumer cuma **mencatat posisi baca mereka sendiri** (offset). Beberapa consumer group **berbeda** bisa membaca **pesan yang sama**, dari titik yang berbeda-beda, tanpa saling mempengaruhi.

Analogi paling pas: message queue tradisional itu seperti **kotak surat fisik** (surat diambil, hilang dari kotak). Kafka itu seperti **rekaman siaran/log file yang bisa diputar ulang (`git log`)** — semua orang yang punya akses bisa "membaca ulang" dari titik manapun dalam sejarahnya, dan membaca tidak menghapus apapun dari log itu.

## Konsep & Sintaks

### Istilah Inti

| Istilah | Definisi | Analogi |
|---|---|---|
| **Broker** | 1 server Kafka yang menyimpan data & melayani request. Kumpulan broker = **cluster** | 1 instance database dalam cluster database terdistribusi |
| **Topic** | Kategori/nama "saluran" tempat pesan dikirim & dibaca (mis. `ecommerce-transactions`) | Nama tabel, atau nama channel pub/sub |
| **Partition** | Topic dibagi jadi beberapa partition — tiap partition adalah log terurut sendiri | Sharding di database — 1 topic "dipecah" biar bisa diproses paralel |
| **Producer** | Aplikasi/script yang **menulis** pesan ke topic | Klien yang melakukan `INSERT` |
| **Consumer** | Aplikasi/script yang **membaca** pesan dari topic | Klien yang melakukan `SELECT` berkelanjutan |
| **Consumer Group** | Sekumpulan consumer yang **berbagi** beban baca 1 topic (tiap partition cuma dibaca 1 consumer dalam grup yang sama) | Worker pool yang membagi tugas dari 1 antrian |
| **Offset** | Posisi/nomor urut pesan dalam 1 partition — penanda "sudah baca sampai mana" | Bookmark halaman terakhir yang dibaca |

### Topic & Partition

```
Topic: ecommerce-transactions

Partition 0: [msg0] [msg1] [msg2] [msg3] --> (urutan terjamin DALAM partition ini)
Partition 1: [msg0] [msg1] [msg2]        --> (urutan terjamin DALAM partition ini)
Partition 2: [msg0] [msg1] [msg2] [msg3] [msg4]
```

Poin krusial yang sering salah dipahami pemula: **Kafka menjamin urutan pesan HANYA di dalam 1 partition, bukan di seluruh topic.** Kalau ada 3 partition, dan pesan-pesan untuk 1 customer tertentu tersebar acak di ke-3 partition itu, **tidak ada jaminan** consumer membacanya dalam urutan waktu aslinya.

**Solusinya**: pesan yang perlu urutannya terjamin (mis. semua transaksi 1 customer yang sama) dikirim dengan **key** yang sama (mis. `customer_id`) — Kafka menjamin pesan dengan key yang sama **selalu masuk ke partition yang sama**, sehingga urutannya terjaga untuk key itu. Ini keputusan desain yang harus diambil producer, bukan otomatis.

### Kenapa Partition Penting untuk Skalabilitas

Karena tiap partition adalah log independen, **banyak consumer** (dalam 1 consumer group) bisa membaca partition-partition yang berbeda **secara paralel** — 1 consumer per partition (maksimal). Kalau topic punya 3 partition, maksimal 3 consumer dalam 1 grup bisa bekerja paralel membaca topic itu; consumer ke-4 dalam grup yang sama akan menganggur (idle) karena tidak ada partition tersisa untuknya.

```
Consumer Group "revenue-aggregator":
  Consumer A -> baca Partition 0
  Consumer B -> baca Partition 1
  Consumer C -> baca Partition 2
  (kalau ditambah Consumer D -> menganggur, tidak ada partition ke-4)
```

Ini alasan jumlah partition sebuah topic jadi keputusan penting sejak awal (walau bisa ditambah belakangan, tidak bisa dikurangi) — jumlah partition menentukan **batas atas** paralelisme konsumsi topic itu.

### Consumer Group — Broadcast vs Load Balancing

- **Consumer dalam grup yang SAMA** → beban dibagi (tiap pesan cuma diproses **1 kali** oleh salah satu consumer di grup itu) — pola load balancing, cocok untuk kasus seperti "banyak worker memproses transaksi paralel".
- **Consumer di grup yang BERBEDA**, membaca topic yang sama → masing-masing grup menerima **salinan penuh** semua pesan (broadcast) — cocok untuk kasus "tim fraud detection dan tim revenue reporting sama-sama butuh baca semua transaksi yang sama, untuk tujuan berbeda-beda, tanpa saling mengganggu."

Ini kemampuan yang **tidak dimiliki** message queue tradisional consume-once: di Kafka, 2 sistem yang sama sekali berbeda bisa membaca ulang riwayat pesan yang sama secara independen, karena pesan tidak "hilang" setelah dibaca satu pihak.

### Offset — Bookmark yang Dikelola Consumer

Kafka **tidak** melacak "pesan ini sudah dibaca semua orang" secara global — yang dilacak adalah **offset per consumer group per partition** ("consumer group X sudah baca sampai posisi ke-N di partition ini"). Ini kenapa consumer group berbeda bisa punya progress baca yang sama sekali berbeda terhadap topic yang sama, dan kenapa 1 consumer group bisa "mundur" (reset offset ke posisi lebih awal) untuk memproses ulang histori pesan — berguna kalau ada bug di logic consumer yang perlu diperbaiki lalu diproses ulang dari awal.

## Kesalahan Umum

1. **Mengira Kafka = message queue biasa yang "lebih cepat".** Perbedaan intinya bukan soal kecepatan, tapi **model retensi & konsumsi pesan** (log yang bisa dibaca ulang vs antrian consume-once) — ini yang membuka use case sama sekali baru (banyak sistem independen baca ulang histori yang sama), bukan cuma soal performa.
2. **Mengira urutan pesan terjamin di seluruh topic.** Cuma terjamin **per partition** — kalau urutan lintas semua pesan penting, harus didesain lewat pemilihan **key** yang tepat saat produce, bukan asumsi otomatis dari Kafka.
3. **Menambah consumer melebihi jumlah partition dengan harapan makin cepat.** Consumer ekstra dalam grup yang sama akan menganggur kalau semua partition sudah "dijatah" ke consumer lain — paralelisme maksimal dibatasi jumlah partition, bukan jumlah consumer yang dijalankan.
4. **Mengira 1 topic harus cuma dibaca 1 sistem.** Salah satu kekuatan utama Kafka justru banyak sistem independen (consumer group berbeda) bisa membaca topic yang sama untuk tujuan masing-masing, tanpa perlu producer tahu/peduli siapa saja yang membaca.

## Latihan

1. Sebuah topic `order-events` dipakai bersama oleh 2 tim: tim **inventory** (update stok tiap ada order) dan tim **analytics** (agregasi revenue harian). Jelaskan setup consumer group yang tepat untuk kedua tim ini, dan kenapa mereka **tidak boleh** berada di consumer group yang sama.
2. Topic `ecommerce-transactions` dibuat dengan 4 partition, tapi cuma dijalankan 1 consumer (dalam 1 consumer group). Apakah ini valid? Apa konsekuensinya dibanding menjalankan 4 consumer?
3. Sebuah tim memproduksi pesan transaksi ke Kafka tanpa menentukan key (jadi Kafka mendistribusikan pesan ke partition secara round-robin/acak). Belakangan mereka sadar butuh urutan transaksi per customer terjamin untuk deteksi fraud. Apa yang perlu diubah, dan kenapa?
4. Kenapa "consumer bisa mundur/reset offset untuk membaca ulang histori" adalah kemampuan yang sangat berguna untuk kasus **memperbaiki bug** di logic consumer? Bandingkan dengan message queue tradisional yang sudah menghapus pesan begitu dikonsumsi.

## Kunci Jawaban & Pembahasan

**1.** Tim inventory dan tim analytics harus berada di **consumer group yang berbeda** (mis. `inventory-service` dan `analytics-aggregator`) — masing-masing grup akan menerima **salinan lengkap** semua pesan di `order-events`, membaca & memproses secara independen sesuai kebutuhan masing-masing (update stok vs agregasi revenue), tanpa saling mengambil "jatah" pesan satu sama lain. Kalau keduanya ditaruh di **grup yang sama**, pesan akan **dibagi** di antara mereka (load balancing) — artinya tim inventory cuma akan menerima **sebagian** transaksi (sisanya "diambil" oleh consumer tim analytics), yang jelas salah karena kedua tim sama-sama butuh melihat **semua** transaksi.

**2.** **Valid** — tidak ada aturan yang mewajibkan jumlah consumer harus sama dengan jumlah partition. Konsekuensinya: 1 consumer itu akan membaca dari **keempat** partition secara bergantian (Kafka client menangani ini otomatis), tapi **tidak ada paralelisme** — semua pemrosesan tetap sekuensial di 1 proses consumer itu, walau datanya sendiri terdistribusi di 4 partition. Menjalankan 4 consumer (dalam grup yang sama) akan membagi keempat partition itu 1-per-consumer, memungkinkan pemrosesan **paralel sungguhan** — throughput total berpotensi 4x lebih tinggi (tergantung apakah bottleneck sebenarnya ada di pemrosesan consumer atau di tempat lain).

**3.** Perlu diubah: producer harus mulai menyertakan **key** (mis. `customer_id`) saat mengirim tiap pesan (`producer.send(topic, key=customer_id, value=event)`), bukan mengirim tanpa key. Dengan key yang konsisten, Kafka akan selalu mengarahkan pesan dengan `customer_id` yang sama ke **partition yang sama**, sehingga urutan pesan untuk customer tersebut (across waktu) terjamin sesuai urutan pengiriman — ini prasyarat mutlak untuk deteksi fraud yang butuh melihat urutan kejadian transaksi per customer secara akurat. Perlu diingat: perubahan ini cuma memengaruhi pesan **baru**, pesan lama yang sudah terlanjur tersebar tanpa key tetap seperti semula di partition asalnya.

**4.** Karena Kafka menyimpan pesan sebagai log yang **tidak hilang** setelah dibaca (beda dari message queue tradisional yang menghapus pesan begitu di-consume), kalau ditemukan bug di logic consumer (mis. rumus agregasi revenue yang salah selama seminggu terakhir), tim cukup **reset offset** consumer group itu mundur ke titik sebelum bug diperkenalkan, lalu jalankan ulang consumer yang sudah diperbaiki — semua pesan yang "salah diproses" sebelumnya bisa **diproses ulang dari log aslinya**, menghasilkan hasil yang benar tanpa kehilangan data apapun. Di message queue tradisional, begitu pesan sudah dikonsumsi (dan salah diproses karena bug), pesan itu **hilang selamanya** dari antrian — memperbaiki dampak bug jadi jauh lebih sulit (butuh sumber data cadangan lain untuk memproses ulang, kalau ada).
