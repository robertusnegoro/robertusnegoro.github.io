---
layout: post
title: Hal Kontraproduktif Ketika Migrasi ke Cloud
date: '2021-01-14 06:49:48'
tags:
- cloud
- culture
- infrastructure
- migrasi
---

Pada medio 2020 lalu, Bukalapak Engineering telah berhasil menyelesaikan proyek yang sangat penting dalam pendewasaan tim, baik secara teknologi, maupun secara organisasi. Proyek penting itu adalah migrasi infrastruktur dari data center on-premise ke cloud provider bernama Google Cloud Platform (GCP). Sebenarnya sudah ada 3 artikel yang lebih dahulu membahas proyek yang secara internal dinamakan Gatotkaca ini, 1 artikel di linkedin yang membahas hal yang [dipelajari oleh seorang TPM saat itu - Ichsan Rahardianto](https://www.linkedin.com/pulse/what-i-learned-out-bukalapaks-cloud-migration-ichsan-rahardianto/), 2 artikel lain di GCP Blog, yang pertama soal cloud migration 101 yang [dijabarkan dengan sangat baik oleh Mohan Krishnan](https://cloud.google.com/blog/products/cloud-migration/guide-to-all-google-cloud-migration-guides), dan terakhir soal pencapaian penting pada proyek Gatotkaca ini dari [sudut pandang seorang Alabsi](https://cloud.google.com/blog/products/databases/migrating-databases-kubernetes-clusters-and-developers-to-cloud).

Karena sudah ada 3 artikel dan mungkin masih banyak artikel lain yang bersifat positif, maka artikel yang saya tulis kali ini hanya akan memuat hal yang perlu dihindari, pola pikir yang salah atau hal bersifat kontraproduktif lainnya.

### Sumber Daya Cloud adalah Tidak Terbatas

Seperti kesabaran, sumber daya (resources) pada Cloud infrastructure dan provider, akan selalu terbatas. Meski kita melihatnya dalam bentuk virtual machines atau container, pada dasarnya semua berjalan di atas mesin fisik seperti pada on-premise. Karena kemudahan antar muka dan skala yang lebih besar saja, kita melihatnya seperti tak ada batasannya. Kealpaan pada perencanaan dapat membuat proses migrasi jadi terganggu, atau minimal cost efficiency jadi tidak tercapai.

Sebuah kesalahan besar ketika kita tidak melakukan inventory dan capacity sizing (pengukuran kapasitas) pada saat memindahkan infrastruktur kita ke cloud provider. Selain untuk dapat mengetahui apakah provider dapat menyerap kebutuhan kita, kesiapan kita dalam pengukuran kapasitas juga akan berguna untuk mendapatkan harga yang lebih terjangkau. Misalkan kita sudah tahu machine size yang kita butuhkan, kita dapat membeli reserved instance atau CUD pada provider. Intinya kita komitmen akan menggunakan dengan kapasitas tertentu, maka in-return, kita mendapatkan harga yang lebih baik.

### Pakai Cloud Langsung Auto Hemat

Meskipun pada beberapa cloud provider seperti AWS dengan reserved instance dan GCP dengan CUD-nya, tidak berarti memindahkan infrastruktur ke cloud provider secara otomatis billing kita akan lebih murah daripada ongkos (baik capex maupun opex) di on-premise. Jika ada anggapan pada cloud kita cenderung akan melupakan capex, anggapan itu tidak sepenuhnya benar, karena kita tetap harus memperhitungkan kapasitas untuk mendapatkan biaya yang kita keluarkan untuk membeli CUD / reserved instances. Mendapatkan harga terbaik dengan cara seperti ini, adalah salah satu cara mendapatkan harga yang lebih baik dan menghemat pengeluaran juga.

Pada migrasi yang dilakukan secara lift and shift, tanpa ada optimasi dan efisiensi, atau melakukan hal yang esensial untuk membuat service lebih "cloud native", juga tidak akan mereduksi tagihan di akhir bulan. Namun pendekatan migrasi adalah pilihan. Akan sulit untuk mendapatkan akselerasi dari segi waktu migrasi dengan cost efficiency dalam waktu bersamaan, namun kedua hal tersebut dapat dicapai secara berimbang dengan pengaturan prioritas, keep the joyful stuff ala Marie Kondo dan manajemen resiko yang baik.

### Network di Cloud Akan Selalu Stabil

Hal ini adalah mitos yang setara dengan keberadaan kuda unicorn, virus corona itu tidak ada konspirasi elit global, atau nama asli Gajah Mada sebenarnya Gaj Ahmad a. Tidak ada penjelasan lebih jauh lagi soal ini. Walau bagaimanapun, ini adalah mitos yang tidak hanya terjadi di cloud namun juga di on-premise. Sebaiknya saat menyusun SLA service sendiri, hal yang paling penting diperhatikan adalah SLA dari dependency yang dimaintain / managed oleh cloud provider. Kemudian secara disiplin menerapkan pendekatan mitigasi network failure seperti circuit breaker, exponential backoff dan sebagainya.

### Performa Aplikasi Otomatis Meningkat (atau Minimal Sama Seperti On-Premise)

Pada beberapa kasus, performa cloud resource seperti disk mempunyai constraint tertentu. Misalkan pada GCP, performa disk dibatasi oleh beberapa hal, misalkan jumlah vCPU, besaran disk sampai dibatasi oleh network throughput. Pada aplikasi atau service yang sangat sensitif terhadap latensi, maka hal - hal kecil seperti ini harus diperhatikan.

Naik atau turunnya performa sebuah service atau aplikasi tidak datang dengan tiba - tiba. Justru dengan migrasi ke cloud, akan ada beberapa hal yang berubah, misalkan constraint pada resource yang disebutkan di atas, latency antar service dan dependency serta faktor - faktor lainnya. Maka dari itu, sebelum melakukan cutover ke cloud, sebaiknya perlu dilakukan load testing pada environment non production (staging atau pre-production) untuk dapat dilihat apakah service yang dipasang di cloud dapat memenuhi target pada metrik - metrik sesuai dengan SLO. Jika terdapat bottleneck di suatu bagian pada service, ada baiknya diselesaikan terlebih dahulu masalahnya sebelum dipindahkan ke cloud.

### Migrasi Wajib Menggunakan Certified Engineer

Tidak mengatakan bahwa pendekatan ini salah, namun sertifikasi memang bukan penentu utama sukses atau tidaknya sebuah proyek migrasi ke cloud. Walau bagaimanapun, cloud provider pada dasarnya menyuguhkan produk dan solusi, yang mana akan selalu dimutakhirkan sesuai dengan roadmap perusahaan, atau kebutuhan pelanggan. Kebolehan suatu organisasi mendapatkan engineer dengan pengetahuan spesifik ke cloud provider yang akan dipakai, jelas akan mengakselerasi kemajuan atau progres dari proyek migrasi. Produk dari cloud provider bukanlah hal yang rumit untuk dipelajari. Memberikan waktu untuk engineer yang sudah ada, untuk melakukan PoC atau eksperimen dengan resource dan biaya yang terkontrol adalah salah satu cara membantu tim untuk lebih familiar dengan teknologi dan produk cloud provider.

### Data di Cloud Tidak Aman

Hal ini sering didengar dari seseorang yang entah awam, atau memang tidak punya pengalaman hidup yang bersinggungan dengan cloud provider. Mulai dari "Data di cloud itu kan seperti taruh data kita di tempat orang, jadi nanti mereka bisa baca data kita" sampai "Data di Cloud itu tidak aman, kan cloud itu publik, nanti bisa dilihat orang". Jika dinalar ada sebagian pernyataan yang benar, misalkan data kita ada di infrastruktur pihak ke tiga, yang mana kita tidak punya kontrol. Hal ini dapat diselesaikan dengan perjanjian NDA atau penyesuaian layanan dengan peraturan (compliance) yang relevan dengan jenis bisnis. Secara umum, kita dapat bercakap dan berkonsultasi dengan Account Manager dari cloud provider.

Untuk pasal keamanan data, tentu saja sangat bergantung bagaimana kita mencegah hal tersebut terjadi terlepas dari bentuk infrastruktur, entah di on-premise atau di cloud. Pada sebagian kasus data breach, umumnya hal tersebut diakibatkan hal - hal yang tidak secara spesifik berkaitan dengan kapabilitas cloud secara khusus, atau kapabilitas infra secara umum, dan lebih sering berkaitan erat dengan prosedur internal penyimpanan data pengguna - seperti menggunakan dump database dan tidak memperhatikan storage tempat penyimpanan tersebut, hingga keteledoran manusia.

Selain hal tersebut di atas, masih banyak hal yang kontraproduktif lainnya, misalkan menghabiskan waktu untuk berdebat mengenai standarisasi atau konvensi tanpa mempunyai dasar data yang kuat, berpikir bahwa best practice itu overrated, dan berbagai hal picik lain soal cloud dan ihwal lainnya.

_Gimana, udah siap pindah ke cloud?_

_Bird migration Photo by [Julia Craice](https://unsplash.com/@jcraice?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/migration?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)_

