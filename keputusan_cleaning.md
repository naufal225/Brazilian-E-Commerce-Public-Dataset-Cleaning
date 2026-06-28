## Keputusan Cleaning

### olist_customers_dataset.csv
- tipe data customer_zip_code_prefix ubah jadi string dengan maximal panjang 5, jika panjang kurang dari 5, dibagian depan tambah angka "0" string
- perbaiki konsistensi nama kota sao

### olist_geolocation_dataset.csv
- tipe data customer_zip_code_prefix ubah jadi string, sama
- perbaiki konsistensi nama kota sao

### olist_products_dataset.csv
- perbaiki typo nama kolom:

product_name_lenght -> product_name_length, 
  product_description_lenght -> product_description_length, 
  product_height_cm -> product_heigth_cm

- perbaiki tipe data:

product_name_lenght -> harusnya integer aja, panjang nama ga mungkin desimal
  product_description_lenght -> integer, ga mungkin desimal
  product_photos_qty -> integer

terdapat null:

product_weight_g                2
product_length_cm               2
product_height_cm               2
product_width_cm                2

product_weight yang 0 atau NaN isi dengan "invalid weight" karena kalau difill bisa menganggu untuk analisis ongkir

difill dengan rata rata dari product dengan category yang sama

610 rows product ada yang null namun sudah dipakai di orders, fill dengan "unknown category" 

ada 2 row yang  product_weight_g, product_length_cm, product_height_cm, product_width_cm, ada yang 1 row null semua, namun 1 row ada yang category name nya ada, yang 1 row null semua angka nya diisi dengan angka rata rata keseluruhan semua


### olist_products_dataset.csv

- 610 product pernah sold padahan category name nya null, keputusannya adalah fill dengan "unknown category"
- rename penamaan kolom: 
  product_name_lenght -> product_name_length, 
  product_description_lenght -> product_description_length, 
  product_height_cm -> product_heigth_cm
- casting tipe data:
  product_name_lenght -> harusnya integer aja, panjang nama ga mungkin desimal
  product_description_lenght -> integer, ga mungkin desimal
  product_photos_qty -> integer
- untuk weight, length, height, width pada row yang category name nya null, kosong, NaN, atau "unknown category" maka fill dengan rata rata, jika ada category nya maka diisi dengan rata rata category nya 

### olist_sellers_dataset.csv

- seller_zip_code_prefix harusnya string

#### sellers, order_payments, order_reviews tidak ada cleaning
