---
title: Hari 1 - Konsep Dasar Data Governance
parent: Minggu 5 - Data Governance
nav_order: 2
---

# Hari 1 — Konsep Dasar Data Governance: Apa, Kenapa, dan Pilar Utamanya

*Senin, 2 jam. Konsep murni — pakai repo `ecommerce-etl-pipeline` (Minggu 3-4) sebagai contoh kasus sepanjang materi.*

## Tujuan Belajar

- [ ] Mendefinisikan data governance dan membedakannya dari data management dan data quality
- [ ] Menjelaskan 4 pilar governance: ownership, policy, standard, stewardship
- [ ] Menjelaskan konsekuensi konkret (bukan cuma teoretis) dari tidak adanya governance
- [ ] Menerapkan pilar governance ke pipeline `ecommerce-etl-pipeline` yang sudah dibangun

## Untuk Instruktur: Mindset Shift

Developer sering mengira "governance" itu birokrasi murni — meeting, dokumen, approval process yang memperlambat kerja. Analogi yang efektif untuk meluruskan ini: **governance itu seperti code review + CODEOWNERS + style guide + access control (RBAC) di software engineering, tapi diterapkan ke data, bukan ke kode.**

| Konsep Software Engineering | Konsep Data Governance |
|---|---|
| `CODEOWNERS` file (siapa review PR ke folder X) | **Ownership** — siapa bertanggung jawab atas dataset X |
| Style guide / linter rules | **Standard** — format, penamaan, tipe data yang disepakati |
| Branch protection rules, approval policy | **Policy** — siapa boleh akses/ubah data apa, dalam kondisi apa |
| Maintainer yang menjaga kualitas & triage issue | **Stewardship** — orang yang aktif menjaga kualitas & kegunaan dataset sehari-hari |

Developer yang sudah kerja di tim menengah-besar biasanya **sudah merasakan langsung** kenapa `CODEOWNERS`/style guide penting (kode jadi kacau tanpa itu begitu tim membesar) — governance data adalah kebutuhan yang **sama persis**, cuma untuk data, bukan kode, dan sayangnya sering diabaikan lebih lama karena akibatnya tidak langsung terlihat seperti bug di production.

## Konsep & Sintaks

### Definisi: Governance vs Management vs Quality

Tiga istilah yang sering tertukar, padahal berbeda level:

| | Cakupan | Pertanyaan yang Dijawab |
|---|---|---|
| **Data Management** | **Operasional** — bagaimana data disimpan, diproses, dipindah | "Bagaimana cara kita membangun & menjalankan pipeline ini?" (ini yang dikerjakan Minggu 3-4) |
| **Data Quality** | **Teknis** — apakah isi data benar | "Apakah data ini valid?" (Great Expectations, Minggu 4) |
| **Data Governance** | **Organisasional** — siapa bertanggung jawab, aturan apa yang berlaku, bagaimana orang menemukan & mempercayai data | "Siapa yang bertanggung jawab kalau data ini salah? Siapa boleh akses? Bagaimana orang lain tahu data ini ada dan valid?" |

Data governance **membungkus** data management dan data quality — dia tidak menggantikan keduanya, tapi menyediakan struktur organisasional yang membuat keduanya bisa dipercaya dan berkelanjutan dalam skala tim/organisasi, bukan cuma benar untuk 1 orang yang paham semua konteksnya di kepala sendiri.

### 4 Pilar Governance

**1. Ownership (Kepemilikan)**

Siapa yang bertanggung jawab atas sebuah dataset — bukan cuma "siapa yang menulis kodenya", tapi siapa yang dihubungi kalau ada pertanyaan/masalah tentang data itu, dan siapa yang berwenang mengubah skemanya.

```
fact_sales     -> Owner: Data Engineering Team (dibangun & dijaga tim ini)
dim_customer   -> Owner: Data Engineering Team, tapi definisi bisnis "customer aktif" -> Sales Team
```

Analogi: sama seperti `CODEOWNERS` di GitHub — tanpa ini, saat ada bug/pertanyaan tentang sebuah modul, tidak ada yang jelas siapa yang harus dihubungi, dan akhirnya semua orang menganggap "bukan tanggung jawab saya".

**2. Policy (Kebijakan)**

Aturan formal tentang **siapa boleh melakukan apa** terhadap data — akses, penggunaan, retensi (berapa lama data disimpan sebelum dihapus).

```
Policy contoh:
- Analyst boleh SELECT dari fact_sales/dim_*, TIDAK boleh UPDATE/DELETE langsung
- Data mentah (raw) disimpan maksimal 2 tahun, lalu diarsipkan
- Kolom customer_id di-mask untuk siapapun di luar tim Data Engineering (lihat hari_5_compliance_privacy.md)
```

**3. Standard (Standar)**

Kesepakatan format, penamaan, dan tipe data yang berlaku **lintas** dataset/tim — supaya data dari sumber berbeda tetap konsisten dan bisa digabungkan.

```
Standard contoh:
- Semua tabel fact diawali "fact_", dimension diawali "dim_" (sudah diikuti sejak Minggu 3)
- Semua kolom tanggal pakai tipe DATE/TIMESTAMP, bukan string
- Semua kolom uang dalam mata uang & 2 desimal yang konsisten
```

Perhatikan: penamaan `fact_sales`/`dim_customer` yang sudah dipraktikkan sejak `materi/minggu_3/hari_2_data_modeling.md` **sebenarnya sudah menerapkan standard** — cuma belum didokumentasikan secara eksplisit sebagai kebijakan yang disepakati. Ini poin bagus untuk peserta sadari: sebagian governance **sudah** mereka praktikkan secara implisit sejak awal, minggu ini tinggal mendokumentasikan & memformalkannya.

**4. Stewardship (Kepengurusan)**

Peran/orang yang **aktif** menjaga kualitas, kegunaan, dan kepatuhan sebuah dataset sehari-hari — beda dari ownership (yang lebih ke tanggung jawab formal), steward itu yang benar-benar **mengerjakan** perawatannya: memantau hasil data quality check, meng-update deskripsi metadata, menjawab pertanyaan pengguna data. Dibahas lebih dalam di `hari_4_stewardship_quality_governance.md`.

### Kenapa Governance Penting — Konsekuensi Konkret Tanpa Itu

Bukan argumen abstrak — ini skenario yang benar-benar terjadi di organisasi tanpa governance yang memadai:

1. **Tidak ada yang tahu data itu ada.** Analyst baru menghabiskan berhari-hari membangun ulang laporan yang **sudah ada** di tabel lain, karena tidak ada catalog yang bisa dicari (topik Hari 2).
2. **Tidak ada yang tahu apa arti sebuah kolom.** Kolom `status` di suatu tabel — status apa? Siapa yang tahu nilai `'C'` di kolom `flag` artinya "cancelled" bukan "confirmed"? Tanpa dokumentasi, pengetahuan ini cuma ada di kepala 1-2 orang, hilang begitu mereka resign.
3. **Tidak ada yang bertanggung jawab saat data salah.** Dashboard menampilkan angka aneh — siapa yang harus diperbaiki? Tanpa ownership yang jelas, ini jadi "bukan tugas siapa-siapa" untuk waktu lama.
4. **Risiko compliance/hukum.** Data PII (customer_id, alamat, dsb) diakses/diekspor tanpa kontrol, berpotensi melanggar regulasi (GDPR/UU PDP — Hari 5) — ini bisa berarti sanksi finansial nyata bagi organisasi.
5. **Data swamp** (istilah dari `materi/minggu_3/hari_1_arsitektur_data.md`) — data lake/warehouse yang penuh tapi tidak dipercaya siapapun, karena tidak ada yang tahu mana data yang valid/terkini/berwenang dipakai.

## Kesalahan Umum

1. **Menganggap governance = birokrasi yang memperlambat kerja.** Governance yang baik justru **mempercepat** kerja jangka panjang (orang tidak perlu tanya-tanya manual/membangun ulang yang sudah ada) — biaya governance itu di depan (waktu setup dokumentasi), keuntungannya terasa belakangan & berkelanjutan, mirip trade-off menulis test/dokumentasi kode.
2. **Mengira governance cuma soal compliance/hukum.** Compliance (Hari 5) cuma **satu bagian** dari governance — ownership, catalog, lineage, standard semuanya bermanfaat langsung untuk produktivitas tim, terlepas dari ada tidaknya tuntutan regulasi.
3. **Mencoba menerapkan governance penuh sekaligus di awal, untuk semua dataset.** Governance yang realistis biasanya bertahap — mulai dari dataset paling kritis/paling sering dipakai, bukan mencoba mendokumentasikan semuanya sekaligus (yang sering berakhir tidak selesai sama sekali).
4. **Menganggap 1 orang/tim bisa memegang semua 4 pilar sendirian untuk semua data.** Ownership dan stewardship sering **terdistribusi** — tim data engineering biasa jadi owner teknis (skema, pipeline), tapi definisi/makna bisnis suatu kolom sering paling dikuasai tim yang datanya berasal dari sana (contoh `dim_customer` di atas: definisi bisnis "customer aktif" ada di tim Sales, bukan Data Engineering).

## Latihan

1. Untuk repo `ecommerce-etl-pipeline`, tentukan (secara hipotetis, karena ini project belajar solo) siapa yang cocok jadi **owner** untuk: (a) skema & pipeline teknis `fact_sales`, (b) definisi bisnis "apa itu transaksi valid vs retur". Jelaskan kenapa keduanya bisa berbeda orang/tim.
2. Rumuskan 1 **policy** dan 1 **standard** konkret untuk repo `ecommerce-etl-pipeline` yang belum ada tapi masuk akal ditambahkan (boleh di luar yang sudah dicontohkan di atas).
3. Jelaskan dengan kata-kata sendiri kenapa "tidak ada bug/error di pipeline" (fokus Minggu 3-4) **tidak** sama dengan "governance sudah baik" — beri 1 contoh skenario konkret di mana pipeline 100% sukses tapi governance-nya buruk.
4. Seorang rekan bilang "kita cuma tim kecil, semua orang tahu semua data, tidak perlu repot-repot governance formal." Kapan menurutmu argumen ini **benar**, dan pada titik apa (ukuran tim/kompleksitas data) argumen ini mulai **tidak lagi valid**?

## Kunci Jawaban & Pembahasan

**1.** (a) **Tim Data Engineering** — merekalah yang membangun & menjaga `spark_jobs/build_star_schema.py`, tahu detail teknis skema, dan yang paling tepat dihubungi kalau ada masalah teknis (pipeline gagal, skema berubah). (b) **Tim bisnis/analytics/finance** yang memakai datanya — merekalah yang paling paham konteks bisnis "kapan sebuah invoice dianggap retur yang sah vs anomali yang perlu diselidiki", keputusan yang **bukan** keputusan teknis murni. Keduanya berbeda karena kepemilikan **teknis** (bagaimana data dibangun & dijaga tetap jalan) dan kepemilikan **bisnis/semantik** (apa arti & definisi yang benar dari data itu) adalah dua tanggung jawab yang butuh keahlian berbeda — developer paling ahli soal "bagaimana", bukan selalu paling ahli soal "apa artinya bagi bisnis".

**2.** Contoh **policy**: "Data mentah (`data/raw/`) tidak boleh di-commit ke git (sudah masuk `.gitignore`), harus diambil ulang dari sumber aslinya — mencegah data sensitif/besar tersimpan permanen di histori repo." Contoh **standard**: "Semua nama file script transformasi harus deskriptif dalam format `verb_object.py` (`clean_transform.py`, `build_star_schema.py`, bukan `script1.py`/`final_v2.py`) — supaya siapapun yang baru buka repo bisa menebak fungsi tiap file tanpa harus membukanya."

**3.** Pipeline `ecommerce_etl_pipeline` bisa saja 100% sukses (task hijau semua, data quality check dari `great_expectations` lolos semua) — tapi kalau **tidak ada dokumentasi** yang menjelaskan kolom `is_return` itu artinya apa, **tidak ada** yang tahu siapa yang harus dihubungi kalau butuh data tambahan, dan **tidak ada** kontrol siapa saja yang boleh mengakses `customer_id` (yang merupakan PII) — governance-nya buruk walau secara teknis pipeline berjalan sempurna. Ini pembeda penting: keberhasilan teknis (task tidak error, data valid) adalah **prasyarat** governance yang baik, bukan **pengganti**-nya.

**4.** Argumen ini **benar** selama tim benar-benar kecil (mis. 2-5 orang) dan **semua** orang di tim itu memang punya visibilitas penuh & terus-menerus ke semua data yang dipakai — dalam kondisi ini, dokumentasi formal memang bisa jadi overhead yang belum sepadan manfaatnya. Argumen ini **mulai tidak valid** begitu salah satu dari ini terjadi: (a) tim bertambah orang baru yang tidak punya konteks historis, (b) ada rotasi/orang keluar dan pengetahuan tacit itu ikut hilang, (c) jumlah dataset bertambah banyak sehingga tidak ada lagi 1 orang yang "tahu semuanya di kepala", atau (d) data mulai diakses tim/departemen lain di luar tim kecil awal itu. Titik baliknya bukan angka pasti, tapi **kapan pengetahuan tentang data berhenti muat di kepala orang-orang yang masih ada di tim** — dan itu biasanya datang lebih cepat dari yang diperkirakan tim yang masih merasa "kecil".
