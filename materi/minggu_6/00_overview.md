---
title: Overview
parent: Minggu 6 - Cloud Platform Fundamentals
nav_order: 1
---

# Modul Minggu 6 — Cloud Platform Fundamentals (untuk Software Developer)

> Pendamping `minggu_6.md` (jadwal & outline). File ini konten pengajaran lengkap: penjelasan konsep, analogi kode, contoh, latihan, kunci jawaban. Lihat juga `materi/minggu_3/` sampai `materi/minggu_5/` — minggu ini memindahkan (sebagian) infrastruktur yang sudah dibangun ke cloud, bukan membangun ulang dari nol.

## Provider yang Dipakai di Modul Ini: GCP

Sesuai `minggu_6.md` bagian "Pilih 1 Provider Dulu": **konsepnya sama di semua cloud provider besar** (storage, compute, IAM, data warehouse selalu ada padanannya), yang beda cuma nama layanan & detail UI/CLI. Modul ini memakai **GCP** sebagai provider utama untuk semua contoh kode & langkah hands-on, dengan alasan yang sama seperti disebut `minggu_6.md`: learning curve BigQuery paling ramah, dan **BigQuery Sandbox** memungkinkan belajar query tanpa perlu setup billing account sama sekali (poin penting untuk menghindari risiko biaya tak terduga).

**Kalau kamu memilih AWS**: tiap konsep di modul ini disertai tabel padanan istilah AWS — logikanya 1:1, cuma nama layanan & sedikit detail CLI yang beda. Jangan pindah-pindah provider di tengah jalan (`minggu_6.md` eksplisit bilang "pilih 1 dulu") — konsistensi provider penting supaya kredensial/setup yang sudah dikonfigurasi di 1 hari tetap dipakai di hari-hari berikutnya.

## Shift Minggu Ini: dari "Kita yang Kelola Infrastruktur" ke "Provider yang Kelola Infrastruktur"

Sepanjang Minggu 1-5, **semua infrastruktur** (Postgres, Airflow, Kafka, OpenMetadata) dijalankan **sendiri** lewat Docker di laptop/mesin sendiri — kita bertanggung jawab penuh: instalasi, patching, resource, backup (yang mana tidak pernah benar-benar disiapkan, karena ini konteks belajar lokal). Minggu ini memperkenalkan **model tanggung jawab yang berbeda**: sebagian infrastruktur (storage, compute, keamanan jaringan fisik) sekarang **dikelola provider**, kita cuma bertanggung jawab atas konfigurasi & data di dalamnya — ini disebut **shared responsibility model**, dibahas detail di `hari_1_konsep_cloud.md`.

Ini bukan sekadar "pindah tempat jalanin Docker" — ada primitif baru yang tidak ada padanannya persis di setup lokal: **object storage** (bukan filesystem biasa), **IAM** (bukan cuma `docker-compose` environment variable), dan **cloud data warehouse** yang memisahkan storage & compute secara arsitektural (beda fundamental dari Postgres lokal yang menyatukan keduanya di 1 mesin).

## Kenapa Bobot Materinya Begini

- **Senin–Kamis**: fondasi konsep cloud (computing model, storage, compute, IAM) — berurutan dari yang paling abstrak (Senin) ke yang paling konkret dipakai di mini project (Kamis: IAM, langsung dipakai Sabtu).
- **Jumat**: cloud data warehouse — sengaja diletakkan setelah IAM & storage, karena data warehouse cloud **butuh** keduanya (data masuk lewat storage, akses diatur lewat IAM) untuk benar-benar dipahami konteksnya.
- **Sabtu–Minggu**: hands-on migrasi pipeline yang sudah ada ke cloud — bukan pipeline baru, murni **migrasi** (mengangkat apa yang sudah dibangun Minggu 3, memindahkan targetnya).
- Minggu ini **lebih padat** dari kapasitas biasa (15-20 jam vs 18 jam) — `minggu_6.md` sudah mengantisipasi ini dengan catatan boleh geser ke Minggu 7 kalau setup akun/billing makan waktu lebih lama dari perkiraan. Ini realistis: setup akun cloud pertama kali (verifikasi identitas, kartu kredit untuk free trial, dsb) sering makan waktu tidak terduga, di luar kendali kecepatan belajar peserta.

## Setup Environment

```bash
# Google Cloud SDK (gcloud CLI)
# Mac: brew install --cask google-cloud-sdk
# atau download installer dari cloud.google.com/sdk

gcloud init                      # login & pilih/buat project
gcloud auth application-default login   # kredensial untuk SDK/client library (dipakai script Python)

pip install google-cloud-storage google-cloud-bigquery
```

- **Buat GCP project baru khusus untuk roadmap ini** (misal `ecommerce-etl-pipeline-belajar`) — jangan pakai project yang sudah ada isinya, supaya gampang dihapus bersih di akhir kalau sudah tidak dipakai, dan supaya biaya/resource yang dipakai jelas terisolasi untuk keperluan belajar.
- **BigQuery Sandbox** (untuk `hari_5_cloud_data_warehouse.md` dan sebagian latihan query) tidak butuh billing account sama sekali — cukup akun Google biasa. **Cloud Storage** (Selasa, `hari_2_object_storage.md`) **butuh** billing account aktif (walau dalam batas free trial credit $300/90 hari untuk akun baru) — perbedaan ini penting diketahui dari awal supaya tidak kaget di tengah jalan.
- **Set budget alert SEBELUM mulai apapun** (lihat `latihan_cloud_migration_mini_project.md` Bagian 1) — sesuai peringatan eksplisit di `minggu_6.md`.

## Dataset & Repo yang Dipakai Minggu Ini

Tidak ada dataset baru — minggu ini memindahkan **output** pipeline Minggu 3 (raw CSV, Parquet hasil `build_star_schema.py`) ke Google Cloud Storage, lalu me-load-nya ke BigQuery. Repo tetap `ecommerce-etl-pipeline`, ditambah folder `cloud/`.

## Struktur Modul

| File | Sesuai Jadwal `minggu_6.md` | Topik |
|---|---|---|
| [`hari_1_konsep_cloud.md`](hari_1_konsep_cloud.md) | Senin, 2 jam | IaaS/PaaS/SaaS, region & availability zone, shared responsibility model |
| [`hari_2_object_storage.md`](hari_2_object_storage.md) | Selasa, 2 jam | Object storage: bucket, storage class/tiering, lifecycle policy |
| [`hari_3_compute.md`](hari_3_compute.md) | Rabu, 2 jam | VM dasar (Compute Engine) + serverless (Cloud Functions) |
| [`hari_4_iam.md`](hari_4_iam.md) | Kamis, 2 jam | IAM: principal, role, policy, service account, least privilege |
| [`hari_5_cloud_data_warehouse.md`](hari_5_cloud_data_warehouse.md) | Jumat, 2 jam | Cloud data warehouse: BigQuery/Redshift/Snowflake, separation of storage & compute |
| [`latihan_cloud_migration_mini_project.md`](latihan_cloud_migration_mini_project.md) | Sabtu (4 jam) + Minggu (4 jam) | Hands-on: setup bucket + IAM, load ke BigQuery, jalankan ulang query Minggu 1-2 |

Struktur tiap file `hari_X` sama dengan minggu-minggu sebelumnya: Tujuan Belajar → Untuk Instruktur → Konsep & Sintaks → Contoh → Kesalahan Umum → Latihan → Kunci Jawaban.

## Catatan Cara Mengajar

- **Tekankan biaya sejak hari pertama.** Beda dari semua tool sebelumnya (Docker lokal = gratis selama laptop kuat), cloud punya risiko **biaya finansial nyata** kalau salah konfigurasi (lupa matikan resource, dsb). Ajari kebiasaan "cek estimasi biaya sebelum klik create" sebagai refleks, bukan cuma sekali diingatkan di awal.
- **Sambungkan terus ke konsep minggu-minggu sebelumnya**: object storage = data lake (`materi/minggu_3/hari_1_arsitektur_data.md`), cloud data warehouse elastis = alasan ELT populer (`materi/minggu_3/hari_3_etl_elt.md`), IAM = penerapan teknis dari access policy yang sudah ditulis di `GOVERNANCE.md` (`materi/minggu_5/latihan_catalog_lineage_mini_project.md`).
- **Jangan buru-buru ke Terraform/infra-as-code** — `cloud/terraform/` di struktur repo `minggu_6.md` eksplisit ditandai **opsional**. Fokus minggu ini paham konsep & bisa setup manual dulu (Console/CLI), infra-as-code itu topik lanjutan yang layak dipelajari terpisah setelah fundamentalnya kuat.
- Total waktu: 5 hari × 2 jam + Sabtu 4 jam + Minggu 4 jam = 18 jam (dengan potensi meleber ke Minggu 7 kalau setup akun makan waktu, sesuai catatan `minggu_6.md`).
