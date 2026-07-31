---
title: Hari 5 - Compliance & Privacy
parent: Minggu 5 - Data Governance
nav_order: 6
---

# Hari 5 — Compliance & Privacy: GDPR/UU PDP, PII, Masking & Anonymization

*Jumat, 2 jam. Level pengantar untuk developer — bukan pelatihan hukum formal. Tujuannya cukup paham istilah & prinsip untuk berdiskusi dengan tim legal/compliance, dan menerapkan teknik dasar di kode.*

## Tujuan Belajar

- [ ] Menjelaskan prinsip inti GDPR dan UU PDP (konteks Indonesia) yang relevan untuk data engineer
- [ ] Mengidentifikasi PII (direct identifier) dan quasi-identifier di dataset `ecommerce-etl-pipeline`
- [ ] Membedakan **masking**, **anonymization**, dan **pseudonymization** — kapan masing-masing dipakai
- [ ] Menerapkan teknik dasar ini dalam kode Python terhadap data Online Retail II

## Untuk Instruktur: Mindset Shift

Developer sering menyamakan "keamanan data" (security — enkripsi, access control, mencegah kebocoran) dengan "privasi data" (privacy — hak individu atas data pribadi mereka, terlepas dari siapa yang punya akses sah). Keduanya **berbeda dan saling melengkapi**: sistem bisa punya security sempurna (tidak ada yang bisa meretas), tapi tetap melanggar privasi kalau, misalnya, data dikumpulkan tanpa izin atau dipakai untuk tujuan yang tidak disetujui pemiliknya.

Analogi yang membantu: security itu seperti **kunci pintu rumah** (mencegah orang tidak berwenang masuk). Privacy itu seperti **aturan tentang apa yang boleh dilakukan pemilik rumah sendiri terhadap barang tamu** yang dititipkan (tidak boleh dipakai sembarangan, harus dikembalikan/dihapus kalau diminta, tidak boleh dikasih ke orang lain tanpa izin) — keduanya penting, tapi menjawab pertanyaan berbeda.

## Konsep & Sintaks

### GDPR — Prinsip Inti yang Relevan untuk Data Engineer

**GDPR** (General Data Protection Regulation, regulasi Uni Eropa, relevan karena Online Retail II adalah data pelanggan UK/Eropa) menetapkan beberapa prinsip yang berdampak langsung ke desain pipeline:

| Prinsip | Artinya | Dampak ke Pipeline |
|---|---|---|
| **Lawfulness & Purpose Limitation** | Data cuma boleh dikumpulkan untuk tujuan yang jelas & sah, tidak dipakai di luar itu tanpa izin baru | Dokumentasikan **kenapa** tiap dataset dikumpulkan (bagian dari business metadata, Hari 2) |
| **Data Minimization** | Cuma kumpulkan/simpan data yang **benar-benar diperlukan**, tidak lebih | Jangan simpan kolom PII "kalau-kalau nanti dibutuhkan" tanpa alasan jelas |
| **Right to Access** | Individu berhak tahu data apa saja tentang mereka yang disimpan | Pipeline idealnya bisa menjawab "tunjukkan semua data tentang customer X" |
| **Right to Erasure ("right to be forgotten")** | Individu berhak minta datanya dihapus | Ini **persis** kasus yang sudah disinggung di `materi/minggu_4/hari_5_incremental_cdc.md` Latihan #3 — incremental load berbasis watermark **tidak cukup** menangkap ini, butuh mekanisme eksplisit (atau CDC) |
| **Breach Notification** | Kebocoran data harus dilaporkan ke otoritas & individu terdampak dalam waktu tertentu | Alasan kuat kenapa access control & masking (di bawah) bukan sekadar "nice to have" |

### UU PDP — Konteks Indonesia

**UU Perlindungan Data Pribadi (UU PDP)** Indonesia mengadopsi prinsip yang sangat mirip dengan GDPR (banyak regulasi privasi dunia memang saling merujuk kerangka yang serupa), dengan istilah lokal:

| Istilah UU PDP | Padanan GDPR |
|---|---|
| **Pengendali Data Pribadi** | *Data Controller* — pihak yang menentukan tujuan & cara pemrosesan data (di konteks kita: perusahaan e-commerce) |
| **Prosesor Data Pribadi** | *Data Processor* — pihak yang memproses data atas instruksi pengendali (bisa jadi tim data engineering sendiri, atau vendor cloud) |
| **Subjek Data Pribadi** | *Data Subject* — individu yang datanya diproses (customer) |
| **Data Pribadi** vs **Data Pribadi Bersifat Spesifik** | Mirip pembagian PII biasa vs kategori sensitif (kesehatan, biometrik, dsb — butuh perlindungan lebih ketat) |

**Poin untuk developer**: terlepas dari yurisdiksi mana yang berlaku (data Online Retail II berbasis UK/Eropa → GDPR relevan; kalau membangun sistem untuk pasar Indonesia → UU PDP relevan), **prinsip teknisnya nyaris sama** — minimalkan data yang disimpan, punya cara menjawab "apa yang kita punya tentang orang ini", dan punya mekanisme masking/anonymization untuk data yang tidak perlu diakses mentah oleh semua pihak.

### Mengidentifikasi PII: Direct Identifier vs Quasi-Identifier

| Jenis | Definisi | Contoh di Online Retail II |
|---|---|---|
| **Direct Identifier** | Langsung mengidentifikasi 1 individu spesifik, sendirian | `Customer ID` |
| **Quasi-Identifier** | Tidak cukup sendirian, tapi **dikombinasikan** dengan atribut lain bisa mempersempit ke 1 individu | `Country` + `InvoiceDate` + pola belanja — kombinasi ini bisa jadi cukup unik untuk "menebak" identitas seseorang walau `Customer ID` sendiri disembunyikan |

**Poin penting yang sering terlewat**: quasi-identifier **berbahaya justru karena terlihat tidak sensitif sendiri-sendiri**. `Country` saja tidak masalah dibagikan — tapi `Country` + `InvoiceDate` + daftar produk yang dibeli, dikombinasikan, bisa jadi cukup spesifik untuk mengidentifikasi 1 orang (terutama kalau kombinasi itu langka, mis. 1 customer dari negara yang jarang muncul, membeli di tanggal spesifik). Ini alasan anonymization sungguhan (bukan cuma menghapus 1 kolom `Customer ID`) perlu mempertimbangkan **kombinasi** kolom, bukan kolom satu-satu.

### 3 Teknik: Masking, Anonymization, Pseudonymization

| Teknik | Reversibel? | Cara Kerja | Kapan Dipakai |
|---|---|---|---|
| **Masking** | Tidak (untuk masking sederhana) | Sembunyikan sebagian nilai (mis. `****1234`), atau ganti dengan nilai acak/placeholder | Data untuk tampilan/demo/testing yang tidak butuh nilai asli sama sekali |
| **Anonymization** | **Tidak, permanen** | Hapus/generalisasi informasi sampai individu **tidak bisa** diidentifikasi lagi, bahkan dikombinasikan dengan data lain | Data untuk analisis agregat/statistik jangka panjang, expor ke pihak eksternal |
| **Pseudonymization** | **Ya, dengan kunci terpisah** | Ganti identifier asli dengan token/ID buatan, tapi ada tabel mapping (disimpan terpisah, akses terbatas) untuk mengembalikannya kalau perlu | Data yang **masih perlu** dihubungkan balik ke individu asli untuk keperluan sah (mis. customer minta hapus data — perlu tahu baris mana yang harus dihapus), tapi tidak semua pihak boleh melihat identitas aslinya |

**Beda krusial**: anonymization **membakar jembatan** (tidak bisa dikembalikan ke data asli, bahkan oleh yang punya kunci sekalipun) — kalau nanti ternyata butuh tahu identitas asli lagi (mis. permintaan Right to Access dari GDPR), **tidak bisa**. Pseudonymization **menyimpan jembatan itu**, cuma dipisah & dikunci — bisa dibuka lagi oleh pihak berwenang saat benar-benar dibutuhkan.

### Contoh Kode: Menerapkan Ketiganya ke `dim_customer`

```python
import hashlib
import pandas as pd

dim_customer = pd.read_parquet("data/warehouse/dim_customer")

# --- MASKING (untuk tampilan/demo, tidak reversibel, tidak butuh nilai asli) ---
def mask_customer_id(customer_id: int) -> str:
    s = str(customer_id)
    return "*" * (len(s) - 2) + s[-2:]   # 17850 -> "***50"

dim_customer["customer_id_masked"] = dim_customer["customer_id"].apply(mask_customer_id)

# --- PSEUDONYMIZATION (reversibel via mapping table terpisah, akses terbatas) ---
# Mapping disimpan TERPISAH dari data yang dibagikan luas -- inilah "kunci"-nya
mapping_table = dim_customer[["customer_id"]].copy()
mapping_table["pseudonym"] = mapping_table["customer_id"].apply(
    lambda cid: hashlib.sha256(f"{cid}-SALT_RAHASIA".encode()).hexdigest()[:16]
)
# mapping_table disimpan di tempat dengan akses SANGAT terbatas (mis. cuma Data Owner)

dim_customer_shared = dim_customer.merge(mapping_table, on="customer_id").drop(columns=["customer_id"])
# dim_customer_shared (cuma berisi "pseudonym", bukan customer_id asli) yang dibagikan ke tim analytics

# --- ANONYMIZATION (permanen, untuk agregat statistik -- tidak ada "kunci" sama sekali) ---
# Generalisasi: ganti Customer ID individual dengan agregat per Country saja
anonymized_summary = (
    dim_customer.merge(pd.read_parquet("data/warehouse/fact_sales"), on="customer_id")
    .groupby("country")
    .agg(total_customers=("customer_id", "nunique"), total_revenue=("revenue", "sum"))
    .reset_index()
)
# Tidak ada cara mengembalikan ini ke data individual -- sudah teragregasi permanen
```

**Catatan penting soal SALT/kunci pseudonymization**: `"SALT_RAHASIA"` di atas **harus** disimpan sebagai secret (environment variable/secret manager), **bukan** ditulis literal di kode seperti contoh ini — contoh di atas disederhanakan untuk tujuan belajar. Kalau salt itu bocor/diketahui, siapapun bisa menghitung ulang hash yang sama dari `customer_id` yang mereka tebak, membuat pseudonymization jadi tidak efektif melindungi identitas.

## Kesalahan Umum

1. **Mengira menghapus 1 kolom (`Customer ID`) saja sudah cukup anonymization.** Seperti dibahas di atas, quasi-identifier yang tersisa (`Country` + `InvoiceDate` + pola belanja) bisa tetap cukup untuk mengidentifikasi individu — anonymization sungguhan butuh mempertimbangkan **kombinasi** kolom, bukan cuma menghapus 1 kolom yang paling jelas.
2. **Memakai anonymization padahal butuhnya pseudonymization** (atau sebaliknya). Kalau ada kemungkinan nanti butuh menghubungkan balik ke individu asli (Right to Access/Erasure dari GDPR/UU PDP), anonymization yang sudah permanen **tidak bisa diperbaiki** — pilih pseudonymization dari awal kalau ada keraguan.
3. **Menyimpan kunci/mapping pseudonymization di tempat yang sama dengan data yang di-pseudonymize.** Ini menghilangkan seluruh manfaatnya — kalau siapapun yang bisa akses data pseudonym juga otomatis bisa akses mapping-nya, itu sama saja tidak ada perlindungan sama sekali.
4. **Menerapkan masking/anonymization di akhir pipeline saja** (setelah semua orang di tim sudah biasa mengakses data mentah). Idealnya, kontrol privasi dirancang **sejak desain skema** (data minimization — Hari 1 GDPR) — bukan ditambal belakangan setelah kebiasaan akses data mentah sudah terlanjur terbentuk di tim.

## Latihan

1. Untuk skema `fact_sales`/`dim_customer` (Minggu 3), identifikasi mana yang **direct identifier**, mana yang **quasi-identifier** (kalau dikombinasikan dengan kolom lain), dan mana yang aman dibagikan tanpa kontrol khusus.
2. Tim data science butuh membangun model rekomendasi produk yang perlu tahu "customer yang sama membeli produk apa saja dari waktu ke waktu" (butuh menghubungkan transaksi ke individu yang sama), tapi tidak boleh tahu identitas asli customer itu siapa. Masking, anonymization, atau pseudonymization yang tepat? Jelaskan kenapa dua opsi lain tidak cukup.
3. Seorang customer meminta datanya dihapus sepenuhnya (Right to Erasure). Jelaskan kenapa ini **lebih mudah** dilakukan kalau sistem memakai pseudonymization dibanding kalau sistem sudah melakukan anonymization penuh sejak awal.
4. Jelaskan ke rekan kerja yang bilang "kita startup kecil, GDPR/UU PDP kan cuma buat perusahaan besar" — apakah ini asumsi yang aman? Kaitkan dengan fakta bahwa Online Retail II adalah data customer UK/Eropa.

## Kunci Jawaban & Pembahasan

**1.** **Direct identifier**: `customer_id` di `dim_customer`/`fact_sales`. **Quasi-identifier**: `country` (di `dim_customer`) dikombinasikan dengan pola transaksi di `fact_sales` (produk yang dibeli, tanggal, frekuensi) — kombinasi ini berpotensi mempersempit ke 1 individu tertentu, terutama untuk `country` yang jarang muncul di data. **Aman dibagikan tanpa kontrol khusus**: data yang sudah teragregasi penuh (mis. total revenue per `country` per bulan, tanpa breakdown per customer) — pada titik agregasi ini, tidak ada lagi jejak individual yang bisa ditelusuri balik.

**2.** **Pseudonymization**. Anonymization **tidak cukup** karena tim data science butuh menghubungkan **beberapa transaksi berbeda ke customer yang sama** dari waktu ke waktu (butuh identifier yang konsisten per customer) — anonymization penuh (mis. agregasi per negara) akan menghilangkan kemampuan ini sama sekali. Masking (mis. `***50`) juga tidak cukup — masking yang menyisakan sebagian karakter bisa jadi tidak konsisten/tidak cukup unik sebagai pengganti ID yang bisa diandalkan untuk grouping. Pseudonymization pas: tiap `customer_id` diganti **token yang konsisten** (`hash(customer_id)` selalu menghasilkan hash yang sama untuk customer yang sama) — tim data science bisa mengelompokkan transaksi per token itu tanpa pernah tahu `customer_id`/identitas asli di baliknya.

**3.** Dengan **pseudonymization**, permintaan hapus tinggal: cari `customer_id` di mapping table (yang aksesnya terbatas), dapatkan pseudonym-nya, lalu hapus/anonymkan semua baris dengan pseudonym itu di seluruh sistem yang memakainya — proses yang terarah dan bisa diverifikasi selesai. Dengan **anonymization penuh** (mis. sudah digeneralisasi jadi agregat per negara sejak awal), **tidak ada cara** menelusuri balik baris mana saja yang berasal dari customer spesifik itu — datanya sudah "melebur" ke agregat dan secara struktural tidak bisa dipisahkan lagi per individu. Ini pengingat konkret kenapa keputusan anonymization vs pseudonymization harus diambil dengan sadar sejak awal, bukan default ke anonymization "karena kedengarannya lebih aman" — kalau ternyata nanti butuh Right to Erasure yang presisi, anonymization yang sudah terlanjur permanen justru jadi masalah.

**4.** **Bukan asumsi yang aman.** GDPR (dan sebagian besar regulasi privasi modern) berlaku berdasarkan **kewarganegaraan/lokasi data subject** (siapa individunya, di mana mereka berada), bukan berdasarkan ukuran perusahaan yang memproses datanya — startup kecil yang memproses data customer dari UK/Eropa (persis kasus Online Retail II) tetap tunduk ke GDPR terlepas dari ukuran timnya. UU PDP juga berlaku untuk pemrosesan data pribadi warga negara Indonesia terlepas skala organisasinya. "Kecil" mungkin memengaruhi **prioritas penegakan** dari regulator secara praktis (perusahaan besar lebih sering jadi sorotan), tapi tidak menghilangkan **kewajiban hukum**-nya — dan lebih penting lagi untuk developer: menerapkan prinsip data minimization, punya kontrol PII, dan tahu cara handle Right to Erasure itu **baik untuk desain sistem** terlepas dari status hukum formalnya, karena mencegah masalah teknis & kepercayaan pengguna, bukan cuma soal kepatuhan regulasi semata.
