---
title: Hari 3 - MongoDB Mendalam
parent: Minggu 8 - NoSQL & Storage Strategy (Capstone)
nav_order: 4
---

# Hari 3 — MongoDB Mendalam: Schema Design, Embedding vs Referencing

*Rabu, 2 jam. Persiapan langsung untuk Tahap 1 mini project Sabtu (migrasi `dim_product`).*

## Tujuan Belajar

- [ ] Menjelaskan struktur dasar MongoDB: database, collection, document (padanannya dari dunia RDBMS)
- [ ] Menulis operasi CRUD dasar dengan `pymongo`
- [ ] Menjelaskan trade-off embedding vs referencing, dan memilih yang tepat untuk kasus konkret
- [ ] Merancang schema document untuk `dim_product` dengan atribut bervariasi per kategori

## Untuk Instruktur

Peserta sudah 7 minggu berpikir dalam kerangka tabel/kolom/`JOIN` (RDBMS) — sesi ini butuh pergeseran mental yang eksplisit diajarkan, bukan diasumsikan otomatis dipahami. Analogi paling efektif: **document = objek dalam bahasa pemrograman** (dict/JSON di Python, bukan baris tabel) — developer yang terbiasa serialize/deserialize objek ke JSON untuk API sudah punya intuisi document model tanpa sadar.

## Konsep & Sintaks

### Struktur Dasar: Padanan dari RDBMS

| RDBMS (PostgreSQL) | MongoDB | Keterangan |
|---|---|---|
| Database | Database | Konsep sama persis |
| Table | **Collection** | Kumpulan document — beda dari table, document di dalamnya **tidak wajib** identik strukturnya |
| Row | **Document** | 1 unit data, berbentuk JSON/BSON (Binary JSON), bukan baris kolom tetap |
| Column | Field | Field bisa **berbeda** antar document dalam collection yang sama |
| Primary Key | `_id` | Otomatis dibuat MongoDB kalau tidak disediakan sendiri (`ObjectId`) |

```python
# RDBMS mental model:
# SELECT stock_code, description FROM dim_product WHERE stock_code = '85123A';

# MongoDB mental model -- document adalah objek, bukan baris:
{
    "_id": ObjectId("..."),
    "stock_code": "85123A",
    "description": "WHITE HANGING HEART T-LIGHT HOLDER",
    "category": "Home Decor",
    "material": "glass"          # <- field ini TIDAK ADA di document produk kategori lain
}
```

### CRUD Dasar dengan `pymongo`

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017/")
db = client["ecommerce_catalog"]
products = db["products"]

# Create
products.insert_one({
    "stock_code": "85123A",
    "description": "WHITE HANGING HEART T-LIGHT HOLDER",
    "category": "Home Decor",
    "material": "glass",
})
products.insert_many([{...}, {...}])   # banyak sekaligus

# Read
products.find_one({"stock_code": "85123A"})
products.find({"category": "Home Decor"})              # semua produk kategori itu
products.find({"category": "Bags", "size_variant": "M"})  # filter nested field

# Update
products.update_one(
    {"stock_code": "85123A"},
    {"$set": {"material": "ceramic"}}
)

# Delete
products.delete_one({"stock_code": "85123A"})
```

Perhatikan: `find({"category": "Home Decor"})` **tetap** bisa query berdasarkan isi field, persis seperti `WHERE category = 'Home Decor'` di SQL — perbedaan dari RDBMS bukan soal "bisa di-query atau tidak", tapi soal **fleksibilitas struktur** antar document.

### Embedding vs Referencing

Ini keputusan desain **paling penting** di MongoDB — padanan konsep dari "kapan normalize, kapan denormalize" di RDBMS (`materi/minggu_3/hari_2_data_modeling.md`), tapi dengan pertimbangan berbeda.

**Embedding** — data terkait disimpan **di dalam** 1 document yang sama (nested):
```json
{
    "stock_code": "85123A",
    "description": "WHITE HANGING HEART T-LIGHT HOLDER",
    "category": "Home Decor",
    "attributes": {                    // <- di-embed, bukan collection terpisah
        "material": "glass",
        "dimensions_cm": {"height": 10, "width": 6}
    },
    "reviews": [                        // <- array di-embed juga
        {"rating": 5, "comment": "Bagus!"},
        {"rating": 4, "comment": "Sesuai deskripsi"}
    ]
}
```

**Referencing** — data terkait disimpan di collection **terpisah**, dihubungkan lewat ID (mirip foreign key):
```json
// collection "products"
{"_id": "85123A", "description": "..."}

// collection "suppliers" (terpisah)
{"_id": ObjectId("..."), "name": "Supplier ABC", "country": "China"}

// document produk MEREFERENSIKAN supplier lewat ID, bukan embed penuh datanya
{"_id": "85123A", "description": "...", "supplier_id": ObjectId("...")}
```

| | Embedding | Referencing |
|---|---|---|
| Kecepatan baca | **Cepat** — 1 query saja, semua data terkait sudah ada di 1 document | Butuh **query tambahan** (atau `$lookup`, mirip `JOIN`) untuk menggabungkan data |
| Kapan cocok | Data yang **selalu diakses bersama**, dan tidak berubah/di-update terpisah secara sering, ukurannya terbatas | Data yang **sering berubah independen**, dipakai bersama banyak document lain (relasi many-to-many), atau ukurannya bisa tumbuh besar tak terbatas |
| Risiko | Duplikasi data kalau informasi yang sama dipakai banyak document (mis. data supplier yang sama di-embed ke ratusan produk — kalau supplier pindah alamat, harus update di ratusan tempat) | Butuh query/`$lookup` tambahan, sedikit lebih lambat dibanding embedding murni |
| Contoh keputusan di `dim_product` | Atribut spesifik produk itu sendiri (`material`, `size_variant`) — **selalu** melekat ke produk itu, wajar di-embed | Kalau ada entitas `supplier` yang dipakai **banyak** produk sekaligus dan sering berubah info-nya sendiri (alamat, kontak) — lebih tepat direferensikan, bukan di-embed ke tiap produk |

**Prinsip umum (rule of thumb)**: *"data that is accessed together should be stored together"* — kalau kamu **hampir selalu** butuh data A dan B bersamaan dan B tidak dipakai entitas lain, embed. Kalau B dipakai **banyak** entitas berbeda atau sering berubah sendiri terlepas dari A, referensikan.

**Untuk mini project `dim_product`** (Sabtu): atribut kategori (`material`, `size_variant`, `warranty_period`, dst) **di-embed langsung** ke document produk — atribut ini melekat penuh ke produk itu sendiri, tidak dipakai entitas lain, dan selalu dibutuhkan bersamaan saat menampilkan 1 produk. Tidak ada kebutuhan referencing di scope mini project ini (desain sengaja disederhanakan, konsisten dengan pola "SCD Type 1 disederhanakan" di Minggu 3).

## Kesalahan Umum

1. **Mengira document harus selalu identik strukturnya, meniru kebiasaan tabel RDBMS.** Ini justru menghilangkan keunggulan utama document model — field yang berbeda antar document (sesuai kategori produk, mis.) itu **fitur**, bukan bug, selama tetap ada field umum yang konsisten (`stock_code`, `description`) untuk query lintas kategori.
2. **Over-embedding — memasukkan semua data terkait ke 1 document sampai ukurannya sangat besar atau datanya terus tumbuh tak terbatas** (mis. embed **semua** riwayat transaksi ke document customer — bisa tumbuh tanpa batas seiring waktu). MongoDB punya batas ukuran dokumen (16MB) — data yang tumbuh tak terbatas lebih cocok direferensikan/disimpan di collection terpisah.
3. **Over-referencing — memecah semua relasi kecil jadi collection terpisah, meniru normalisasi RDBMS penuh.** Ini menghilangkan manfaat performa document model (baca 1 collection saja) dan menambah kompleksitas `$lookup` yang tidak perlu untuk data yang sebenarnya selalu diakses bersama.
4. **Lupa bahwa `_id` bersifat unik & immutable setelah dibuat.** Kalau ingin pakai `stock_code` sebagai identifier utama (bukan `ObjectId` otomatis), set eksplisit `_id: stock_code` saat insert — lebih natural untuk kasus migrasi dari data yang sudah punya primary key jelas seperti `dim_product`.

## Latihan

1. Rancang schema document untuk 3 produk `dim_product` dari kategori berbeda (buat sendiri datanya, boleh fiktif): 1 elektronik (dengan `warranty_period`), 1 fashion (dengan `size_variant`), 1 kategori lain bebas dengan atribut yang masuk akal untuknya.
2. Untuk kasus berikut, tentukan embedding atau referencing, dan jelaskan alasannya: (a) alamat pengiriman yang melekat ke 1 pesanan spesifik, (b) data 1 kategori produk (nama kategori, deskripsi kategori) yang dipakai **ribuan** produk berbeda dan sesekali di-update namanya.
3. Tulis query `pymongo` untuk mencari semua produk kategori "Bags" yang punya `size_variant` mengandung ukuran `"L"`.
4. Kenapa keputusan "embed atribut kategori langsung ke document produk" (bukan bikin collection `categories` terpisah lalu referensikan) dianggap "cukup" untuk skala mini project ini — dalam skenario seperti apa keputusan ini **perlu** ditinjau ulang kalau sistemnya berkembang lebih besar?

## Kunci Jawaban & Pembahasan

**1.** Contoh:
```json
{"_id": "ELEC001", "description": "Wireless Mouse", "category": "Electronics", "warranty_period": "12 months", "voltage": "5V"}
{"_id": "FASH001", "description": "Cotton Tote Bag", "category": "Bags", "size_variant": ["S", "M", "L"], "material": "canvas"}
{"_id": "BOOK001", "description": "Data Engineering 101", "category": "Books", "author": "Jane Doe", "isbn": "978-..."}
```
Ketiganya hidup di collection yang sama (`products`), field umum (`_id`/`description`/`category`) konsisten di semua, tapi field spesifik-kategori berbeda — ini persis fleksibilitas yang jadi alasan document model dipilih di `hari_1_konsep_nosql.md`.

**2.** (a) **Embedding** — alamat pengiriman **selalu** melekat & diakses bersama 1 pesanan spesifik, tidak dipakai pesanan lain, dan tidak perlu diupdate terpisah dari pesanan itu (kalaupun alamat user berubah di masa depan, pesanan **lama** tetap harus menyimpan alamat **saat pesanan itu dibuat** — justru bagus di-embed supaya "membeku" sesuai kondisi saat itu). (b) **Referencing** — data kategori dipakai **ribuan** produk sekaligus (relasi one-to-many besar) dan **berubah sendiri** terlepas dari produk manapun (update nama kategori 1x harus tercermin ke semua produk) — kalau di-embed ke tiap produk, update nama kategori berarti harus mengubah ribuan document sekaligus (duplikasi & inkonsistensi tinggi), jauh lebih baik disimpan 1x di collection `categories` dan direferensikan lewat ID.

**3.**
```python
products.find({"category": "Bags", "size_variant": "L"})
```
MongoDB otomatis mencocokkan kalau `size_variant` adalah **array** dan salah satu elemennya sama dengan `"L"` — tidak perlu operator khusus untuk pencocokan sederhana seperti ini terhadap array (`$in`/`$elemMatch` baru dibutuhkan untuk kondisi yang lebih kompleks, mis. mencocokkan beberapa kriteria sekaligus di dalam 1 elemen array objek).

**4.** Dianggap cukup untuk skala mini project karena jumlah kategori yang dipakai terbatas & jarang berubah namanya (beda dari skenario nomor 2 di atas, yang eksplisit mensyaratkan "dipakai ribuan produk, sesekali di-update") — untuk kebutuhan latihan & demonstrasi document model, kompleksitas referencing tidak sepadan manfaatnya di skala ini, sama seperti keputusan "SCD Type 1 disederhanakan" di Minggu 3 yang sengaja tidak dibuat lebih rumit dari yang dibutuhkan skala roadmap ini. Keputusan ini **perlu ditinjau ulang** kalau: (a) jumlah kategori bertambah signifikan dan sering di-update metadatanya (nama, deskripsi, gambar banner kategori) — duplikasi ke banyak produk jadi masalah nyata; (b) butuh menampilkan/mengelola kategori sebagai entitas mandiri di aplikasi (halaman "kelola kategori" terpisah dari halaman produk) — referencing jadi jauh lebih natural untuk kebutuhan itu.
