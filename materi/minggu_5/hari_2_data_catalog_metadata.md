---
title: Hari 2 - Data Catalog & Metadata Management
parent: Minggu 5 - Data Governance
nav_order: 3
---

# Hari 2 — Data Catalog & Metadata Management

*Selasa, 2 jam. Konsep + perbandingan tools — setup hands-on OpenMetadata baru Sabtu (`latihan_catalog_lineage_mini_project.md`).*

## Tujuan Belajar

- [ ] Menjelaskan apa itu data catalog dan masalah konkret yang diselesaikannya
- [ ] Membedakan 3 jenis metadata: technical, business, operational
- [ ] Membandingkan tools data catalog populer (DataHub, Amundsen, OpenMetadata, Alation) dan kapan masing-masing relevan
- [ ] Merancang metadata (ketiga jenisnya) untuk tabel `fact_sales`/`dim_customer` dari pipeline yang sudah dibangun

## Untuk Instruktur: Mindset Shift

Analogi paling langsung: **data catalog adalah mesin pencari (search engine) + dokumentasi API, tapi untuk dataset, bukan untuk endpoint kode.** Developer sudah punya pengalaman kuat soal ini: bayangkan API besar tanpa Swagger/OpenAPI docs — endpoint-nya mungkin berfungsi sempurna, tapi tidak ada yang tahu itu ada, apa yang diterima/dikembalikan, atau siapa yang harus dihubungi kalau ada masalah. Data catalog menyelesaikan masalah **yang sama persis**, untuk tabel/dataset.

**Metadata** = "data tentang data" — istilah yang sering terasa sirkular/membingungkan buat pemula. Cara paling konkret menjelaskannya: metadata adalah **semua informasi tentang sebuah dataset selain isi baitnya sendiri** — skema, deskripsi, siapa pemiliknya, kapan terakhir diupdate, seberapa sering dipakai. Developer sudah familiar dengan analognya: metadata file (ukuran, tanggal modifikasi, permission) vs isi file itu sendiri — konsepnya identik, cuma levelnya di dataset, bukan file individual.

## Konsep & Sintaks

### Masalah yang Diselesaikan Data Catalog

Tanpa catalog, "menemukan data" biasanya berarti: tanya di Slack channel tim, cari di dokumentasi lama yang mungkin sudah usang, atau baca kode pipeline langsung untuk menebak skema. Ini **tidak scalable** begitu jumlah dataset & jumlah orang yang butuh akses bertambah — persis masalah yang sudah dibahas Hari 1 poin "tidak ada yang tahu data itu ada".

Data catalog adalah **pusat pencarian & dokumentasi** yang menjawab, untuk tiap dataset:
- Dataset apa saja yang ada, dan skemanya apa?
- Apa arti tiap kolom (dalam bahasa bisnis, bukan cuma nama teknis)?
- Siapa pemiliknya, kapan terakhir diupdate, seberapa sering dipakai?
- Dari mana asalnya (lineage — Hari 3), dan siapa/apa yang bergantung padanya?
- Ada tag/klasifikasi khusus (mis. PII — Hari 5)?

### 3 Jenis Metadata

| Jenis | Isi | Contoh untuk `fact_sales` | Biasanya Berasal Dari |
|---|---|---|---|
| **Technical Metadata** | Skema, tipe data, constraint, ukuran tabel, partisi | Kolom `customer_id INT`, `revenue NUMERIC(10,2)`, jumlah baris, ukuran storage | **Otomatis** — hasil introspeksi langsung ke database/warehouse (`information_schema`, dsb) |
| **Business Metadata** | Deskripsi dalam bahasa manusia, definisi bisnis, glossary term, tag klasifikasi | "Tabel ini berisi 1 baris per item transaksi. `is_return=True` berarti pembatalan/retur." | **Manual** — ditulis manusia (data steward, analyst) yang paham konteks bisnisnya |
| **Operational Metadata** | Statistik penggunaan, riwayat run pipeline, freshness, siapa yang query kapan | "Terakhir diupdate: hari ini 02:15 (dari DAG `ecommerce_etl_pipeline`), diquery 47x minggu ini" | **Otomatis** — dari log sistem, integrasi dengan orkestrator (Airflow), query history |

Pembagian ini penting karena **cara mendapatkannya beda**: technical & operational metadata idealnya **selalu otomatis** (kalau manual, akan cepat basi/tidak akurat begitu skema berubah) — sementara business metadata **hampir selalu butuh input manusia**, karena "apa arti kolom ini bagi bisnis" bukan sesuatu yang bisa disimpulkan otomatis dari database.

### Perbandingan Tools

| Tool | Karakteristik | Kapan Relevan |
|---|---|---|
| **DataHub** (LinkedIn, open-source) | Arsitektur berbasis event streaming (Kafka di baliknya — koneksi konseptual ke `materi/minggu_4/hari_4_pengantar_kafka.md`), sangat scalable, ekosistem plugin luas | Organisasi besar, banyak sumber data heterogen, butuh integrasi custom mendalam |
| **Amundsen** (Lyft, open-source) | Fokus ke **search & discovery** (mirip mesin pencari data), UI sederhana, lineage lebih terbatas dibanding DataHub | Tim yang prioritas utamanya "gampang menemukan data", bukan governance penuh |
| **OpenMetadata** (open-source) | All-in-one: catalog + lineage + data quality + glossary dalam 1 platform, quickstart Docker relatif ringan | **Tim kecil–menengah yang ingin 1 tool serba bisa** — inilah alasan dipilih untuk hands-on roadmap ini |
| **Alation** (komersial) | Fitur enterprise matang (governance workflow, kolaborasi), support berbayar | Organisasi besar dengan budget & kebutuhan compliance ketat |

**Kenapa OpenMetadata dipilih untuk hands-on** (`latihan_catalog_lineage_mini_project.md`): bukan karena "yang terbaik" secara mutlak — DataHub dan Amundsen sama validnya di dunia kerja nyata — tapi karena kombinasi setup Docker yang lebih ringan dan cakupan fitur (catalog + lineage + tag PII) yang paling pas untuk **satu** sesi hands-on akhir pekan. Konsep yang dipelajari (metadata, ownership, tag, lineage) **transfer penuh** ke tool manapun yang dipakai di tempat kerja nanti.

### Contoh: Merancang Metadata untuk `fact_sales`

```yaml
# Technical (biasanya otomatis dari introspeksi database)
table: fact_sales
columns:
  - invoice: VARCHAR
  - stock_code: VARCHAR
  - customer_id: INT
  - date_id: INT
  - quantity: INT
  - unit_price: NUMERIC(10,2)
  - revenue: NUMERIC(10,2)
  - is_return: BOOLEAN
row_count: ~400000   # tergantung data aktual

# Business (ditulis manual oleh steward/tim data engineering)
description: >
  1 baris = 1 item produk dalam 1 invoice (grain: order item level).
  is_return=True menandakan invoice yang diawali huruf 'C' (cancellation/retur).
owner: Data Engineering Team
tags: [PII]   # karena mengandung customer_id -- lihat hari_5_compliance_privacy.md
glossary_terms: ["Revenue = quantity x unit_price, TIDAK termasuk baris is_return=True untuk laporan penjualan bersih"]

# Operational (otomatis dari integrasi Airflow/query log)
last_updated: 2026-01-15T02:15:00
updated_by_pipeline: ecommerce_etl_pipeline
query_count_last_7_days: 47
```

## Kesalahan Umum

1. **Mengisi business metadata sekali lalu tidak pernah diupdate lagi.** Deskripsi yang basi (tidak mencerminkan perubahan skema/logic terbaru) lebih berbahaya daripada tidak ada deskripsi sama sekali — orang akan percaya penjelasan yang sudah salah. Update business metadata harus jadi bagian rutin dari proses perubahan skema, bukan aktivitas sekali jadi.
2. **Mencoba mengisi semua metadata secara manual**, termasuk yang seharusnya otomatis (technical/operational). Ini tidak scalable dan cepat basi — technical/operational metadata harus di-generate dari sistem (ingestion otomatis dari database/orkestrator), manusia cuma mengurus business metadata.
3. **Menganggap catalog = dokumentasi statis sekali tulis** (seperti wiki lama yang tidak pernah dibuka lagi). Catalog yang berfungsi baik terintegrasi ke **alur kerja aktif** (terhubung ke pipeline sungguhan, diupdate otomatis, dicari & dipakai orang tiap hari) — bukan dokumen terpisah yang gampang dilupakan.
4. **Memilih tool catalog berdasarkan fitur terbanyak, bukan kecocokan dengan ukuran tim & kebutuhan.** Tool enterprise (Alation, DataHub skala penuh) bisa jadi overkill dan berat dioperasikan untuk tim kecil — pertimbangkan effort maintenance tool itu sendiri, bukan cuma daftar fiturnya.

## Latihan

1. Untuk tabel `dim_date` (Minggu 3), rancang metadata untuk ketiga jenis (technical, business, operational) — minimal 2 item per jenis.
2. Jelaskan kenapa "jumlah baris di `fact_sales`" adalah technical metadata, sementara "revenue tidak termasuk baris retur untuk laporan penjualan bersih" adalah business metadata — walau keduanya sama-sama "fakta tentang data".
3. Tim kecil (5 orang) dengan ~10 dataset mempertimbangkan antara Amundsen dan DataHub. Faktor apa yang sebaiknya jadi pertimbangan utama mereka, dan tool mana yang lebih masuk akal untuk skala ini? Jelaskan.
4. Rekan kerja bilang "kita sudah punya README.md di tiap folder yang jelaskan tabelnya, jadi tidak perlu data catalog terpisah." Bandingkan kelebihan catalog dibanding README biasa — kapan README saja **cukup**, dan kapan **tidak lagi cukup**?

## Kunci Jawaban & Pembahasan

**1.**
```
Technical: kolom (date_id INT, full_date DATE, quarter INT, is_weekend BOOLEAN), jumlah baris (~jumlah hari yang di-generate)
Business: deskripsi ("1 baris = 1 tanggal kalender, digenerate penuh untuk rentang beberapa tahun ke depan, dipakai untuk join filter/group by berbasis waktu"), owner (Data Engineering Team)
Operational: kapan terakhir di-generate ulang, dipakai oleh berapa query/dashboard (kalau ada integrasi query log)
```

**2.** "Jumlah baris" adalah fakta yang bisa **dihitung langsung** dari data itu sendiri (`SELECT COUNT(*)`) — tidak butuh pengetahuan konteks bisnis apapun, murni hasil introspeksi teknis, makanya technical metadata. "Revenue tidak termasuk baris retur untuk laporan penjualan bersih" **tidak bisa** disimpulkan dari data itu sendiri — itu adalah **keputusan/kesepakatan bisnis** tentang bagaimana angka itu seharusnya diinterpretasikan dan dipakai (seseorang harus tahu & memutuskan definisi "penjualan bersih" itu apa), makanya business metadata. Aturan pembeda praktis: kalau bisa dijawab hanya dengan query ke data itu sendiri tanpa konteks tambahan → technical; kalau butuh pengetahuan/kesepakatan di luar data mentahnya → business.

**3.** Faktor utama: **kompleksitas kebutuhan vs effort maintenance**. Untuk tim 5 orang dengan ~10 dataset, kebutuhan utamanya kemungkinan besar "gampang mencari & memahami dataset yang ada" — bukan governance workflow kompleks atau integrasi lintas puluhan sumber data heterogen. **Amundsen** lebih masuk akal di sini: fokusnya pas dengan kebutuhan (search & discovery sederhana), dan effort setup/maintenance jauh lebih ringan dibanding DataHub yang dirancang untuk skala jauh lebih besar (arsitektur event-streaming DataHub baru terasa manfaatnya kalau jumlah sumber data & volume perubahan metadata sudah besar). Memilih DataHub di skala ini berisiko menghabiskan lebih banyak waktu mengelola tool-nya sendiri dibanding manfaat yang didapat tim.

**4.** README **cukup** selama jumlah dataset masih sedikit, tidak ada kebutuhan **mencari** di antara banyak dataset (developer masih hafal semua nama folder/file), dan tidak butuh fitur seperti tag PII/lineage otomatis/statistik penggunaan. README **tidak lagi cukup** begitu: (a) jumlah dataset sudah terlalu banyak untuk diingat manual, butuh **pencarian** (README tidak punya search index terpusat — orang harus tahu duluan di folder mana harus cari), (b) butuh melihat **lineage** (README statis tidak bisa menunjukkan "tabel ini asalnya dari mana, dipakai oleh apa saja" secara otomatis terhubung ke sistem sungguhan), (c) butuh **kontrol** seperti tag PII yang bisa dicari/difilter lintas semua dataset sekaligus, atau (d) README-nya sendiri mulai basi karena tidak ada mekanisme yang memastikan README terupdate seiring skema berubah (catalog yang terintegrasi ke database bisa mendeteksi skema berubah otomatis, README tidak).
