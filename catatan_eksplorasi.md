# Catatan Teknis — Eksplorasi Data untuk Data Engineering

## Definisi

**Eksplorasi data** adalah proses mendiagnosis kondisi sebuah dataset sebelum
melakukan pembersihan atau transformasi apapun. Tujuannya bukan menghasilkan
insight bisnis, melainkan menghasilkan **peta masalah** yang lengkap dan
terdokumentasi — termasuk masalah yang terlihat jelas maupun yang tersembunyi.

Eksplorasi adalah **diagnosis**, bukan **operasi**. Dokter tidak langsung
membedah tanpa diagnosis. Engineer tidak langsung cleaning tanpa eksplorasi.

---

## Framework 5 Dimensi

Setiap kali menghadapi dataset baru, lakukan pengecekan dalam urutan berikut.
Urutan penting — dimensi sebelumnya menjadi fondasi untuk dimensi berikutnya.

---

### Dimensi 1 — Struktur

**Definisi:** Pemahaman dasar tentang bentuk fisik dataset — berapa baris,
berapa kolom, tipe data tiap kolom, dan apakah nama kolom sudah konsisten.

**Yang dicek:**
- `df.shape` → jumlah baris dan kolom
- `df.dtypes` → tipe data tiap kolom
- `df.columns` → nama-nama kolom
- `df.head(10)` → sampel 10 baris pertama

**Pertanyaan yang harus dijawab:**
- Apakah tipe data tiap kolom sudah sesuai realita? (contoh: kolom tanggal
  bertipe object/string padahal seharusnya datetime)
- Ada nama kolom yang typo, ambigu, atau tidak konsisten?
- Kolom mana yang perlu diinterpretasikan lebih dalam sebelum bisa
  mengambil keputusan?

**Contoh temuan di Olist:**
- `product_name_lenght` dan `product_description_lenght` → typo, seharusnya
  `length`
- `customer_zip_code_prefix` dtype `int64` → seharusnya string karena ini
  kode pos, bukan angka yang dihitung
- `shipping_limit_date` dtype `object` → seharusnya datetime

---

### Dimensi 2 — Kelengkapan

**Definisi:** Seberapa lengkap data yang ada. Kelengkapan bukan hanya soal
menghitung null — yang lebih penting adalah memahami **mengapa** null itu ada.

**Yang dicek:**
```python
df.isna().sum()
(df.isna().sum() / len(df) * 100).round(2)  # persentase null per kolom
```

**Jenis-jenis null — ini yang sering tidak dipahami pemula:**

| Jenis | Artinya | Contoh | Penanganan |
|---|---|---|---|
| Null expected — belum terjadi | Kondisi bisnis yang valid | `order_delivered_customer_date` null untuk order yang masih dikirim | Biarkan, pipeline harus aman terhadapnya |
| Null expected — tidak relevan | Kolom memang opsional | `review_comment_title` null karena komentar opsional | Biarkan |
| Null dirty — data hilang | Data seharusnya ada tapi tidak terekam | `product_weight_g` null padahal semua produk punya berat | Handle: fill atau flag |
| Null dirty — kesalahan input | Kesalahan saat data dimasukkan | ID pelanggan null | Investigasi lebih dalam |

**Pertanyaan yang harus dijawab untuk setiap kolom ber-null:**
1. Berapa persentasenya? (5% vs 60% = prioritas berbeda)
2. Apakah null ini expected dari konteks bisnis?
3. Apa dampaknya ke pipeline jika dibiarkan null?

**Contoh temuan di Olist:**
- `order_delivered_customer_date`: 2.965 null (2,98%) → **expected**,
  order belum/tidak delivered
- `review_comment_title`: 87.656 null (88%) → **expected**, komentar opsional
- `product_weight_g`: 2 null → **dirty**, semua produk punya berat fisik

---

### Dimensi 3 — Konsistensi

**Definisi:** Apakah nilai-nilai dalam dataset ditulis dengan cara yang
seragam. Ada dua jenis inkonsistensi:

**Duplikat exact** — baris yang identik persis di semua kolom.
```python
df.duplicated().sum()
```

**Duplikat logis** — data yang merepresentasikan hal yang sama tapi ditulis
berbeda. Ini yang lebih berbahaya karena tidak terdeteksi oleh
`duplicated()`.

```python
df['kolom_kategori'].value_counts()  # lihat variasi penulisan
```

**Contoh duplikat logis di Olist:**
- `geolocation_city`: `"sao paulo"` dan `"são paulo"` → kota yang sama,
  ditulis berbeda karena inkonsistensi encoding karakter (ASCII vs UTF-8)
- Dampak: `GROUP BY city` akan menghitung dua kota berbeda padahal satu

**Duplikat yang valid:**
- `geolocation`: 261.831 duplikat ZIP code → bukan dirty data. Satu area
  ZIP code mencakup wilayah geografis, bukan satu titik, sehingga wajar ada
  banyak koordinat lat/lng yang berbeda untuk ZIP code yang sama.

**Pertanyaan yang harus dijawab:**
- Duplikat ini exact atau logis?
- Kalau logis — format mana yang benar?
- Kalau exact — mana yang harus dipertahankan dan berdasarkan apa?

---

### Dimensi 4 — Validitas

**Definisi:** Apakah nilai yang ada secara teknis valid (bukan null, bukan
duplikat) tapi secara domain/bisnis tidak masuk akal.

Ini adalah dimensi "happy path trap" — banyak pemula berhenti di dimensi 2
dan 3 karena data terlihat lengkap dan tidak duplikat. Engineer profesional
selalu tanya: **apakah nilai ini mungkin secara fisik/bisnis?**

```python
df[['kolom_numerik']].describe()  # lihat min, max, mean, percentile
df['kolom_kategori'].value_counts()  # lihat semua nilai unik
```

**Kategori nilai tidak valid:**

| Kategori | Contoh | Cara deteksi |
|---|---|---|
| Impossible value | `product_weight_g = 0` (produk tidak mungkin 0 gram) | `min()` di bawah threshold logis |
| Out of range | `review_score = 7` (skala 1–5) | `value_counts()` di luar range valid |
| Nilai mencurigakan | `payment_type = "not_defined"` | `value_counts()` lalu investigasi lebih dalam |
| Outlier ekstrem | Harga produk Rp 6.735 vs rata-rata Rp 120 | `describe()` → gap besar antara max dan 75th percentile |

**Contoh temuan di Olist:**
- `product_weight_g` min = 0 → impossible, produk fisik tidak mungkin 0 gram
- `payment_type = "not_defined"`: 3 baris → investigasi lebih dalam →
  ternyata semua `order_status = canceled` dan `payment_value = 0` →
  kondisi bisnis valid, bukan dirty data
- `freight_value` min = 0 → valid, bisa gratis ongkir

**Pelajaran penting:** Nilai yang mencurigakan tidak otomatis dirty data.
Selalu investigasi konteks bisnis dulu sebelum memutuskan.

---

### Dimensi 5 — Relasi antar tabel

**Definisi:** Apakah foreign key di satu tabel punya pasangannya di tabel
lain. Ini hanya relevan untuk dataset multi-tabel.

Di database production, ada **foreign key constraint** yang otomatis mencegah
data tanpa pasangan masuk. Di file CSV tidak ada constraint apapun — jadi
pengecekan ini harus dilakukan manual.

**Istilah:** Record yang foreign key-nya tidak punya pasangan di tabel lain
disebut **orphan record**.

```python
# Cek apakah semua order_id di payments ada di tabel orders
orphan = (~df_payments['order_id'].isin(df_orders['order_id'])).sum()
print(f"Orphan payments: {orphan}")
```

**Notasi yang dipakai:**
- `isin(koleksi)` → cek apakah nilai ada dalam koleksi → hasil: True/False
- `~` → membalik True/False (NOT operator)
- `.sum()` → hitung berapa yang True (berapa orphan)

**Dampak orphan record ke pipeline:**
- JOIN antara dua tabel akan menghasilkan hasil yang tidak lengkap atau error
- Aggregate seperti total revenue bisa under-count
- Data yang tidak bisa di-join = data yang tidak berguna

**Pengecekan di Olist:**
```python
(~df_payments['order_id'].isin(df_orders['order_id'])).sum()   # 0
(~df_items['order_id'].isin(df_orders['order_id'])).sum()      # 0
(~df_items['product_id'].isin(df_products['product_id'])).sum() # 0
(~df_items['seller_id'].isin(df_sellers['seller_id'])).sum()   # 0
(~df_orders['customer_id'].isin(df_customers['customer_id'])).sum() # 0
```
Semua 0 → integritas relasi bersih.

---

## Level masalah data

Masalah data tidak hanya ada di satu level. Penting untuk membedakan:

**Level nilai (cell):** Satu nilai spesifik yang bermasalah.
Contoh: `product_weight_g = 0` di satu baris tertentu.

**Level kolom:** Seluruh kolom punya masalah sistematis.
Contoh: `customer_zip_code_prefix` dtype `int64` — semua nilai di kolom ini
berpotensi kehilangan leading zero.

**Level tabel (antar tabel):** Masalah yang hanya terlihat saat dua tabel
dibandingkan. Contoh: orphan records, inkonsistensi nama kota yang sama.

---

## Output eksplorasi yang benar

Eksplorasi dianggap selesai bukan saat semua kode dijalankan, tapi saat
setiap temuan sudah didokumentasikan dengan format:

```markdown
**[nama kolom]** — [ringkasan masalah]
Temuan: [apa yang ditemukan dari data]
Kesimpulan: [dirty data atau kondisi valid?]
Keputusan: [DROP / FILL / FLAG / BIARKAN]
Alasan: [mengapa keputusan itu diambil]
Action: [apa yang akan dilakukan di tahap cleaning]
```

Tidak boleh ada keputusan tanpa alasan tertulis. Dua minggu kemudian kamu
tidak akan ingat kenapa kamu memutuskan sesuatu.

---

## Checklist eksplorasi cepat

Gunakan ini sebagai SOP setiap kali menghadapi dataset baru:

- [ ] D1: Cek shape, dtypes, columns, head — ada tipe data yang salah?
- [ ] D1: Ada nama kolom typo atau ambigu?
- [ ] D2: Cek null per kolom — seberapa banyak? Expected atau dirty?
- [ ] D3: Cek duplikat exact — berapa? Mana yang dipertahankan?
- [ ] D3: Cek nilai kategorikal — ada variasi penulisan untuk hal yang sama?
- [ ] D4: Cek describe() kolom numerik — ada min/max yang tidak masuk akal?
- [ ] D4: Cek value_counts() kolom kategorikal — ada nilai aneh?
- [ ] D5: Cek foreign key — ada orphan record?
- [ ] Semua temuan terdokumentasi dengan keputusan dan alasan