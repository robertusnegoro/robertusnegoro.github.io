---
layout: post
title: 'Catatan Kafka : Replikasi'
date: '2020-04-06 09:39:27'
tags:
- kafka
- dokumentasi
- infrastructure
---

Apache Kafka adalah salah satu sistem yang sering dipergunakan untuk membangun data pipeline dan / atau sistem streaming data. Saya sengaja mencatat beberapa bagian penting sebagai sarana untuk mempelajari dan mulai familiar dengan sistem ini.

Salah satu bagian penting dari Kafka adalah Sistem **Replikasi**. Sistem replikasi pada Kafka adalah bagian penting untuk menjamin durabilitas dan reliabilitas dari Kafka itu sendiri. Setiap topic dan partition pada Kafka direplikasi di 2 jenis replika : **Leader Replica, Follower Replica**. Kafka menggunakan Leader Replica untuk menjamin konsistensi pada setiap replica partitions karena leader replica (atau Leader saja), akan menerima request producer dan consumer serta memastikan para follower terus ter-update terhadap leader.

Pada skema produce, maka ada 2 macam request berkaitan dengan produce ini : Produce request dan Fetch Request. Produce request adalah request dari client berupa message untuk di "antri" kan ke kafka brokers, sedangkan fetch request, adalah request consume dan produce dari followers kepada leader.Dari sisi client, set konfigurasi acks untuk memastikan konsistensi. Ack bisa 1, -1 atau 0. 1, semua atau fire and forget. Pada beberapa blog berisikan benchmark, sering digunakan ack 0, yang mana pastinya akan menghasilkan grafik yang "indah" pula.

Bicara soal replikasi di kafka, ada istilah ISR atau In-Sync Replica, istilah untuk kafka brokers yang mempunyai data yang sama pada sebuah partisi topik tertentu. Misalkan pada sebuah topik ditetapkan replication factornya 3, maka akan ada 1 leader replica dan 2 follower replica yang mempunyai data yang sama per partisi pada topik tersebut.Jumlah minimal in-sync replica dapat dikonfigurasi untuk menentukan seberapa reliable cluster kafka tersebut. Secara default, min-insync-replica adalah 1, artinya dengan replication factor 3, kita masih dapat melakukan consume dan produce jika ada 2 node yang mati pada partisi tersebut. Hal ini diluar pembahasan konsistensi dan hubungannya dengan ack.

Replica dapat dikatakan dalam status "in-sync" jika memenuhi 3 hal ini :   
1. tetap mempunyai sesi aktif ke zookeeper. broker tetap mengirimkan signal heartbeat ke zookeeper dalam waktu 6 detik (dapat dikonfigurasi pada zookeeper.session.timeout.ms )  
2. melakukan fetch request yang berhasil ke leader dalam kurun waktu 10 detik ( tergantung konfigurasi replica.lag.time.max.ms)  
3. menerima message terakhir dalam rentang 10 detik (poin nomor 2) dari leader.

> Dalam skema ack all, maka sebuah request produce baru dikatakan sukses (committed), jika semua replica telah mengkonfirmasi telah menerima message. Untuk lebih jelasnya dapat dilihat pada thread ini di situs stackoverflow : [https://stackoverflow.com/questions/50689177/kafka-ack-all-and-min-isr](https://stackoverflow.com/questions/50689177/kafka-ack-all-and-min-isr)

Konfigurasi semacam replication factor, min insync replica akan menentukan seberapa reliable sebuah kafka cluster. Dengan replication factor yang semakin banyak, tentu meningkatkan availability dari kafka tersebut, semakin lengkap jika ack nya di set ke all, sehingga memastikan semua replika mendapatkan message tersebut. Namun konfigurasi ini juga akan berdampak pada kecepatan / latency request yang semakin meningkat. Berikut konfigurasi broker yang berimplikasi terhadap replikasi dan hubungannya dengan reliabilitas serta kecepatan :

### Replication Factor

Pada kafka, sebuah topic dapat dibagi menjadi beberapa partition. dan replikasi terjadi di level partition bukan di topic nya. Jika partition di atur pada parameter num.partitions, maka untuk replication per partition dikenal dengan istilah replication factor, sebuah istilah yang kerap digunakan pada database yang mempunyai sistem replikasi misalkan Cassandra.Replication Factor menentukan seberapa banyak sebuah partition pada topic direplikasi. Misalkan pada topic bernama topic\_a , mempunyai 3 partisi. Dan replication factor di set ke angka 3, itu artinya topic\_a - partition 0 akan disimpan di 3 broker yang berbeda, begitu juga partition 1 dan seterusnya. Sebuah replication factor X, akan mengijinkan cluster kafka untuk kehilangan X-1 brokers dan tetap dapat menerima request produce dan consume (read / write). Selain memastika setiap replica partition ada di broker yang berbeda, sebaiknya juga pastikan partition tersebut disebar juga di broker yang berada di rak server yang berbeda. Hal ini dapat diterjemahkan juga menjadi multiple Availability Zone pada cloud platform.

### Unclean Leader Election

Jika sebuah leader replica kemudian tidak dapat dikontak atau mati, salah satu dari ISR akan segera menggantikannya sebagai leader replica baru. Cara pemilihan (election) ini disebut "clean" election, karena menggaransi tidak ada data yang hilang. Tapi apa yang terjadi ketika tidak ada lagi ISR yang tersisa?Partition akan tetap offline sampai replica - replica tadi kembali hidup atau online. Jika kita mengijinkan replica yang out-of-sync untuk jadi leader, maka kita akan kehilangan beberapa message yang dikirim di interval waktu saat ISR tadi bertumbangan.Mengeset unclean.leader.election.enable menjadi true - di mana ini adalah default value-nya, akan mengijinkan out-of-sync replica menjadi leader (unclean election), namun beresiko pada kehilangan message seperti yang dipaparkan sebelumnya.Namun tanpa adanya konfigurasi ini (set ke false), maka availability jadi lebih rendah karena kita harus menunggu leader replica lama kembali online. Sebaliknya jika kita membahas kualitas data dan konsistensi, maka ada baiknya mengubah konfigurasi unclean leader election ini menjadi false.

### Min In-Sync Replica

Pada contoh di atas, kita menetapkan RF (Replication Factor) menjadi 3, maka ada kemungkinan kita hanya mempunyai 1 ISR di cluster. Kita bisa saja menggunakan ack all namun tetap saja akan menunggu sisa ISR untuk kembali online sehingga produce yang kita kirimkan dapat di commit. Pada kasus ini, kita diminta untuk memilih consistency dan availability. Jika kita mengeset min.insync.replicas menjadi 2, maka kita hanya dapat menulis ke topic partition tersebut jika 2 dari 3 replica sedang online. Artinya jika ada 1 mesin mati, kita masih bisa menuliskan message. Namun jika 2 mesin mati / offline, maka yang terjadi client / producer tidak dapat mengirimkan produce request (mendapat error _NotEnoughReplicasException_ ).

