---
layout: post
title: Euro 2020, Inggris dan Downtime
date: '2021-06-15 04:35:24'
tags:
- sre
- football
- euro-2020
- cloud
- culture
---

Bagi para pecinta sepakbola, selama beberapa pekan ke depan, kita semua akan dimanjakan dengan tontonan olahraga populer nan berkualitas dari dua kompetisi antar negara. Yang pertama Copa America 2021 - turnamen yang digelar oleh CONMEBOL dengan peserta negara - negara Amerika Selatan. Turnamen ini disiarkan secara daring oleh vidio.com dan sudah dibuka dengan kemenangan Brasil sebagai juara bertahan. Yang satu lagi, Euro 2020 - turnamen yang digelar oleh UEFA, diikuti oleh negara benua eropa termasuk Rusia. Sebagai episentrum liga - liga terbaik dunia, pagelaran ini ditunggu oleh masyarakat. Euro 2020 yang digelar di tahun 2021 ini, disiarkan oleh mola.tv, sebuah platform OTT daring yang masih cukup muda, dan didukung oleh pabrikan besar ternama, yaitu Polytron. Euro 2020 sudah dimulai terlebih dahulu dan dibuka dengan kemenangan Italia atas Turki, pada tanggal 12 Juni 2021 lalu.

Euro sebagai turnamen dengan sejarah panjang ini, belum genap berjalan sepekan, namun sudah banyak hal terjadi di beberapa hari kebelakang ini. Mulai dari kemenangan telak Italia, insiden terjatuhnya Christian Eriksen di lapangan (get well soon, Eriksen!), laga dramatis Belanda melawan Ukraina, selebrasi rasial dari Arnautovic sampai gol indah Patrick Schick. Yang paling epik dan dirasakan sangat mengesalkan (termasuk saya), adalah insiden pada saat laga perdana Inggris di kompetisi Euro 2020 ini. Laga tersebut sangat ditunggu dan diantisipasi warga pecinta bola, telaknya lagi, laga itu jatuh di akhir pekan. Bukan insiden di dalam lapangan, melainkan, terjadinya gangguan yang membuat para pelanggan Mola.tv tidak bisa menyaksikan pasukan Gareth Southgate. Saya berulang kali mencoba untuk masuk melalui web dan android app, namun hasilnya saya tetap mendapatkan gangguan yang sama.

Akhir pekan, Inggris dan waktu primer (prime time) adalah kombinasi cukup mematikan untuk platform streaming semacam Mola.tv . Seperti yang kita tahu, Inggris dan Liga Primer-nya adalah liga terpopuler di negara kita. Hal yang sama juga pernah terjadi ketika laga Malaysia melawan Indonesia, Persija melawan Persib di platform saingannya, vidio.com. Bedanya, vidio.com tidak mengalami insiden seburuk mola.tv. Sebagai orang di luar organisasi Mola, tentu saya tidak tahu penyebab utama dari insiden di hari itu. Yang bisa saya bayangkan, saat itu Mola sedang mengumpulkan tim teknis terkait dan berusaha "memadamkan" api yang sudah terlanjur menjalar.

Sebenarnya untuk pengguna yang sudah terlanjur membuka atau mungkin para pengakses awal halaman Ingriss melawan Kroasia saat itu, akan dapat menikmati tayangan tersebut, asalkan tidak dimuat ulang lamannya. Sehingga ada beberapa pengguna Mola yang merasa tidak ada insiden atau hambatan seperti pengguna kebanyakan. Sekali lagi, ini bukanlah penyebab sesungguhnya, atau bahkan bukan hipotesa, namun sering kali pada kejadian lumpuhnya halaman streaming yang disertai dengan melonjaknya trafik, jarang terjadi bottleneck pada "streaming" nya itu sendiri. Biasanya live streaming sudah ditangani oleh CDN sebagai komponen utama yang sanggup menangani trafik besar sekalipun.

Lonjakan trafik ini biasanya berakibat cukup fatal terhadap service backend seperti API endpoint yang menangani proses bisnis dan fitur lain. Misalkan endpoint untuk mengecek legitimasi user, paket user, menyediakan umpan balik sebuah URL live stream yang tepat, sampai pada fitur live chat. Pada kasus mola.tv, tidak terdapat fitur live chat seperti di youtube atau vidio.com. Jika pada saat kejadian, anda membuka mola lewat web dan membuka developer tools, maka dapat dilihat ada beberapa API endpoint yang mengembalikan respon dengan kode error.

Tanpa mengetahui apa detil yang terjadi di tim teknis mola.tv, penulis merasa bahwa pada setiap even besar, penting dilakukan beberapa persiapan, termasuk melibatkan tim bisnis dan produk. Biasanya tim tersebut mampu memprediksi seberapa besar trafik yang akan masuk dan menonton tayangan tersebut. Kemudian persiapan teknis apa yang bisa dilakukan tim infrastruktur bersama pemrogram?

### Mempersiapkan _War Room_

Pada tim teknis, sebaiknya mempersiapkan tim untuk bersiap (on-call standby) pada saat waktu berlangsungnya even besar tersebut. Dalam kasus super ideal, organisasi mempunyai tim _techops_ yang dapat mengumpulkan pihak - pihak terkait ke dalam sebuah _war room_. Sehingga ketika terjadi insiden, semua pihak yang berkompeten, dapat dipanggil di satu tempat saja. Ini juga dapat memperpendek MTTR (mean time to resolve) pada insiden.

### Autoscaling Tak Bisa Diandalkan

Menurut pengalaman, kita tak lagi sanggup mengandalkan fitur autoscaling yang umumnya tersedia di platform - plaform orkestrasi atau cloud. Fitur autoscaling tak lagi handal untuk mengatasi gelombang kerusakan eksponensial akibat lonjakan trafik yang secara ekstrim sangat drastis.   
Sebelum terjadinya even, hendaknya tim teknis mulai mempersiapkan peladen yang sudah _hot standby_ untuk melayani jumlah trafik yang melonjak tinggi. Selain itu, penting pula untuk menggarisbawahi metrik - metrik yang penting untuk dipantau, termasuk oleh pemrogram.

### Memprediksi Bottleneck pada Dependensi

Setelah mengetahui bahwa _autoscaling_ tidak dapat diandalkan. Tim teknis juga harus dapat memprediksi limitasi dari sistem - sistem lain yang menjadi dependensi dari services yang akan diakses oleh pengguna. Entah itu database, atau sistem lain, bisa saja tidak mampu melayani trafik masuk sesuai ekspektasi, sehingga membutuhkan perubahan konfigurasi sebelum even besar dimulai. Biasanya tak mudah menambah kapasitas (_scaling up_) dependensi, terutama _stateful dependency._ &nbsp;

Beberapa hari setelah terjadi insiden tersebut, saya pribadi melihat Mola melakukan _crisis management_ dengan sangat baik, yaitu memberikan layanan gratis (liga primer Inggris) sebagai kompensasi atas insiden Inggris vs Kroasia.

Dari persiapan di atas, mungkin ada yang kurang. Sila ditambahkan jika ada yang kurang dengan merespon laman ini ke kontak[at]robee.id

