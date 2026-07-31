---
title: Hari 1 - Python Dasar
parent: Minggu 2 - Python Fundamentals
nav_order: 2
---

# Hari 1 — Python Dasar: Variabel, Tipe Data, Control Flow, Function

*Senin, 2 jam. Setup & dataset: lihat `00_overview.md`.*

> Sesi ini seharusnya terasa **cepat** untuk peserta — mereka sudah tahu apa itu variabel, if/else, loop, dan function dari bahasa lain. Fokus sesi ini murni **penerjemahan sintaks + gotcha khas Python**, bukan pengajaran konsep dari nol.

## Tujuan Belajar

- [ ] Menulis variabel & tipe data dasar Python dengan idiom yang benar (f-string, snake_case)
- [ ] Menulis control flow ala Python (`for` iterasi langsung, bukan index-based; truthiness)
- [ ] Menulis function dengan default argument, keyword argument, `*args`/`**kwargs`
- [ ] Mengenali & menghindari jebakan **mutable default argument** — bug klasik yang bahkan sering mengecoh developer berpengalaman

## Untuk Instruktur: Cara Mengajar Sesi Ini

Buka dengan pertanyaan langsung: *"Bahasa apa yang paling kamu kuasai? Oke — anggap sesi ini kamus terjemahan dari bahasa itu ke Python."* Jangan jelaskan ulang apa itu variabel atau kenapa kita butuh function. Yang perlu ditekankan justru kebalikannya: **hal-hal yang terlihat familiar tapi berperilaku beda** dari bahasa lain. Itulah isi seluruh sesi ini.

## Konsep & Sintaks

### Variabel & Tipe Data — Dynamic Typing

```python
name = "Andi Wijaya"      # str
price = 15.00              # float
quantity = 2                # int
is_active = True            # bool (bukan "true"/"false" huruf kecil)
notes = None                 # None, bukan null/nil/undefined
```

Tidak ada deklarasi tipe eksplisit — tipe ditentukan saat runtime dan **bisa berubah** (variabel yang sama bisa di-assign ulang ke tipe lain). Ini beda fundamental kalau asalnya dari bahasa statically-typed (Java, C#, Go, TypeScript strict mode). Tidak salah, cuma beda filosofi: Python percaya pada "duck typing" — yang penting objeknya *berperilaku* sesuai kebutuhan, bukan *dideklarasikan* sebagai tipe tertentu.

**f-string** (Python 3.6+) — cara idiomatik format string, mirip template literal di JS:
```python
print(f"{name} beli {quantity} unit seharga ${price:.2f}")
# Andi Wijaya beli 2 unit seharga $15.00
```

### Control Flow — Blok Ditentukan Indentation, Bukan `{}`

```python
if price > 100:
    tier = "premium"
elif price > 20:
    tier = "standard"
else:
    tier = "budget"
```

**Wajib konsisten pakai spasi (4 spasi standar/PEP8), jangan campur tab & spasi** — ini bisa bikin `IndentationError` atau (lebih parah) bug diam-diam yang beda editor bisa render beda.

`for` loop di Python iterasi **langsung ke elemen**, bukan berbasis index seperti `for (int i=0; i<n; i++)`:
```python
for product in products:          # iterasi langsung ke tiap elemen
    print(product["product_name"])

for i, product in enumerate(products):   # kalau butuh index juga
    print(i, product["product_name"])

for i in range(5):                # kalau memang butuh angka 0..4
    print(i)
```

### Truthiness — Gotcha Penting

Python menganggap beberapa nilai sebagai "falsy" tanpa perlu perbandingan eksplisit: `0`, `0.0`, `""` (string kosong), `[]`, `{}`, `None` — semua dianggap `False` dalam konteks boolean.

```python
cart = []
if cart:                 # False kalau cart kosong — idiomatik Python
    print("ada item")
else:
    print("keranjang kosong")

# BUKAN salah, tapi tidak idiomatik:
if len(cart) > 0:
    ...
```

Gotcha: developer dari bahasa yang lebih strict (mis. harus eksplisit `if cart.length > 0`) sering menulis versi verbose-nya — bukan salah, tapi kode Python idiomatik lebih suka bentuk singkat di atas. Sebaliknya, **hati-hati** kalau `0` adalah nilai valid yang ingin dibedakan dari "tidak ada nilai": `if quantity:` akan `False` juga kalau `quantity = 0`, padahal `0` itu valid secara bisnis (order dengan quantity 0 beda makna dari order yang belum diisi qty-nya/`None`). Kalau butuh beda perlakuan, harus eksplisit: `if quantity is not None:`.

### Function

```python
def calculate_total(quantity, unit_price, discount=0):
    """Default argument: discount opsional, default 0."""
    return quantity * unit_price * (1 - discount)

calculate_total(2, 15.00)                 # positional
calculate_total(2, 15.00, discount=0.1)   # keyword argument — self-documenting
calculate_total(quantity=2, unit_price=15.00, discount=0.1)  # semua keyword, urutan bebas
```

`*args` (kumpulan positional argument jadi tuple) dan `**kwargs` (kumpulan keyword argument jadi dict) — dipakai saat jumlah argumen tidak pasti:
```python
def total_of_prices(*prices):
    return sum(prices)

total_of_prices(15.00, 45.00, 25.00)   # 85.0

def describe_product(**attrs):
    return ", ".join(f"{k}={v}" for k, v in attrs.items())

describe_product(name="Wireless Mouse", category="Electronics", price=15.00)
# "name=Wireless Mouse, category=Electronics, price=15.00"
```

Return beberapa nilai sekaligus (sebenarnya return 1 tuple, tapi bisa langsung di-unpack):
```python
def min_max_price(products):
    prices = [p["unit_price"] for p in products]
    return min(prices), max(prices)

cheapest, priciest = min_max_price(products)
```

## Kesalahan Umum

1. **Mutable default argument — bug paling terkenal di Python, sering mengecoh developer berpengalaman sekalipun.**
   ```python
   # SALAH — default list dibuat SEKALI saat function didefinisikan, bukan tiap dipanggil
   def add_to_cart(item, cart=[]):
       cart.append(item)
       return cart

   add_to_cart("Mouse")           # ['Mouse']
   add_to_cart("Keyboard")        # ['Mouse', 'Keyboard'] <- BUKAN ['Keyboard']!
   ```
   Default argument dievaluasi **sekali** saat `def` dijalankan, lalu objek yang sama (list yang sama di memori) dipakai ulang di setiap pemanggilan tanpa argumen itu. Solusi standar:
   ```python
   def add_to_cart(item, cart=None):
       if cart is None:
           cart = []
       cart.append(item)
       return cart
   ```

2. **`/` vs `//`.** Python 3, `/` selalu menghasilkan `float` walau kedua operand `int` (`5 / 2 == 2.5`). Kalau butuh pembagian bulat (floor division), pakai `//` (`5 // 2 == 2`). Ini beda dari bahasa seperti Java/C di mana `int / int` otomatis truncate.

3. **`==` vs `is`.** `==` membandingkan **nilai**, `is` membandingkan **identitas objek** (apakah objek yang persis sama di memori). Idiom Python: selalu `is None` / `is not None`, **bukan** `== None` (walau `== None` biasanya juga "jalan", ini bukan konvensi yang benar dan bisa salah untuk kasus custom object dengan `__eq__` yang di-override).

4. **Snake_case, bukan camelCase.** Konvensi Python (PEP8) untuk nama variabel & function adalah `snake_case` (`calculate_total`, bukan `calculateTotal`). Developer dari JS/Java kadang otomatis nulis camelCase — bukan error, tapi akan mencolok sebagai kode "bukan idiomatik Python" di code review.

## Latihan

Pakai data produk berikut (7 produk, sama seperti `data/products.csv`):
```python
products = [
    {"product_id": 1, "product_name": "Wireless Mouse", "category": "Electronics", "unit_price": 15.00},
    {"product_id": 2, "product_name": "Mechanical Keyboard", "category": "Electronics", "unit_price": 45.00},
    {"product_id": 3, "product_name": "USB-C Hub", "category": "Electronics", "unit_price": 25.00},
    {"product_id": 4, "product_name": "Office Chair", "category": "Furniture", "unit_price": 120.00},
    {"product_id": 5, "product_name": "Notebook A5", "category": "Stationery", "unit_price": 3.50},
    {"product_id": 6, "product_name": "Coffee Mug", "category": "Kitchen", "unit_price": 8.00},
    {"product_id": 7, "product_name": "Desk Lamp", "category": "Furniture", "unit_price": 30.00},
]
```

1. Loop lewat `products`, cetak `product_name` untuk produk dengan `unit_price > 20`, format: `f"{product_name}: ${unit_price:.2f}"`.
2. Tulis function `apply_discount(price, percent=10)` yang mengembalikan harga setelah diskon `percent`% (default 10%).
3. Tulis function `total_cart_value(*prices)` yang menerima sejumlah harga sebagai `*args` dan mengembalikan totalnya.
4. Tulis function `describe(**kwargs)` yang menerima keyword argument bebas dan mengembalikan string `"key1=value1, key2=value2"`. Panggil dengan `describe(nama="Mouse", kategori="Electronics")`.
5. Cari & jelaskan bug di kode berikut, lalu perbaiki:
   ```python
   def add_category_tag(product, tags=[]):
       tags.append(product["category"])
       return tags
   ```

## Kunci Jawaban & Pembahasan

**1.**
```python
for p in products:
    if p["unit_price"] > 20:
        print(f'{p["product_name"]}: ${p["unit_price"]:.2f}')
```
Hasil: Mechanical Keyboard ($45.00), USB-C Hub ($25.00), Office Chair ($120.00), Desk Lamp ($30.00).

**2.**
```python
def apply_discount(price, percent=10):
    return price * (1 - percent / 100)

apply_discount(100)        # 90.0
apply_discount(100, 20)    # 80.0
```

**3.**
```python
def total_cart_value(*prices):
    return sum(prices)

total_cart_value(15.00, 45.00, 25.00)   # 85.0
```

**4.**
```python
def describe(**kwargs):
    return ", ".join(f"{k}={v}" for k, v in kwargs.items())

describe(nama="Mouse", kategori="Electronics")
# "nama=Mouse, kategori=Electronics"
```

**5.**
Bug: `tags=[]` adalah mutable default argument — list `tags` yang sama dipakai ulang di **setiap pemanggilan function** yang tidak eksplisit memberi `tags`, jadi tag dari pemanggilan sebelumnya akan terus menumpuk dan "bocor" ke pemanggilan berikutnya. Contoh kalau tidak sadar:
```python
add_category_tag(products[0])   # ['Electronics']
add_category_tag(products[3])   # ['Electronics', 'Furniture'] <- bukan ['Furniture']!
```
Perbaikan:
```python
def add_category_tag(product, tags=None):
    if tags is None:
        tags = []
    tags.append(product["category"])
    return tags
```
