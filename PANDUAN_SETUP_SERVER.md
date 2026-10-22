# 🚀 Panduan Setup Lokal & Deploy Server VR TI FTS UIKA Bogor

Dokumentasi lengkap untuk menjalankan aplikasi secara lokal dan persiapan deployment ke server **`vr.ti.fts.uika-bogor.ac.id`**.

---

## 📂 Struktur Target di Server (`/var/www`)

Di server `root@server-vr-ti:/var/www#`, struktur folder yang ideal adalah:

```text
/var/www/
├── html/                     <-- Berisi index.html gerbang awal (Portal Pilihan VR)
├── lab-peminatan-vr/         <-- Aplikasi VR yang sudah ada
└── jelajah/                  <-- Aplikasi Metaverse Jelajah Alam (index.html + folder klip/)
```

---

## 1. 💻 Cara Setup & Menjalankan di Lokal (Development)

> ⚠️ **Penting:** Jangan membuka file `index.html` dengan klik ganda (*double click* / `file://`). Browser akan memblokir tekstur video 3D dan Web Audio API karena kebijakan keamanan CORS. Gunakan local HTTP server.

Pilih salah satu cara termudah berikut dari terminal folder proyek:

### Cara A: Menggunakan Python (Sudah terpasang di sistem Anda)
Buka PowerShell / Terminal di folder ini (`d:\Repo\Metaverse-Jelajah Alam`):
```bash
python -m http.server 8000
```
Buka browser: **`http://localhost:8000`**

### Cara B: Menggunakan Node.js / NPX
```bash
npx serve .
# atau
npx http-server -p 8000
```

### Cara C: Menggunakan Ekstensi VS Code
Jika menggunakan VS Code, pasang ekstensi **Live Server**, klik kanan pada `index.html` lalu pilih **"Open with Live Server"**.

---

## 2. 🌐 Persiapan Upload ke Server

Pastikan folder yang diunggah ke server mencakup:
1. `index.html` (Aplikasi 3D Jelajah Alam)
2. Folder `klip/` (Seluruh 26 file `.mp4`)

### Langkah A: Upload Folder Proyek ke Server
Dari komputer lokal, jalankan perintah `scp` atau `rsync` (atau gunakan FileZilla / WinSCP):

```bash
# Contoh menggunakan SCP dari PowerShell/CMD:
scp -r "d:\Repo\Metaverse-Jelajah Alam\*" root@server-vr-ti:/var/www/jelajah/
```
*Atau buat folder `/var/www/jelajah` terlebih dahulu di server:*
```bash
# Di server:
mkdir -p /var/www/jelajah
```

### Langkah B: Pasang Gerbang Awal (Portal Gateway)
File `portal/index.html` (atau `gateway-index.html`) yang telah dibuat berfungsi sebagai portal pemilihan VR.
Copy file tersebut ke document root utama server:

```bash
# Di server:
cp /var/www/jelajah/gateway-index.html /var/www/html/index.html
```

### Langkah C: Atur Hak Akses File (Permissions)
Di server Ubuntu/Debian, berikan izin ke user web server (`www-data`):

```bash
chown -R www-data:www-data /var/www/jelajah
chmod -R 755 /var/www/jelajah
chown -R www-data:www-data /var/www/html
chmod -R 755 /var/www/html
```

---

## 3. ⚙️ Konfigurasi Web Server (Nginx)

Buka file konfigurasi Nginx untuk domain `vr.ti.fts.uika-bogor.ac.id`:
```bash
nano /etc/nginx/sites-available/vr.ti.fts.uika-bogor.ac.id
# atau di: /etc/nginx/sites-available/default
```

Pastikan blok server berisi routing berikut:

```nginx
server {
    listen 80;
    listen 443 ssl http2;
    server_name vr.ti.fts.uika-bogor.ac.id;

    # Dokumen root utama untuk Gerbang Awal
    root /var/www/html;
    index index.html;

    # 1. GERBANG AWAL (PORTAL)
    location / {
        try_files $uri $uri/ =404;
    }

    # 2. APLIKASI VR 1: LAB PEMINATAN
    location = /lab-peminatan-vr {
        return 301 /lab-peminatan-vr/;
    }
    location /lab-peminatan-vr/ {
        alias /var/www/lab-peminatan-vr/;
        index index.html;
        try_files $uri $uri/ =404;
    }

    # 3. APLIKASI VR 2: JELAJAH KEAJAIBAN ALAM
    location = /jelajah {
        return 301 /jelajah/;
    }
    location /jelajah/ {
        alias /var/www/jelajah/;
        index index.html;
        try_files $uri $uri/ =404;

        # Optimasi streaming potongan film video 3D
        location ~* \.mp4$ {
            add_header Accept-Ranges bytes;
            mp4;
            mp4_buffer_size 1m;
            mp4_max_buffer_size 50m;
            expires 30d;
            add_header Cache-Control "public, no-transform";
        }
    }
}
```

Uji dan reload Nginx:
```bash
nginx -t
systemctl reload nginx
```

---

## 4. 🔗 Skema URL Hasil Akhir

| Halaman | URL | Keterangan |
|---|---|---|
| **Gerbang Awal (Portal)** | `https://vr.ti.fts.uika-bogor.ac.id/` | Memilih aplikasi VR yang mau dibuka |
| **VR Lab Peminatan TI** | `https://vr.ti.fts.uika-bogor.ac.id/lab-peminatan-vr/` | Aplikasi VR Lab Peminatan yang sudah ada |
| **Jelajah Alam 3D** | `https://vr.ti.fts.uika-bogor.ac.id/jelajah/` | Metaverse 3D Spasial Jelajah Keajaiban Alam |
