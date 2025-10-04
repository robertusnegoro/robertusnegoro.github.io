---
layout: post
title: "Belajar Setup AWS Application Load Balancer dengan AWS CLI"
date: 2025-10-04 10:00:00 +0700
tags:
- aws
- alb
- load-balancer
- cli
- infrastructure
- networking
- learning
---

![AWS Load Balancer](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?ixlib=rb-4.0.3&auto=format&fit=crop&w=1200&q=80)

Baru-baru ini saya memutuskan untuk belajar lebih dalam tentang AWS Application Load Balancer (ALB) dengan mencoba setup dari nol menggunakan AWS CLI. Meskipun sudah familiar dengan konsep load balancing, ternyata ada banyak detail teknis yang perlu dipahami ketika melakukan setup manual via command line.

Pengalaman ini memberikan insight yang cukup menarik tentang bagaimana AWS mengelola infrastruktur load balancing di balik layar, dan juga beberapa gotcha yang tidak terduga.

## Persiapan Environment

Sebelum memulai eksperimen, saya perlu memastikan beberapa hal:

**1. AWS CLI sudah terinstall**
Saya cek dulu versi yang terinstall:
```bash
aws --version
```

**2. AWS credentials sudah dikonfigurasi**
```bash
aws configure
```

Saya menggunakan region us-east-1 dan output format json untuk konsistensi.

**3. Infrastructure yang sudah ada:**
- VPC ID (vpc-xxxxxxxxxxxxxxxxx)
- Dua subnet dalam VPC yang sama
- Security group yang mengizinkan inbound HTTP (port 80)
- Dua instance EC2 yang menjalankan web server (Apache)

Saya sengaja menggunakan environment yang sudah ada untuk meminimalisir biaya eksperimen.

## Membuat Load Balancer

Langkah pertama adalah membuat Application Load Balancer. Saya memilih ALB karena lebih modern dibandingkan Classic Load Balancer dan mendukung advanced routing features.

```bash
aws elbv2 create-load-balancer \
  --name my-LB \
  --subnets subnet-aaaa1111 subnet-bbbb2222 \
  --security-groups sg-cccc3333
```

Yang menarik, command ini langsung menghasilkan output JSON dengan LoadBalancerArn:
```
arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-LB/abcd1234efgh5678
```

Saya menyimpan ARN ini karena akan digunakan di hampir semua command selanjutnya. Ini adalah pattern yang konsisten di AWS CLI - setiap resource yang dibuat akan menghasilkan ARN yang unik.

## Membuat Target Groups

Konsep target group ini cukup baru bagi saya. Ternyata ALB tidak langsung mengarahkan traffic ke instance, tapi melalui target group dulu. Ini memberikan fleksibilitas untuk mengelompokkan instance berdasarkan kriteria tertentu.

Saya membuat dua target group untuk eksperimen:

```bash
aws elbv2 create-target-group \
  --name TG1 \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-xxxxxxxxxxxxxxxxx

aws elbv2 create-target-group \
  --name TG2 \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-xxxxxxxxxxxxxxxxx
```

Setiap target group menghasilkan ARN yang unik:
```
arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/TG1/aaaa1111bbbb2222
arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/TG2/cccc3333dddd4444
```

Saya mulai memahami mengapa AWS menggunakan ARN - ini memungkinkan referensi yang unik dan konsisten di seluruh layanan.

## Registrasi EC2 Instances ke Target Groups

Ini adalah bagian yang cukup tricky. Saya perlu mendaftarkan instance EC2 ke target group yang sesuai. Awalnya saya bingung, kenapa tidak bisa langsung assign instance ke load balancer?

Ternyata konsepnya adalah: Load Balancer → Target Group → EC2 Instances

```bash
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/TG1/aaaa1111bbbb2222 \
  --targets Id=i-1111aaaa2222bbbb

aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/TG2/cccc3333dddd4444 \
  --targets Id=i-3333cccc4444dddd
```

Sekarang TG1 memiliki instance1, TG2 memiliki instance2. Saya mulai melihat pola arsitektur AWS yang lebih modular dibandingkan solusi tradisional.

## Membuat Listener untuk Load Balancer

Ini adalah bagian yang paling membingungkan bagi saya. Load balancer tanpa listener tidak akan menerima traffic sama sekali! Listener adalah "telinga" yang mendengarkan request masuk.

```bash
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-LB/abcd1234efgh5678 \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/TG1/aaaa1111bbbb2222
```

Saya set TG1 sebagai default action, artinya semua traffic yang tidak match dengan rule lain akan pergi ke TG1. Ini adalah konsep "catch-all" yang cukup berguna.

## Membuat Rules untuk Path-Based Routing

Ini adalah fitur yang paling menarik bagi saya! ALB bisa melakukan routing berdasarkan path pattern. Saya ingin mencoba skenario:

- `/images/*` → TG1 (untuk static assets)
- `/work/*` → TG2 (untuk aplikasi work)

```bash
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:listener/app/my-LB/abcd1234efgh5678/xyz9876 \
  --priority 10 \
  --conditions Field=path-pattern,Values='/images/*' \
  --actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/TG1/aaaa1111bbbb2222

aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:listener/app/my-LB/abcd1234efgh5678/xyz9876 \
  --priority 5 \
  --conditions Field=path-pattern,Values='/work/*' \
  --actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/TG2/cccc3333dddd4444
```

Yang menarik, priority 5 akan dievaluasi sebelum priority 10. Saya sempat bingung dengan urutan ini, tapi ternyata AWS menggunakan sistem priority yang lebih rendah = lebih tinggi prioritasnya.

## Verifikasi Target Health

Setelah semua setup, saya penasaran apakah instance saya benar-benar sehat dan siap menerima traffic. ALB punya health check mechanism yang otomatis.

```bash
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/TG1/aaaa1111bbbb2222

aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/TG2/cccc3333dddd4444
```

Ketika saya melihat `"state": "healthy"`, rasanya lega! Ini berarti instance saya benar-benar siap dan ALB akan mengirim traffic ke sana. Health check ini adalah fitur yang sangat penting untuk production environment.

## Testing Load Balancer

Moment of truth! Saya buka DNS name dari load balancer (bisa ditemukan di output create-load-balancer command) dan mulai testing:

- `http://<LB-DNS>/images/` → seharusnya pergi ke TG1 backend
- `http://<LB-DNS>/work/` → seharusnya pergi ke TG2 backend

Dan ternyata... berhasil! Saya bisa melihat response yang berbeda dari kedua backend. Ini adalah momen yang cukup memuaskan setelah setup yang cukup panjang.

Yang menarik, ALB juga memberikan response header `X-Amzn-Trace-Id` yang bisa digunakan untuk tracing request flow.

## Cleanup Resources

Setelah eksperimen selesai, saya harus cleanup semua resources agar tidak kena charge. Ini adalah bagian yang tidak kalah penting!

**Hapus Rules:**
```bash
aws elbv2 delete-rule \
  --rule-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:listener-rule/app/my-LB/abcd1234efgh5678/xyz9876/priority10

aws elbv2 delete-rule \
  --rule-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:listener-rule/app/my-LB/abcd1234efgh5678/xyz9876/priority5
```

**Hapus Listener:**
```bash
aws elbv2 delete-listener \
  --listener-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:listener/app/my-LB/abcd1234efgh5678/xyz9876
```

**Hapus Load Balancer:**
```bash
aws elbv2 delete-load-balancer \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-LB/abcd1234efgh5678
```

Saya harus menunggu sampai load balancer benar-benar terhapus (ini memakan waktu 1-2 menit). AWS tidak mengizinkan penghapusan target group jika masih ada dependency.

**Hapus Target Groups:**
```bash
aws elbv2 delete-target-group \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/TG1/aaaa1111bbbb2222

aws elbv2 delete-target-group \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/TG2/cccc3333dddd4444
```

Environment ALB saya sekarang sudah bersih dan tidak akan kena charge lagi.

## Refleksi dan Pembelajaran

Eksperimen ini memberikan beberapa insight menarik:

1. **Modularity**: AWS ALB menggunakan arsitektur yang sangat modular dengan komponen yang saling terpisah (Load Balancer → Listener → Rules → Target Groups → Instances)

2. **ARN System**: Setiap resource memiliki ARN yang unik, memungkinkan referensi yang konsisten di seluruh AWS ecosystem

3. **Health Checks**: ALB secara otomatis melakukan health check dan hanya mengirim traffic ke instance yang sehat

4. **Path-based Routing**: Fitur ini sangat powerful untuk microservices architecture

5. **Cost Awareness**: Cleanup yang proper sangat penting untuk menghindari unexpected charges

Yang paling mengejutkan adalah betapa kompleksnya setup yang terlihat sederhana di AWS Console. Melakukan setup via CLI memberikan pemahaman yang lebih dalam tentang bagaimana AWS mengelola infrastruktur di balik layar.