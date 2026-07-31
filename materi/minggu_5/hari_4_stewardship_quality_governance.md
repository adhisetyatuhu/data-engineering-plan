---
title: Hari 4 - Data Stewardship & Quality Governance
parent: Minggu 5 - Data Governance
nav_order: 5
---

# Hari 4 — Data Quality dalam Konteks Governance & Data Stewardship

*Kamis, 2 jam. Menyambungkan langsung ke `materi/minggu_4/hari_1_data_quality_dimensions.md` dan `hari_2_great_expectations.md` — kalau belum familiar dengan itu, baca ulang dulu sebelum lanjut.*

## Tujuan Belajar

- [ ] Menjelaskan kenapa data quality adalah **bagian dari** governance, bukan topik terpisah
- [ ] Membedakan peran Data Owner, Data Steward, dan Data Custodian
- [ ] Merancang proses eskalasi: apa yang terjadi (secara organisasional, bukan cuma teknis) saat `data_quality_check` gagal
- [ ] Menjelaskan konsep quality score/badge di data catalog dan kegunaannya

## Untuk Instruktur: Mindset Shift

Minggu 4 menjawab **"bagaimana cara mendeteksi data buruk secara teknis"** (Great Expectations, fail-fast). Minggu ini melengkapi sisi yang belum dijawab: **"begitu terdeteksi, siapa yang menangani, dan bagaimana orang lain tahu ada masalah?"** Great Expectations bisa membuat `data_quality_check` gagal merah di Airflow UI — tapi kalau tidak ada **proses** yang jelas siapa yang harus melihat kegagalan itu dan bertindak, alert tersebut cuma jadi "noise merah" yang lama-lama diabaikan (persis peringatan yang sudah disebut di `materi/minggu_4/hari_2_great_expectations.md` poin Kesalahan Umum #2: expectation yang terlalu berisik membuat orang berhenti percaya).

Analogi yang pas: ini sama seperti bedanya **CI test yang gagal** dengan **proses on-call/incident response** di software engineering. Test yang gagal (Great Expectations) itu **deteksi teknis**. Siapa yang di-page, siapa yang triase, siapa yang punya otoritas memutuskan "ini boleh diabaikan sementara atau harus diperbaiki sekarang" — itu **proses organisasional** (governance), dan tanpanya, test yang gagal pun tidak banyak berguna.

## Konsep & Sintaks

### Data Quality sebagai Bagian dari Governance, Bukan Topik Terpisah

Ingat kembali definisi Hari 1: governance menjawab "siapa bertanggung jawab, aturan apa yang berlaku, bagaimana data dipercaya". Data quality (Minggu 4) adalah **mekanisme teknis** untuk menegakkan salah satu bagian dari itu — "bagaimana data dipercaya" secara konkret diukur lewat expectation yang lolos/gagal. Tapi expectation itu sendiri **adalah** sebuah kebijakan (policy, dari Hari 1) — `expect_column_values_to_not_be_null("customer_id")` adalah **kesepakatan formal** bahwa `customer_id` yang kosong dianggap tidak dapat diterima, persis definisi *policy*.

```
Governance (Hari 1)         Diterapkan lewat...          Contoh Konkret
Policy                  ->  Expectation GE            -> "customer_id tidak boleh null"
Ownership               ->  Siapa yang di-notify        -> on_failure_callback -> Data Steward fact_sales
Stewardship              -> Siapa yang triase & perbaiki -> Proses eskalasi (di bawah)
Standard                 -> Definisi "valid" yang disepakati -> ecommerce_suite.json sebagai dokumentasi hidup
```

### 3 Peran: Owner, Steward, Custodian

Istilah yang sering tertukar — perbedaannya penting untuk merancang proses eskalasi yang jelas:

| Peran | Tanggung Jawab | Analogi Software Engineering |
|---|---|---|
| **Data Owner** | Akuntabilitas **akhir** atas dataset — biasanya level manajerial, memutuskan kebijakan besar (siapa boleh akses, berapa lama retensi) | Engineering Manager/Tech Lead sebuah tim — bertanggung jawab atas hasil akhir, tidak selalu mengerjakan detail sehari-hari |
| **Data Steward** | Menjaga kualitas & kegunaan sehari-hari — orang yang **benar-benar** dihubungi kalau `data_quality_check` gagal, meng-update deskripsi metadata, menjawab pertanyaan pengguna data | Maintainer aktif sebuah repo — yang benar-benar review PR & triase issue hari ke hari |
| **Data Custodian** | Menjaga infrastruktur **teknis** tempat data disimpan/diproses — akses database, backup, keamanan infrastruktur | Platform/Infra engineer — menjaga server/database tetap jalan, tapi belum tentu paham isi/makna datanya |

Untuk `ecommerce-etl-pipeline` (konteks belajar solo, tapi baik dirancang seolah-olah tim sungguhan):
```
fact_sales:
  Owner:     Head of Data (akuntabilitas keputusan besar: siapa boleh akses PII, dsb)
  Steward:   Data Engineer yang membangun pipeline ini (triase kegagalan data_quality_check,
             update deskripsi di catalog, jadi kontak utama pertanyaan sehari-hari)
  Custodian: (kalau infrastruktur dikelola tim terpisah) Platform Engineering Team --
             menjaga Postgres/Airflow/Kafka tetap jalan, tanpa perlu paham arti bisnis datanya
```

Dalam tim kecil (seperti kebanyakan skenario portfolio individu), 1 orang bisa memegang ketiga peran sekaligus — tapi **tetap berguna** memisahkan konsepnya secara eksplisit, karena begitu tim membesar, peran-peran ini yang pertama kali perlu didistribusikan ke orang berbeda.

### Proses Eskalasi Saat Data Quality Check Gagal

Ini yang melengkapi `on_failure_callback` dari `materi/minggu_4/hari_3_airflow_lanjutan.md` — alert teknisnya sudah ada, sekarang perlu **proses** yang jelas setelah alert terkirim:

```
1. data_quality_check gagal -> on_failure_callback terkirim (Slack/email)
2. Data Steward fact_sales menerima alert, TRIASE dalam waktu yang disepakati (mis. 1 hari kerja):
   a. Apakah ini kegagalan SUNGGUHAN (data memang rusak) -> perbaiki sumber/pipeline, re-run
   b. Apakah ini FALSE POSITIVE (expectation terlalu ketat/tidak relevan lagi) -> revisi
      ecommerce_suite.json, dokumentasikan alasan perubahan
   c. Apakah ini masalah yang butuh keputusan Data Owner (mis. sumber data eksternal berubah
      format permanen, perlu keputusan bisnis) -> eskalasi ke Owner
3. Hasil triase didokumentasikan (bahkan sesederhana log/issue tracker) -- supaya kegagalan
   serupa di masa depan tidak perlu ditriase dari nol lagi
```

Poin penting: **tidak semua kegagalan berarti "data rusak"** — kadang kegagalan berarti **aturan (expectation)** yang sudah usang dan perlu direvisi (mis. daftar `country` valid di `materi/minggu_4/latihan_dq_streaming_mini_project.md` perlu ditambah kalau bisnis mulai beroperasi di negara baru). Data Steward-lah yang punya konteks untuk membedakan ini — inilah kenapa perannya tidak bisa digantikan sepenuhnya oleh otomasi.

### Quality Score/Badge di Data Catalog

Tool catalog modern (termasuk OpenMetadata yang dipakai di `latihan_catalog_lineage_mini_project.md`) bisa menampilkan **skor/badge kualitas** langsung di halaman sebuah dataset — biasanya dihitung dari persentase expectation yang lolos pada validasi terakhir, ditampilkan sebagai indikator visual (hijau/kuning/merah) tanpa orang perlu membuka laporan Great Expectations secara terpisah.

**Kegunaannya**: seseorang yang **menemukan** `fact_sales` lewat catalog (Hari 2) bisa langsung tahu "apakah data ini bisa dipercaya untuk dipakai sekarang" **sebelum** mulai memakainya — menyambungkan langsung hasil kerja teknis Minggu 4 (Great Expectations) ke kegunaan discovery Minggu 5 (catalog). Tanpa integrasi ini, hasil validasi GE cuma terlihat di log Airflow — berguna untuk debugging, tapi tidak terlihat oleh orang yang sekadar mencari dataset untuk dipakai.

## Kesalahan Umum

1. **Menganggap Data Owner harus menangani semua kegagalan teknis sehari-hari.** Ini justru peran Data Steward — Owner idealnya cuma dilibatkan untuk keputusan besar/eskalasi, kalau setiap kegagalan kecil langsung ke Owner, prosesnya jadi bottleneck dan Owner kewalahan menangani hal-hal yang seharusnya bisa ditriase di level Steward.
2. **Tidak mendokumentasikan hasil triase.** Kegagalan yang sama (mis. negara baru muncul di data, bikin `expect_column_values_to_be_in_set` gagal) bisa terjadi berulang kali — tanpa dokumentasi triase sebelumnya, tiap kejadian ditangani dari nol seolah baru pertama kali terjadi.
3. **Memperlakukan semua expectation yang gagal dengan urutan prioritas yang sama.** Kegagalan `expect_column_values_to_not_be_null("customer_id")` (completeness, data benar-benar hilang) biasanya lebih mendesak dibanding kegagalan `expect_column_values_to_be_in_set("country", ...)` yang mungkin cuma berarti daftar negaranya perlu diupdate — proses eskalasi yang baik membedakan urgensi, bukan memperlakukan semua alert setara.
4. **Tidak ada quality score/indikator yang terlihat di catalog**, sehingga orang yang menemukan dataset lewat pencarian catalog tidak tahu riwayat kualitasnya sampai mereka sudah terlanjur mulai memakai datanya dan menemukan masalah sendiri.

## Latihan

1. `expect_column_values_to_be_in_set("country", COUNTRY_ALLOWLIST)` di `ecommerce_suite.json` (Minggu 4) gagal karena muncul transaksi dari negara baru yang sah (bisnis baru saja mulai melayani pasar baru). Jelaskan langkah triase yang tepat, dan siapa (peran apa) yang berwenang memutuskan tindakan akhirnya.
2. Untuk repo `ecommerce-etl-pipeline`, tuliskan (secara hipotetis) siapa/peran apa yang cocok jadi Owner, Steward, dan Custodian untuk **setiap** komponen berikut: (a) skema `fact_sales`, (b) infrastruktur Airflow/Postgres yang menjalankannya, (c) keputusan kebijakan retensi data mentah.
3. Jelaskan kenapa "quality score hijau" di catalog **tidak** sama dengan "data ini 100% benar secara bisnis" — apa batasan dari skor yang cuma berdasarkan expectation yang sudah ditulis?
4. Bandingkan: proses eskalasi tanpa dokumentasi triase vs dengan dokumentasi triase, untuk kegagalan yang **berulang** (terjadi berkali-kali dalam setahun). Kuantifikasi (secara kasar) kenapa dokumentasi menghemat waktu tim dalam jangka panjang.

## Kunci Jawaban & Pembahasan

**1.** Triase: ini adalah kasus **false positive relatif terhadap kondisi bisnis yang berubah** (bukan data rusak) — data yang masuk memang **valid**, cuma aturan `COUNTRY_ALLOWLIST` yang sudah usang (belum mencakup pasar baru). Langkah: Data Steward mengonfirmasi ke tim bisnis bahwa ekspansi ke negara baru itu memang sah, lalu **merevisi** `ecommerce_suite.json` menambahkan negara baru ke allowlist, mendokumentasikan alasan perubahan (kapan, kenapa). Untuk perubahan **aturan/kebijakan** seperti ini (bukan sekadar "jalankan ulang pipeline"), idealnya tetap ada persetujuan/sepengetahuan **Data Owner** — karena mengubah definisi "valid" adalah perubahan kebijakan (policy, Hari 1), bukan cuma perbaikan bug teknis murni yang bisa diputuskan sendiri oleh siapapun yang kebetulan menerima alert.

**2.**
```
(a) Skema fact_sales:      Owner: Head of Data Engineering | Steward: Data Engineer pemilik pipeline
(b) Infrastruktur teknis:  Custodian: Platform/Infra Engineering Team
(c) Kebijakan retensi:      Owner: Head of Data (dengan kemungkinan melibatkan tim Legal/Compliance
                            untuk keputusan yang menyentuh regulasi -- lihat hari_5_compliance_privacy.md)
```

**3.** Quality score yang dihitung dari expectation yang lolos/gagal **cuma sebaik expectation yang sudah ditulis** — kalau ada aturan bisnis penting yang belum pernah diterjemahkan jadi expectation (mis. tidak ada expectation yang mengecek "total revenue bulanan tidak boleh melonjak >500% dari bulan sebelumnya tanpa alasan jelas" — sebuah anomali bisnis yang valid secara skema tapi janggal secara konteks), data bisa tetap "hijau" (semua expectation yang **ada** lolos) padahal ada masalah nyata yang belum tertangkap sama sekali. Skor hijau berarti "memenuhi semua aturan yang **sudah kita definisikan**", bukan jaminan absolut "benar secara bisnis dalam segala hal" — batasan ini penting disadari supaya orang tidak terlalu percaya diri hanya karena melihat badge hijau.

**4.** Tanpa dokumentasi triase: tiap kali kegagalan serupa terjadi (mis. 3x setahun, kasus "negara baru muncul"), Steward yang menerima alert (bisa jadi orang berbeda tiap kali kalau ada rotasi tim) harus **menginvestigasi dari nol** — memastikan ini bukan data rusak, mengonfirmasi ke tim bisnis, baru memutuskan revisi expectation — proses yang mungkin makan beberapa jam tiap kali. Dengan dokumentasi triase (catatan singkat: "kegagalan `country` allowlist biasanya berarti ekspansi pasar baru yang sah, cek dengan tim Sales sebelum revisi suite"), Steward berikutnya bisa langsung mengenali pola ini dari riwayat, mempercepat triase jadi hitungan menit, bukan jam. Untuk kegagalan yang memang berulang, penghematan ini terakumulasi signifikan — investasi kecil menulis dokumentasi triase (beberapa menit sekali) menghemat waktu triase berulang (jam) tiap kali kejadian serupa muncul lagi, sama seperti manfaat menulis runbook incident response di software engineering.
