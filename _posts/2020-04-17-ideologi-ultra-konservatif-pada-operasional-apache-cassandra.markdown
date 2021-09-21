---
layout: post
title: Ideologi Ultra Konservatif pada Operasional Apache Cassandra
date: '2020-04-17 07:56:02'
tags:
- infrastructure
- database
- cassandra
---

> _konservatif/kon·ser·va·tif/ /konsérvatif/ a 1 kolot; 2 bersikap mempertahankan keadaan, kebiasaan, dan tradisi yang berlaku_

## Konservatif pada Token Range

Saat pertama kali saya dan tim menggunakan cassandra, saat itu versi paling lama yang kami pergunakan adalah 2.0.x , versi di mana sebenarnya cassandra sudah mengenal konsep vnodes. Sebuah konsep yang mengijinkan alokasi jumlah partisi dan pembagian letak partisi, secara otomatis diatur oleh cassandra itu sendiri. Namun, atas alasan yang cukup konservatif, saat itu, salah satu engineer di tim menolak gagasan penggunaan vnodes.

Meski kami sudah menguji gabungan multi DC cassandra dengan fixed token dan properti file snitch, dengan DC lain yang menggunakan vnodes. Alasannya sederhana, kita membutuhkan sebuah database cluster yang mudah untuk ditebak perilaku penyimpanannya. Hal ini masuk akal. Kami membutuhkan prediksi yang tepat, akan partisi dalam 1 token range atau gabungannya dengan replication factor, sehingga saat nanti kami harus melakukan rolling restart untuk keperluan patch atau update, kami tahu node mana yang harus kami maintain terlebih dahulu, dan mana node berikutnya.

Selain itu, tim akan terbantu ketika ingin melakukan repair data. Karena, tentu saja, kita tahu kemana kita akan mereferensi data. Kita juga akan dapat dengan mudah melakukan repair pada token range tertentu saja. Dengan pembagian yang jelas, ini memang akan mempermudah maintenance pada Cassandra.

Hal lain yaitu jaminan lebih baik pada keseimbangan data yang tersebar di antara nodes dalam 1 Data Center. Meski ini bergantung kembali kepada seberapa seimbang kita membagi sebuah token range utuh kepada jumlah nodes yang kita punya dalam 1 Data Center, namun hal ini setidaknya mengurangi resiko penumpukan token pada titik / node tertentu.

## Konservatif pada cara Scaling

Pada 2 perusahaan yang berbeda, saya tetap terus menghindari penggunaan vnodes dan hal itu juga berimplikasi pada keputusan untuk melakukan scaling cluster. Metode scaling cluster cassandra yang saya lebih sukai adalah dengan cara membuat data center baru dengan jumlah node yang lebih banyak, untuk kemudian menggantikan data center existing atau lama yang lebih kecil.

Kemudian data center lama ini akan di decommision setelah service atau aplikasi sudah mengarahkan query request ke data center baru. Untuk dapat melakukan ini, kita harus mengimplementasikan consistency level pada service atau driver aplikasi yang bersifat local, entah itu local one, local quorum dan seterusnya. Selain consistency level, kita juga harus membatasi query hanya terisolasi pada 1 data center.

Implementasi isolasi ini dapat dilakukan dengan mengenable opsi semacam DCAware kemudian mengunci query pada localDc, yaitu data center yang lama - pada saat data center baru sedang disiapkan atau di-rebuild dan kemudian kepada data center baru, setelah data center baru siap. Hal ini sangat memudahkan scaling dan terminasi pada cluster.

Pada vnodes, scaling dapat dilakukan dengan menambah node dan mengubah token. Hal ini berimbas pada pergeseran data yang akan terjadi secara live, karena kepemilikan token berubah. Pada konsep static token juga sangat dihindari metode serupa, yaitu dengan cara menambah node dan menggeser atau mengatur ulang partisi. Jika cluster hanya mempunya 1 data center saja, maka 1 data center yang kemungkinan sedang dipakai oleh service di production, juga akan terbebani dengan operasi internal saat menggeser data di berbagai node ini.

Photo by [lcs \_vgt](https://unsplash.com/@lcs_vgt?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/ultra?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

