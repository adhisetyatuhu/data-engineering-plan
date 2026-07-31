---
title: Hari 3 - Data Lineage
parent: Minggu 5 - Data Governance
nav_order: 4
---

# Hari 3 — Data Lineage: Melacak Asal-Usul & Perjalanan Data

*Rabu, 2 jam. Konsep — dipetakan langsung ke pipeline `ecommerce-etl-pipeline` yang sudah dibangun Minggu 3-4.*

## Tujuan Belajar

- [ ] Menjelaskan apa itu data lineage dan 2 kegunaan utamanya: audit trail dan impact analysis
- [ ] Membedakan lineage table-level dan column-level
- [ ] Memetakan lineage penuh pipeline `ecommerce-etl-pipeline` dari sumber sampai warehouse
- [ ] Menjelaskan cara-cara lineage bisa didapat (manual vs otomatis) beserta trade-off-nya

## Untuk Instruktur: Mindset Shift

Analogi paling langsung: **data lineage adalah `git blame` + dependency graph, tapi untuk data, bukan kode.** `git blame` menjawab "baris kode ini berasal dari commit/orang mana"; dependency graph (`npm ls`, import graph) menjawab "modul ini bergantung ke apa, dan apa yang bergantung padanya". Data lineage menjawab pertanyaan yang sama persis untuk data: **dari mana asalnya**, dan **apa yang akan terdampak** kalau data ini berubah.

Developer yang pernah mengalami "saya ubah 1 fungsi kecil, ternyata dipakai di 5 tempat lain yang tidak saya sadari, dan semuanya rusak" akan langsung paham **kenapa** lineage penting untuk data — masalahnya identik, cuma levelnya di tabel/kolom, bukan fungsi/modul.

## Konsep & Sintaks

### 2 Kegunaan Utama Lineage

**1. Audit Trail (Debugging Ke Belakang)**

Pertanyaan: *"angka di dashboard ini salah — dari mana asalnya, di langkah mana kemungkinan errornya muncul?"*

Tanpa lineage, menjawab ini berarti membaca kode pipeline satu-satu dari nol, menebak-nebak urutan transformasi. Dengan lineage yang terdokumentasi, langsung terlihat jalur lengkapnya:

```
online_retail_II.csv (raw)
    -> clean_transform.py (dropna, dedup, filter Price>0)
    -> build_star_schema.py (split jadi fact/dim)
    -> fact_sales (Postgres)
    -> dashboard revenue harian
```

Kalau dashboard salah, lineage ini langsung mempersempit **di langkah mana** kemungkinan bug berada — persis seperti `git bisect` mempersempit commit mana yang memperkenalkan bug.

**2. Impact Analysis (Dampak Ke Depan)**

Pertanyaan kebalikannya: *"kalau saya ubah kolom `Price` di sumber raw jadi format baru, apa saja yang akan terdampak?"*

Lineage yang sama, dibaca **maju** (bukan mundur), menjawab ini: semua yang ada "di bawah" `Price` di rantai lineage (kolom `revenue`, `fact_sales.unit_price`, dashboard yang memakainya) berpotensi terdampak. Ini kegunaan yang sangat konkret sebelum melakukan perubahan skema — analog dengan "find usages" di IDE sebelum rename sebuah fungsi/variabel, supaya tahu semua tempat yang perlu ikut disesuaikan **sebelum** mengubah, bukan menemukannya lewat kegagalan di production.

### Table-Level vs Column-Level Lineage

| | Table-Level Lineage | Column-Level Lineage |
|---|---|---|
| Granularitas | "Tabel A → Tabel B" | "Kolom `Price` di Tabel A → Kolom `unit_price` di Tabel B" |
| Kemudahan didapat | Lebih mudah (bisa dari sekadar tahu tabel apa dibaca/ditulis 1 job) | Lebih sulit (butuh parsing logic transformasi sampai level ekspresi kolom) |
| Kegunaan | Cukup untuk pertanyaan "tabel ini datang dari mana secara umum" | Dibutuhkan untuk pertanyaan presisi "kalau kolom spesifik ini berubah, kolom spesifik mana saja yang terdampak" |
| Contoh tools yang mendukung otomatis | Kebanyakan tool catalog (OpenMetadata, DataHub) | Sebagian tool (tergantung integrasi — butuh parsing SQL/kode transformasi, bukan cuma tahu I/O job) |

Untuk pipeline `ecommerce-etl-pipeline`, table-level lineage-nya:
```
online_retail_II.csv → retail_clean (staging) → fact_sales/dim_customer/dim_product/dim_date → (query analisis/dashboard)
```

Column-level lineage untuk 1 kolom spesifik (`revenue`):
```
Quantity (raw CSV) ─┐
                     ├─→ revenue (clean_transform.py: quantity * price) → fact_sales.revenue
Price (raw CSV)    ─┘
```

### Bagaimana Lineage Didapat

**1. Manual (diagram/dokumentasi ditulis tangan)** — yang dipakai di mini project minggu ini (`latihan_catalog_lineage_mini_project.md`), karena paling sederhana disetup dan tetap valid untuk pipeline berskala kecil-menengah yang jumlah langkahnya masih bisa dipetakan manusia. Kelemahan: **bisa basi** — begitu pipeline berubah, diagram manual tidak otomatis ikut update, butuh disiplin mengupdatenya.

**2. Otomatis lewat parsing SQL** — beberapa tool catalog bisa membaca query SQL yang dijalankan (dari log warehouse) dan menyimpulkan lineage dari situ (`INSERT INTO fact_sales SELECT ... FROM retail_clean` → tool tahu `fact_sales` berasal dari `retail_clean`). Bekerja baik untuk pipeline berbasis SQL/ELT (`materi/minggu_3/hari_3_etl_elt.md`), kurang bekerja untuk transformasi yang logic-nya ada di kode (PySpark) bukan SQL.

**3. Otomatis lewat instrumentasi pipeline (mis. OpenLineage)** — standar terbuka yang diintegrasikan langsung ke orkestrator (Airflow) dan engine pemrosesan (Spark), memancarkan event lineage **setiap kali** task/job jalan, tanpa perlu parsing SQL manual. Ini pendekatan paling robust & selalu up-to-date, tapi butuh effort setup tambahan (instalasi provider/listener khusus) — disinggung sebagai opsi lanjutan di `latihan_catalog_lineage_mini_project.md`, tidak wajib untuk mini project dasar.

**Rekomendasi praktis untuk skala roadmap ini**: mulai dari **manual** (diagram jelas, didokumentasikan di README) — ini sudah memberi manfaat besar (audit trail & impact analysis dasar) dengan effort setup minimal. Otomasi (poin 2-3) baru sepadan diinvestasikan kalau pipeline sudah cukup kompleks/sering berubah sehingga menjaga diagram manual tetap akurat jadi beban tersendiri.

## Kesalahan Umum

1. **Membuat diagram lineage sekali lalu tidak pernah diupdate.** Sama seperti business metadata (Hari 2), lineage yang basi (tidak mencerminkan pipeline terbaru) bisa lebih menyesatkan daripada tidak ada lineage sama sekali — orang mengambil keputusan berdasarkan peta yang sudah salah.
2. **Mengira lineage cuma berguna untuk debugging (audit trail), lupa impact analysis.** Impact analysis justru yang paling mencegah insiden **sebelum** terjadi (dicek sebelum melakukan perubahan), sementara audit trail baru dipakai **setelah** ada masalah — keduanya penting, tapi impact analysis punya nilai pencegahan yang sering terlewat.
3. **Berhenti di table-level padahal butuhnya column-level.** Untuk pertanyaan presisi ("kolom mana **persis** yang terdampak kalau saya ubah format `Price`?"), table-level lineage cuma bilang "seluruh `fact_sales` mungkin terdampak" — terlalu kasar, bisa membuat orang terlalu hati-hati (takut mengubah apapun) atau sebaliknya kurang hati-hati (tidak sadar kolom spesifik yang benar-benar terpengaruh).
4. **Mendokumentasikan lineage tapi tidak pernah dihubungkan ke catalog/dokumentasi lain.** Lineage yang berdiri sendiri (file gambar terpisah yang tidak terhubung ke deskripsi tabel di catalog) kurang termanfaatkan dibanding yang terintegrasi — orang yang sedang melihat metadata sebuah tabel di catalog harusnya bisa langsung melihat lineage-nya juga, tanpa harus tahu ada file terpisah untuk dicari.

## Latihan

1. Gambarkan (dalam teks/ASCII, tidak perlu gambar sungguhan) table-level lineage lengkap untuk `dim_customer`, dari file sumber sampai ke tabel itu ada di Postgres.
2. Tim ingin mengubah `clean_transform.py` supaya kolom `Price` yang tadinya boleh `>= 0` menjadi wajib `> 0` (lebih ketat). Pakai lineage untuk menjelaskan langkah apa yang harus dicek **sebelum** perubahan ini di-deploy, supaya tidak ada yang rusak tanpa disadari (impact analysis).
3. Dashboard revenue bulanan tiba-tiba menunjukkan angka yang jauh lebih rendah dari biasanya. Jelaskan bagaimana lineage (audit trail) membantu mempersempit pencarian penyebabnya, dibanding harus membaca ulang seluruh kode pipeline dari awal.
4. Jelaskan kenapa pipeline `ecommerce-etl-pipeline` (PySpark + Postgres, bukan murni SQL/ELT) lebih cocok memakai pendekatan **manual** atau **OpenLineage** untuk lineage-nya, dibanding pendekatan **parsing SQL** — kaitkan dengan `materi/minggu_3/hari_3_etl_elt.md`.

## Kunci Jawaban & Pembahasan

**1.**
```
online_retail_II.csv (raw, kolom "Customer ID", "Country")
    -> clean_transform.py (dropna Customer ID, drop_duplicates, filter Price>0)
    -> retail_clean (staging parquet)
    -> build_star_schema.py (SELECT DISTINCT Customer ID, Country)
    -> dim_customer (parquet)
    -> load_to_warehouse task (Airflow DAG)
    -> dim_customer (tabel Postgres)
```

**2.** Berdasarkan lineage, `Price` mengalir ke: `revenue` (`quantity * price`, dihitung di `clean_transform.py`) → `fact_sales.unit_price` dan `fact_sales.revenue` → dan (via GE) expectation `expect_column_pair_values_to_be_equal("revenue", "computed_total")` di `great_expectations/expectations/ecommerce_suite.json` (`materi/minggu_4/latihan_dq_streaming_mini_project.md`). Sebelum deploy perubahan ini, yang perlu dicek: (a) apakah ada baris `Price == 0` yang sebelumnya **valid lolos** filter tapi sekarang akan **terbuang** — cek dulu berapa baris yang akan terpengaruh dan apakah pembuangan itu memang diinginkan secara bisnis, (b) apakah expectation GE terkait `unit_price`/`revenue` masih relevan/perlu disesuaikan, (c) apakah dashboard/laporan yang bergantung ke `fact_sales` perlu diberi tahu row count-nya mungkin sedikit berkurang. Ini persis manfaat impact analysis: mencegah kejutan **sebelum** perubahan dijalankan, bukan menemukannya lewat komplain user setelah deploy.

**3.** Dengan lineage table-level (`raw CSV → retail_clean → fact_sales → dashboard`), pencarian bisa langsung dipersempit: cek dulu apakah `fact_sales` di Postgres punya row count yang wajar (kalau tiba-tiba jauh lebih sedikit dari biasanya, kemungkinan masalah ada di `load_to_warehouse` atau lebih ke hulu). Kalau row count `fact_sales` normal tapi total revenue-nya rendah, kemungkinan masalah ada di perhitungan `revenue` itu sendiri (`clean_transform.py`) atau di data quality check yang lolos padahal seharusnya tidak (celah di `ecommerce_suite.json`). Tanpa lineage, developer harus membaca ulang **seluruh** rantai kode dari nol tanpa arahan tentang di mana kemungkinan besar masalahnya — dengan lineage, pencarian jadi terarah, dimulai dari titik yang paling mungkin, mirip cara `git bisect` mempersempit rentang commit yang perlu diperiksa dibanding memeriksa semua commit satu-satu.

**4.** Pipeline ini memakai **PySpark** untuk transformasi (`clean_transform.py`, `build_star_schema.py`) — logic transformasinya berupa **kode Python/DataFrame API**, bukan query SQL yang dijalankan di warehouse (ini sudah dijelaskan sebagai pola **ETL**, bukan ELT, di `materi/minggu_3/hari_3_etl_elt.md`). Tool lineage berbasis **parsing SQL** cuma bisa membaca query SQL — tidak bisa "membaca" logic PySpark yang ditulis sebagai kode Python biasa (`.filter()`, `.withColumn()`, dst.), jadi pendekatan itu tidak akan menangkap lineage dari langkah transformasi Spark-nya. **Manual** (menulis diagram sendiri berdasarkan pemahaman kode) atau **OpenLineage** (yang punya integrasi resmi ke Spark, memancarkan event lineage langsung dari dalam eksekusi job Spark, bukan dari parsing teks SQL) jauh lebih cocok untuk arsitektur pipeline seperti ini.
