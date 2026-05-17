```markdown
# 🚀 EPrints 3.4 (Core Zero) Docker Setup for Windows: From Zero to Hero

Repository ini menyediakan panduan lengkap, berkas konfigurasi, dan langkah-langkah penyelesaian masalah (*troubleshooting*) untuk menjalankan **EPrints 3.4 Core Zero** di atas **Docker Desktop Windows**. Seluruh konfigurasi di bawah ini telah dioptimalkan untuk mengatasi masalah izin akses (*permission denied*), dependensi Perl yang hilang, serta masalah *cookie session* pada browser modern.

---

## ⚠️ PENTING: Peringatan Versi EPrints
Sebelum memulai instalasi, pastikan Anda **selalu mengunduh source code EPrints versi terbaru** langsung dari repositori resmi EPrints atau situs resminya. Menggunakan versi yang terlalu usang dapat menyebabkan kerentanan keamanan dan ketidakcocokan dengan modul Perl modern pada kontainer Linux.

---

## 📋 Prasyarat Sistem
1. **Docker Desktop for Windows** telah terpasang dan berjalan (menggunakan WSL 2 backend sangat direkomendasikan).
   * 🌐 Unduh installer resmi di sini: [Docker Desktop Windows Installation](https://docs.docker.com/desktop/setup/install/windows-install/)
2. Ekstrak seluruh *source code* tarball EPrints Core Zero Anda langsung di dalam folder kerja utama, misalnya: `C:\Eprints-Docker\`. 
   * *Catatan: Pastikan folder seperti `bin`, `cfg`, dan `cgi` berada langsung di root folder kerja tersebut (bukan terbungkus di dalam sub-folder lagi).*

---

## 📂 Struktur Direktori Proyek
Pastikan struktur folder di laptop Anda terlihat seperti ini sebelum melakukan build:
```text
C:\Eprints-Docker\
├── bin/
├── cfg/
├── cgi/
├── perl_lib/
├── ... (file bawaan EPrints lainnya)
├── Dockerfile
└── docker-compose.yml

```

---

## 🛠️ Berkas Konfigurasi Utama

### 1. `Dockerfile`

Buat berkas bernama `Dockerfile` (tanpa ekstensi) di dalam folder `C:\Eprints-Docker\` dan masukkan kode bersih berikut:

```dockerfile
FROM ubuntu:20.04

# Mencegah pop-up interaktif selama instalasi paket Ubuntu
ENV DEBIAN_FRONTEND=noninteractive

# 1. Install Apache, Perl, dan seluruh dependensi wajib EPrints 3.4
RUN apt-get update && apt-get install -y \
    apache2 \
    libapache2-mod-perl2 \
    perl \
    libdbd-mysql-perl \
    libxml-libxml-perl \
    libxml-xslt-perl \
    libio-string-perl \
    libcgi-pm-perl \
    libjson-perl \
    liburi-perl \
    libmime-lite-perl \
    libtext-unidecode-perl \
    mysql-client \
    && rm -rf /var/lib/apt/lists/*

# 2. Atur ServerName global agar Apache tidak memunculkan warning/error
RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf

# 3. Buat folder tujuan sesuai dengan jalur bawaan EPrints
RUN mkdir -p /opt/eprints3

# 4. Buat user dan grup khusus 'eprints' demi standar keamanan
RUN groupadd eprints && useradd -g eprints -s /bin/bash -d /opt/eprints3 eprints

# 5. Salin semua source code EPrints dari laptop ke folder /opt/eprints3 di kontainer
COPY . /opt/eprints3

# 6. Berikan hak akses penuh folder /opt/eprints3 ke user eprints
RUN chown -R eprints:eprints /opt/eprints3

# Expose port web standar
EXPOSE 80

# Jalankan Apache secara terus-menerus di foreground
CMD ["apachectl", "-D", "FOREGROUND"]

```

### 2. `docker-compose.yml`

Buat berkas bernama `docker-compose.yml` di folder yang sama, lalu masukkan konfigurasi multi-kontainer berikut:

```yaml
services:
  eprints:
    build: .
    container_name: eprints_app
    ports:
      - "80:80"
    environment:
      - EPRINTS_HOSTNAME=myrepo.local
    depends_on:
      - db

  db:
    image: mysql:5.7
    container_name: eprints_db
    environment:
      MYSQL_ROOT_PASSWORD: admin123
      MYSQL_DATABASE: eprints
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:

```

---

## 🚀 Langkah Demi Langkah Instalasi (0 sampai Z)

### Langkah 1: Build dan Jalankan Kontainer

Buka **Command Prompt (CMD)**, masuk ke direktori proyek, lalu rakit kontainer aplikasinya:

```cmd
cd C:\Eprints-Docker
docker compose up --build -d

```

*Pastikan kedua kontainer (`eprints_app` dan `eprints_db`) berstatus `Started` / `Running`.*

### Langkah 2: Jalankan Wizard Konfigurasi EPrints

Eksekusi perintah interaktif pembuat repositori dengan menggunakan identitas user `eprints` (bukan root):

```cmd
docker exec -u eprints -it eprints_app /opt/eprints3/bin/epadmin create zero

```

### Langkah 3: Panduan Pengisian Layar (Wizard Isian)

Isi pertanyaan interaktif yang muncul di layar terminal Anda dengan panduan berikut:

1. **Archive ID?** Ketik **`myrepo`** lalu Enter.
2. **Configure vital settings? [yes] ?** Langsung tekan **Enter**.
3. **Host Name?** Ketik **`myrepo.local`** lalu Enter. *(Jangan gunakan `localhost` agar tidak diblokir oleh fitur session cookie pada browser modern).*
4. **HTTP Port? [80] ?** Langsung tekan **Enter**.
5. **Alias (enter # when done) [#] ?** Langsung tekan **Enter** untuk melewati.
6. **Path? [/] ?** Langsung tekan **Enter**.
7. **HTTPS Hostname [] ?** Langsung tekan **Enter** (kosongkan).
8. **Administrator Email?** Masukkan email Anda (misal: `admin@example.ac.id`) lalu Enter.
9. **Archive Name?** Masukkan nama repositori Anda (misal: `Repo Jurnal Utama`) lalu Enter.
10. **Organisation Name?** Masukkan nama institusi/kampus Anda lalu Enter.
11. **Write these core settings? [yes] ?** Langsung tekan **Enter**.
12. **Configure database? [yes] ?** Langsung tekan **Enter**.
13. **Database Name [myrepo] ?** Langsung tekan **Enter**.
14. **MySQL Host [localhost] ?** ⚠️ **WAJIB KETIK: `db**` lalu Enter (mengarahkan ke kontainer database).
15. **MySQL Port & Socket?** Langsung tekan **Enter** pada kedua pertanyaan tersebut untuk memilih default.
16. **Database User [myrepo] ?** Langsung tekan **Enter**.
17. **Database Password?** Langsung tekan **Enter** untuk menggunakan password acak bawaan yang aman.
18. **Database Engine [InnoDB] ?** Langsung tekan **Enter**.
19. **Write these database settings? [yes] ?** Langsung tekan **Enter**.
20. **Create database "myrepo" [yes] ?** Langsung tekan **Enter**.
21. **Database Superuser Username [root] ?** Langsung tekan **Enter**.
22. **Database Superuser Password?** Ketik secara buta **`admin123`** lalu Enter. *(Karakter tidak akan muncul di layar saat diketik, ini adalah fitur keamanan normal).*
23. **Create database tables? [yes] ?** Langsung tekan **Enter**.
24. **Create an initial user? [yes] ?** Langsung tekan **Enter** untuk membuat user login web pertama Anda.
* **Username:** Ketik **`admin`** lalu Enter.
* **User type:** Ketik **`admin`** lalu Enter.
* **Password:** Ketik secara buta **`admin123`** (atau password pilihan Anda) lalu Enter.
* **Email:** Masukkan email admin lalu Enter.


25. **Do you want to build the static web pages? [yes] ?** Langsung tekan **Enter**.
26. **Do you want to update the apache config files? [yes] ?** Langsung tekan **Enter**.

### Langkah 4: Hubungkan EPrints ke Web Server Apache

Setelah kembali ke prompt biasa (`C:\Eprints-Docker>`), jalankan rentetan perintah ini untuk mengaktifkan konfigurasi EPrints ke Apache dan mematikan halaman default Ubuntu:

```cmd
docker exec eprints_app sh -c "echo 'Include /opt/eprints3/cfg/apache.conf' >> /etc/apache2/apache2.conf"
docker exec eprints_app a2dissite 000-default
docker exec eprints_app apachectl graceful

```

### Langkah 5: Daftarkan Domain Lokal di Windows

Karena kita menggunakan hostname `myrepo.local`, Windows perlu diarahkan agar mengenali domain tersebut secara lokal:

1. Cari **Notepad** di Windows Start Menu, klik kanan lalu pilih **Run as Administrator**.
2. Tekan **Ctrl + O**, pada kolom *File name*, tempel jalur ini lalu Enter:
```text
C:\Windows\System32\drivers\etc\hosts

```


3. Gulir ke baris paling bawah berkas, tambahkan konfigurasi berikut:
```text
127.0.0.1    myrepo.local

```


4. Simpan (**Ctrl + S**) dan tutup Notepad.

---

## 🛠️ Buku Saku Penyelesaian Masalah (Troubleshooting Log)

### 🚨 Masalah 1: Kontainer Tiba-Tiba Mati (*Crash/Not Running*) Sehabis Build

* **Penyebab:** Menyertakan baris perintah `Include /opt/eprints3/cfg/apache.conf` di dalam `Dockerfile` secara prematur sebelum perintah `epadmin create` dijalankan. Apache akan langsung mati karena mencari berkas konfigurasi repositori yang belum terbentuk.
* **Solusi:** Hapus baris *Include* dari `Dockerfile` dan pindahkan proses eksekusinya sebagai instruksi pasca-instalasi (*post-installation steps*) lewat CMD setelah repositori sukses dikonfigurasi.

### 🚨 Masalah 2: Pesan Error `Permission denied` Saat Pembuatan Dokumen Web / `auto.js`

* **Penyebab:** Proses pembuatan repositori dijalankan menggunakan user `eprints`, namun web server Apache berjalan di bawah kepemilikan user bawaan Linux bernama `www-data`. Apache tidak memiliki izin untuk menulis dokumen baru ke folder arsip milik user `eprints`.
* **Solusi:** Longgarkan izin akses folder archives agar dapat diakses bersama secara lokal dengan mengeksekusi perintah berikut di CMD:
```cmd
docker exec eprints_app chmod -R 777 /opt/eprints3/archives
docker exec eprints_app apachectl graceful

```



### 🚨 Masalah 3: Tampilan Website Rusak Hancur / Tanpa Desain (Tampilan "Majapahit")

* **Penyebab:** EPrints gagal memuat aset berkas CSS (*style*) dan JavaScript karena proses *generate* yang sempat terinterupsi atau terhambat masalah perizinan folder sebelumnya.
* **Solusi:** Paksa sistem EPrints untuk memproduksi ulang seluruh aset halaman statis, muat ulang Apache, kemudian lakukan pembersihan memori cache pada browser:
1. Jalankan perintah kompilasi aset statis di CMD:
```cmd
docker exec -u eprints -it eprints_app /opt/eprints3/bin/generate_static myrepo
docker exec eprints_app apachectl graceful

```


2. Buka browser pada alamat `http://myrepo.local`, lalu lakukan pembersihan cache mendalam dengan menekan tombol **Ctrl + F5** (atau **Cmd + Shift + R** pada Mac).



---

## 🎉 Selesai

Buka browser Anda dan akses halaman utama melalui alamat:
👉 **`http://myrepo.local`**

Gunakan akun administrator (**Username:** `admin` | **Password:** `admin123`) untuk masuk ke dalam ruang kerja repositori digital Anda.

```

```
