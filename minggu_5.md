# Minggu 5 — Data Governance

## Breakdown Harian (±10-15 jam materi, slack tersedia untuk buffer)

| Hari | Jam | Materi |
|---|---|---|
| Senin | 2 jam | Konsep dasar data governance: apa itu, kenapa penting, komponen utama (ownership, policy, standard) |
| Selasa | 2 jam | Data catalog & metadata management: konsep, contoh tools (DataHub, Amundsen, Alation) |
| Rabu | 2 jam | Data lineage: konsep tracking asal-usul & perjalanan data, kenapa penting untuk audit & debugging |
| Kamis | 2 jam | Data quality dalam konteks governance (hubungkan dengan Great Expectations Minggu 4) + data stewardship |
| Jumat | 2 jam | Compliance & privacy: GDPR overview, PII handling, data masking/anonymization |
| Sabtu | 4 jam | Hands-on: setup DataHub/OpenMetadata lokal via Docker, eksplor fitur catalog |
| Minggu | 4 jam | Hands-on: dokumentasikan metadata & lineage dari pipeline Minggu 3-4 |

## Detail Topik
1. **Konsep Dasar Governance** — pilar: ownership, policy, standard, stewardship; perbedaan governance vs data management vs data quality
2. **Data Catalog & Metadata** — technical vs business vs operational metadata
3. **Data Lineage** — audit trail, impact analysis, column-level vs table-level lineage
4. **Data Stewardship & Quality Governance** — menghubungkan dengan Great Expectations Minggu 4
5. **Compliance & Privacy** — GDPR/UU PDP, PII handling, masking/anonymization/pseudonymization

## Sumber Belajar
- "Data Governance: The Definitive Guide" (O'Reilly) — bab awal, atau ringkasan DAMA-DMBOK
- DataHub docs "Quickstart", atau OpenMetadata docs
- Artikel/video overview (Seattle Data Guy, Data Engineering Weekly)
- Ringkasan resmi UU PDP dari Kominfo (konteks lokal Indonesia)

## Target di Akhir Minggu 5
- Menjelaskan komponen utama data governance dan kenapa penting
- Menjelaskan konsep data catalog, metadata, dan lineage
- Setup dasar tools data catalog open-source
- Mengidentifikasi PII dan teknik dasar anonymization

---

## Mini Project: "Data Catalog, Lineage & Governance Layer untuk E-Commerce Pipeline"

**Kenapa case ini?** Melengkapi narasi portfolio — bukan cuma "bisa bangun pipeline", tapi paham bagaimana data dikelola dan dijaga kepatuhannya, skill pembeda level mid-senior.

> Lanjutkan repo `ecommerce-etl-pipeline`, tambahkan folder governance baru.

### Tahap 1: Setup Data Catalog
- Setup OpenMetadata atau DataHub via Docker
- Ingest metadata dari warehouse (SQLite/PostgreSQL) yang sudah dibangun Minggu 3-4
- Lengkapi metadata manual di UI: business description tiap tabel, tag kolom PII, assign owner

### Tahap 2: Data Lineage Documentation
- Dokumentasikan lineage: raw CSV → Spark transform → star schema → warehouse
- Kalau tools support auto-lineage dari Airflow, integrasikan
- Kalau tidak, buat manual lineage diagram (draw.io)

### Tahap 3: Governance Policy Document
Buat `GOVERNANCE.md`:
- **Data ownership**: siapa bertanggung jawab atas tiap dataset
- **Data quality policy**: rule yang harus dipenuhi (reuse dari Great Expectations)
- **PII handling policy**: kolom sensitif, cara handle (misal masking `customer_id` untuk export eksternal)
- **Access policy sederhana**: siapa boleh akses data apa (hipotetis)

### Struktur Repo (update dari Minggu 4)
```
ecommerce-etl-pipeline/
├── README.md
├── GOVERNANCE.md                    # baru
├── dags/
├── spark_jobs/
├── great_expectations/
├── streaming-demo/
├── governance/
│   ├── docker-compose-openmetadata.yml
│   └── lineage_diagram.png
├── data/
└── diagrams/
    ├── star_schema.png
    ├── pipeline_architecture_v2.png
    └── data_lineage.png             # baru
```

### README yang Perlu Diupdate
- Section baru "Data Governance" — screenshot catalog UI (metadata & tag PII terisi)
- Link ke `GOVERNANCE.md`
- Diagram lineage end-to-end

### Alokasi Waktu (±8 jam dari slack tersedia)
- Setup OpenMetadata/DataHub + ingest metadata: 3 jam
- Lengkapi metadata, tag PII, ownership: 1.5 jam
- Buat lineage diagram: 1.5 jam
- Tulis `GOVERNANCE.md`: 2 jam
