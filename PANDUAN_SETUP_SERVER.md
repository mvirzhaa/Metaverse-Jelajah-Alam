# 🚀 Panduan Setup Lokal & Deploy Server VR TI FTS UIKA Bogor
### Domain: `https://vr.ti.fts.uika-bogor.ac.id/`

Panduan ini mencakup pengelolaan **3 Aplikasi VR & Metaverse** dalam satu server yang sama:
1. 🏛️ **VR Lab Peminatan TI** (Aplikasi WebRTC/Socket existing - Port `8080`)
2. 🌍 **Metaverse Jelajah Keajaiban Alam** (Web 3D Three.js Statis - Subpath `/jelajah/`)
3. 🏬 **Virtual Mall Botani Square 3D** (`virtual-mall-botani` - Node.js + WebSocket Port `8090` - Subpath `/botani/`)

---

## 📂 1. Struktur Folder di Server (`/var/www`)

Di server `root@server-vr-ti:/var/www#`, susunan foldernya adalah:

```text
/var/www/
├── html/                     <-- Berisi index.html Gerbang Awal (Portal 3 Pilihan VR)
├── lab-peminatan-vr/         <-- Aplikasi VR Lab Peminatan existing (Port 8080)
├── jelajah/                  <-- Aplikasi Metaverse Jelajah Alam (index.html + folder klip/)
└── virtual-mall-botani/      <-- Aplikasi Virtual Mall Botani Square (server.js, public/, data/)
```

---

## 💻 2. Setup Lokal untuk Virtual Mall Botani Square

Aplikasi `virtual-mall-botani` adalah aplikasi **Node.js** mandiri dengan dukungan WebSockets multi-user.

### Langkah Menjalankan di Lokal:
1. Buka folder proyek di terminal / PowerShell:
   ```bash
   cd "d:\Repo\virtualmall-botani v1\virtual-mall-botani"
   ```
2. Install dependency:
   ```bash
   npm install
   ```
3. Jalankan server:
   ```bash
   npm start
   ```
4. Buka di browser:
   - **Mall 3D Pengunjung**: `http://localhost:8090`
   - **Panel Admin Mall**: `http://localhost:8090/admin`

---

## 🌐 3. Proses Upload & Setup `virtual-mall-botani` ke Server

### Langkah A: Upload Folder ke Server
Dari komputer lokal, jalankan perintah `scp` (atau gunakan WinSCP / FileZilla):
```bash
# Upload folder virtual-mall-botani ke server
scp -r "d:\Repo\virtualmall-botani v1\virtual-mall-botani" root@server-vr-ti:/var/www/virtual-mall-botani
```

### Langkah B: Install Dependency & Jalankan di Server
Masuk ke terminal server (`root@server-vr-ti`):
```bash
cd /var/www/virtual-mall-botani
npm install

# Jalankan service di background menggunakan PM2 agar otomatis hidup kembali saat server restart:
npm install -g pm2
pm2 start server.js --name "mall-botani"
pm2 save
pm2 startup
```

> 💡 **Cek Status Service:**
> ```bash
> pm2 status
> # Cek log jika ada kendala:
> pm2 logs mall-botani
> ```

### Langkah C: Atur Hak Akses File (Permissions)
```bash
chown -R www-data:www-data /var/www/virtual-mall-botani
chmod -R 755 /var/www/virtual-mall-botani
```

---

## 🚪 4. Update Gerbang Awal (Portal `index.html`)

File [portal/index.html](file:///d:/Repo/Metaverse-Jelajah%20Alam/portal/index.html) (atau [gateway-index.html](file:///d:/Repo/Metaverse-Jelajah%20Alam/gateway-index.html)) sudah diperbarui dengan **3 kartu VR interaktif** (termasuk Virtual Mall Botani Square).

Upload dan pasang ke document root `/var/www/html/index.html`:
```bash
# Di server:
cp /var/www/jelajah/gateway-index.html /var/www/html/index.html
chown www-data:www-data /var/www/html/index.html
```

---

## ⚙️ 5. Konfigurasi Lengkap Nginx di Server

Buka file konfigurasi Nginx di server:
```bash
nano /etc/nginx/sites-available/vr.ti.fts.uika-bogor.ac.id
```

Ganti seluruh isinya dengan konfigurasi terintegrasi berikut:

```nginx
server {
    server_name vr.ti.fts.uika-bogor.ac.id;
    client_max_body_size 100M;

    # ================================================================
    # 1. GERBANG AWAL (PORTAL 3 VR) PADA HALAMAN UTAMA (/)
    # ================================================================
    location = / {
        root /var/www/html;
        index index.html;
        try_files /index.html =404;
    }

    # ================================================================
    # 2. APLIKASI VR 1: JELAJAH KEAJAIBAN ALAM (Subpath /jelajah/)
    # ================================================================
    location = /jelajah {
        return 301 /jelajah/;
    }

    location /jelajah/ {
        alias /var/www/jelajah/;
        index index.html;
        try_files $uri $uri/ =404;

        # Optimasi streaming video 3D mp4
        location ~* \.mp4$ {
            add_header Accept-Ranges bytes;
            mp4;
            mp4_buffer_size 1m;
            mp4_max_buffer_size 50m;
            expires 30d;
            add_header Cache-Control "public, no-transform";
        }
    }

    # ================================================================
    # 3. APLIKASI VR 2: VIRTUAL MALL BOTANI SQUARE (Subpath /botani/)
    # Port Backend: 8090
    # ================================================================
    location = /botani {
        return 301 /botani/;
    }

    location /botani/ {
        # Tanda slash "/" di akhir WAJIB untuk me-rewrite path ke root port 8090
        proxy_pass http://127.0.0.1:8090/;
        proxy_http_version 1.1;

        # WebSocket wajib untuk pergerakan avatar, chat, dan WebRTC voice
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;

        proxy_redirect / /botani/;
    }

    # ================================================================
    # 4. APLIKASI VR 3: LAB PEMINATAN TI (Subpath /lab-peminatan-vr/)
    # Port Backend: 8080
    # ================================================================
    location = /lab-peminatan-vr {
        return 301 /lab-peminatan-vr/;
    }

    location /lab-peminatan-vr/ {
        proxy_pass http://127.0.0.1:8080/;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;

        proxy_redirect / /lab-peminatan-vr/;
    }

    # ================================================================
    # 5. FALLBACK PROXY UNTUK ASSETS UMUM PORT 8080
    # ================================================================
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }

    listen 443 ssl; # managed by Certbot
    listen [::]:443 ssl ipv6only=on; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/vr.ti.fts.uika-bogor.ac.id/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/vr.ti.fts.uika-bogor.ac.id/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

server {
    if ($host = vr.ti.fts.uika-bogor.ac.id) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    listen 80;
    listen [::]:80;
    server_name vr.ti.fts.uika-bogor.ac.id;
    return 404; # managed by Certbot
}
```

Uji dan reload Nginx di server:
```bash
nginx -t && systemctl reload nginx
```

---

## 🔗 6. Peta URL Hasil Akhir di Server

| Tujuan | URL Akses | Tipe Layanan |
|---|---|---|
| 🚪 **Gerbang Awal (Portal 3 VR)** | `https://vr.ti.fts.uika-bogor.ac.id/` | File Statis `/var/www/html/index.html` |
| 🌍 **Metaverse Jelajah Alam** | `https://vr.ti.fts.uika-bogor.ac.id/jelajah/` | File Statis Three.js + MP4 `/var/www/jelajah/` |
| 🏬 **Virtual Mall Botani Square** | `https://vr.ti.fts.uika-bogor.ac.id/botani/` | Node.js + WebSocket (Port `8090`) |
| 🔑 **Admin Mall Botani** | `https://vr.ti.fts.uika-bogor.ac.id/botani/admin` | Panel Admin Mall Botani |
| 🏛️ **VR Lab Peminatan TI** | `https://vr.ti.fts.uika-bogor.ac.id/lab-peminatan-vr/` | Node.js + WebSocket (Port `8080`) |
