---
title: Hari 2 - Tipe-Tipe NoSQL
parent: Minggu 8 - NoSQL & Storage Strategy (Capstone)
nav_order: 3
---

# Hari 2 — Tipe-Tipe NoSQL: Key-Value, Document, Column-Family, Graph

*Selasa, 2 jam. "NoSQL" bukan 1 tipe database — sesi ini memetakan 4 tipe utama & kapan masing-masing dipilih.*

## Tujuan Belajar

- [ ] Menjelaskan 4 tipe utama NoSQL: key-value, document, column-family, graph
- [ ] Menjelaskan use case yang paling pas untuk masing-masing tipe
- [ ] Menempatkan MongoDB (document) dan Redis (key-value) — 2 tool yang akan dipakai minggu ini — dalam peta ini
- [ ] Menjelaskan kenapa 1 kebutuhan (mis. e-commerce) sering butuh **lebih dari 1** tipe NoSQL sekaligus (polyglot)

## Untuk Instruktur

Sesi ini murni peta konsep, bukan hands-on (hands-on MongoDB & Redis mulai `hari_3`/`hari_4`). Manfaatkan untuk membangun kerangka **kapan pakai tipe apa** sebelum peserta terjun ke syntax — supaya nanti saat menulis kode MongoDB/Redis, mereka paham **kenapa** tool itu yang dipilih untuk kasus tertentu, bukan cuma mengikuti instruksi.

## Konsep & Sintaks

### Peta 4 Tipe Utama NoSQL

```
                    NoSQL
    ┌───────────┬───────────┬───────────┬───────────┐
    │ Key-Value │ Document  │ Column-   │ Graph     │
    │           │           │ Family    │           │
    │  Redis    │ MongoDB   │ Cassandra │ Neo4j     │
    └───────────┴───────────┴───────────┴───────────┘
     Paling         Skema         Write-      Relasi
     sederhana,     fleksibel     heavy,      kompleks
     paling cepat   per dokumen   scale       antar
                                  besar       entitas
```

| Tipe | Model Data | Contoh Tool | Use Case Ideal |
|---|---|---|---|
| **Key-Value** | Pasangan `key → value` sederhana, value biasanya "buram" (tidak di-query berdasarkan isinya) | Redis, DynamoDB | Caching, session store, rate limiting — akses **super cepat** berdasarkan key yang sudah diketahui |
| **Document** | Dokumen semi-terstruktur (biasanya JSON/BSON), field bisa bervariasi antar dokumen dalam koleksi yang sama | MongoDB, Couchbase | Data dengan struktur bervariasi/hierarkis (katalog produk, profil user dengan atribut beda-beda) |
| **Column-Family** | Mirip tabel, tapi dioptimalkan untuk **menulis** dalam volume sangat besar & terdistribusi, kolom bisa berbeda per baris | Cassandra, HBase | Time-series/log skala masif, write-heavy workload (data sensor IoT, event tracking volume tinggi) |
| **Graph** | Node (entitas) & edge (relasi) sebagai warga kelas satu, dioptimalkan untuk **traversal relasi kompleks** | Neo4j, Amazon Neptune | Rekomendasi produk berbasis relasi ("orang yang membeli X juga membeli Y, yang terhubung ke Z..."), social network, fraud detection |

### Key-Value: Paling Sederhana, Paling Cepat

Model paling minimal: `SET key value` dan `GET key` — tidak ada query berdasarkan **isi** value (value diperlakukan seperti kotak hitam, kecuali dipakai struktur data khusus seperti Redis hash/list/set). Karena modelnya sesederhana ini, key-value store bisa jadi **sangat** cepat (biasanya beroperasi penuh di memory, seperti Redis) — trade-off-nya: kalau butuh mencari berdasarkan **isi** data (bukan cuma key yang sudah diketahui), key-value store bukan pilihan tepat.

**Analogi**: locker penyimpanan di stasiun — kamu **harus** tahu nomor locker-nya (key) untuk mengambil isinya (value); tidak ada cara "cari locker yang isinya baju warna biru" tanpa tahu nomornya dulu.

### Document: Struktur Fleksibel, Tetap Bisa Di-Query

Beda dari key-value, document store menyimpan **struktur** (biasanya JSON/BSON) yang **bisa** di-query berdasarkan isi field-nya (`db.products.find({category: "Electronics"})`) — tapi field antar dokumen dalam 1 koleksi **tidak wajib** identik, beda fundamental dari kolom tabel RDBMS yang kaku.

```json
// dokumen 1 (elektronik) dan dokumen 2 (fashion) hidup di koleksi yang SAMA
{ "stock_code": "85123A", "description": "T-LIGHT HOLDER", "category": "Home Decor", "material": "glass" }
{ "stock_code": "22423",  "description": "TOTE BAG",       "category": "Bags",       "size_variant": ["S", "M", "L"] }
```

Detail penuh (embedding vs referencing, kapan pakai document) di `hari_3_mongodb.md`.

### Column-Family: Dioptimalkan untuk Write Besar & Terdistribusi

Sekilas mirip tabel (baris & kolom), tapi didesain sejak awal untuk **scale out** ke ratusan node dan menampung **volume tulis** yang sangat tinggi (bukan cuma volume baca) — tiap baris **bisa** punya kolom berbeda (fleksibel seperti document, tapi model penyimpanan fisiknya dioptimalkan per-kolom, bukan per-baris). Contoh kasus nyata: menyimpan data sensor IoT dari jutaan perangkat yang mengirim data tiap detik, atau log event skala sangat besar. **Di luar cakupan hands-on minggu ini** (tidak dipakai di mini project) — disebut di sini supaya peta 4 tipe NoSQL lengkap, dan dikenali kalau muncul dalam konteks kerja nyata nanti.

### Graph: Relasi sebagai Warga Kelas Satu

RDBMS **bisa** menyimpan relasi lewat foreign key & `JOIN` (seperti star schema Minggu 3), tapi untuk **relasi yang sangat kompleks & berlapis** (traversal beberapa "hop" — "teman dari teman yang juga membeli produk yang sama"), `JOIN` berulang jadi mahal & rumit ditulis. Graph database menyimpan relasi (edge) sebagai struktur data **utama**, bukan sekadar foreign key — traversal lintas banyak relasi jadi jauh lebih natural & cepat. Contoh kasus nyata: sistem rekomendasi ("orang yang beli X juga beli Y"), deteksi fraud (pola transaksi mencurigakan antar banyak akun terhubung). **Di luar cakupan hands-on minggu ini** juga — disebut untuk melengkapi peta.

### Kenapa 1 Sistem Sering Butuh Lebih dari 1 Tipe (Polyglot)

Kasus e-commerce di repo `ecommerce-etl-pipeline` sendiri adalah contoh nyata: **RDBMS** (PostgreSQL/BigQuery) untuk `fact_sales`/`dim_*` yang terstruktur & butuh `JOIN` analitik, **Document** (MongoDB) untuk katalog produk yang atributnya bervariasi per kategori, **Key-Value** (Redis) untuk caching hasil query yang sering diakses. Tidak ada 1 tool yang optimal untuk **semua** kebutuhan ini sekaligus — memaksakan 1 tool untuk semuanya berarti mengorbankan optimalitas di sebagian kebutuhan demi kesederhanaan operasional. Trade-off sebaliknya (banyak tool = lebih optimal per kebutuhan, tapi lebih kompleks dioperasikan/dijaga konsistensinya) adalah inti keputusan yang dibahas `hari_5_storage_strategy.md`.

## Kesalahan Umum

1. **Memilih tipe NoSQL berdasarkan "yang paling populer/sering disebut", bukan karakteristik data & pola akses.** MongoDB terkenal tidak berarti dia jawaban tepat untuk semua kasus non-RDBMS — kalau kebutuhannya cuma caching key-value sederhana, MongoDB jadi over-engineered dibanding Redis.
2. **Mengira document store "menggantikan" RDBMS sepenuhnya untuk data yang punya relasi.** Document store **bisa** menyimpan relasi (lewat referencing, `hari_3_mongodb.md`), tapi tidak seefisien RDBMS untuk relasi kompleks multi-arah — `fact_sales` yang berelasi ke 3 dimension table sekaligus tetap lebih natural di RDBMS.
3. **Mengira Redis cuma bisa menyimpan string sederhana.** Redis punya struktur data lebih kaya dari sekadar key-value string murni: hash, list, set, sorted set — dibahas detail `hari_4_redis.md`.
4. **Menganggap "pakai banyak tipe database sekaligus" selalu lebih baik tanpa pertimbangan biaya operasional.** Tiap tool tambahan = tambahan kompleksitas: perlu di-monitor, di-backup, dijaga konsistensi datanya dengan sumber lain — keputusan polyglot harus didasari kebutuhan nyata, bukan "supaya portfolio terlihat lebih banyak tool".

## Latihan

1. Untuk masing-masing kasus berikut, tentukan tipe NoSQL yang paling cocok (key-value/document/column-family/graph), dan jelaskan alasannya: (a) menyimpan session login user yang harus diakses dalam hitungan milidetik, (b) katalog produk marketplace dengan atribut sangat bervariasi antar kategori, (c) rekomendasi "pengguna yang membeli ini juga membeli itu" berdasarkan jaringan pembelian yang kompleks, (d) menyimpan data sensor IoT dari 1 juta perangkat yang mengirim pembacaan tiap detik.
2. Kenapa `dim_product` (Minggu 3, `stock_code`/`description`) **belum** benar-benar butuh MongoDB selama roadmap ini berjalan (Minggu 1-7) — apa yang berubah di Minggu 8 sehingga MongoDB sekarang jadi relevan (lihat kembali `minggu_8.md` Tahap 1)?
3. Redis (key-value) dan MongoDB (document) sama-sama akan dipakai minggu ini. Jelaskan kenapa keduanya **tidak saling menggantikan** — kalau MongoDB sudah bisa menyimpan data fleksibel, kenapa masih butuh Redis terpisah untuk caching?
4. Rancang (secara konsep, tidak perlu kode): kalau `ecommerce-etl-pipeline` suatu saat butuh fitur "rekomendasi produk" berbasis pola pembelian pelanggan, tipe NoSQL apa yang paling relevan dipertimbangkan, dan kenapa RDBMS (`fact_sales` yang sudah ada) kurang ideal untuk kasus spesifik ini?

## Kunci Jawaban & Pembahasan

**1.** (a) **Key-value** (Redis) — akses berdasarkan key (session ID) yang sudah diketahui, butuh kecepatan ekstrem, tidak butuh query berdasarkan isi session. (b) **Document** (MongoDB) — struktur atribut bervariasi antar kategori produk, tapi tetap butuh di-query berdasarkan field (harga, kategori, dst), cocok document model. (c) **Graph** (Neo4j) — traversal relasi berlapis antar banyak entitas (customer-produk-customer lain) adalah kekuatan utama graph database, akan sangat mahal/rumit kalau dipaksa pakai `JOIN` berulang di RDBMS. (d) **Column-family** (Cassandra) — volume tulis sangat besar & terdistribusi dari banyak sumber (1 juta perangkat), karakteristik utama yang dioptimalkan column-family store.

**2.** `dim_product` **belum** butuh MongoDB sebelumnya karena strukturnya memang sederhana & seragam (`stock_code`, `description` — 2 kolom saja, tidak ada variasi struktural antar produk) — RDBMS bekerja sempurna untuk itu, sesuai `hari_1_konsep_nosql.md`. Yang berubah di Minggu 8 (`minggu_8.md` Tahap 1): mini project **sengaja** memperkaya `dim_product` dengan atribut yang **bervariasi per kategori** (elektronik butuh `warranty_period`, fashion butuh `size_variant`) — begitu kebutuhannya berubah jadi "atribut variatif per jenis data", karakteristik itu langsung cocok dengan kekuatan document model, sesuai kerangka berpikir yang dibangun hari ini.

**3.** Keduanya **tidak saling menggantikan** karena dioptimalkan untuk kebutuhan yang berbeda secara fundamental: MongoDB dioptimalkan untuk menyimpan data **terstruktur secara fleksibel** yang perlu di-query berdasarkan isi field-nya (`find({category: "Electronics"})`) — bukan untuk kecepatan akses ekstrem berdasarkan key tunggal yang sudah diketahui. Redis dioptimalkan justru untuk itu: akses `GET`/`SET` berbasis key yang sangat cepat (operasi di memory), tapi **tidak** dirancang untuk query kompleks berdasarkan isi value. Pola yang umum (dan akan dipraktikkan Minggu, `hari_4_redis.md`): **MongoDB sebagai sumber data**, **Redis sebagai cache** di depannya — query mahal ke MongoDB (atau database manapun) dijalankan sekali, hasilnya disimpan sementara di Redis dengan key yang mudah ditebak, request berikutnya untuk data yang sama cukup `GET` dari Redis tanpa perlu query ulang ke sumber data.

**4.** **Graph database** (Neo4j atau sejenisnya) paling relevan dipertimbangkan — kebutuhan "rekomendasi berbasis pola pembelian" pada dasarnya adalah pertanyaan **traversal relasi**: "pelanggan yang membeli produk A, produk apa lagi yang sering dibeli pelanggan **lain** yang juga membeli A?" — pertanyaan berlapis seperti ini di RDBMS berarti `JOIN` `fact_sales` ke dirinya sendiri berkali-kali (self-join kompleks) yang performanya menurun drastis seiring bertambahnya "lapisan" relasi yang ditelusuri. Graph database menyimpan hubungan pelanggan-produk sebagai edge yang bisa ditelusuri langsung tanpa `JOIN` berulang, jauh lebih natural & cepat untuk pertanyaan berbasis relasi berlapis seperti ini — meski, konsisten dengan `hari_5_storage_strategy.md` nanti, `fact_sales` di PostgreSQL/BigQuery **tetap** dipertahankan untuk kebutuhan analitik historis biasa (total revenue, RFM) yang memang sudah optimal di sana; graph database di sini jadi **tambahan** untuk use case spesifik, bukan pengganti.
