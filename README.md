# Analisis Performa Penjualan Trek Musik Digital dan Tren Pendapatan Bulanan (Chinook)

Mini project Data Analyst & Business Intelligence (Celerates): membangun **data warehouse** dan **data mart** penjualan trek musik dari database **Chinook** menggunakan **Pentaho Data Integration (PDI)** dan **PostgreSQL**, mulai dari staging, dimensi, fakta, sampai data mart.

---

## 1. Tim

**Kelompok 3**

| No | Nama | NIM |
|----|------|-----|
| 1 | Nagatan Alif Putra Silahen | 24031554086 |
| 2 | Nabil Putra Yuan | 24031554126 |
| 3 | Meisya Natasafira | 24031554174 |
| 4 | Cut Azzahra Firdausi Syakiena Nazwa | 24031554180 |
| 5 | Kafka Praya Firmansyah | 24031554182 |

---

## 2. Latar Belakang

Data penjualan toko musik digital seperti Chinook tersimpan di banyak tabel operasional (OLTP) yang terpisah: `invoice`, `invoice_line`, `track`, `album`, `artist`, `genre`, dan `media_type`. Untuk menjawab pertanyaan sederhana seperti "berapa pendapatan bulan ini?" atau "genre apa yang paling laku?", kita harus menggabungkan banyak tabel sekaligus. Hal ini merepotkan dan sulit dipakai untuk analisis berulang.

Karena itu dibangun data warehouse dengan **star schema** dan dua data mart yang siap dianalisis. Database Chinook dipilih karena ukurannya pas, relasinya kaya, dan mewakili sistem toko media digital di dunia nyata.

**Pertanyaan bisnis yang dijawab (berdasarkan hasil data mart):**

| # | Data mart | Pertanyaan | Jawaban singkat |
|---|-----------|------------|-----------------|
| 1 | `datamart_track_monthly` | Genre dan artis apa yang paling berkontribusi terhadap penjualan? | Genre **Rock** (835 penjualan, 826,65 atau sekitar 35,5% pendapatan), disusul Latin dan Metal. Artis terlaris **Iron Maiden** (140 penjualan, 138,60), disusul U2 dan Metallica |
| 2 | `datamart_revenue_monthly` | Bagaimana tren pendapatan per bulan dan per tahun? | Total 2.240 trek terjual dengan pendapatan **2.328,60** (Jan 2021 s.d. Des 2025). Tren stabil, rata-rata sekitar 37 trek dan 38,81 per bulan. Tertinggi 2022 (481,45), terendah 2021 (449,46) |

---

## 3. Alur Pengerjaan (Pipeline)

```mermaid
flowchart LR
    A[(OLTP Chinook<br/>PostgreSQL)] --> B[Staging<br/>staging_chinook]
    B --> C[Dimensi dan Fact<br/>Star Schema]
    C --> D[Data Mart<br/>track_monthly dan revenue_monthly]
    D --> E[CSV / Excel / BI]
```

| Tahap | Tool | Keterangan |
|-------|------|------------|
| Sumber | PostgreSQL | Dump Chinook (11 tabel) |
| Staging | Pentaho (Job dan Transformation) | Salin data apa adanya, dengan log jumlah baris |
| Dimensi dan Fact | Pentaho | Cleaning, standarisasi, surrogate key, lookup |
| Data mart | Pentaho | Gabungan fact dan dimensi, ringkasan bulanan |
| Penyajian | Text file output (CSV) dan Excel | Hasil siap dipakai |

---

## 4. Isi Database (Chinook)

Database Chinook berisi data toko musik digital dengan **11 tabel**:

| Tabel | Baris | Penjelasan singkat |
|-------|------:|--------------------|
| `invoice_line` | 2.240 | Rincian isi tiap pembelian: trek, harga satuan, jumlah. Dasar tabel fakta |
| `invoice` | 412 | Faktur tiap transaksi: pelanggan, tanggal, alamat penagihan, total |
| `track` | 3.503 | Daftar trek: nama, komposer, durasi (ms), ukuran (byte), harga |
| `album` | 347 | Daftar album dan artis pemiliknya |
| `artist` | 275 | Daftar artis atau musisi |
| `genre` | 25 | Daftar genre musik (Rock, Jazz, Latin, dan lainnya) |
| `media_type` | 5 | Jenis file trek (MPEG audio, AAC audio, video) |
| `customer` | 59 | Data pelanggan dan karyawan yang menanganinya |
| `employee` | 8 | Data karyawan, jabatan, dan atasan (`reports_to`) |
| `playlist` | 18 | Daftar putar |
| `playlist_track` | 8.715 | Penghubung playlist dan trek (banyak ke banyak) |

Rentang data transaksi: **1 Januari 2021 sampai 22 Desember 2025 (60 bulan)**.

---

## 5. Struktur Database

Struktur yang dijelaskan di sini hanya tabel yang dikerjakan pada proyek ini, yaitu 7 tabel sumber, 1 tabel fakta, 6 tabel dimensi, dan 2 data mart.

### 5.1 Struktur Sumber (OLTP)

Tujuh dari 11 tabel Chinook dipakai sebagai sumber data warehouse. Tabel `customer`, `employee`, `playlist`, dan `playlist_track` tidak dipakai karena analisis berfokus pada penjualan trek.

| Tabel | Kolom yang dipakai | Primary key | Foreign key |
|-------|--------------------|-------------|-------------|
| `artist` | `artist_id`, `name` | `artist_id` | - |
| `album` | `album_id`, `title`, `artist_id` | `album_id` | `artist_id` ke `artist` |
| `genre` | `genre_id`, `name` | `genre_id` | - |
| `media_type` | `media_type_id`, `name` | `media_type_id` | - |
| `track` | `track_id`, `name`, `album_id`, `media_type_id`, `genre_id`, `composer`, `milliseconds`, `bytes`, `unit_price` | `track_id` | `album_id` ke `album`, `genre_id` ke `genre`, `media_type_id` ke `media_type` |
| `invoice` | `invoice_id`, `invoice_date` | `invoice_id` | - |
| `invoice_line` | `invoice_line_id`, `invoice_id`, `track_id`, `unit_price`, `quantity` | `invoice_line_id` | `invoice_id` ke `invoice`, `track_id` ke `track` |

**Penjelasan relasi:**

| Relasi | Penjelasan |
|--------|------------|
| `artist` 1 ke banyak `album` | Satu artis dapat memiliki banyak album |
| `album` 1 ke banyak `track` | Satu album berisi banyak trek |
| `genre` 1 ke banyak `track` | Satu genre mengelompokkan banyak trek |
| `media_type` 1 ke banyak `track` | Satu jenis media (format file) dipakai banyak trek |
| `track` 1 ke banyak `invoice_line` | Satu trek bisa terjual berkali-kali |
| `invoice` 1 ke banyak `invoice_line` | Satu invoice berisi satu atau lebih item trek |

Rantai `artist` ke `album` ke `track` ke `invoice_line` inilah yang digabungkan saat membangun tabel fakta, dan `invoice` hanya dipakai untuk mengambil tanggal transaksi.

### 5.2 Struktur Star Schema (Data Warehouse)

Data warehouse memakai **star schema**: satu tabel fakta di tengah yang dikelilingi enam tabel dimensi. Setiap dimensi memiliki surrogate key berawalan `sk_` yang menjadi penghubung ke tabel fakta.

**Hubungan tabel fakta dengan dimensi:**

| Tabel fakta | Tabel dimensi | Kunci penghubung |
|-------------|---------------|------------------|
| `fact_invoice` | `dim_track` | `sk_track` |
| `fact_invoice` | `dim_album` | `sk_album` |
| `fact_invoice` | `dim_artist` | `sk_artist` |
| `fact_invoice` | `dim_genre` | `sk_genre` |
| `fact_invoice` | `dim_media_type` | `sk_media_type` |
| `fact_invoice` | `dim_date` | `sk_invoicedate` (ke `sk_date`) |

**Tabel fakta `fact_invoice`** (grain: satu baris per item invoice, 2.240 baris)

| Kolom | Penjelasan |
|-------|------------|
| `transaksi_id` | Nomor urut baris transaksi (primary key) |
| `invoice_id` | Nomor invoice, disimpan apa adanya tanpa tabel dimensi |
| `sk_invoicedate` | Kunci ke `dim_date` |
| `sk_track` | Kunci ke `dim_track` |
| `sk_album` | Kunci ke `dim_album` |
| `sk_artist` | Kunci ke `dim_artist` |
| `sk_genre` | Kunci ke `dim_genre` |
| `sk_media_type` | Kunci ke `dim_media_type` |
| `jumlah_terjual` | Ukuran: jumlah trek yang terjual |
| `revenue` | Ukuran: pendapatan (harga satuan x jumlah) |

**Tabel dimensi**

| Tabel | Sumber | Kolom | Penjelasan |
|-------|--------|-------|------------|
| `dim_track` | `track` | `sk_track`, `track_id`, `track_name`, `track_composer`, `album_id`, `genre_id`, `media_type_id`, `track_sec`, `track_size_mb`, `track_category`, `unit_price` | Atribut trek: nama, komposer, durasi dalam detik, ukuran dalam MB, kategori durasi, dan harga satuan. `album_id`, `genre_id`, dan `media_type_id` disimpan sebagai penghubung lookup |
| `dim_album` | `album` | `sk_album`, `album_id`, `album_title`, `artist_id` | Judul album, dan `artist_id` sebagai penghubung lookup ke artis |
| `dim_artist` | `artist` | `sk_artist`, `artist_id`, `artist_name` | Nama artis |
| `dim_genre` | `genre` | `sk_genre`, `genre_id`, `genre_name` | Nama genre |
| `dim_media_type` | `media_type` | `sk_media_type`, `media_type_id`, `media_type_name` | Jenis media (format file) trek |
| `dim_date` | dibuat sendiri | `sk_date`, `date_id`, `full_date`, `day_of_month`, `month_number`, `month_name`, `quarter`, `year`, `day_of_week`, `day_name`, `is_weekend` | 2.000 hari berurutan mulai 2021-01-01, dengan tahun, kuartal, bulan, nama bulan, nama hari, dan penanda akhir pekan. Dibuat sendiri karena OLTP tidak punya tabel tanggal |

**Catatan desain:**
- Setiap dimensi dihubungkan langsung ke tabel fakta lewat `sk_` (bukan snowflake), sehingga analisis cukup satu langkah penggabungan dari fakta ke dimensi.
- `album_id`, `genre_id`, `media_type_id`, dan `artist_id` disimpan di dimensi hanya sebagai penghubung saat membangun tabel fakta (Database lookup berantai: track, album, artis).
- Dimensi memakai SCD tipe 1 (data ditimpa), karena data Chinook bersifat statis.

### 5.3 Struktur Data Mart

Data mart dibentuk dari tabel fakta dan dimensi di atas, dalam dua bentuk dengan tingkat detail berbeda.

| Data mart | Grain | Isi kolom | Penjelasan |
|-----------|-------|-----------|------------|
| `datamart_track_monthly` | Satu baris per item invoice (2.240 baris) | `track_name`, `track_composer`, `track_sec`, `track_size_mb`, `track_category`, `unit_price`, `album_title`, `artist_name`, `genre_name`, `media_type_name`, `full_date`, `year`, `quarter`, `month_number`, `month_name`, `day_name`, `jumlah_terjual`, `revenue` | Data mart rinci yang sudah memuat nama dan atribut, jadi tidak perlu penggabungan tabel lagi. Dipakai untuk analisis per trek, album, artis, genre, dan jenis media |
| `datamart_revenue_monthly` | Satu baris per tahun dan bulan (60 baris) | `year`, `month_number`, `month_name`, `jumlah_terjual`, `revenue` | Data mart ringkasan hasil penjumlahan per bulan. Dipakai untuk memantau tren penjualan dan pendapatan dari waktu ke waktu |

---

## 6. Langkah Pengerjaan 

1. **Eksplorasi dan profiling data.** Memeriksa 11 tabel: jumlah baris, nilai kosong, duplikat, format, dan keutuhan relasi.
2. **Staging.** Menyalin tabel sumber apa adanya ke schema `staging` lewat job Pentaho dinamis, lengkap dengan log jumlah baris.
3. **Cleaning dan standarisasi.** Trim spasi, rapikan spasi ganda (regex), isi `composer` kosong dengan `Unknown`, ubah nama kolom, konversi durasi dan ukuran file.
4. **Membangun dimensi.** Membuat 6 tabel dimensi (`dim_track`, `dim_album`, `dim_artist`, `dim_genre`, `dim_media_type`, `dim_date`) dengan surrogate key `sk_`.
5. **Membangun tabel fakta.** Menggabungkan `invoice_line` dan `invoice`, lookup ke semua dimensi, hitung `revenue` dan `jumlah_terjual` menjadi `fact_invoice`.
6. **Membangun data mart.** Menggabungkan fakta dan dimensi menjadi `datamart_track_monthly`, lalu meringkasnya per bulan menjadi `datamart_revenue_monthly`.

---

## 7. Implementasi

### 7.1 Staging

Staging memakai dua job dan tiga transformation di Pentaho. Data sengaja dibiarkan sama persis dengan sumber, dan pembersihan dilakukan di tahap dimensi supaya data mentah tetap bisa diperiksa.

| Komponen | Fungsi |
|----------|--------|
| `main_staging_chinook` (Job) | Job utama: Start, `tf_list_table_chinook`, `execute_data_chinook`, Success |
| `tf_list_table_chinook` | Membuat daftar tabel OLTP yang belum tercatat di `log.log_table_entry` (Merge join, Filter rows, Copy rows to result) |
| `execute_data_chinook` (Job) | Dijalankan per tabel: `tf_read_data_chinook`, hitung target, tentukan `logical`, catat tabel masuk |
| `tf_read_data_chinook` | Membaca satu tabel sumber, mengosongkan tabel tujuan, menulis ke `staging`, mencatat jumlah baris sumber |
| `tf_tabel_masuk_chinook` | Mencatat nama tabel dan waktu masuk ke `log.log_table_entry` |

Tabel log:

```sql
CREATE SCHEMA IF NOT EXISTS log;

CREATE TABLE log.staging_log (
  table_name      VARCHAR(100),
  source_records  INTEGER,
  source_date     TIMESTAMP,
  target_records  INTEGER,
  target_date     TIMESTAMP,
  etl_date_start  TIMESTAMP,
  logical         VARCHAR(20)   -- Success jika source = target, selain itu To be Evaluated
);

CREATE TABLE log.log_table_entry (
  table_name VARCHAR(100),
  input_date TIMESTAMP
);
```

### 7.2 Star Schema

**Cleaning dan standarisasi di dimensi:**

| Operasi | Step Pentaho | Dikenakan pada |
|---------|--------------|----------------|
| Rapikan spasi ganda | Replace in string (regex `\s+`) | `track.name`, `track.composer` |
| Hapus spasi di tepi | String operations (trim) | semua kolom teks dimensi |
| Isi NULL | If field value is null | `composer` menjadi `Unknown` (977 baris) |
| Konversi satuan | Calculator | `milliseconds` menjadi `track_sec` (dibagi 1.000), `bytes` menjadi `track_size_mb` (dibagi 1.048.576) |
| Kategori durasi | Number range | `track_category`: Pendek (< 180 detik), Sedang (180 s.d. < 300), Panjang (300 s.d. < 600), Sangat Panjang (>= 600) |
| Surrogate key | Add sequence | `sk_track`, `sk_album`, `sk_artist`, `sk_genre`, `sk_media_type`, `sk_date` |
| Muat dimensi | Insert / update | Key: ID asli tiap dimensi |
| Dimensi tanggal | Generate rows, Calculator, Formula, Value Mapper | 2.000 hari mulai 2021-01-01 |

**Alur tabel fakta `fact_invoice`:**

1. Table input `invoice_line` (`invoice_id`, `track_id`, `quantity`, `unit_price`)
2. Stream lookup ke `invoice` untuk mengambil `invoice_date`
3. Database lookup berantai: `dim_track` (membawa `album_id`, `genre_id`, `media_type_id`), `dim_album` (membawa `artist_id`), `dim_artist`, `dim_genre`, `dim_media_type`, lalu `dim_date` (`full_date` = `invoice_date`)
4. Calculator: `revenue = unit_price x quantity`, `jumlah_terjual = quantity`
5. Select values (buang kolom bantu), Add sequence `transaksi_id`, Table output

### 7.3 Data Mart

**`datamart_track_monthly`:** `fact_invoice` dibaca, lalu Stream lookup ke tiap dimensi (track, album, artist, genre, media type, date) untuk mengambil nama dan atribut. Setelah Select values, hasilnya ditulis ke tabel dan ke file CSV.

**`datamart_revenue_monthly`:** `fact_invoice` digabung dengan `dim_date`, lalu **Group by** `year`, `month_number`, `month_name` dengan `SUM(jumlah_terjual)` dan `SUM(revenue)`.

---

## 8. Hasil dan Penjelasan

**Ringkasan angka:**

| Ukuran | Nilai |
|--------|------:|
| Jumlah transaksi (baris fakta) | 2.240 |
| Jumlah invoice | 412 |
| Total pendapatan | 2.328,60 |
| Rentang waktu | Jan 2021 s.d. Des 2025 (60 bulan) |
| Trek berbeda yang terjual | 1.984 dari 3.503 |
| Rata-rata per bulan | 37,3 trek, pendapatan 38,81 |

**Pendapatan per tahun:**

| Tahun | Jumlah terjual | Pendapatan |
|-------|---------------:|-----------:|
| 2021 | 454 | 449,46 |
| 2022 | 455 | 481,45 |
| 2023 | 442 | 469,58 |
| 2024 | 447 | 477,53 |
| 2025 | 442 | 450,58 |

Bulan dengan pendapatan tertinggi: Januari 2022 (38 terjual, 52,62). Terendah: November 2023 (24 terjual, 23,76).

**Genre teratas berdasarkan pendapatan:**

| Genre | Jumlah terjual | Pendapatan |
|-------|---------------:|-----------:|
| Rock | 835 | 826,65 |
| Latin | 386 | 382,14 |
| Metal | 264 | 261,36 |
| Alternative & Punk | 244 | 241,56 |
| TV Shows | 47 | 93,53 |

**Artis teratas berdasarkan pendapatan:** Iron Maiden (138,60), U2 (105,93), Metallica (90,09), Led Zeppelin (86,13), Lost (81,59).

**Penjelasan:**
- Penjualan relatif datar dari tahun ke tahun, tanpa lonjakan musiman yang mencolok.
- Rock mendominasi penjualan. Katalog genre Rock, Latin, dan Metal menyumbang sebagian besar pendapatan.
- Konten video (TV Shows, Drama, dan sejenisnya) berharga 1,99 sehingga kontribusi pendapatannya lebih besar dari porsi jumlah penjualannya.
- Hampir separuh katalog belum pernah terjual, sehingga ada peluang promosi seperti diskon, paket album, atau rekomendasi trek.

Contoh query dari data mart ringkasan:

```sql
SELECT "year", month_number, month_name,
       SUM(jumlah_terjual) AS jumlah_terjual,
       SUM(revenue)        AS revenue
FROM datamart.datamart_revenue_monthly
GROUP BY "year", month_number, month_name
ORDER BY "year", month_number;
```

---

## 9. Keterbatasan

## 9. Keterbatasan

- **Cakupan analisis hanya pada penjualan trek.** Dari 11 tabel Chinook, hanya 7 tabel yang dibawa ke data warehouse, yaitu `invoice`, `invoice_line`, `track`, `album`, `artist`, `genre`, dan `media_type`. Tabel `customer`, `employee`, `playlist`, dan `playlist_track` tidak dipakai, sehingga analisis yang berkaitan dengan pelanggan, karyawan, atau playlist, seperti pelanggan dengan pembelian terbesar atau kinerja karyawan, belum bisa dilakukan dengan data mart ini.
- **Kualitas data pada sumber belum sepenuhnya lengkap.** Sebanyak 977 dari 3.503 trek (sekitar 27,9%) tidak memiliki data komposer, dan nilai kosong tersebut diganti dengan `Unknown`, sehingga analisis berdasarkan komposer kurang akurat. Selain itu, ada 6 pasang trek yang nama dan albumnya sama tetapi `track_id`-nya berbeda. Keduanya tetap disimpan dan dihitung terpisah, jadi penjualan trek yang sebenarnya sama bisa terbagi ke dua baris. Kolom yang banyak kosong di tabel yang tidak dipakai (seperti `company`, `fax`, dan `billing_state`) tidak dibersihkan karena tidak dibawa ke data warehouse.
- **Periode dan cakupan penjualan terbatas.** Data transaksi hanya mencakup 1 Januari 2021 sampai 22 Desember 2025 (60 bulan), sehingga tren di luar periode ini tidak dapat dianalisis. Sebanyak 1.519 dari 3.503 trek (sekitar 43,4%) juga belum pernah terjual, sehingga analisis performa per trek hanya mencerminkan trek yang pernah dibeli, dan data penjualan trek yang belum terjual tidak muncul di tabel fakta.
