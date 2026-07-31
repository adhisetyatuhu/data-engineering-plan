---
title: Hari 4 - IAM (Identity and Access Management)
parent: Minggu 6 - Cloud Platform Fundamentals
nav_order: 5
---

# Hari 4 — IAM: User, Role, Policy, Service Account, Least Privilege

*Kamis, 2 jam. Konsep + contoh CLI — service account yang dirancang di sini langsung dipakai Sabtu (`latihan_cloud_migration_mini_project.md`).*

> Ini penerapan **teknis** dari yang sudah dibahas konseptual di `materi/minggu_5/hari_1_konsep_governance.md` (pilar "policy") dan `materi/minggu_5/latihan_catalog_lineage_mini_project.md` (`GOVERNANCE.md` bagian "Access Policy") — kebijakan yang sudah ditulis di dokumen sekarang benar-benar **ditegakkan** lewat konfigurasi cloud.

## Tujuan Belajar

- [ ] Menjelaskan komponen IAM: principal, role, policy (binding)
- [ ] Menjelaskan service account dan bedanya dari akun user biasa
- [ ] Menerapkan prinsip least privilege ke skenario pipeline `ecommerce-etl-pipeline`
- [ ] Membuat service account dengan role yang tepat (dipakai langsung di mini project)

## Untuk Instruktur: Mindset Shift

Analogi paling langsung: IAM itu **RBAC (Role-Based Access Control)** yang developer sudah kenal dari desain sistem aplikasi (mis. tabel `users`, `roles`, `permissions` di database aplikasi) — cuma levelnya sekarang di infrastruktur cloud, bukan di dalam aplikasi yang kamu bangun sendiri.

**Service account** adalah konsep yang paling sering asing buat developer baru cloud — analogi yang pas: ini seperti **API key/service token** yang developer sudah kenal (mis. token yang dipakai CI/CD pipeline untuk deploy, atau API key yang dipakai backend server untuk memanggil layanan pihak ketiga) — identitas untuk **program/mesin**, bukan untuk manusia yang login lewat browser.

## Konsep & Sintaks

### 3 Komponen Inti

| Komponen | Definisi | Analogi RBAC Aplikasi |
|---|---|---|
| **Principal** | "Siapa" — user, group, atau service account | Baris di tabel `users` |
| **Role** | "Kumpulan permission" — mis. `Storage Object Viewer` (cuma boleh baca object), `BigQuery Data Editor` (boleh tulis ke BigQuery) | Baris di tabel `roles` |
| **Policy (Binding)** | "Aturan" yang menghubungkan principal + role + resource — "principal X punya role Y di resource Z" | Baris di tabel `user_roles`/`permissions` |

```
Policy: {
  principal: "etl-pipeline-sa@project.iam.gserviceaccount.com",
  role: "roles/storage.objectViewer",
  resource: "gs://ecommerce-data-lake"
}
```
Artinya: service account `etl-pipeline-sa` **boleh membaca** (tidak menulis/menghapus) object di bucket `ecommerce-data-lake` — dan **tidak** punya akses ke resource lain apapun, kecuali ada binding lain yang eksplisit menyatakannya.

### Predefined Role vs Custom Role

| | Predefined Role | Custom Role |
|---|---|---|
| Contoh | `roles/storage.objectViewer`, `roles/bigquery.dataEditor` | Kombinasi permission spesifik yang kamu tentukan sendiri |
| Kelebihan | Siap pakai, dikelola/diupdate provider, sudah teruji granularitasnya | Presisi penuh — cuma permission yang benar-benar dibutuhkan |
| Kelemahan | Kadang terlalu luas/sempit untuk kebutuhan spesifik | Perlu effort merancang & maintain sendiri |

**Rekomendasi praktis**: mulai dari predefined role (lebih cepat, cukup untuk kebanyakan kasus termasuk mini project ini) — custom role baru dipertimbangkan kalau predefined role yang tersedia konsisten terlalu luas untuk kebutuhan spesifik.

### Service Account — "Robot User"

Service account adalah identitas untuk **aplikasi/pipeline**, bukan manusia — tidak punya password, tidak login lewat browser, diautentikasi lewat **kunci** (key) atau mekanisme lain yang lebih aman (dibahas di bawah).

```bash
# Buat service account
gcloud iam service-accounts create etl-pipeline-sa \
  --display-name="ETL Pipeline Service Account"

# Assign role -- least privilege: HANYA object viewer di bucket spesifik, bukan role luas
gsutil iam ch \
  serviceAccount:etl-pipeline-sa@PROJECT_ID.iam.gserviceaccount.com:roles/storage.objectViewer \
  gs://ecommerce-data-lake

# Assign role BigQuery -- HANYA data editor di dataset spesifik, bukan seluruh project
bq add-iam-policy-binding \
  --member=serviceAccount:etl-pipeline-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/bigquery.dataEditor \
  ecommerce_warehouse
```

**Kenapa pakai service account, bukan akun personal, untuk pipeline** (poin eksplisit `minggu_6.md` Tahap 2): kalau pipeline memakai kredensial akun personal, (a) pipeline berhenti berfungsi kalau akun itu di-nonaktifkan/orangnya resign, (b) tidak ada jejak audit yang jelas ("siapa/apa" yang melakukan aksi tertentu — akun personal dipakai manusia **dan** pipeline jadi tercampur), dan (c) akun personal biasanya punya akses lebih luas dari yang dibutuhkan pipeline (melanggar least privilege). Service account **khusus per keperluan** menyelesaikan ketiganya — identitas terpisah, jelas jejak auditnya, dan bisa dibatasi persis sesuai kebutuhan.

### Least Privilege

Prinsip: berikan **hanya** permission yang benar-benar dibutuhkan untuk menjalankan tugas, tidak lebih — bukan "berikan akses luas dulu, batasi kalau ada masalah" (pendekatan yang terbalik dan berisiko).

```
SALAH (terlalu luas):
  etl-pipeline-sa -> roles/owner (akses PENUH ke seluruh project -- storage, compute, IAM, billing, semuanya)

BENAR (least privilege):
  etl-pipeline-sa -> roles/storage.objectViewer  PADA bucket ecommerce-data-lake SAJA
  etl-pipeline-sa -> roles/bigquery.dataEditor   PADA dataset ecommerce_warehouse SAJA
```

Kenapa ini penting bukan cuma soal "kerapian": kalau kredensial service account itu **bocor** (ter-commit ke git tidak sengaja, atau tereksploitasi lewat celah lain), dampaknya **dibatasi** sebatas permission yang diberikan — service account dengan `roles/storage.objectViewer` yang bocor cuma bisa dipakai untuk **membaca** 1 bucket tertentu, bukan menghapus seluruh project. Ini pola yang sama dengan **blast radius** di software engineering — desain sistem supaya kegagalan/kompromi di 1 titik tidak otomatis merembet ke seluruh sistem.

### Menyimpan Kredensial Service Account dengan Aman

```bash
# Cara LAMA (masih umum, tapi berisiko) -- download JSON key
gcloud iam service-accounts keys create key.json \
  --iam-account=etl-pipeline-sa@PROJECT_ID.iam.gserviceaccount.com
```

**JSON key adalah credential jangka panjang yang sangat sensitif** — persis level risiko yang sama dengan `SALT_RAHASIA` di `materi/minggu_5/hari_5_compliance_privacy.md` atau API key yang ter-hardcode: kalau bocor (ter-commit ke git, misalnya), siapapun yang punya file itu bisa memakainya **sampai key-nya dicabut manual**. **Selalu** tambahkan `*.json`/nama key spesifik ke `.gitignore`, dan pertimbangkan pendekatan yang lebih modern: **Workload Identity Federation** (untuk beban kerja yang jalan di luar GCP, mis. CI/CD) atau **attached service account** (untuk beban kerja yang jalan **di dalam** GCP, mis. Cloud Function/Compute Engine — service account otomatis "melekat" ke resource itu, tanpa perlu key file yang bisa bocor sama sekali).

## Kesalahan Umum

1. **Memakai role `Owner`/`Editor` luas untuk service account pipeline** karena "lebih gampang, tidak perlu pikirkan detail permission". Ini pelanggaran least privilege paling umum — persis kesalahan yang dicontohkan di atas, dan salah satu penyebab paling sering insiden keamanan cloud (kredensial dengan akses terlalu luas yang bocor).
2. **Meng-commit JSON key service account ke repository git.** Ini setara membocorkan password — begitu masuk histori git, dianggap **sudah bocor permanen** (bahkan setelah dihapus dari commit terbaru, tetap ada di histori kecuali di-rewrite) dan key itu harus segera **dicabut/revoke**, bukan sekadar dihapus dari file.
3. **Memberi akses di level project, padahal cuma butuh di level resource spesifik.** Mis. memberi `roles/storage.objectViewer` di **level project** (berarti bisa baca **semua** bucket di project itu) padahal cuma butuh baca 1 bucket tertentu — selalu scope permission ke resource paling spesifik yang memenuhi kebutuhan.
4. **Berbagi 1 service account untuk banyak keperluan berbeda** (mis. 1 service account dipakai untuk pipeline ETL **dan** untuk dashboard BI **dan** untuk testing). Ini menghilangkan manfaat audit trail yang jelas (poin di atas) dan membuat blast radius kalau bocor jadi lebih luas — pisahkan service account per keperluan/pipeline.

## Latihan

1. Rancang service account + role untuk `ecommerce-etl-pipeline` yang mencakup **seluruh** kebutuhan aksesnya: baca raw data dari bucket, tulis hasil processed ke bucket, tulis ke BigQuery. Tulis sebagai daftar binding (principal-role-resource), pastikan tetap least privilege.
2. Seorang developer memakai akun personalnya sendiri untuk menjalankan DAG Airflow yang mengakses BigQuery, karena "lebih cepat, tidak perlu setup service account dulu". Jelaskan 2 risiko konkret dari pendekatan ini yang baru akan terasa **belakangan**, bukan langsung terlihat saat itu.
3. Bandingkan JSON key file vs attached service account (untuk beban kerja yang jalan di GCP, mis. Cloud Function) — kenapa attached service account secara fundamental lebih aman, bukan cuma "lebih praktis"?
4. Jelaskan hubungan antara IAM policy yang dirancang minggu ini dengan "Access Policy" yang sudah ditulis di `GOVERNANCE.md` Minggu 5 — bagian mana dari access policy hipotetis itu yang sekarang bisa **benar-benar ditegakkan** secara teknis, dan bagian mana yang masih perlu penegakan di luar IAM (mis. lewat proses/kebijakan organisasi)?

## Kunci Jawaban & Pembahasan

**1.**
```
Principal: etl-pipeline-sa@PROJECT_ID.iam.gserviceaccount.com

Binding 1: roles/storage.objectViewer  PADA gs://ecommerce-data-lake/raw/*  (baca raw data)
Binding 2: roles/storage.objectCreator PADA gs://ecommerce-data-lake/processed/*  (tulis hasil processed)
Binding 3: roles/bigquery.dataEditor   PADA dataset ecommerce_warehouse  (tulis ke BigQuery)
Binding 4: roles/bigquery.jobUser      PADA project (dibutuhkan untuk MENJALANKAN load job ke BigQuery)
```
Perhatikan **tidak** ada `roles/owner`/`roles/editor` luas, dan akses storage dipisah baca (`raw/`) vs tulis (`processed/`) sesuai kebutuhan aktual tiap tahap pipeline — bukan 1 role besar yang mengizinkan baca-tulis di seluruh bucket.

**2.** Risiko 1: **pipeline berhenti berfungsi** kalau akun personal itu di-nonaktifkan/password berubah/orangnya resign/cuti panjang — karena pipeline bergantung ke identitas manusia yang tidak dirancang untuk "selalu ada". Risiko 2: **audit trail yang membingungkan** — begitu ada aktivitas mencurigakan atau perlu ditelusuri "siapa/apa yang melakukan operasi X di BigQuery", log akan menunjukkan akun personal itu, padahal bisa jadi itu memang si developer login manual, **atau** pipeline otomatis yang kebetulan memakai kredensial yang sama — tidak ada cara membedakan keduanya dari log, mempersulit investigasi/debugging di kemudian hari.

**3.** JSON key adalah **file fisik** yang bisa disalin, dikirim, ter-commit tidak sengaja ke git, tersimpan di laptop yang bisa dicuri/diretas — begitu file itu ada di luar kontrol, tidak ada cara mencegah pemakaiannya sampai key itu di-revoke secara eksplisit. **Attached service account** tidak melibatkan file kredensial sama sekali — identitas "melekat" langsung ke resource cloud (mis. Cloud Function) dan diautentikasi lewat mekanisme internal provider yang **tidak pernah meninggalkan infrastruktur cloud itu sendiri** dalam bentuk yang bisa disalin/dicuri seperti file biasa. Ini bukan cuma "lebih praktis" (walau memang juga lebih praktis, tidak perlu mengelola file key) — secara fundamental **menghilangkan seluruh kategori risiko** (key file bocor) karena tidak ada key file yang perlu dijaga sama sekali.

**4.** Bagian access policy `GOVERNANCE.md` yang **bisa ditegakkan teknis lewat IAM**: baris seperti "Data Engineer: Full access" dan "Analyst: SELECT only" ke `fact_sales`/`dim_*` — ini persis bisa diterjemahkan jadi IAM role (`roles/bigquery.dataEditor` untuk Data Engineer, `roles/bigquery.dataViewer` untuk Analyst) yang **secara otomatis** mencegah akses di luar itu, bukan cuma aturan tertulis yang bergantung kepatuhan sukarela. Bagian yang **masih butuh penegakan di luar IAM**: aturan seperti "kolom `customer_id` di-pseudonymize untuk data yang diekspor ke tim eksternal" — IAM bisa membatasi **siapa boleh akses tabel apa**, tapi tidak otomatis tahu bahwa "tabel yang sama harus ditampilkan berbeda (pseudonymized) tergantung siapa yang mengakses" — ini butuh mekanisme tambahan (mis. view terpisah dengan kolom yang sudah di-mask untuk role tertentu, column-level security BigQuery, atau proses ekspor yang secara eksplisit menerapkan masking sebelum data keluar) — IAM adalah **satu lapisan** penegakan, bukan solusi tunggal untuk semua kebijakan governance yang lebih granular dari sekadar "boleh akses tabel ini atau tidak".
