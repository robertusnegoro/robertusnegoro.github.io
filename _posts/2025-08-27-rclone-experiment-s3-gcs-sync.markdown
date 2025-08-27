---
layout: post
title: "Eksperimen Rclone untuk Sinkronisasi Data antar Cloud Provider"
date: 2025-08-27 10:00:00 +0700
tags:
- rclone
- aws
- gcp
- s3
- gcs
- infrastructure
- cloud
---

![Cloud Data Synchronization](https://images.unsplash.com/photo-1451187580459-43490279c0fa?ixlib=rb-4.0.3&auto=format&fit=crop&w=1200&q=80)

Sebagai bagian dari eksperimen pribadi untuk memahami berbagai tool sinkronisasi cloud storage, saya tertarik untuk mencoba kemampuan rclone dalam memindahkan data antara Google Cloud Storage (GCS) bucket dan Amazon S3 bucket. Meskipun tersedia managed service seperti AWS DataSync atau GCP Storage Transfer Service, eksperimen ini bertujuan untuk mengevaluasi rclone sebagai alternatif yang lebih cost-effective.

Rclone, sebagai sebuah command line tool yang cukup matang untuk sinkronisasi cloud storage, ternyata memberikan hasil yang cukup memuaskan dengan beberapa catatan penting yang perlu diperhatikan dalam eksperimen ini.

## Konfigurasi Rclone

Untuk dapat melakukan sinkronisasi antar cloud provider, terlebih dahulu perlu dilakukan konfigurasi rclone untuk kedua layanan cloud storage. Alih-alih menggunakan mode interaktif, saya memilih pendekatan command line agar konfigurasi dapat direproduksi dengan mudah.

### Konfigurasi Google Cloud Storage

```bash
rclone config create gcs "google cloud storage" \
  project_number "123456789"
```

Hal penting yang perlu diperhatikan adalah penggunaan `project_number` dan bukan `project_id`. Kesalahan umum yang sering terjadi adalah mencampurkan kedua parameter ini, yang dapat menyebabkan kegagalan otentikasi.

### Konfigurasi AWS S3

```bash
rclone config create aws s3 \
  provider AWS \
  env_auth true \
  region ap-southeast-3 \
  endpoint https://s3.ap-southeast-3.amazonaws.com
```

Pada konfigurasi S3, saya menggunakan `env_auth true` yang memungkinkan rclone mengambil credentials dari environment variables (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, dll). Pendekatan ini lebih aman dibandingkan dengan menyimpan credentials secara langsung di file konfigurasi.

Region `ap-southeast-3` adalah Jakarta region yang memiliki performa cukup baik untuk user di Indonesia.

## Verifikasi Akses Bucket

Sebelum memulai proses sinkronisasi, langkah yang baik adalah memverifikasi terlebih dahulu bahwa rclone dapat mengakses kedua bucket:

```bash
rclone lsd gcs:gcp-bucket
rclone lsd aws:aws-bucket
```

Command `lsd` (list directories) berguna untuk memverifikasi akses dan melihat struktur folder di dalam bucket.

## Proses Sinkronisasi Data

Berikut adalah command utama yang digunakan untuk melakukan sinkronisasi:

```bash
rclone sync gcs:gcp-bucket aws:aws-bucket  \
  --transfers=64 \
  --checkers=64 \
  --multi-thread-streams=16 \
  --multi-thread-cutoff=100M \
  --buffer-size=256M \
  --s3-chunk-size=64M \
  --s3-upload-concurrency=32 \
  --fast-list \
  --progress --stats=30s
```

Berikut penjelasan detail dari masing-masing parameter:

- `--transfers=64`: Jumlah file transfer yang berjalan secara paralel
- `--checkers=64`: Jumlah checker yang berjalan paralel untuk membandingkan source dan destination
- `--multi-thread-streams=16`: Jumlah stream per file untuk file berukuran besar
- `--multi-thread-cutoff=100M`: File yang lebih besar dari nilai ini akan dipecah menjadi multiple streams
- `--buffer-size=256M`: Buffer size untuk operasi read
- `--s3-chunk-size=64M`: Ukuran chunk untuk S3 multipart uploads
- `--s3-upload-concurrency=32`: Jumlah concurrent S3 uploads
- `--fast-list`: Menggunakan recursive listing (lebih cepat tapi menggunakan memori lebih banyak)
- `--progress --stats=30s`: Menampilkan progress setiap 30 detik

Perlu diperhatikan bahwa pengaturan ini cukup agresif dan dapat membebani network atau mencapai API rate limits. Disarankan untuk memulai dengan nilai yang lebih rendah dan melakukan penyesuaian secara bertahap.

## Verifikasi Hasil Sinkronisasi

Setelah proses sinkronisasi selesai, langkah penting selanjutnya adalah melakukan verifikasi:

```bash
rclone size aws:aws-bucket
rclone size gcs:gcp-bucket
```

Command ini akan menampilkan total ukuran dan jumlah objek di masing-masing bucket. Kedua nilai tersebut seharusnya sama jika proses sinkronisasi berhasil.

## Perbandingan dengan Managed Solutions

### AWS DataSync

**Keunggulan DataSync:**
- Layanan yang sepenuhnya dikelola oleh AWS
- Built-in data validation
- Integrasi yang lebih baik dengan layanan AWS lainnya
- Automatic retry mechanism
- Bandwidth throttling
- Integrasi dengan CloudWatch untuk monitoring

**Kekurangan DataSync:**
- Biaya yang lebih mahal, terutama untuk dataset besar
- Opsi konfigurasi yang lebih terbatas
- Vendor lock-in dengan ekosistem AWS
- Customization yang terbatas

### GCP Storage Transfer Service

**Keunggulan Storage Transfer Service:**
- Integrasi native dengan GCP
- Scheduled transfers
- Built-in monitoring dan logging
- Tidak perlu mengelola infrastruktur
- Automatic resumption untuk transfer yang gagal

**Kekurangan Storage Transfer Service:**
- Terbatas pada kombinasi source/destination tertentu
- Kontrol yang lebih sedikit terhadap parameter transfer
- Dapat menjadi mahal untuk transfer yang sering
- Terkadang lebih lambat dibandingkan rclone yang dioptimasi

## Keunggulan Menggunakan Rclone

1. **Cost Effective**: Hanya perlu membayar compute instance tempat rclone berjalan, ditambah biaya data transfer
2. **Highly Configurable**: Banyak parameter yang dapat disesuaikan untuk optimasi performa
3. **Multi-Cloud Native**: Kompatibel dengan hampir semua cloud storage provider
4. **Open Source**: Gratis dan dapat dikembangkan lebih lanjut
5. **Powerful Features**: Enkripsi, kompresi, filtering, dan berbagai fitur lainnya
6. **Local Control**: Dapat dijalankan dari infrastruktur sendiri

## Kekurangan dan Hal yang Perlu Diperhatikan

1. **Complexity**: Memerlukan pemahaman terhadap berbagai parameter dan implikasinya
2. **No Built-in Monitoring**: Perlu setup monitoring dan alerting secara manual
3. **Network Dependency**: Performa sangat bergantung pada koneksi network yang tersedia
4. **Resource Usage**: Dapat mengonsumsi CPU dan memory yang signifikan dengan pengaturan agresif
5. **Error Handling**: Memerlukan logic retry manual untuk production workloads
6. **Rate Limiting**: Mudah mencapai API rate limits dengan pengaturan yang terlalu agresif

## Pembelajaran dari Eksperimen

1. **Mulai dengan Pengaturan Konservatif**: Mulai dengan transfer rate yang rendah dan tingkatkan secara bertahap
2. **Monitor Resource Usage**: Perhatikan penggunaan CPU, memory, dan network
3. **Test dengan Dataset Kecil**: Selalu validasi pendekatan dengan subset data terlebih dahulu
4. **Backup adalah Prioritas**: Pastikan memiliki backup sebelum melakukan migrasi besar
5. **Network Matters**: Bandwidth dan latency berdampak signifikan terhadap performa

## Kesimpulan

Rclone terbukti merupakan tool yang powerful untuk sinkronisasi data antar cloud provider. Meskipun managed solutions menyediakan integrasi yang lebih baik dan operational overhead yang lebih rendah, rclone memberikan kontrol yang lebih besar dan dapat menjadi pilihan yang lebih ekonomis untuk dataset berukuran besar.

Kunci utamanya adalah memahami kebutuhan spesifik:
- Untuk migrasi one-time dengan constraint biaya: rclone dapat menjadi pilihan ideal
- Untuk production workloads yang berkelanjutan: managed solutions mungkin pilihan yang lebih baik
- Untuk skenario multi-cloud yang kompleks: fleksibilitas rclone sangat bermanfaat

Seperti biasa, lakukan testing secara menyeluruh sebelum deployment production. Dan ingat, dengan power yang besar datang tanggung jawab yang besar pula - pengaturan rclone yang agresif dapat dengan mudah membebani sistem jika tidak dikonfigurasi dengan tepat.
