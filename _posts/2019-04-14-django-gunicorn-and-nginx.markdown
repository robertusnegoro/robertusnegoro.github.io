---
layout: post
title: Django, Gunicorn and Nginx
date: '2019-04-14 15:34:25'
tags:
- django
- python
- nginx
- gunicorn
- wsgi
- tutorial
- dokumentasi
- infrastructure
---

Sebenarnya postingan pertamax dengan judul "Silo atau Limbo" itu post yang sudah ada dari jaman domain vanjava.web.id. Harusnya ini postingan pertamax yang sesungguhnya. Pertamax pertamax mulu kek masih main Kaskus aja.

Postingan kali ini saya mau mendokumentasikan bagaimana konfigurasi [Django Framework](https://docs.djangoproject.com/en/2.2/) (yang pada saat tulisan ini dibuat, versi stable terakhir adalah 2.2) di sistem Operasi Linux [Ubuntu](https://www.ubuntu.com/download) versi 18.04 LTS dengan [Gunicorn](https://gunicorn.org/) sebagai WSGI server dan [Nginx](https://docs.nginx.com/) sebagai garda terdepan.

Sebenarnya dokumentasi ini tidak hanya berlaku untuk Django Framework saja, tapi (mungkin) juga masih applicable untuk python based web framework lainnya seperti Flask, CherryPy dan lain sebagainya.

### Instalasi Django

Untuk memasang django, dapat dimulai memasang beberapa dependensi

    sudo apt update
    sudo apt install python3 python3-venv

Kemudian buatlah python3 virtual environment. Mengapa harus virtual environment? Baca ini saja. [Python Virtual environment](https://www.geeksforgeeks.org/python-virtual-environment/).

    python3 -m venv /path/to/virtualenv
    # contoh
    python3 -m venv /opt/robee-env/

Setelah virtual environment dibuat, aktifkan dan pasang django **di dalam virtual environment**

    source /path/to/virtualenv/bin/activate
    pip install django

Mengapa ditebalkan bagian "di dalam virtual environment", sekali lagi baca url ini [Python Virtual environment](https://www.geeksforgeeks.org/python-virtual-environment/).

Masih di dalam virtual environment, pastikan django terpasang dengan baik

    >>> import django
    >>> print(django.get_version())
    2.2

Dengan asumsi source code project django kita letakkan di _/path/to/mydjangoproject/_ maka :

    cd /path/to
    django-admin startproject mydjangoproject

maka nanti akan ada django project baru pada `/path/to/mydjangoproject/` bersama dengan struktur django di dalamnya. Untuk menguji apakah django sudah dapat serve :

    cd /path/to/mydjangoproject/
    python manage.py runserver 0.0.0.0:8080

kemudian cobalah untuk mengakses `http://localhost:8080` dan akan didapati lama seperti ini

<figure class="kg-card kg-image-card"><img src="/content/images/2019/04/image.png" class="kg-image" alt loading="lazy"></figure>

Untuk membuat sebuah aplikasi django dan berbagai fungsi atau fitur lainnya, silakan dilanjutkan untuk mempelajarinya di sini : [Django Framework](https://docs.djangoproject.com)

### Gunicorn

Gunicorn sendiri merupakan sebuah WSGI HTTP Server yang dapat berfungsi meneruskan client request ke python. Laman resminya bisa dipantau di sini : [https://gunicorn.org/](https://gunicorn.org/)

Mengapa harus WSGI Server?

Sebenarnya setiap python web frameworks punya built-in http server sendiri, misalkan Django dengan `manage.py runserver` atau Flask dengan `flask run` . Tapi secara general, built in http server tersebut hanya diperuntukkan pada level development saja. Sedangkan untuk production, WSGI server lebih diutamakan karena kemampuannya untuk spawn multi proses untuk kemudian diteruskan ke multi CPU, kemudian mempunyai fitur logging yang lebih baik (meski ini juga masih _debatable_), lebih dapat dioptimasi untuk performa yang lebih baik lagi. Khususnya untuk Gunicorn, WSGI HTTP Server ini juga mudah sekali dikonfigurasi.

Untuk memasang gunicorn, pastikan kita masih di python virtual environment yang tadi sudah kita buat.

    source /path/to/virtualenv/bin/activate
    pip install gunicorn

masih di virtualenv, kita juga bisa menjalankan django di atas gunicorn

    cd /path/to/mydjangoproject/
    gunicorn mydjangoproject.wsgi -b 0.0.0.0:8081
    
    [2019-04-14 22:01:54 +0700] [18174] [INFO] Starting gunicorn 19.9.0
    [2019-04-14 22:01:54 +0700] [18174] [INFO] Listening at: http://0.0.0.0:8081 (18174)
    [2019-04-14 22:01:54 +0700] [18174] [INFO] Using worker: sync
    [2019-04-14 22:01:54 +0700] [18177] [INFO] Booting worker with pid: 18177

kemudian kita dapat mengakses django yang dijalankan di atas gunicorn di http://localhost:8081

_Nah_ untuk men-_daemonize_ gunicorn pada systemd Ubuntu, maka kita bisa melakukan ini

Tulis socket file baru di sini `sudo vim /etc/systemd/system/gunicorn.socket`

    [Unit]
    Description=gunicorn socket
    
    [Socket]
    ListenStream=/run/gunicorn.sock
    
    [Install]
    WantedBy=sockets.target

Kemudian buat file untuk service nya `sudo vim /etc/systemd/system/gunicorn.service`

    [Unit]
    Description=gunicorn daemon
    Requires=gunicorn.socket
    After=network.target
    
    [Service]
    User=namauser
    Group=namavalidgroup
    WorkingDirectory=/path/to/mydjangoproject
    EnvironmentFile=/path/to/virtualenv
    ExecStart=/path/to/virtualenv/bin/gunicorn \
              --access-logfile /path/to/logfile \
              --workers 3 \
              --bind unix:/run/gunicorn.sock \
              mydjangoproject.wsgi:application
    
    [Install]
    WantedBy=multi-user.target

Untuk namauser, namavalidgroup dan path ke logfile dapat diubah sesuai dengan kondisi masing-masing.

Untuk menyalakan service gunicorn

    sudo systemctl daemon-reload
    # start gunicorn with mydjangoproject
    sudo systemctl start gunicorn.service
    # mematikan gunicorn
    sudo systemctl stop gunicorn.service
    # melihat status gunicorn
    sudo systemctl status gunicorn.service
    # merestart service gunicorn
    sudo systemctl restart gunicorn.service

### Nginx Reverse Proxy

Menggunakan Nginx (engine x, bukan _nging_) sebagai reverse proxy, bukan hanya untuk reverse proxy saja. Namun karena beberapa kebutuhan seperti terminasi SSL Certificate, menerapkan limit burst, melakukan load balancing jika memiliki lebih dari 1 backend gunicorn dan fitur nginx lainnya. Dalam skala yang lebih besar, tentu saja nginx saja tidak cukup. Jika diperlukan application load balancer seperti AWS ELB atau ALB atau HTTPS Proxy di GCP dan lain sebagainya, maka hal tersebut juga bisa jadi pilihan. Terutama jika membicarakan sistem microservice yang kompleks, maka solusinya bisa saja menggunakan Envoy proxy, Finagle, Proxygen atau Linkerd. Tapi kali ini mengesampingkan kompleksitas itu, penulis hanya ingin menggunakan nginx saja.

Pemasangan dan konfigurasi dasar nginx sudah banyak dibahas di lain tempat. Maka pada bagian ini penulis hanya berfokus pada vhost config yang menerapkan konsep reverse proxy dari nginx ke gunicorn. Maka buat file vhost baru `sudo vim /etc/nginx/sites-available/mydjangoproject`

    server {
        listen 80;
        server_name namaserver;
    
        location = /favicon.ico { access_log off; log_not_found off; }
        location / {
            include proxy_params;
            proxy_pass http://unix:/run/gunicorn.sock;
        }
    }

Demi menghindari log favicon yang deras, biasanya saya pribadi membuang log akses favicon. Untuk namaserver dapat diganti dengan IP server kita atau nama domain yang diarahkan record-nya ke IP server kita.

Konfigurasi sudah selesai, dan django dapat bekerja di port 80 dan diserve oleh Nginx yang melakukan reverse proxy terhadap gunicorn. Pada kesempatan lain akan dibahas juga bagaimana pengelolaan file media upload, asset static pada Django Framework (baca: _jika ada waktu_).

Good bye now, be nice!

