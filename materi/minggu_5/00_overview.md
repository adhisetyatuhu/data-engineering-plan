---
title: Overview
parent: Minggu 5 - Data Governance
nav_order: 1
---

# Modul Minggu 5 — Data Governance (untuk Software Developer)

> Pendamping `minggu_5.md` (jadwal & outline). File ini konten pengajaran lengkap: penjelasan konsep, analogi kode, contoh, latihan, kunci jawaban. Lihat juga `materi/minggu_3/` dan `materi/minggu_4/` — minggu ini adalah **lapisan tambahan** di atas pipeline yang sudah dibangun, bukan project baru.

## Shift Minggu Ini: dari "Datanya Benar" ke "Datanya Bisa Dipercaya & Dikelola"

Minggu 4 menjawab pertanyaan **"apakah data ini valid?"** (lewat Great Expectations). Minggu 5 menjawab pertanyaan yang lebih luas: **"siapa yang bertanggung jawab atas data ini, dari mana asalnya, siapa boleh melihatnya, dan bagaimana orang lain di organisasi tahu data ini ada dan bisa dipercaya?"** — pertanyaan yang tidak terjawab walau pipeline-nya sudah sempurna secara teknis.

Ini bukan materi "tools baru" dalam artian yang sama seperti Spark/Airflow/Kafka. Governance lebih ke **proses, dokumentasi, dan kesepakatan** — tools (data catalog) di sini cuma alat bantu untuk menegakkan proses itu, bukan inti dari topiknya sendiri. Ini juga alasan kenapa minggu ini punya jam materi konsep sedikit lebih ringan (10-15 jam vs 18 jam) dengan slack buffer — governance itu topik yang lebih "lebar" dari "dalam", banyak yang perlu diketahui secara garis besar, tidak semuanya butuh hands-on mendalam seperti Spark/Airflow.

Analogi yang bisa dipakai berulang sepanjang minggu ini: kalau Minggu 1-4 itu tentang **membangun sistem yang jalan dan benar**, Minggu 5 itu tentang **menulis dokumentasi, README, kebijakan akses, dan code ownership** untuk sistem itu — hal-hal yang developer berpengalaman tahu **sama pentingnya** dengan kode itu sendiri begitu sebuah sistem dipakai lebih dari 1 orang, dalam jangka panjang.

## Kenapa Bobot Materinya Begini

- **Senin–Rabu**: fondasi konseptual governance, catalog, lineage — berurutan karena tiap topik membangun di atas yang sebelumnya (governance butuh catalog untuk didokumentasikan, catalog butuh lineage untuk lengkap).
- **Kamis**: sengaja diletakkan setelah lineage, karena menyambungkan balik ke Great Expectations (Minggu 4) — data quality itu **bagian dari** governance, bukan topik terpisah. Kalau dibahas duluan, koneksi ini kurang terasa.
- **Jumat**: compliance & privasi — topik yang paling "hukum/kebijakan", ditutup di akhir minggu konsep sebelum masuk hands-on.
- **Sabtu–Minggu**: hands-on setup data catalog + dokumentasi lineage & kebijakan — proporsinya sengaja lebih ke **dokumentasi** dibanding coding dibanding minggu-minggu sebelumnya, mencerminkan sifat asli pekerjaan governance.

## Setup Environment

```bash
# OpenMetadata (direkomendasikan dibanding DataHub untuk konteks belajar -- setup lebih ringan)
mkdir -p governance && cd governance
curl -sL -o docker-compose-openmetadata.yml \
  https://github.com/open-metadata/OpenMetadata/releases/download/1.5.0-release/docker-compose.yml
docker compose -f docker-compose-openmetadata.yml up -d

# CLI ingestion framework (opsional, dipakai di latihan_catalog_lineage_mini_project.md)
pip install "openmetadata-ingestion[postgres]"
```

- **OpenMetadata** dipilih sebagai tool utama (bukan DataHub) murni pertimbangan praktis untuk belajar mandiri: kebutuhan resource lebih ringan dan quickstart Docker-nya lebih langsung siap pakai. Konsepnya (catalog, metadata, lineage, tag) **sama** antara OpenMetadata, DataHub, Amundsen — kalau di tempat kerja nanti pakai tool berbeda, pengetahuan minggu ini tetap transfer, cuma UI/detail setup yang beda (dibahas perbandingannya di `hari_2_data_catalog_metadata.md`).
- Butuh RAM cukup (idealnya 6-8GB dialokasikan ke Docker) — OpenMetadata menjalankan beberapa service sekaligus (server, MySQL, Elasticsearch, Airflow internal untuk ingestion). Kalau resource laptop terbatas, alternatifnya didiskusikan di `latihan_catalog_lineage_mini_project.md`.

## Dataset & Repo yang Dipakai Minggu Ini

Tidak ada dataset baru — minggu ini bekerja **sepenuhnya** di atas apa yang sudah ada: warehouse Postgres (`dim_customer`, `dim_product`, `dim_date`, `fact_sales`) dari `materi/minggu_3/`, dan pipeline yang sudah punya data quality gate dari `materi/minggu_4/`. Kalau keduanya belum jalan sukses, selesaikan itu dulu — governance mendokumentasikan & mengatur sesuatu yang **sudah ada**, tidak bisa berdiri sendiri tanpa itu.

## Struktur Modul

| File | Sesuai Jadwal `minggu_5.md` | Topik |
|---|---|---|
| [`hari_1_konsep_governance.md`](hari_1_konsep_governance.md) | Senin, 2 jam | Apa itu governance, kenapa penting, pilar (ownership, policy, standard, stewardship) |
| [`hari_2_data_catalog_metadata.md`](hari_2_data_catalog_metadata.md) | Selasa, 2 jam | Data catalog, metadata (technical/business/operational), perbandingan tools |
| [`hari_3_data_lineage.md`](hari_3_data_lineage.md) | Rabu, 2 jam | Lineage: audit trail, impact analysis, table-level vs column-level |
| [`hari_4_stewardship_quality_governance.md`](hari_4_stewardship_quality_governance.md) | Kamis, 2 jam | Data quality dalam konteks governance (sambungan Great Expectations Minggu 4), data stewardship |
| [`hari_5_compliance_privacy.md`](hari_5_compliance_privacy.md) | Jumat, 2 jam | GDPR/UU PDP, PII, masking/anonymization/pseudonymization |
| [`latihan_catalog_lineage_mini_project.md`](latihan_catalog_lineage_mini_project.md) | Sabtu (4 jam) + Minggu (4 jam) | Hands-on: setup OpenMetadata, ingest metadata, dokumentasi lineage, `GOVERNANCE.md` |

Struktur tiap file `hari_X` sama dengan minggu-minggu sebelumnya: Tujuan Belajar → Untuk Instruktur → Konsep & Sintaks → Contoh → Kesalahan Umum → Latihan → Kunci Jawaban.

## Catatan Cara Mengajar

- **Ini minggu paling "non-teknis" sejauh ini** — jangan salah kira itu berarti lebih mudah/kurang penting. Justru topik seperti ini yang sering dianggap remeh oleh developer junior, padahal jadi pembeda nyata level mid-senior di dunia kerja data engineering (persis alasan di `minggu_5.md`: "skill pembeda level mid-senior").
- **Selalu tarik balik ke pipeline yang sudah dibangun** — governance yang diajarkan abstrak/generik gampang terasa teoretis. Tiap konsep (ownership, lineage, PII) sebaiknya langsung ditanya: "kalau diterapkan ke `fact_sales`/`dim_customer` kita, bentuknya seperti apa?"
- **Compliance (Hari 5) bukan sesi hukum formal** — cukup level "developer perlu tahu istilah dan prinsip dasarnya untuk bisa berdiskusi dengan tim legal/compliance", bukan menjadikan peserta ahli hukum privasi data.
- Total waktu: 5 hari × 2 jam + Sabtu 4 jam + Minggu 4 jam = 18 jam (dengan slack buffer sesuai `minggu_5.md`, karena materinya sendiri diperkirakan 10-15 jam).
