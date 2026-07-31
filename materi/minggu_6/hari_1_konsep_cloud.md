---
title: Hari 1 - Konsep Dasar Cloud Computing
parent: Minggu 6 - Cloud Platform Fundamentals
nav_order: 2
---

# Hari 1 — Konsep Dasar Cloud Computing: IaaS/PaaS/SaaS, Region & AZ, Shared Responsibility

*Senin, 2 jam. Konsep murni — belum butuh akun cloud aktif, cukup dipahami dulu sebelum setup Sabtu.*

## Tujuan Belajar

- [ ] Menjelaskan spektrum IaaS/PaaS/SaaS dan menempatkan tools yang sudah dipakai (Postgres lokal, BigQuery, Airflow) di spektrum itu
- [ ] Menjelaskan region dan availability zone, serta kenapa keduanya penting untuk desain sistem
- [ ] Menjelaskan shared responsibility model dan bagaimana pembagiannya berubah tergantung jenis layanan
- [ ] Menjelaskan elastisitas & pay-as-you-go sebagai perbedaan fundamental dari infrastruktur on-premise

## Untuk Instruktur: Mindset Shift

Analogi paling efektif untuk IaaS/PaaS/SaaS: **menyewa tempat tinggal**, dengan level "berapa banyak yang diurus sendiri vs diurus pemilik" sebagai sumbunya.

| Level | Analogi Sewa | Analogi Cloud | Yang Kamu Urus | Yang Provider Urus |
|---|---|---|---|---|
| **IaaS** | Sewa apartemen kosong | Virtual Machine (Compute Engine/EC2) | OS, patching, aplikasi, data | Listrik, gedung, jaringan fisik |
| **PaaS** | Sewa apartemen full-furnished + maintenance | Cloud Functions, Cloud Run, managed Kubernetes | Kode aplikasi, konfigurasi | OS, runtime, patching, scaling |
| **SaaS** | Hotel dengan room service penuh | BigQuery, Gmail, Salesforce | Data & konfigurasi pemakaian | Semuanya — infrastruktur, aplikasi, maintenance |

Developer yang sudah pernah pakai Heroku/Vercel/Render sudah punya intuisi PaaS (`git push`, tidak perlu urus server). Developer yang pernah setup VPS sendiri (DigitalOcean droplet, EC2 manual) sudah punya intuisi IaaS. Poin yang perlu ditekankan: **makin ke atas (SaaS), makin sedikit yang kamu kontrol, tapi makin sedikit juga yang harus kamu urus** — bukan salah satu "lebih baik", tapi trade-off kontrol vs kemudahan operasional.

## Konsep & Sintaks

### Memetakan Tools yang Sudah Dipakai ke Spektrum IaaS/PaaS/SaaS

Ini yang paling membantu peserta punya pijakan konkret:

```
IaaS  ---------------------------------------------------  SaaS
 |                    |                    |                 |
VM kosong      Postgres lokal        Cloud Functions      BigQuery
(Compute        di Docker            (kode saja,          (query saja,
Engine/EC2)     (kamu urus OS,        provider urus         provider urus
                image, patching,      semua infra)          semua storage,
                resource)                                    compute, scaling)
```

Postgres yang dipakai sejak `materi/minggu_1/00_overview.md` (`docker run postgres:16`) itu **bukan** cloud service sama sekali — itu software yang kamu jalankan sendiri di infrastruktur sendiri (laptop). Kalau Postgres itu **disewa** dari cloud provider sebagai *managed database* (mis. Cloud SQL/RDS), itu masuk kategori PaaS — provider yang urus patching, backup, high availability, kamu cuma urus skema & data.

### Region & Availability Zone

**Region** = area geografis (mis. `us-central1` di Iowa, `asia-southeast1` di Singapura) — dipilih berdasarkan kedekatan dengan user (latency) dan kadang regulasi (data residency — data warga negara tertentu harus disimpan di region tertentu, menyambung ke `materi/minggu_5/hari_5_compliance_privacy.md`).

**Availability Zone (AZ)** = 1 atau lebih data center **fisik terpisah** di dalam 1 region, dengan sumber listrik & jaringan independen satu sama lain.

```
Region: asia-southeast1 (Singapura)
├── Zone asia-southeast1-a  (data center fisik 1)
├── Zone asia-southeast1-b  (data center fisik 2)
└── Zone asia-southeast1-c  (data center fisik 3)
```

**Kenapa AZ penting**: kalau 1 AZ mengalami gangguan (pemadaman listrik, kebakaran, dsb — jarang tapi nyata terjadi), AZ lain di region yang sama **tetap jalan normal**, karena infrastrukturnya benar-benar terisolasi fisik. Ini analog langsung dengan pola **replikasi terdistribusi** yang developer sudah kenal dari desain sistem (multi-node database cluster, load balancer dengan beberapa backend) — taruh replika di lebih dari 1 AZ = versi cloud dari "jangan taruh semua telur di 1 keranjang". Layanan *managed* seperti BigQuery **otomatis** mereplikasi data lintas beberapa zone tanpa kamu perlu konfigurasi apapun — salah satu manfaat besar SaaS/PaaS dibanding mengurus sendiri (IaaS).

### Shared Responsibility Model

Prinsip inti: **keamanan & operasional adalah tanggung jawab bersama** antara cloud provider dan pengguna, dan **pembagiannya bergeser** tergantung level layanan (IaaS/PaaS/SaaS).

```
                    IaaS              PaaS              SaaS
Data & akses     -> KAMU              KAMU              KAMU
Konfigurasi app  -> KAMU              KAMU              (terbatas)
Runtime/OS       -> KAMU              PROVIDER          PROVIDER
Patching         -> KAMU              PROVIDER          PROVIDER
Infra fisik      -> PROVIDER          PROVIDER          PROVIDER
```

**Yang TIDAK PERNAH berpindah ke provider, di level manapun**: keamanan **data & konfigurasi akses** (siapa boleh baca/tulis apa) selalu jadi tanggung jawab pengguna — ini sebabnya IAM (`hari_4_iam.md`) tetap krusial dipelajari walau memakai layanan SaaS penuh seperti BigQuery. Kesalahpahaman paling umum & paling berbahaya: mengira "karena ini cloud, keamanannya otomatis diurus provider" — provider menjamin **infrastrukturnya** aman (data center, hypervisor, jaringan fisik), tapi **konfigurasi yang kamu buat di atasnya** (bucket yang sengaja/tidak sengaja dibuat publik, IAM role yang kelewat luas) sepenuhnya tanggung jawab pengguna. Insiden kebocoran data cloud yang sering muncul di berita hampir selalu karena **kesalahan konfigurasi pengguna** (bucket publik, credential bocor), bukan provider yang diretas.

### Elastisitas & Pay-as-you-go

Beda fundamental dari infrastruktur on-premise (beli server fisik, kapasitasnya tetap sampai upgrade manual): cloud resource bisa **naik-turun otomatis** sesuai beban (elastisitas), dan **dibayar sesuai pemakaian aktual**, bukan kapasitas maksimum yang disiapkan.

Analogi: on-premise itu seperti **beli mobil** (biaya tetap besar di depan, kapasitas tetap, dipakai atau tidak tetap bayar perawatan). Cloud pay-as-you-go itu seperti **naik taksi/ride-hailing** (bayar sesuai jarak dipakai, tidak ada beban saat tidak dipakai, tapi bisa lebih mahal per-unit kalau dipakai terus-menerus dalam volume besar). Ini kenapa cloud data warehouse (`hari_5_cloud_data_warehouse.md`) sangat menarik untuk beban kerja analitik yang **tidak konstan** (query berat sesekali, sepi di waktu lain) — kamu tidak bayar untuk compute yang menganggur, beda dari mengurus server sendiri yang harus disiapkan untuk beban puncak walau jarang terjadi.

## Kesalahan Umum

1. **Mengira semua layanan cloud otomatis "aman" tanpa konfigurasi tambahan.** Sudah dibahas di atas — shared responsibility model berarti keamanan **konfigurasi** (akses, network rules) tetap tanggung jawab pengguna di level manapun.
2. **Memilih region cuma berdasarkan "yang paling murah" tanpa mempertimbangkan latency ke user atau regulasi data residency.** Region yang jauh dari user mayoritas bisa berarti latency tinggi; beberapa regulasi (termasuk konteks UU PDP, `materi/minggu_5/hari_5_compliance_privacy.md`) mensyaratkan data warga tertentu disimpan di wilayah tertentu.
3. **Mengira 1 AZ sudah cukup untuk redundansi.** Kalau butuh high availability sungguhan, resource perlu tersebar di **lebih dari 1 AZ** — 1 AZ tunggal tetap punya titik kegagalan tunggal (single point of failure) di level infrastruktur fisik itu.
4. **Membiarkan resource IaaS (VM) menyala terus tanpa dipakai**, lalu kaget dengan tagihan — beda dari on-premise (biaya sudah "sunk cost" di awal), cloud pay-as-you-go berarti **biaya terus berjalan selama resource menyala**, terlepas dipakai atau tidak. Ini alasan peringatan budget alert di `latihan_cloud_migration_mini_project.md` sangat serius, bukan formalitas.

## Latihan

1. Tempatkan tools berikut di spektrum IaaS/PaaS/SaaS, dan jelaskan alasannya: (a) Airflow yang dijalankan sendiri via `docker compose` (Minggu 3), (b) hipotetis kalau Airflow dijalankan lewat **Cloud Composer** (versi terkelola dari GCP), (c) BigQuery.
2. Sebuah startup memutuskan menaruh **semua** infrastrukturnya di 1 AZ tunggal demi menghemat biaya. Jelaskan risiko konkret dari keputusan ini, dan kapan risiko itu **sepadan** untuk diambil (kalau ada skenario di mana itu masuk akal).
3. Sebuah insiden kebocoran data terjadi karena bucket Cloud Storage sengaja dibuat "public" untuk memudahkan testing, lalu lupa dikembalikan ke private. Menurut shared responsibility model, ini salah siapa — provider atau pengguna? Jelaskan.
4. Bandingkan biaya menjalankan Postgres 24/7 di VM sendiri (IaaS) vs memakai BigQuery yang cuma dibayar per-query — untuk beban kerja "query analitik berat, tapi cuma dijalankan 2 jam per hari, sisanya idle". Mana yang lebih hemat, dan kenapa elastisitas jadi faktor penentu di sini?

## Kunci Jawaban & Pembahasan

**1.** (a) **Bukan cloud service sama sekali** — ini software yang dijalankan sendiri di infrastruktur sendiri (laptop lewat Docker), paling dekat konsepnya ke IaaS kalau nanti dipindah ke VM cloud (kamu tetap urus semua patching/scaling Airflow-nya sendiri). (b) **PaaS** — Cloud Composer adalah Airflow yang **dikelola** GCP: kamu cuma urus DAG & konfigurasi, GCP yang urus infrastruktur di baliknya (scaling scheduler/worker, patching versi Airflow, high availability). (c) **SaaS** (atau kadang disebut "serverless" sebagai variasi PaaS/SaaS lebih spesifik) — kamu cuma menulis query & mengelola data, tidak ada compute/infrastruktur apapun yang perlu kamu provision atau kelola sama sekali.

**2.** Risiko: kalau AZ tunggal itu mengalami gangguan (pemadaman listrik, masalah jaringan data center), **seluruh** layanan startup itu ikut down bersamaan — tidak ada redundansi sama sekali. Risiko ini **sepadan diambil** untuk skenario seperti: lingkungan development/testing (bukan production, downtime tidak berdampak ke user sungguhan), atau tahap sangat awal startup (pre-product-market-fit) di mana kecepatan & biaya minimal jauh lebih prioritas dibanding uptime sempurna, dan tim sadar penuh trade-off ini sebagai keputusan sementara, bukan permanen — bukan keputusan yang tepat untuk sistem production yang sudah punya user aktif dan SLA yang harus dijaga.

**3.** Ini **salah pengguna**, bukan provider — sesuai shared responsibility model, provider bertanggung jawab atas keamanan **infrastruktur fisik & platform** (bucket yang dibuat, secara teknis, memang berfungsi & aman dari sisi platform), tapi **konfigurasi akses** (siapa boleh baca bucket itu — publik atau tidak) sepenuhnya keputusan & tanggung jawab pengguna. Ini persis kesalahan umum #1 yang dibahas di atas, dan pola insiden yang sangat umum terjadi di dunia nyata — mayoritas "kebocoran data cloud" yang jadi berita besar disebabkan kesalahan konfigurasi seperti ini, bukan provider yang gagal menjaga keamanan platformnya sendiri.

**4.** **BigQuery (pay-as-you-go) lebih hemat** untuk pola beban kerja ini. VM yang menjalankan Postgres 24/7 tetap dikenakan biaya penuh **sepanjang waktu**, termasuk 22 jam di mana VM itu sebenarnya idle/menganggur (biaya storage + compute tetap berjalan walau tidak ada query masuk). BigQuery, karena dibayar **per-query** (biasanya berdasarkan volume data yang di-scan), cuma menimbulkan biaya saat **benar-benar dipakai** — untuk beban kerja yang cuma aktif 2 jam sehari, ini bisa jauh lebih murah karena tidak ada biaya untuk 22 jam idle itu. Elastisitas jadi faktor penentu karena beban kerja ini **tidak konstan** — kalau beban kerjanya justru konstan 24/7 dengan volume tinggi dan stabil, VM yang disewa dengan komitmen jangka panjang (reserved instance) kadang justru bisa lebih murah per-unit dibanding pay-as-you-go murni; keputusan ini selalu bergantung **pola** pemakaian, bukan aturan mutlak "cloud serverless selalu lebih murah".
