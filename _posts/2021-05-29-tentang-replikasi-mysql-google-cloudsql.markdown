---
layout: post
title: Tentang Replikasi Eksternal dari dan ke MySQL Google CloudSQL
date: '2021-05-29 16:33:11'
tags:
- database
- cloud
- gcp
- mysql
- infrastructure
---

Penulis ingin mengawali catatan kali ini dengan kisah jenaka, betapa lucunya peraturan negeri ini mengenai pusat data untuk industri finansial, tapi takut dan mungkin nirfaedah. Semoga seseorang yang bekerja di bidang yang sama, nanti, akan mampu menuliskan betapa jenakanya peraturan tersebut.

Ya, meski kita tahu di Google CloudSQL (baik postgresql maupun mysql) kita dapat membuat replika, namun oleh karena aturan bahwa sebuah perusahaan yang bergerak di bidang finansial, harus membuat pusat data untuk pemulihan bencana (_Disaster Recovery Data Center),_ maka pada kasus ini, penulis dan tim infrastruktur, ditugaskan untuk mempersiapkan sebuah set komplit pada pusat data lain. Karena pusat data DR, maka harus selain Google Cloud Platform (sudah dipakai sebagai pusat data utama), bisa saja di sebuah VM di penyedia lain, seperti Alibaba Cloud, Biznet Gio/Neo atau merek lain. Pada aktivitas ini, komponen yang paling menjadi sorotan adalah klaster basis data, dalam hal ini MySQL Database Cluster.

Tulisan ini tidak akan menjelaskan langkah per langkah, namun hanya merekam hal penting yang mungkin nanti akan membantu penulis kembali, jika menemui hal yang sama ke depannya. Pada tulisan ini, versi MySQL yang penulis gunakan adalah versi 8.0. Untuk detail petunjuk, akan penulis lampirkan tautan referensi di akhir artikel. Pada aspek yang lebih detil, penulis berkonsultasi dengan rekan ahli lainnya, yaitu [Hofid Mashudi](https://id.linkedin.com/in/hofid) dan [Agus Syafaat](https://id.linkedin.com/in/agsyafaat) (terima kasih teman temanku) \* _gestur saranghaeyo_ \*.

### Replikasi ke Replika Eksternal

Pada skenario ini, kita akan mempunyai MySQL _primary instance_ di Google CloudSQL sedangkan untuk _replica instances_ ada di CloudSQL sendiri dan sebuah VM (sebuah VM berisi MySQL yang dipasang dan dikonfigurasi sendiri tanpa _managed service_).

    mysql vm replica ---- mysql cloudsql primary 
                             -- cloudsql replica

Selain membuat MySQL di Google CloudSQL dan VM, langkah - langkahnya secara garis besar adalah sebagai berikut :

1. Membuat 1 _user_ untuk replikasi dengan _permission_ `REPLICATION SLAVE` atau `REPLICATION CLIENT` pada _primary instance_ di CloudSQL
2. Memastikan antara VM _replica_ dan CloudSQL dapat berkomunikasi di port 3306
3. Ekpor data dari _primary instance_ (CloudSQL) dengan mempertahankan GTID
4. Impor data dari hasil ekspor di atas ke replika di VM
5. Menyalin _server certificate_ dari _primary instance_ di CloudSQL ke VM _replica_.
6. Set master di VM _replica_ dengan contoh `CHANGE MASTER TO MASTER_HOST = '', MASTER_PORT = 3306, MASTER_USER = '', MASTER_PASSWORD = '<replica user's password>, MASTER_AUTO_POSITION = 1, MASTER_SSL = 1, MASTER_SSL_CA = '/path/to/server-ca.pem';`
7. `START REPLICA` dijalankan di VM _replica_

Cukup simpel dan tidak berbelit. Namun ada beberapa hal yang perlu diperhatikan pula. Pada kasus ini, kita harus memastikan `replication-ignore-db` pada replika eksternal harus dapat diset dan dipasang `mysql` sebagai nilainya. `replication-ignore-db = mysql`. Sehingga, jika kita ingin melakukan replikasi misalkan antara mysql di cloudsql dan amazon RDS, kita harus pastikan dulu RDS support setup parameter ini atau tidak, jika tidak, maka replikasi tak dapat dilakukan. [1]

Hal lain, adalah memastikan parameter `read_only` di set ke **ON** , pada replika. Jika perlu, setelah setup selesai, kita set `super_read_only` untuk menghindari "_errant transaction_" [2]

### Replikasi dari Eksternal (_External Master/Primary_)

Salah satu aspek penilaian auditor terhadap kesiapan sistem, pada produk - produk teknologi finansial (tekfin / _fintech_) salah satunya adalah _data center swing_ atau secara sederhana, melakukan simulasi jika terjadi bencana, dan pusat data cadangan akan menjadi yang utama. Dari sisi _database_, maka hal ini sesederhana membalik skenario di atas dari cloudsql sebagai _primary instance_ dan VM sebagai replika. Sehingga VM sekarang sebagai _primary instance_ sedangkan mysql di CloudSQL menjadi replika. Prosedur lengkap ada di referensi bawah [3], namun secara ringkas dapat ditulis sebagai berikut :

1. Membuat 1 _user_ untuk replikasi dengan _permission_ `REPLICATION SLAVE` atau `REPLICATION CLIENT` pada VM mysql replika
2. Membuat "_ **source representation** _" pada Google CloudSQL [4]
3. Melakukan prosedur _demote_ pada mysql instance di CloudSQL (yang tadinya adalah primary)
4. eksekusi `STOP REPLICA` dan `RESET REPLICA ALL` pada mysql di VM (yang saat ini jadi master)
5. set parameter `read_only` (dan `super_read_only`) menjadi **OFF** pada mysql di VM

Sesederhana itu. Pada aktivitas ini, yang paling harus diperhatikan adalah pada saat pembuatan _source representation_ dan prosedur _demotion_ , untuk :

1. memastikan memasukkan _server certificate_ milik mysql di VM dengan tepat
2. memastikan mysql di CloudSQL tidak punya CLoudSQL _replica_ lagi. [5]

Jika terdapat replika pada sisi CloudSQL, maka yang terjadi, akan ada _return error_ seperti ini :

    {
      "error": {
        "code": 400,
        "message": "The requested operation is not valid for an instance that has a replica.",
        "errors": [
          {
            "message": "The requested operation is not valid for an instance that has a replica.",
            "domain": "global",
            "reason": "badRequest"
          }
        ]
      }
    }

### Monitoring

Pada kedua aktivitas di atas, pastikan kondisi replikasi selalu dalam pantauan.

- Tetap memantau `Second Behind Source` tetap 0 di hasil output dari perintah `SHOW REPLICA STATUS\G`
- Cek kondisi replika dari sisi _primary instance_ `SHOW SLAVE HOSTS`
- Cek GTID di `SHOW MASTER STATUS`
- Jika memungkinkan, pantau metrik ini pada kakas pemantauan (_monitoring tools_) seperti Grafana + Prometheus atau Stackdriver Monitoring atau AWS Cloudwatch atau Datadog dll.
- Pastikan retensi binlog pada _primary instance_ dihitung dengan benar disesuaikan kebutuhan replikasi dan kondisi _network_

Jika ada yang pertanyaan atau masukan, dengan senang hati dapat berdikusi di kontak[at]robee.id.   
  
\* _nonton drakor lagi_ \*

### Referensi:

1. [https://cloud.google.com/sql/docs/mysql/replication/configure-external-replica](https://cloud.google.com/sql/docs/mysql/replication/configure-external-replica)
2. [https://cloud.google.com/sql/docs/mysql/replication/replication-from-external](https://cloud.google.com/sql/docs/mysql/replication/replication-from-external) [3], [4], [5]
3. [https://cloud.google.com/sql/docs/mysql/replication](https://cloud.google.com/sql/docs/mysql/replication) [1]
4. [https://cloud.google.com/sql/docs/mysql/import-export/exporting](https://cloud.google.com/sql/docs/mysql/import-export/exporting)
5. [https://dzone.com/articles/mysql-replication-errant-transactions-in-gtid-base](https://dzone.com/articles/mysql-replication-errant-transactions-in-gtid-base) [2]

Post image by [Kelly Sikkema](https://unsplash.com/@kellysikkema?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/note?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

