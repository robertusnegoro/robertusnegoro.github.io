---
layout: post
title: Ekspansi Prometheus Storage dengan TimescaleDB
date: '2019-08-29 07:07:55'
tags:
- postgresql
- timescaledb
- prometheus
- grafana
- monitoring
- devops
- sre
- infrastructure
---

> Prometheus adalah monitoring tools yang hampir tidak pernah saya lihat, sampai suatu ketika saya harus bersinggungan dengan Kubernetes.

Prometheus, secara "mutlak" dipakai oleh Kubernetes dan lingkungannya, termasuk Istio yang masyhur itu, sebagai monitoring tools atau sederhananya pengepul angka dari metrics perkakas - perkakas kubernetes. Dalam konfigurasi yang ringkas, maka prometheus akan dipasang sebagai sidecar dari pod atau sebagai statefulset yang mengambil metrics dari pod, Istio atau metrics yang diekspose oleh exporter. Misalkan redis exporter.

Jika di zoom ke bagian monitoring saja, maka rute nya akan jadi begini :

<!--kg-card-begin: markdown-->

    Exporter / Istio Telemetry -> Prometheus -> (Optional) Stackdriver
                                             -> Grafana

<!--kg-card-end: markdown-->

Pada prakteknya, prometheus sering kewalahan dipakai sebagai database timeseries baik ingest dari metrics source maupun untuk diplot dalam bentuk grafik di kakas semacam Grafana. Prometheus akan menggunakan sumber daya, seperti beban memori yang terlalu besar. Dan penumpukan data pada file yang diambil dari persistent disk juga membuat Prometheus semakin hari semakin bloated, gemuk - tanpa nada atau niatan body shaming. Sifatnya yang cenderung stateful ini membuat Prometheus sulit untuk di-scale. Adapun solusi scaling up Prometheus adalah dengan dua solusi, yang pertama adalah mode federated. Yang artinya akan ada lebih dari 1 prometheus dengan konsep DC dan sebagainya, namun kelemahan mode ini adalah tingkat kerumitan dalam pengaturan distribusi node prometheus dan ketika di-query tetap menuju ke 1 node prometheus, maka kebuntuan penggunaan sumber daya memori tetap terjadi. Karena tetap saja tidak terdistribusi pembagian bebannya. Solusi ke-dua adalah dengan cara sharding, alias membagi job scrape ke dalam beberapa prometheus yang berdiri secara terpisah.

Dalam permasalahan tersebut, akhirnya muncul beberapa solusi untuk mengatasi kebuntuan sumber daya, misalkan yang paling populer saat tulisan ini dibuat adalah [Thanos](https://thanos.io). Thanos menjanjikan pemetaan fungsi yang tadinya dibebankan semua pada prometheus menjadi lebih terbagi-bagi berdasarkan fungsinya. Thanos mempunyai thanos sidecar sebagai shipper, ada thanos query yang nanti akan menerima user request, ada thanos compactor untuk memelihara file yang dihantarkan shipper ke penyimpanan pihak ketiga (third party storage, seperti GCS, S3 atau Ceph backed storage). Dengan banyaknya komponen ini, Thanos mempunyai kelemahan yaitu tingkat kerumitan yang cukup tinggi. Belum tentu juga menjamin komponen Thanos Query ini tidak mengalami kebuntuan dari segi penggunaan memori.

Untuk itu saya berpikir sedikit lebih sederhana. Mengapa tidak menghantarkan data dari Prometheus ke penyimpanan secara remote (remote write dan remote read), kemudian Grafana tinggal melakukan query ke penyimpanan tersebut. Penyimpanan di sini diartikan sebagai basis data (database), yang didukung oleh Grafana sebagai datasource, misalkan InfluxDB atau PostgreSQL.

Pilihan percobaan jatuh kepada PostgreSQL gara - gara saya hanya menginginkan teknologi yang lebih familiar buat saya pribadi. Namun ternyata PostgreSQL tidak dapat berdiri sendiri, jika kita ingin menjadikannya sebagai timeseries database. PostgreSQL butuh plugin Timescale yang menjadikan kombinasi ini sebagai TimescaleDB.

Tulisan ini tidak akan menjelaskan langkah perlangkah untuk mengonfigurasi sebuah server atau banyak server menjadi tempat penyimpanan metrics dari Prometheus. Karena langkah tersebut dapat ditemukan di sini : " **Getting started with Prometheus and TimescaleDB**" ([https://docs.timescale.com/v0.10/tutorials/prometheus-adapter](https://docs.timescale.com/v0.10/tutorials/prometheus-adapter)). Saya hanya akan membahasnya secara konseptual dan membagikan pengalaman saat mengimplementasikannya.

Dengan menggunakan TimescaleDB, maka rutenya menjadi seperti ini :

<!--kg-card-begin: markdown-->

    Exporter / Istio Telemetry -> Prometheus -> Prometheus PostgreSQL Adapter -> PostgreSQL server + Timescale + Pg_prometheus -> Grafana

<!--kg-card-end: markdown-->

Jika dibandingkan dengan Thanos, saya rasa solusi ini lebih sederhana dan masuk akal jika ditilik dari segi kesederhanaan. Hal ini sedikit berbeda dengan Thanos yang menyebutkan bahwa solusi mereka didesain dengan konsep KISS (Keep It Simple, Stupid), menurut saya sih Thanos tidak simple.

Pada solusi TimescaleDB sebagai "long term database for Prometheus" ini, maka kita perlu mempersiapkan (selain Prometheus dan Grafana):

1. PostgreSQL  
PostgreSQL yang dipersiapkan bisa pakai versi stabil terakhir (saat ini 11) dengan tambahan library timescaledb dan Pg\_prometheus.
2. Prometheus PostgreSQL adapter  
Adapter yang akan berbicara di protokol http dan menerima ingest dari Prometheus

Kedua hal di atas dapat disatukan pada satu server atau dipisah. Sesederhana bumbu kacang pada sate, dipisah atau dicampur. Pada prometheus akan ditambahkan baris konfigurasi untuk melakukan remote write dan remote read ke Prometheus PostgreSQL Adapter.

Saya kira dengan diaktifkannya remote write, prometheus akan berhenti menulis ke penyimpanan permanennya, namun ternyata tidak. Prometheus tetap menulis file TSDB yang dipakainya sebagai simpanan lokal. Nah, berarti solusi ini tidak akan berdampak pada membeludaknya ukuran database di Prometheus. Namun setidaknya dapat mencapai tujuan untuk menyelamatkan data prometheus jika memang terjadi apa-apa terhadap Prometheus-nya.

Yang perlu diperhatikan juga adalah penggunaan query timeseries pada grafana yang pada praktiknya adalah query PostgreSQL dengan tambahan makro time series atau time bucket di sana. Query tidak sesederhana PromQL yang dengan mudahnya melakukan perhitungan aritmatika hanya dengan 1 line query. Bahkan sampai tulisan ini saya buat, saya belum dapat menemukan semua pengganti sepadan syntax PromQL misalkan irate. Namun hal ini dapat diatasi dengan riset pada metrics tertentu yang membutuhkan query yang (mungkin) kompleks. Dengan 'kemudahan' dan 'popularitas' SQL, seharusnya ini tidak menjadi masalah di kemudian hari.

Referensi :

1. [https://blog.timescale.com/blog/sql-nosql-data-storage-for-prometheus-devops-monitoring-postgresql-timescaledb-time-series-3cde27fd1e07/](https://blog.timescale.com/blog/sql-nosql-data-storage-for-prometheus-devops-monitoring-postgresql-timescaledb-time-series-3cde27fd1e07/)
2. [https://docs.timescale.com/v0.10/tutorials/prometheus-adapter](https://docs.timescale.com/v0.10/tutorials/prometheus-adapter)
