---
layout: post
title: Flash Lineage OS ke Oneplus 3 dengan Ubuntu 18.04 Desktop LTS
date: '2019-04-22 17:21:03'
tags:
- android
- lineageos
- custom-rom
- iseng
- oneplus
---

[Do it with your own risk.] Berikut ini dokumentasi yang saya lakukan saat flashing Lineage OS ke Oneplus 3. Mungkin juga berlaku untuk Oneplus 3T. Hal ini worth untuk dilakukan mengingat Oneplus 3 sudah di ambang masa obsolete support nya untuk mendapatkan update Oxygen OS.

Yang perlu dipersiapkan :

1. Ubuntu 18.04 Desktop LTS
2. Lineage OS yang dapat diunduh di sini : [https://download.lineageos.org/oneplus3](https://download.lineageos.org/oneplus3)
3. TWRP image dari sini : [https://dl.twrp.me/oneplus3/twrp-3.3.0-1-oneplus3.img.html](https://dl.twrp.me/oneplus3/twrp-3.3.0-1-oneplus3.img.html)
4. GApps. kecuali seorang pemboikot Google :) dapat diperoleh dari sini : [https://opengapps.org/](https://opengapps.org/)

### Persiapan

Sebelum semuanya dilakukan, backup data penting ke cloud. Skenario yang saya lakukan adalah dengan mem-format internal storage tanpa menyisakan data.

Pada Ubuntu 18.04 tersebut, pasang android tools yang nanti akan dibutuhkan

    sudo apt update
    sudo apt install android-tools-adb android-tools-fastboot

### Unlock Bootloader

Pada Oneplus 3, buka Setting. Kemudian pada About Phone, tap di Build Number sampai keluar pesan semacam "You are now a Developer".

Pada Developer menu, aktifkan hal berikut:

- Enable USB Debugging
- Enable OEM Unlock / Allow Bootloader Unlock
- Enable Advance Reboot

Matikan Oneplus 3, lalu nyalakan kembali dengan menekan Volume Up + Power Button secara bersama. Setelah masuk mode fastboot, hubungkan Oneplus 3 dengan komputer dengan Ubuntu tadi.

Bukalah aplikasi Terminal atau Terminator, lalu gunakan perintah ini untuk mendeteksi apakah device sudah dikenali `fastboot devices` .

Untuk meng-unlock bootloader, gunakan perintah ini `fastboot oem unlock` pada step ini saja, data anda akan hilang.

### Flash TWRP

Kemudian flash TWRP yang tadi didapatkan dari sini : [https://dl.twrp.me/oneplus3/twrp-3.3.0-1-oneplus3.img.html](https://dl.twrp.me/oneplus3/twrp-3.3.0-1-oneplus3.img.html)

    fastboot flash recovery <namafile_TWRP_disini>.img
    fastboot boot <namafile_TWRP_disini>.img

### Hapus ini Hapus itu

[Achtung! Data akan diformat!]

Jika TWRP menanyakan password untuk decrypt, lewati saja (tekan cancel). Kemudian pada menu **Wipe** - Format Data. Sebelum dilakukan, ada baiknya dibackup sendiri datanya. Ingat, saya memang berencana untuk menghapus partisi agar tidak diributkan dengan urusan decrypt partition nantinya. Hapus Cache juga jika perlu.

### Flash ROM

Masih di TWRP, tap back dan masuk menu Advanced. Kemudian pilih ADB Sideload.

Dengan posisi Oneplus 3 terhubung ke Ubuntu, buka terminator, dan lakukan sideload 2 image file yaitu image Lineage OS yang diunduh dari sini [https://download.lineageos.org/oneplus3](https://download.lineageos.org/oneplus3) dan GApps dari sini [https://opengapps.org/](https://opengapps.org/)

    adb sideload lineage-16.0-20190420-nightly-oneplus3-signed.zip
    adb sideload open_gapps-arm64-9.0-stock-20190420.zip

setelah terpasang, jangan lupa untuk mengunci / lock boot loader dengan masuk kembali ke fastboot pada Oneplus 3 dan kirim perintah ini `fastboot oem lock`

Saya sendiri mengulang - ulang beberapa perintah di atas dan menemui permasalahan seperti ini :

- Lineage terpasang, boot, tapi tidak dapat melakukan decrypt storage (maka saya format habis saja)
- Lineage terpasang, tapi somehow stuck (maka akhirnya saya coba untuk lock kembali bootloader)

Saya pun baru tahu bahwa Android 9 punya masalah pada decrypt data yang kita encrypt di Android versi sebelumnya. [https://forum.xda-developers.com/oneplus-3/oneplus-3--3t-cross-device-development/rom-lineageos-16-0-oneplus-3-3t-t3866517](https://forum.xda-developers.com/oneplus-3/oneplus-3--3t-cross-device-development/rom-lineageos-16-0-oneplus-3-3t-t3866517)

Selamat datang Lineage!

[Update 6 April 2020]

- Untuk upgrade perlu install firmware modem baru : [https://www.androidexplained.com/oneplus-3-upgrade-modem/](https://www.androidexplained.com/oneplus-3-upgrade-modem/)
- Install firmware sebelum sideload lineageos
