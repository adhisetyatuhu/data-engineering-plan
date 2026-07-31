---
title: Hari 5 - Storage Strategy Decision Framework
parent: Minggu 8 - NoSQL & Storage Strategy (Capstone)
nav_order: 6
---

# Hari 5 — Storage Strategy: Kapan Pakai SQL vs NoSQL vs Data Lake vs Data Warehouse

*Jumat, 2 jam. Sintesis dari seluruh roadmap 8 minggu — persiapan langsung menulis `STORAGE_STRATEGY.md` di mini project Sabtu-Minggu.*

## Tujuan Belajar

- [ ] Menyusun kerangka keputusan (decision framework) untuk memilih jenis storage berdasarkan karakteristik data & pola akses
- [ ] Memetakan **setiap** tool yang sudah dipakai sejak Minggu 1 (Postgres, Parquet/GCS, BigQuery, MongoDB, Redis) ke framework ini
- [ ] Menjelaskan trade-off polyglot persistence: optimalitas per-kebutuhan vs kompleksitas operasional
- [ ] Menulis tabel keputusan "kenapa data X disimpan di Y, bukan di Z" untuk kasus konkret

## Untuk Instruktur

Ini sesi **paling penting** secara portfolio, meski paling sedikit kode barunya — kemampuan menjelaskan **kenapa** memilih tool tertentu (bukan cuma bisa memakainya) adalah yang paling sering ditanyakan di interview data engineering. Dorong peserta menjawab pertanyaan latihan dengan **bahasa mereka sendiri**, bukan menghafal definisi — kalau mereka bisa menjelaskan pilihan storage untuk skenario yang **belum pernah dibahas persis** di materi manapun, itu tanda framework-nya benar-benar dipahami, bukan dihafal.

## Konsep & Sintaks

### 4 Kategori Storage yang Sudah Dipakai Sepanjang Roadmap

Sebelum masuk framework baru, kenali dulu: **seluruh** 8 minggu roadmap ini sebenarnya sudah memakai representasi dari 4 kategori storage utama, tanpa selalu disebut eksplisit sebagai "kategori":

| Kategori | Tool yang Sudah Dipakai | Kapan Diperkenalkan |
|---|---|---|
| **RDBMS (SQL)** | PostgreSQL (`pg-belajar`) | Minggu 1 |
| **Data Lake** | Google Cloud Storage (raw + processed Parquet) | Minggu 6 (`hari_2_object_storage.md`) |
| **Data Warehouse** | BigQuery | Minggu 6 (`hari_5_cloud_data_warehouse.md`) |
| **NoSQL** | MongoDB (document), Redis (key-value) | Minggu 8 (minggu ini) |

### Decision Framework: 4 Pertanyaan Kunci

Framework ini bukan flowchart kaku "jawab A maka pakai X" — tapi 4 pertanyaan yang, dijawab bersama, mengarahkan ke pilihan yang masuk akal:

**1. Seberapa terstruktur datanya, dan seberapa penting relasinya?**
```
Sangat terstruktur,           Semi-terstruktur,          Sangat tidak terstruktur
relasi jelas & penting        struktur bervariasi        (raw file, log, gambar)
        │                            │                            │
        v                            v                            v
   RDBMS (SQL)                 Document (MongoDB)            Data Lake (GCS)
   fact_sales, dim_*           dim_product + atribut          raw CSV, backup
   (Minggu 3)                  variatif (Minggu 8)            Parquet (Minggu 6)
```

**2. Untuk apa data ini akan dipakai — transaksi operasional, analitik historis, atau akses cepat berulang?**
```
Transaksi operasional        Analitik historis skala          Akses super cepat,
(butuh ACID, konsistensi     besar (agregasi, JOIN            berulang, data
kuat per transaksi)          lintas jutaan baris)              boleh "sementara"
        │                            │                            │
        v                            v                            v
   RDBMS (SQL)                Data Warehouse (BigQuery)        Key-Value (Redis)
                               separation storage/compute        caching (Minggu 8)
                               (Minggu 6)
```

**3. Seberapa besar & seberapa cepat data ini tumbuh?**
- Kecil-menengah, tumbuh stabil & terprediksi → RDBMS cukup (`pg-belajar` selama ini menangani `fact_sales` tanpa masalah)
- Sangat besar, perlu retensi jangka panjang murah → Data Lake (storage class lifecycle policy, `hari_2_object_storage.md` Minggu 6)
- Volume query tinggi tapi tidak konstan (elastis) → Data Warehouse (pay-as-you-go, Minggu 6)

**4. Apakah data ini bisa "dibangun ulang" dari sumber lain kalau hilang?**
- Ya, murni turunan/cache dari sumber lain → Redis cocok (kalau hilang, tinggal query ulang sumbernya)
- Tidak, ini sumber kebenaran (source of truth) → **harus** di storage dengan jaminan durabilitas (RDBMS, Data Warehouse, Data Lake) — **tidak boleh** cuma ada di Redis

### Tabel Perbandingan Lengkap

| | RDBMS (SQL) | Data Lake | Data Warehouse | NoSQL (Document) | NoSQL (Key-Value) |
|---|---|---|---|---|---|
| Contoh di roadmap | PostgreSQL | GCS | BigQuery | MongoDB | Redis |
| Struktur data | Kaku, terdefinisi | Bebas (raw file apapun) | Terstruktur (tabel kolumnar) | Fleksibel per dokumen | Key-value sederhana |
| Skala biaya storage | Sedang-mahal (per disk mesin) | **Sangat murah** (object storage) | Murah (dipisah dari compute) | Sedang | Mahal per-GB (in-memory) |
| Kecepatan baca | Cepat untuk data ukuran wajar | Lambat (perlu proses dulu, bukan query langsung) | Cepat untuk agregasi besar | Cepat untuk dokumen tunggal | **Tercepat** (in-memory) |
| Butuh proses sebelum berguna? | Tidak, langsung ter-query | **Ya** — raw file butuh diproses (Spark, Minggu 3) sebelum dianalisis | Tidak, langsung ter-query | Tidak | Tidak |
| Use case di roadmap | `fact_sales`/`dim_*` operasional | Raw CSV + Parquet backup | Analitik historis skala besar | Katalog produk fleksibel | Cache query mahal |

### Polyglot Persistence: Trade-Off yang Harus Disadari

**Manfaat**: tiap kebutuhan dilayani tool yang **memang dirancang** untuknya — performa & kesesuaian optimal per use case, dibanding memaksakan 1 tool generik untuk semua.

**Biaya**: 
1. **Kompleksitas operasional** — makin banyak tool, makin banyak yang perlu di-monitor, di-backup, dijaga versinya, dan dipahami tim (bandingkan: Minggu 1-2 cuma perlu paham Postgres, Minggu 8 butuh paham Postgres **+** GCS **+** BigQuery **+** MongoDB **+** Redis).
2. **Konsistensi data lintas sistem** — begitu data yang "sama" ada di lebih dari 1 tempat (mis. `dim_product` di PostgreSQL **dan** MongoDB), ada risiko keduanya jadi **tidak sinkron** kalau salah satu di-update tapi yang lain tidak — butuh strategi eksplisit (siapa jadi *source of truth*, bagaimana propagasi update) yang tidak dibutuhkan kalau cuma 1 sistem.
3. **Kurva belajar tim** — anggota tim baru harus paham **beberapa** tool sekaligus untuk berkontribusi penuh, bukan cuma 1.

**Prinsip pengambilan keputusan yang sehat**: tambahkan tool baru **hanya** kalau kebutuhan konkretnya sudah jelas terasa (seperti mini project ini — `dim_product` **terasa nyata** butuh fleksibilitas skema, query analitik **terasa nyata** mahal diulang) — bukan "menambahkan tool karena terlihat bagus di portfolio". Ini prinsip yang sama yang sudah dipegang sejak `materi/minggu_6/00_overview.md` soal Terraform ("jangan buru-buru ke infra-as-code") — kompleksitas harus dijustifikasi kebutuhan nyata, bukan ditambahkan preventif.

## Kesalahan Umum

1. **Menganggap framework ini sebagai aturan kaku "data jenis X SELALU harus di tool Y".** Ini kerangka berpikir untuk **mempertimbangkan trade-off**, bukan lookup table otomatis — data yang sama bisa punya jawaban berbeda tergantung konteks (skala tim, budget, kebutuhan spesifik) yang tidak selalu tertangkap 4 pertanyaan generik di atas.
2. **Memilih storage berdasarkan tool yang paling familiar, bukan karakteristik kebutuhan.** Godaan umum: "saya sudah jago Postgres, jadi taruh semua di Postgres" — valid untuk skala kecil (dan memang begitu strategi Minggu 1-5 roadmap ini), tapi perlu ditinjau ulang begitu skala/kebutuhan berubah signifikan (seperti yang terjadi bertahap Minggu 6 dan 8).
3. **Lupa bahwa keputusan storage bisa (dan wajar) berubah seiring waktu.** Roadmap ini sendiri contoh nyata: Minggu 1-5 cuma Postgres, Minggu 6 menambah data lake+warehouse, Minggu 8 menambah NoSQL — bukan karena rencana awal salah, tapi karena kebutuhan **berkembang** dan keputusan storage yang tepat di awal (1 Postgres, sederhana) memang seharusnya berbeda dari keputusan yang tepat di akhir (polyglot, kompleks) — mengenali **kapan** waktu yang tepat untuk transisi itu sendiri adalah skill.
4. **Menambah tool NoSQL/data lake padahal RDBMS tunggal masih lebih dari cukup untuk skala & kebutuhan nyata.** Kembali ke prinsip "jangan tambah kompleksitas preventif" — kalau `pg-belajar` masih menangani semua kebutuhan dengan baik, memaksakan migrasi ke polyglot cuma menambah biaya operasional tanpa manfaat nyata.

## Latihan

1. Sebuah tim kecil (2 data engineer) mengelola e-commerce dengan skala data sedang (jutaan baris transaksi/tahun, bukan miliaran) — mereka mempertimbangkan migrasi penuh ke arsitektur polyglot (RDBMS + data lake + warehouse + MongoDB + Redis) seperti roadmap ini. Beri rekomendasi: kapan ini **sepadan**, dan kapan ini **prematur** untuk skala tim & data seperti ini?
2. Terapkan 4 pertanyaan framework di atas untuk memutuskan storage yang tepat bagi: log aplikasi mentah (jutaan baris teks per hari, jarang di-query kecuali saat debugging insiden).
3. `dim_product` sekarang ada di **dua** tempat: PostgreSQL/BigQuery (Minggu 3/6) **dan** MongoDB (Minggu 8). Jelaskan risiko konkret dari duplikasi ini, dan rancang (secara konsep) siapa yang sebaiknya jadi *source of truth*.
4. Jelaskan ke calon rekruter/interviewer (dalam 3-4 kalimat): kenapa keputusan storage di repo `ecommerce-etl-pipeline` berubah dari "1 Postgres" (Minggu 1) jadi "5 tool berbeda" (Minggu 8) — apa yang membuat progresi ini terlihat sebagai **keputusan sadar bertahap**, bukan sekadar "menumpuk tool baru tiap minggu karena kurikulum menyuruh".

## Kunci Jawaban & Pembahasan

**1.** **Prematur** untuk skala ini, kecuali ada kebutuhan **konkret** yang sudah terasa nyata (bukan cuma "siapa tahu nanti butuh"). Untuk jutaan baris/tahun dengan tim 2 orang, PostgreSQL tunggal (mungkin dengan sedikit optimasi index/partisi) kemungkinan besar **masih lebih dari cukup** — migrasi ke polyglot penuh akan menambah beban operasional signifikan (5 sistem untuk dipelihara 2 orang) tanpa manfaat performa yang benar-benar terasa di skala ini. **Sepadan** dilakukan kalau muncul kebutuhan spesifik yang jelas: mis. tim mulai kewalahan menangani query analitik berat yang mengganggu performa transaksi (waktunya pertimbangkan data warehouse terpisah), atau katalog produk mulai punya struktur sangat bervariasi yang menyulitkan skema tabel (waktunya pertimbangkan MongoDB) — keputusan diambil **reaktif terhadap masalah nyata**, bukan **proaktif meniru arsitektur roadmap belajar** yang memang sengaja mencakup semua tool untuk tujuan pembelajaran.

**2.** (1) Struktur: **sangat tidak terstruktur** (teks bebas) → condong Data Lake. (2) Kegunaan: bukan transaksi, bukan analitik rutin — cuma dibaca **sesekali** saat insiden → tidak butuh kecepatan query seperti warehouse. (3) Volume: **sangat besar** (jutaan baris/hari) dan terus bertambah → butuh storage murah untuk volume besar, RDBMS akan sangat mahal untuk ini. (4) Bisa dibangun ulang? Umumnya tidak (log historis tidak bisa "dibuat ulang" begitu hilang) tapi juga bukan data kritikal harian → toleransi terhadap latency akses tinggi. **Kesimpulan**: **Data Lake** (GCS dengan lifecycle policy, `hari_2_object_storage.md`) adalah pilihan paling tepat — murah untuk volume besar, tidak butuh struktur ketat, dan wajar diakses jarang/lambat karena cuma dibutuhkan saat debugging.

**3.** **Risiko konkret**: kalau `dim_product` di PostgreSQL/BigQuery di-update (mis. deskripsi produk diperbaiki) tapi versi di MongoDB **tidak** ikut di-update (atau sebaliknya), kedua sistem akan menampilkan informasi **berbeda** untuk produk yang sama — pengguna/laporan yang mengakses sistem berbeda bisa mendapat jawaban yang tidak konsisten, masalah kepercayaan data yang serius. **Rancangan source of truth**: PostgreSQL/BigQuery (star schema, hasil `build_star_schema.py`) sebaiknya tetap jadi **source of truth** untuk data produk inti (`stock_code`, `description`) karena itu tempat data ini **pertama kali** dihasilkan dari pipeline utama (Minggu 3) — MongoDB berperan sebagai **proyeksi turunan** yang diperkaya atribut kategori (mirip pola ETL: PostgreSQL/BigQuery sumber, MongoDB hasil transformasi/pengayaan khusus untuk kebutuhan katalog). Proses migrasi (`nosql/mongo_migration.py`, mini project) idealnya dijalankan **berkala** (bukan sekali manual selamanya) supaya perubahan di sumber tetap ter-propagasi ke MongoDB — poin desain yang baik disadari meski di luar scope implementasi wajib mini project ini.

**4.** Contoh jawaban: "Repo ini dimulai dengan 1 PostgreSQL karena di awal, kebutuhannya sederhana — data terstruktur, volume kecil, 1 tim kecil. Seiring kebutuhan berkembang — volume data besar butuh storage murah (data lake), analitik berat butuh compute elastis (data warehouse), katalog produk butuh skema fleksibel (MongoDB), query mahal butuh diakses cepat berulang (Redis) — tiap tool ditambahkan **spesifik** untuk menjawab keterbatasan nyata yang dialami, bukan ditambahkan sekaligus di awal. `STORAGE_STRATEGY.md` mendokumentasikan alasan tiap keputusan ini, supaya siapa pun yang membaca repo ini paham bukan cuma **apa** yang dipakai, tapi **kenapa** — itu yang saya anggap lebih penting ditunjukkan dibanding sekadar daftar tool yang saya kuasai."
