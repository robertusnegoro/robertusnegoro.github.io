---
layout: post
title: Right Application for Right Purpose
date: '2020-01-02 03:39:13'
tags:
- infrastructure
- cloud
- software-architecture
- microservice
---

Sebenarnya penulis ingin membuat tulisan yang membahas tahun lalu dan apa yang telah terjadi, dilanjutkan dengan cita - cita dan target di tahun ini, mengingat itu hal normal yang mungkin setiap penulis atau orang lain lakukan di awal tahun. Tapi mungkin ada baiknya saya menulis refleksi mendasar tentang teknologi yang tidak terlalu teknis, tapi tetap menarik untuk dibahas.

Mengawali tahun ini dengan berbagai target yang sudah ditentukan di dokumen OKR (_Objectives and Key Results_), pada organisasi masing - masing. Biasanya semua dibuat di akhir masa OKR sebelumnya, dalam hal ini Q4 2019. Di kantor saya yang sekarang ini, ada banyak misi yang harus diselesaikan oleh tim teknologi. Misi tersebut sudah dituangkan dan diperjelas dalam dokumen, presentasi dan berbagai bentuk pesan lain, dari pemimpin organisasi hingga pemimpin bagian masing - masing. Untuk mengejar target misi tersebut, kami bukan hanya berlari dengan mengerjakan tugas, namun membawa bertumpuk - tumpuk pengalaman yang sudah didapatkan di masa sebelumnya.

Salah satu pengalaman yang paling saya ingat adalah bagaimana, di masa lalu, organisasi dan bagian di dalamnya melakukan beberapa kesalahan yang fundamental, hingga pada akhirnya ada harga yang harus dibayar di kemudian hari. Kesalahan tersebut bisa berupa kesalahan metode, kesalahan prosedur hingga kesalahan arsitektural. Dan hal paling mendasar adalah kesalahan arsitektural.

Kesalahan arsitektural dalam membangun perangkat lunak yang melayani trafik yang sangat tinggi, memerlukan pendalaman masalah yang sangat intens. Karena banyak akibat yang harus dihadapi, termasuk akibat yang tidak diantisipasi dari awal. Dalam kesalahan arsitektural, sering kali ditemui kecerobohan dalam memilih aplikasi perangkat lunak tertentu untuk menyelesaikan masalah spesifik.

Misalkan saja memilih menggunakan aplikasi _batch stream_ untuk penyimpanan semi permanen, atau bahkan permanen. Memilih menggunakan aplikasi penyimpanan pada memori _volatile_ sebagai kakas basis data. Hal ini terjadi bahkan kerap terjadi karena berpikir segala macam permasalahan performa, dapat diatasi dengan mengoptimalkan aplikasi dari sisi penunjang atau dari sisi infrastruktur. Berpikir bahwa sumber daya tidak terbatas. Tentunya pemikiran ini tidak sepenuhnya benar, karena jika hal ini terjadi di ekosistem yang dibangun di _cloud infrastructure_, maka segalanya menjadi amat terbatas.

Pemilihan aplikasi penunjang seperti _database system, caching system, queue system_ dan sebagainya, memerlukan pengetahuan akan kinerja aplikasi yang dibuat, cara kerja aplikasi penjunjang tersebut dan operasional dibalik aplikasi penjunjang itu semua. Hal ini juga tetap diperlukan bila aplikasi penjunjang tersebut disediakan oleh pihak lain atau yang kerap disebut _managed service_.

Kesalahan pemilihan _tech stack_ ini selain setumpuk _technical debt_ tapi juga permasalahan performa yang pada akhirnya dikonversi menjadi pemborosan biaya secara langsung. Bukan lagi biaya manusia lagi, tapi biaya sumber daya seperti ongkos server atau bahkan server dan bandwidth. Tidak selamanya kita dapat menambal kekurangan aplikasi dengan menambahkan CPU atau alokasi memori ke mesin. Seperti tidak selamanya uang akan menyelamatkan performa aplikasi.

Sebuah organisasi yang mencari profit dengan teknologi sebagai tulang punggung, tidak harus memiliki seorang dengan titel Arsitek, entah itu _software architect, infrastructure architect_ dan lain sebagainya. Namun ada baiknya mempunyai tim ad-hoc yang dapat dibentuk jika dibutuhkan, untuk membahas arsitektur sebuah _service_ baru atau merombak arsitektur _service_ lama yang bermasalah. Seperti dalam sebuah pemerintahan, Presiden terkadang membentuk tim pencari fakta yang sifatnya tidak permanen. Tentunya tim tersebut harus berisikan insinyur - insinyur teknologi yang berkompeten serta jika memungkinkan, mempunyai pengalaman yang sangat bervariasi.

Setelah mengetahui bahwa kesalahan arsitektural adalah kesalahan yang sangat mendasar, semoga di tahun ini (2020) ke depan, tidak ada lagi yang menggunakan _[Redis](https://redis.io/)_ sebagai _database system_ (yang _persistent_ dan dalam jumlah data yang besar) lagi.

.... Aamiin ....

