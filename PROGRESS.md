## 23-06-2026
### Dimensi 1 — Struktur
- 9 file CSV, total relasi berpusat di orders
- Typo nama kolom: product_name_lenght, product_description_lenght 
  (seharusnya: length)
- customer_zip_code_prefix dan seller_zip_code_prefix dtype int64, 
  seharusnya string
-order_item_id, nomor urut item dalam 1 order

refine detail:

#### olist_customers_dataset.csv
- terdiri dari 5 kolom, customer_id (str), customer_unique_id (str), customer_zip_code_prefix (int64), customer_city (str), customer_state (str)
- 99441 baris data
- 0 duplikatb

catatan perbaikan: 
- tipe data customer_zip_code_prefix harusnya string karena ini kode pos dan nilai depan nya bisa 0, seperti "0123" contohnya dan itu tidak untuk dihitung

#### olist_geolocation_dataset.csv
- terdiri dari 5 kolom, geolocation_zip_code_prefix (int64), geolocation_lat (float64), geolocation_lng (float64), geolocation_city (str), geolocation_state (str)
- 1000163 baris data
- 261831 duplikat

catatan perbaikan:
- tipe data customer_zip_code_prefix harusnya string karena ini kode pos dan nilai depan nya bisa 0, seperti "0123" contohnya dan itu tidak untuk dihitung

#### olist_order_items_dataset.csv
- terdiri dari 7 kolom, order_id (str), order_item_id (int64), product_id (str), seller_id (str), shipping_limit_date (str), price (float64), freight_value (float64)
- 112650 baris data

#### olist_order_payments_dataset.csv
- terdiri dari 5 kolom, order_id (str), payment_sequential (int64), payment_type (str), payment_installments (int64), payment_value (float64)
- 103886 baris data

#### olist_order_reviews_dataset.csv
- terdiri dari 7 kolom, review_id (str), order_id (str), review_score (int64), review_comment_title (str), review_comment_message (str), review_creation_date (str), review_answer_timestamp (str)
- 99224 baris data

#### olist_orders_dataset.csv
- terdiri dari 8 kolom, order_id (str), customer_id (str), order_status (str), order_purchase_timestamp (str), order_approved_at (str), order_delivered_carrier_date (str), order_delivered_customer_date (str), order_estimated_delivery_date (str)
- 99441 baris data

#### olist_products_dataset.csv
- terdiri dari 9 kolom, product_id (str), product_category_name (str), product_name_lenght (float64), product_description_lenght (float64), product_photos_qty (float64), product_weight_g (float64), product_length_cm (float64), product_height_cm (float64), product_width_cm (float64)
- 32951 baris data

catatan perbaikan: 
- typo penamaan kolom: 
  product_name_lenght -> product_name_length, 
  product_description_lenght -> product_description_length, 
  product_weigth_g -> product_weigth_g
  product_height_cm -> product_heigth_cm
- tipe data tidak cocok:
  product_name_lenght -> harusnya integer aja, panjang nama ga mungkin desimal
  product_description_lenght -> integer, ga mungkin desimal
  product_photos_qty -> integer

pengecekan lebih lanjut:
product_category_name         610
product_name_lenght           610
product_description_lenght    610
product_photos_qty            610
product_weight_g                2
product_length_cm               2
product_height_cm               2
product_width_cm                2
- 610 null, dan null 2 di sisa kolom berikutnya, punya dugaan ini ga expected, perlu dicek lebih lanjut di dimensi 2

#### olist_sellers_dataset.csv
- terdiri dari 4 kolom, seller_id (str), seller_zip_code_prefix (int64), seller_city (str), seller_state (str)
- 3095 baris data

catatan perbaikan:
- seller_zip_code_prefix harusnya string

#### product_category_name_translation.csv
- terdiri dari 2 kolom, product_category_name (str), product_category_name_english (str)
- 71 baris data

### Dimensi 2 — Kelengkapan
- order_approved_at: 160 null → expected, order belum diapprove
- order_delivered_carrier_date: 1.783 null → expected, belum dikirim
- order_delivered_customer_date: 2.965 null → expected, belum diterima
- product_weight_g, dimensi: 2 null → perlu di-handle
- review_comment_title: 87.656 null → expected, komentar opsional
- review_comment_message: 58.247 null → expected, komentar opsional
- product_category_name: 610 null → semua 610 produk pernah terjual
  Keputusan: FLAG — isi dengan 'unknown_category'
  Alasan: tidak bisa drop karena produknya ada di order_items,
          tidak bisa fill dengan kategori lain karena tidak ada informasi,
          'unknown_category' membuat data tetap terhitung tanpa memalsukan kategori

refine detail:

#### olist_order_reviews_dataset.csv
- terdapat 87656 null untuk review_comment_title dan 58247 untuk review_comment_message -> expected null, review comment title dan message bisa saja null, user bisa hanya kasih bintang atau rating saja

#### olist_orders_dataset.csv
- order_approved_at 160 null -> expected null, secara bussiness logic bisa karena orderan belum diacc
- order_delivered_carrier_date 1783 null -> expected null, secara bussiness logic bisa karena orderan belum dikirim
- order_delivered_customer_date 2965 null -> expected null, secara bussiness logic bisa karena orderan belum sampai ke customer

**orders — timestamp columns** — dtype str
Keputusan: BIARKAN untuk sekarang
Alasan: belum ada use case operasi berbasis waktu
Action: cast ke datetime saat dibutuhkan untuk analisis tren atau durasi delivery

#### olist_products_dataset.csv
product_category_name         610
product_name_lenght           610
product_description_lenght    610
product_photos_qty            610 -> ke 4 null ini tidak expected dan 610 ini null semua di semua kolomnya
product_weight_g                2
product_length_cm               2
product_height_cm               2
product_width_cm                2

df_products[['product_category_name', 'product_name_lenght', 'product_description_lenght', 'product_photos_qty', 'product_weight_g', 'product_length_cm', 'product_height_cm', 'product_width_cm']].isna().sum()

product_category_name         610
product_name_lenght           610
product_description_lenght    610
product_photos_qty            610
product_weight_g                2
product_length_cm               2
product_height_cm               2
product_width_cm                2
dtype: int64

df_products[['product_weight_g', 'product_length_cm', 'product_height_cm', 'product_width_cm']].isna().sum()

product_weight_g     2
product_length_cm    2
product_height_cm    2
product_width_cm     2
dtype: int64

df_products[['product_category_name', 'product_name_lenght', 'product_description_lenght', 'product_photos_qty']].isna().sum()

product_category_name         610
product_name_lenght           610
product_description_lenght    610
product_photos_qty            610
dtype: int64

null_product_ids = null_category['product_id']
sold = null_product_ids.isin(df_items['product_id']).sum()

sold = 610, 610 null name product pernah terjual 

### Dimensi 3 — Konsistensi
- geolocation_city: inkonsistensi encoding 
  ("sao paulo" vs "são paulo") → standarisasi di cleaning
- payment_type: 3 baris "not_defined" → semua canceled, 
  payment_value 0 → valid, biarkan
- geolocation: 261.831 duplikat → valid, satu ZIP bisa 
  banyak koordinat

#### olist_geolocation_dataset.csv
- geolocation_zip_code_prefix
24220    1146
24230    1102
38400     965
35500     907
11680     879
         ... 
97843       1
97960       1
98320       1
98825       1
99839       1
-> valid duplicate

- geolocation_city sao paulo tidak konsisten

sao paulo                      135800
são paulo                       24918
sao paulo das missoes              23
sao paulo do potengi               16
sãopaulo                            2
morro de sao paulo                  2
são paulo do potengi                2
são paulo das missões               2
sa£o paulo                          1
morro de são paulo                  1
são paulo de olivença               1
sao paulo de olivenca               1
Name: count, dtype: int64

### Dimensi 4 — Validitas
- product_weight_g min = 0 → impossible, produk tidak mungkin 
  0 gram → flag sebagai invalid
- freight_value min = 0 → valid, gratis ongkir
- review_score range 1-5 → bersih, tidak ada nilai di luar range
### Dimensi 5 — Relasi
- Semua foreign key valid, 0 orphan records

**order_reviews — duplicate order_id**
Temuan: 547 order punya lebih dari 1 review
Pola: beragam — sistem resend, customer update score, bug submission
Keputusan: Dedup, admbil yang terakhir
Alasan: Buat jaga jaga barang kali kepake nanti buat analisis sentimen