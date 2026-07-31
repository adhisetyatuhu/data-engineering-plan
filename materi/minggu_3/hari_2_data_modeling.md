---
title: Hari 2 - Data Modeling (Star Schema)
parent: Minggu 3 - Arsitektur Data & Big Data Pipeline
nav_order: 3
---

# Hari 2 — Data Modeling: Star Schema, Snowflake Schema, Fact & Dimension Table

*Selasa, 2 jam. Dataset: dataset mini Minggu 1 (`customers`, `products`, `orders`, `order_items`) — lihat `materi/minggu_1/00_overview.md`.*

> Hasil hari ini **langsung dipakai** di Tahap 1 mini project (`latihan_pipeline_mini_project.md`) — bedanya di mini project skemanya dirancang untuk Online Retail II, bukan dataset mini. Perlakukan latihan hari ini sebagai "gladi bersih" dengan data yang sudah familiar sebelum menghadapi data sungguhan yang lebih besar & kotor.

## Tujuan Belajar

- [ ] Menjelaskan konsep **fact table** dan **dimension table**, serta apa itu **grain** (tingkat detail 1 baris)
- [ ] Merancang star schema dari skema OLTP normalized
- [ ] Menjelaskan beda star schema vs snowflake schema, dan kapan masing-masing lebih dipilih
- [ ] Menjelaskan konsep **surrogate key** dan kenapa dipakai selain primary key asli
- [ ] Mengenali (level pengantar) apa itu **Slowly Changing Dimension (SCD)**

## Untuk Instruktur: Mindset Shift

Developer sudah jago normalisasi (3NF) dari pengalaman merancang database aplikasi — dataset `customers`/`orders`/`order_items` di Minggu 1 itu contohnya. Star schema modeling itu **kebalikannya secara sengaja**: bukan menghindari duplikasi, tapi **menerima** duplikasi demi kecepatan baca. Ini sering terasa "salah" buat developer yang baru pertama kali lihat — tekankan bahwa ini trade-off sadar (baca Hari 1: OLAP dominan baca, bukan tulis), bukan orang yang lupa cara normalisasi.

Analogi kode yang bisa dipakai: bayangkan API response yang di-*denormalize* sengaja (`{"order_id": 1, "customer_name": "Andi", "product_name": "Mouse", ...}` — nama customer & produk ikut disalin ke tiap object) supaya frontend tidak perlu request tambahan buat resolve nama dari ID. Itu keputusan yang sama persis dengan alasan star schema: sengaja duplikasi data supaya "pembaca" (query analitik / frontend) tidak perlu kerja ekstra (`JOIN` / request tambahan) tiap kali baca.

## Konsep & Sintaks

### Fact Table vs Dimension Table

| | Fact Table | Dimension Table |
|---|---|---|
| Isi | **Kejadian/transaksi** yang terukur — angka yang bisa dijumlah/dirata-rata (measures) | **Konteks** dari kejadian itu — atribut deskriptif |
| Contoh kolom | `quantity`, `unit_price`, `total_amount` | `product_name`, `category`, `country`, `customer_name` |
| Ukuran tabel | Biasanya **jauh lebih besar** (1 baris = 1 transaksi/event) | Biasanya jauh lebih kecil (1 baris = 1 entitas unik) |
| Berubah? | Baris baru terus ditambah (append), jarang di-update | Bisa berubah (customer pindah negara, produk ganti kategori) — ini yang memicu topik SCD |

Analogi paling gampang buat developer: fact table itu seperti **log/event table** (`page_view`, `order_placed` — banyak baris, terus bertambah, tiap baris berisi angka/ID). Dimension table itu seperti **tabel referensi/lookup** (`users`, `products` — jumlah baris relatif stabil, isinya deskripsi).

### Grain — Konsep Terpenting yang Sering Dilewatkan

**Grain** = jawaban dari "1 baris di fact table ini mewakili apa, persis?". Ini keputusan pertama yang **wajib** dijawab sebelum merancang fact table apapun, karena semua desain kolom mengikuti keputusan ini.

Contoh untuk kasus kita: grain `fact_sales` bisa jadi beberapa pilihan berbeda —
- **1 baris = 1 item dalam 1 order** (grain paling detail — sama seperti tabel `order_items` di Minggu 1) — ini yang akan dipakai di mini project, karena paling fleksibel untuk agregasi ke level manapun di atasnya.
- **1 baris = 1 order** (sudah teragregasi per order — kehilangan detail produk per baris)
- **1 baris = 1 customer per hari** (sudah teragregasi lebih jauh — kehilangan detail order individual)

**Aturan praktis**: pilih grain **paling detail** yang masih masuk akal untuk kebutuhan bisnis. Alasannya: dari grain detail, agregasi ke level yang lebih kasar (per order, per hari, per bulan) selalu bisa dilakukan lewat `GROUP BY` saat query. Sebaliknya, dari grain yang sudah kasar, **tidak bisa** kembali ke detail yang sudah hilang. Ini analog dengan *lossy vs lossless compression* di dunia programming — grain detail itu lossless (semua opsi masih terbuka), grain kasar itu lossy (sudah membuang informasi).

### Star Schema

Satu fact table di tengah, dikelilingi dimension table yang **langsung** terhubung ke fact table (tidak ada tabel dimensi yang terhubung ke dimensi lain) — bentuknya kalau digambar memang menyerupai bintang.

```
        dim_customer
             |
dim_date -- fact_sales -- dim_product
             |
          dim_date
```

Rancangan untuk dataset mini Minggu 1 (grain: 1 baris = 1 baris `order_items`):

```
fact_sales
├── sales_id       (surrogate key)
├── date_id    FK -> dim_date
├── customer_id FK -> dim_customer
├── product_id  FK -> dim_product
├── order_id       (degenerate dimension, lihat catatan bawah)
├── quantity
├── unit_price
└── total_amount  (quantity * unit_price)

dim_customer                    dim_product                  dim_date
├── customer_id PK               ├── product_id PK             ├── date_id PK
├── customer_name                ├── product_name               ├── full_date
└── country                      └── category                   ├── day
                                                                  ├── month
                                                                  ├── quarter
                                                                  └── year
```

**Degenerate dimension** — istilah untuk kolom seperti `order_id` di atas: nilainya berasal dari sumber transaksi, tapi tidak punya dimension table sendiri (tidak ada atribut deskriptif tambahan tentang "order" selain ID-nya sendiri) — cukup disimpan langsung di fact table, tidak perlu tabel terpisah.

**`dim_date`** — dimension yang hampir selalu ada di star schema manapun, walau terlihat "aneh" bagi developer di awal (kenapa tidak simpan `DATE` biasa saja?). Alasannya: `dim_date` memungkinkan query filter/group by atribut kalender kompleks (`quarter`, `is_weekend`, `fiscal_year`) tanpa perlu fungsi tanggal berat di tiap query — tabelnya di-generate sekali (biasanya berisi semua tanggal untuk beberapa tahun ke depan), lalu dipakai berulang.

### Snowflake Schema

Variasi star schema di mana dimension table **dinormalisasi lebih lanjut** — mis. `dim_product` dipecah jadi `dim_product` + `dim_category` terpisah (`dim_product.category_id` → `dim_category`).

```
dim_category -- dim_product -- fact_sales
```

| | Star Schema | Snowflake Schema |
|---|---|---|
| Dimension | Denormalized (1 tabel per dimensi, flat) | Dimension dipecah lebih lanjut jadi sub-dimensi |
| Query | Lebih sedikit `JOIN` → lebih simpel & cepat dibaca | Lebih banyak `JOIN` → sedikit lebih kompleks |
| Storage | Sedikit lebih boros (duplikasi `category` di tiap baris produk) | Lebih hemat (kategori cuma disimpan sekali) |
| Kapan dipilih | **Default pilihan** untuk kebanyakan kasus — prioritas kecepatan & kesederhanaan query | Kalau dimensi punya hierarki kompleks & sering berubah, atau storage jadi concern nyata |

**Rekomendasi praktis untuk mini project**: pakai **star schema murni** (bukan snowflake). Untuk skala data & kompleksitas Online Retail II, keuntungan snowflake (hemat storage) tidak sebanding dengan biaya kompleksitas query tambahan — ini keputusan yang sama yang diambil mayoritas data warehouse modern (storage murah, `JOIN` ekstra yang lebih mahal secara compute).

### Surrogate Key vs Natural Key

- **Natural key** — ID yang sudah ada dari sistem sumber (mis. `Customer ID` dari Online Retail II, `product_id` dari Minggu 1).
- **Surrogate key** — ID baru yang dibuat **khusus** untuk warehouse (biasanya integer auto-increment), tidak punya arti bisnis sama sekali.

Kenapa tidak pakai natural key saja di warehouse? Beberapa alasan praktis:
1. Natural key kadang berubah format/hilang unik-nya saat sumber data berganti sistem.
2. Natural key dari sumber berbeda bisa bentrok (integrasi 2 sistem sumber yang kebetulan sama-sama pakai ID `1001` untuk entitas berbeda).
3. Untuk **Slowly Changing Dimension** (di bawah), 1 natural key bisa punya **beberapa** baris di dimension table (versi historis) — surrogate key memastikan tiap versi tetap unik.

### Slowly Changing Dimension (SCD) — Pengantar

Pertanyaan: kalau seorang customer pindah negara, apa yang terjadi ke baris mereka di `dim_customer`?

- **SCD Type 1** — timpa saja (`UPDATE` langsung). Histori lama hilang. Cocok kalau histori atribut itu tidak penting secara bisnis (mis. typo nama yang dikoreksi).
- **SCD Type 2** — jangan timpa, **tambah baris baru** dengan surrogate key baru, tandai baris lama sebagai tidak aktif (kolom `is_current` / `valid_from`/`valid_to`). Histori lengkap tersimpan — fact table lama tetap merujuk ke versi dimensi yang berlaku saat transaksi itu terjadi.

Ini baru **pengantar** — SCD Type 2 butuh surrogate key (poin di atas) dan akan dipraktikkan lebih dalam saat topik Data Governance & versioning data di Minggu 5. Untuk mini project Minggu 3, cukup pakai SCD Type 1 (skema sederhana, replace penuh tiap kali pipeline jalan) — disebut eksplisit di `latihan_pipeline_mini_project.md`.

## Contoh: Menerjemahkan Skema Normalized ke Star Schema

Skema asal (normalized, dari Minggu 1):
```
customers (customer_id PK, customer_name, country, signup_date)
products  (product_id PK, product_name, category, unit_price)
orders    (order_id PK, customer_id FK, order_date, status)
order_items (order_item_id PK, order_id FK, product_id FK, quantity, unit_price)
```

Langkah merancang star schema:
1. **Tentukan grain** — pilih level `order_items` (paling detail, 19 baris).
2. **Identifikasi measures (fact)** — angka yang mau dijumlah/dirata-rata: `quantity`, `unit_price`, dan turunan `quantity * unit_price` (`total_amount`).
3. **Identifikasi dimensions** — semua atribut deskriptif yang "menjelaskan" tiap baris fact: siapa customernya (`dim_customer`), produk apa (`dim_product`), kapan (`dim_date` — dari `order_date`).
4. **Flatten tiap dimension** — `products.category` cukup jadi kolom langsung di `dim_product` (bukan tabel `dim_category` terpisah — pilih star, bukan snowflake, sesuai rekomendasi di atas).

Hasilnya persis rancangan `fact_sales`/`dim_customer`/`dim_product`/`dim_date` di bagian "Star Schema" di atas.

## Kesalahan Umum

1. **Tidak menentukan grain di awal.** Tanpa grain yang jelas, mudah tergoda mencampur level detail berbeda dalam satu fact table (mis. sebagian baris mewakili 1 item, sebagian lagi sudah teragregasi per order) — hasilnya agregasi jadi salah (double count atau under count).
2. **Menormalisasi dimension table seperti OLTP (jadi snowflake tanpa sadar).** Ingat: tujuan star schema adalah kecepatan baca, bukan efisiensi storage. Kalau ragu, flatten dulu.
3. **Memasukkan measure (angka yang bisa dijumlah) ke dimension table**, atau sebaliknya memasukkan atribut deskriptif ke fact table. Aturan cepat: kalau masuk akal di-`SUM()`/`AVG()`, itu measure → fact table. Kalau dipakai untuk filter/group by ("per kategori", "per negara"), itu atribut deskriptif → dimension table.
4. **Lupa `dim_date` dan malah simpan tanggal mentah di fact table.** Ini bekerja untuk kasus sederhana, tapi kehilangan kemudahan filter/group by atribut kalender (quarter, is_weekend) yang sudah "siap pakai" di `dim_date`.

## Latihan

1. Tentukan grain untuk kebutuhan bisnis berikut: "tim finance mau tahu total revenue per customer per bulan, tidak butuh detail produk". Apakah grain-nya sama dengan yang dipakai di atas? Jelaskan.
2. Untuk dataset mini Minggu 1, rancang `dim_date` — kolom apa saja yang perlu ada supaya bisa menjawab pertanyaan "total revenue per kuartal" dan "revenue di hari kerja vs akhir pekan"?
3. `orders.status` (`completed`/`cancelled`/`refunded`) sebaiknya masuk ke fact table atau dimension table? Jelaskan alasannya memakai aturan measure vs atribut deskriptif.
4. Perusahaan mau mulai melacak histori perubahan `category` produk (karena kadang produk dipindah kategori, dan tim ingin tahu laporan revenue "seperti apa kategorinya saat transaksi terjadi", bukan kategori terbaru). Skema SCD apa yang cocok, dan apa konsekuensinya ke desain `dim_product` (termasuk soal surrogate key)?
5. Kapan snowflake schema justru lebih masuk akal dibanding star schema untuk kasus seperti `dim_product`? Beri contoh skenario konkret.

## Kunci Jawaban & Pembahasan

**1.** Grain yang diminta di sini **lebih kasar**: "1 baris = 1 customer per bulan" (bukan 1 baris = 1 item order). Ini **beda** dari grain di atas (1 baris = 1 order item). Tapi ingat aturan praktis: lebih baik tetap rancang & simpan `fact_sales` di grain paling detail (1 order item), lalu buat **agregat/view/table turunan** ("per customer per bulan") dari situ lewat `GROUP BY` — bukan merancang fact table utama langsung di grain kasar. Alasannya: kalau nanti ada kebutuhan baru (misal "per customer per bulan per kategori produk"), fact table detail masih bisa menjawabnya tanpa perlu rancang ulang, sementara fact table yang sudah telanjur kasar sejak awal tidak bisa.

**2.**
```
dim_date (date_id PK, full_date, day, day_of_week, month, month_name, quarter, year, is_weekend)
```
`quarter` untuk jawab pertanyaan pertama langsung lewat `GROUP BY quarter`. `is_weekend` (boolean, dihitung dari `day_of_week`) untuk jawab pertanyaan kedua tanpa perlu fungsi tanggal on-the-fly di tiap query — nilainya sudah "siap pakai" begitu tabel dibuat sekali (biasanya lewat script generate tanggal, bukan diisi manual).

**3.** `status` adalah **atribut deskriptif** yang dipakai untuk **memfilter/mengelompokkan** transaksi (mis. `WHERE status = 'completed'` — persis pola yang dipakai berulang kali di `materi/minggu_1/`), bukan angka yang dijumlah/dirata-rata. Karena grain fact table di sini adalah per order item dan status ditentukan di level order (bukan per item), pilihan paling umum: simpan `status` langsung di `fact_sales` sebagai kolom deskriptif (bukan bikin `dim_status` terpisah — jumlah nilai unik-nya kecil, tidak butuh dimension table sendiri; ini pola yang kadang disebut *junk dimension* kalau digabung dengan flag-flag kecil lain, tapi untuk 1 kolom saja cukup taruh langsung di fact table).

**4.** Ini kasus **SCD Type 2** — histori kategori lama harus tetap tersambung ke transaksi historisnya, bukan ditimpa. Konsekuensi ke `dim_product`: butuh **surrogate key** (`product_key`, bukan `product_id` natural key langsung) karena 1 produk (1 `product_id`) sekarang bisa punya **beberapa baris** di `dim_product` (satu per periode kategori berbeda), ditambah kolom `valid_from`, `valid_to`, `is_current`. `fact_sales` merujuk ke `product_key` (surrogate, versi yang berlaku saat transaksi terjadi), bukan `product_id` langsung — supaya laporan revenue lama tetap menunjukkan kategori yang benar secara historis, bukan ikut berubah setiap kategori produk di-update.

**5.** Snowflake lebih masuk akal kalau dimension punya **hierarki yang dalam dan sering berubah independen** dari entitas induknya — contoh: `dim_product` dengan hierarki `product → sub_category → category → department`, di mana nama `department`/`category` sering direvisi oleh tim marketing dan dipakai bersama oleh ribuan produk. Kalau di-flatten (star), 1 kali revisi nama kategori berarti harus `UPDATE` ribuan baris `dim_product` (atau bikin banyak baris SCD baru untuk perubahan yang sebenarnya tidak terkait produknya sendiri). Dengan snowflake, cukup `UPDATE` 1 baris di `dim_category`, otomatis "berlaku" untuk semua produk yang mereferensikannya. Untuk skala & kompleksitas `dim_product` di mini project kita (kategori jarang berubah, jumlah kategori sedikit), trade-off ini belum sepadan — makanya rekomendasi tetap star schema murni.
