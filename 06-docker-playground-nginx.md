# 06 — Docker Playground & Aplikasi Web Nginx Sederhana

**Durasi estimasi:** 45–60 menit  
**Level:** Pemula–Menengah  
**Prasyarat:** Docker sudah terinstal di VM ([lihat panduan](05-install-docker.md))

---

## Tujuan Lab

- Eksplorasi perintah Docker secara hands-on
- Deploy web server Nginx menggunakan Docker
- Membuat custom halaman web di dalam container
- Menggunakan Docker Volume untuk persistent data
- Menulis Dockerfile untuk build image sendiri
- Menggunakan Docker Compose untuk multi-service

---

## Skenario Lab

```
  ┌─────────────────────────────────────────┐
  │           VM Ubuntu (Host)              │
  │                                         │
  │  ┌─────────────────────────────────┐   │
  │  │       Docker Engine             │   │
  │  │                                 │   │
  │  │  ┌──────────┐  ┌──────────┐    │   │
  │  │  │ Nginx    │  │ Nginx    │    │   │
  │  │  │ Container│  │ Container│    │   │
  │  │  │  :8080   │  │  :8081   │    │   │
  │  │  └──────────┘  └──────────┘    │   │
  │  └─────────────────────────────────┘   │
  └─────────────────────────────────────────┘
         ▲                    ▲
    http://IP:8080       http://IP:8081
    (dari browser host / laptop)
```

---

## Bagian 1 — Playground Docker Dasar

### 1.1 Eksplorasi Image Nginx

```bash
# Download image Nginx versi terbaru
docker pull nginx:latest

# Download versi spesifik
docker pull nginx:1.25

# Lihat image yang tersedia
docker images

# Inspeksi detail image
docker inspect nginx:latest
```

### 1.2 Jalankan Container Nginx Pertama

```bash
# Jalankan Nginx di background, port 8080
docker run -d \
  --name nginx-01 \
  -p 8080:80 \
  nginx:latest

# Cek container berjalan
docker ps
```

**Test akses dari host (laptop):**

Di browser laptop, buka:  
`http://IP-VM:8080`

Cari IP VM terlebih dahulu:
```bash
ip addr show | grep "inet " | grep -v 127.0.0.1
```

Jika berhasil, akan tampil halaman **"Welcome to nginx!"**

### 1.3 Lihat Log Nginx

```bash
# Lihat access log Nginx
docker logs nginx-01

# Ikuti log secara real-time (tekan Ctrl+C untuk berhenti)
docker logs -f nginx-01
```

### 1.4 Masuk ke dalam Container

```bash
# Buka shell bash di dalam container
docker exec -it nginx-01 bash

# Di dalam container:
ls /usr/share/nginx/html/    # lokasi file web
cat /etc/nginx/nginx.conf    # konfigurasi Nginx
nginx -v                     # versi Nginx

# Keluar dari container
exit
```

---

## Bagian 2 — Custom Halaman Web dengan Volume

### 2.1 Buat Folder dan File HTML di Host

```bash
# Buat direktori untuk file web
mkdir -p ~/lab-nginx/html

# Buat halaman index.html custom
cat > ~/lab-nginx/html/index.html << 'EOF'
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Lab Docker - IT Operations</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
            background: #f0f4f8;
        }
        .card {
            background: white;
            padding: 30px;
            border-radius: 10px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        h1 { color: #2d3748; }
        .badge {
            display: inline-block;
            background: #4299e1;
            color: white;
            padding: 5px 15px;
            border-radius: 20px;
            font-size: 14px;
        }
        .info { margin: 20px 0; padding: 15px; background: #ebf8ff; border-radius: 8px; }
    </style>
</head>
<body>
    <div class="card">
        <h1>🐳 Docker Lab — IT Operations</h1>
        <span class="badge">Nginx Container</span>
        <div class="info">
            <p><strong>Status:</strong> ✅ Berjalan di dalam Docker Container</p>
            <p><strong>Web Server:</strong> Nginx (Latest)</p>
            <p><strong>Dibuat oleh:</strong> Tim IT Operations</p>
        </div>
        <h2>Tentang Lab Ini</h2>
        <p>Halaman ini dijalankan di dalam Docker container Nginx.
        File HTML di-mount dari host menggunakan Docker Volume.</p>
        <p>Ini membuktikan bahwa kita bisa mengelola konten web
        tanpa masuk ke dalam container.</p>
    </div>
</body>
</html>
EOF
```

### 2.2 Jalankan Nginx dengan Volume

```bash
# Hentikan dan hapus container sebelumnya jika ada
docker stop nginx-01
docker rm nginx-01

# Jalankan ulang dengan volume mount
docker run -d \
  --name nginx-01 \
  -p 8080:80 \
  -v ~/lab-nginx/html:/usr/share/nginx/html:ro \
  nginx:latest
```

Akses kembali `http://IP-VM:8080` — sekarang tampil halaman custom!

### 2.3 Edit File Tanpa Restart Container

```bash
# Edit file di host
nano ~/lab-nginx/html/index.html

# Ubah teks apapun, simpan, lalu refresh browser
# Tidak perlu restart container!
```

---

## Bagian 3 — Build Image Custom dengan Dockerfile

### 3.1 Struktur Proyek

```bash
mkdir -p ~/lab-nginx/myapp
cd ~/lab-nginx/myapp
```

Struktur folder yang akan dibuat:
```
myapp/
├── Dockerfile
├── nginx.conf
└── html/
    ├── index.html
    └── style.css
```

### 3.2 Buat File HTML dan CSS

```bash
mkdir html

cat > html/index.html << 'EOF'
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>MyApp - Docker Lab</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <div class="container">
        <header>
            <h1>🚀 MyApp</h1>
            <p class="subtitle">Dibangun dengan Docker</p>
        </header>
        <main>
            <div class="card">
                <h2>Server Info</h2>
                <ul>
                    <li>Web Server: Nginx</li>
                    <li>Container: Docker</li>
                    <li>Environment: Lab IT Ops</li>
                </ul>
            </div>
            <div class="card">
                <h2>Docker Commands</h2>
                <code>docker build -t myapp .</code><br>
                <code>docker run -d -p 8081:80 myapp</code>
            </div>
        </main>
        <footer>
            <p>IT Operations Lab &copy; 2025</p>
        </footer>
    </div>
</body>
</html>
EOF

cat > html/style.css << 'EOF'
* { box-sizing: border-box; margin: 0; padding: 0; }
body {
    font-family: 'Segoe UI', sans-serif;
    background: linear-gradient(135deg, #1a1a2e, #16213e);
    color: #e0e0e0;
    min-height: 100vh;
    padding: 20px;
}
.container { max-width: 900px; margin: 0 auto; }
header {
    text-align: center;
    padding: 40px 0;
}
header h1 { font-size: 48px; color: #4fc3f7; }
.subtitle { color: #90caf9; margin-top: 10px; }
main {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 20px;
    margin: 30px 0;
}
.card {
    background: rgba(255,255,255,0.05);
    padding: 25px;
    border-radius: 12px;
    border: 1px solid rgba(255,255,255,0.1);
}
.card h2 { color: #4fc3f7; margin-bottom: 15px; }
.card ul { list-style: none; }
.card ul li { padding: 5px 0; }
.card ul li::before { content: "▸ "; color: #4fc3f7; }
code {
    display: block;
    background: rgba(0,0,0,0.3);
    padding: 8px 12px;
    border-radius: 6px;
    font-size: 13px;
    margin: 5px 0;
    color: #a5d6a7;
}
footer {
    text-align: center;
    padding: 20px;
    color: #607d8b;
}
EOF
```

### 3.3 Buat Konfigurasi Nginx Custom

```bash
cat > nginx.conf << 'EOF'
server {
    listen 80;
    server_name localhost;
    
    root /usr/share/nginx/html;
    index index.html;
    
    location / {
        try_files $uri $uri/ =404;
    }
    
    # Tambah header keamanan dasar
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    
    # Gzip compression
    gzip on;
    gzip_types text/html text/css application/javascript;
}
EOF
```

### 3.4 Buat Dockerfile

```bash
cat > Dockerfile << 'EOF'
# Gunakan image Nginx Alpine (lebih ringan)
FROM nginx:alpine

# Label metadata
LABEL maintainer="IT Operations Lab"
LABEL version="1.0"
LABEL description="Custom Nginx web server untuk lab Docker"

# Hapus konfigurasi default Nginx
RUN rm /etc/nginx/conf.d/default.conf

# Salin konfigurasi custom
COPY nginx.conf /etc/nginx/conf.d/

# Salin file web
COPY html/ /usr/share/nginx/html/

# Ekspos port 80
EXPOSE 80

# Jalankan Nginx di foreground
CMD ["nginx", "-g", "daemon off;"]
EOF
```

### 3.5 Build Image

```bash
# Pastikan di dalam direktori myapp
cd ~/lab-nginx/myapp

# Build image dengan tag
docker build -t myapp:1.0 .

# Lihat proses build step by step
# Setiap baris di Dockerfile = satu layer

# Verifikasi image terbuat
docker images | grep myapp
```

### 3.6 Jalankan Container dari Image Custom

```bash
docker run -d \
  --name myapp-container \
  -p 8081:80 \
  myapp:1.0
```

Akses di browser: `http://IP-VM:8081`

---

## Bagian 4 — Docker Compose (Multi-Container)

### 4.1 Buat File docker-compose.yml

```bash
mkdir -p ~/lab-compose
cd ~/lab-compose

cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  # Web server utama
  web:
    image: nginx:alpine
    container_name: compose-web
    ports:
      - "8090:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
    restart: unless-stopped
    networks:
      - lab-network

  # Web server backup
  web-backup:
    image: nginx:alpine
    container_name: compose-web-backup
    ports:
      - "8091:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
    restart: unless-stopped
    networks:
      - lab-network

networks:
  lab-network:
    driver: bridge
EOF
```

### 4.2 Buat Konten HTML

```bash
mkdir html

cat > html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Docker Compose Lab</title>
    <style>
        body { font-family: Arial; text-align: center; padding: 50px; background: #e8f5e9; }
        h1 { color: #2e7d32; }
        .box { background: white; padding: 20px; border-radius: 10px; display: inline-block; }
    </style>
</head>
<body>
    <div class="box">
        <h1>✅ Docker Compose Berjalan!</h1>
        <p>Multi-container environment berhasil dideploy.</p>
        <p><em>Dijalankan dengan: docker compose up -d</em></p>
    </div>
</body>
</html>
EOF
```

### 4.3 Operasi Docker Compose

```bash
cd ~/lab-compose

# Jalankan semua service
docker compose up -d

# Lihat status service
docker compose ps

# Lihat log semua service
docker compose logs

# Lihat log service tertentu
docker compose logs web

# Stop semua service
docker compose stop

# Start kembali
docker compose start

# Restart service tertentu
docker compose restart web

# Hapus semua container (data volume tetap ada)
docker compose down

# Hapus termasuk volume
docker compose down -v
```

---

## Bagian 5 — Latihan Mandiri

### Latihan 1 — Ubah Konten Web Tanpa Rebuild

1. Edit file `~/lab-nginx/html/index.html`
2. Tambahkan tanggal hari ini di halaman
3. Refresh browser — apakah perubahan langsung terlihat?

### Latihan 2 — Jalankan Dua Versi Nginx Bersamaan

```bash
# Nginx versi latest
docker run -d --name nginx-latest -p 8080:80 nginx:latest

# Nginx versi 1.24 (versi lama)
docker run -d --name nginx-old -p 8082:80 nginx:1.24

# Bandingkan header response-nya
curl -I http://localhost:8080
curl -I http://localhost:8082
```

### Latihan 3 — Resource Monitoring

```bash
# Jalankan beberapa container
docker run -d --name n1 nginx
docker run -d --name n2 nginx
docker run -d --name n3 nginx

# Monitor semua container
docker stats

# Lihat penggunaan disk Docker
docker system df
```

### Latihan 4 — Build dan Modifikasi Image

1. Modifikasi file `html/index.html` di dalam `~/lab-nginx/myapp/`
2. Build ulang image dengan tag baru: `myapp:2.0`
3. Jalankan container dari image baru di port 8083
4. Bandingkan tampilan versi 1.0 dan 2.0

---

## Ringkasan Perintah Lab Ini

```bash
# Pull dan jalankan Nginx
docker run -d -p 8080:80 --name nginx-01 nginx

# Dengan volume
docker run -d -p 8080:80 -v ~/html:/usr/share/nginx/html:ro nginx

# Build image custom
docker build -t myapp:1.0 .

# Jalankan image custom
docker run -d -p 8081:80 --name myapp myapp:1.0

# Docker Compose
docker compose up -d
docker compose down
```

---

## Troubleshooting

| Masalah | Solusi |
|---|---|
| Port 8080 sudah dipakai | Ganti ke port lain misalnya 8082 |
| Browser tidak bisa akses | Cek IP VM, pastikan firewall tidak memblokir |
| `docker build` gagal | Pastikan berada di direktori yang berisi Dockerfile |
| Volume kosong / konten tidak muncul | Cek path volume: pastikan direktori ada dan ada isinya |
| Container langsung exit | Jalankan `docker logs <nama>` untuk lihat error |

---

## Ringkasan

✅ Nginx berjalan di Docker container  
✅ Custom halaman web dengan volume mount  
✅ Image custom dibangun dengan Dockerfile  
✅ Multi-container dengan Docker Compose  

**Lanjutkan ke:** [07 — Create LXC Container di Proxmox](07-proxmox-lxc-setup.md)
