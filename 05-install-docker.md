# 05 — Instalasi Docker di Linux VM (Ubuntu/Debian)

**Durasi estimasi:** 20–30 menit  
**Level:** Pemula  
**Prasyarat:** VM Ubuntu 22.04 sudah berjalan ([lihat panduan import](03-import-vm-ova.md)), akses internet dari VM

---

## Tujuan Lab

- Menginstal Docker Engine di VM Ubuntu/Debian
- Menjalankan perintah Docker dasar
- Memahami konsep Image, Container, dan Registry

---

## Konsep Dasar Docker

```
Registry (Docker Hub)
      │
      │  docker pull
      ▼
   Image                ← Template read-only (blueprint)
      │
      │  docker run
      ▼
  Container             ← Instance yang berjalan dari Image
```

| Istilah | Analogi | Penjelasan |
|---|---|---|
| **Image** | ISO / Template VM | Blueprint yang berisi OS + aplikasi |
| **Container** | VM yang berjalan | Instance aktif dari image |
| **Registry** | App Store | Tempat menyimpan & download image |
| **Docker Hub** | Registry publik | Repository image resmi Docker |
| **Dockerfile** | Resep masakan | Script untuk build image custom |

---

## Langkah 1 — Update Sistem

Login ke VM Ubuntu, kemudian:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Langkah 2 — Install Docker Engine

Gunakan skrip instalasi resmi dari Docker (metode paling mudah dan up-to-date):

### Metode A — Skrip Otomatis (Rekomendasi untuk Lab)

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Tunggu proses selesai (2–5 menit tergantung koneksi).

### Metode B — Manual via APT Repository

```bash
# 1. Install dependensi
sudo apt install -y ca-certificates curl gnupg lsb-release

# 2. Tambah GPG key Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 3. Tambah repository Docker
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 4. Update dan install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

## Langkah 3 — Konfigurasi Post-Install

### 3.1 Jalankan Docker Tanpa sudo

Secara default, Docker memerlukan `sudo`. Tambahkan user ke grup `docker`:

```bash
sudo usermod -aG docker $USER
```

Terapkan perubahan grup (pilih salah satu):
```bash
# Opsi 1: Logout dan login kembali (disarankan)
exit
# Login kembali, lalu cek

# Opsi 2: Aktifkan tanpa logout
newgrp docker
```

### 3.2 Aktifkan Docker Service Otomatis

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Cek status Docker:
```bash
sudo systemctl status docker
```

Output yang diharapkan:
```
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled)
     Active: active (running) since ...
```

---

## Langkah 4 — Verifikasi Instalasi

### 4.1 Cek Versi Docker

```bash
docker --version
```

Output:
```
Docker version 26.x.x, build xxxxxxx
```

```bash
docker compose version
```

Output:
```
Docker Compose version v2.x.x
```

### 4.2 Jalankan Container Hello World

```bash
docker run hello-world
```

Output yang diharapkan:
```
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image...
```

---

## Langkah 5 — Perintah Docker Dasar

### 5.1 Manajemen Image

```bash
# Download image dari Docker Hub
docker pull ubuntu:22.04
docker pull nginx:latest
docker pull alpine:latest

# Lihat daftar image yang sudah diunduh
docker images
# atau
docker image ls

# Hapus image
docker rmi hello-world
docker image rm ubuntu:22.04
```

### 5.2 Manajemen Container

```bash
# Jalankan container interaktif (masuk ke shell Ubuntu)
docker run -it ubuntu:22.04 bash

# Jalankan container di background (detached mode)
docker run -d --name webserver nginx:latest

# Lihat container yang sedang berjalan
docker ps

# Lihat semua container (termasuk yang sudah stop)
docker ps -a

# Stop container
docker stop webserver

# Start container yang sudah ada
docker start webserver

# Restart container
docker restart webserver

# Hapus container
docker rm webserver

# Masuk ke container yang sedang berjalan
docker exec -it webserver bash
```

### 5.3 Melihat Log Container

```bash
# Lihat log container
docker logs webserver

# Lihat log secara real-time (follow)
docker logs -f webserver

# Lihat 50 baris terakhir
docker logs --tail 50 webserver
```

### 5.4 Informasi Container dan Image

```bash
# Detail informasi container
docker inspect webserver

# Statistik penggunaan resource (CPU, RAM, Network)
docker stats

# Statistik satu container
docker stats webserver

# Proses di dalam container
docker top webserver
```

### 5.5 Pembersihan (Cleanup)

```bash
# Hapus semua container yang sudah stop
docker container prune

# Hapus semua image yang tidak digunakan
docker image prune

# Hapus semua resource yang tidak digunakan (container, network, image, cache)
docker system prune

# Lihat penggunaan disk Docker
docker system df
```

---

## Langkah 6 — Memahami Volume dan Port Mapping

### 6.1 Port Mapping

Membuka port container agar bisa diakses dari host:

```bash
# Sintaks: docker run -p <port_host>:<port_container>
docker run -d -p 8080:80 --name web nginx

# Akses dari host: http://localhost:8080 atau http://IP-VM:8080
```

### 6.2 Volume

Menyimpan data agar tetap ada meski container dihapus:

```bash
# Sintaks: docker run -v <path_host>:<path_container>
docker run -d \
  -p 8080:80 \
  -v /home/ubuntu/html:/usr/share/nginx/html \
  --name web nginx
```

---

## Langkah 7 — Install Docker Compose (Verifikasi)

Docker Compose sudah termasuk dalam instalasi Docker Engine modern (sebagai plugin):

```bash
docker compose version
```

Jika tidak tersedia, install terpisah:
```bash
sudo apt install -y docker-compose-plugin
```

---

## Cheat Sheet Perintah Docker

```
╔══════════════════════════════════════════════════════════╗
║                   DOCKER CHEAT SHEET                    ║
╠══════════════════════════════════════════════════════════╣
║  IMAGE                                                   ║
║  docker pull <image>          Download image            ║
║  docker images                List semua image          ║
║  docker rmi <image>           Hapus image               ║
║  docker build -t <name> .     Build image dari Dockerfile║
╠══════════════════════════════════════════════════════════╣
║  CONTAINER                                               ║
║  docker run <image>           Buat & jalankan container ║
║  docker run -d <image>        Mode background           ║
║  docker run -it <image> bash  Mode interaktif           ║
║  docker ps                    List container aktif      ║
║  docker ps -a                 List semua container      ║
║  docker stop <container>      Stop container            ║
║  docker rm <container>        Hapus container           ║
║  docker exec -it <c> bash     Masuk ke container        ║
║  docker logs <container>      Lihat log                 ║
╠══════════════════════════════════════════════════════════╣
║  SYSTEM                                                  ║
║  docker system df             Penggunaan disk           ║
║  docker system prune          Bersihkan semua unused    ║
║  docker stats                 Monitor resource          ║
╚══════════════════════════════════════════════════════════╝
```

---

## Troubleshooting Umum

| Masalah | Solusi |
|---|---|
| `permission denied` saat docker run | Jalankan `sudo usermod -aG docker $USER` lalu re-login |
| `Cannot connect to Docker daemon` | Jalankan `sudo systemctl start docker` |
| `pull access denied` | Cek nama image dan tag, atau `docker logout` lalu coba lagi |
| Container langsung exit | Cek log dengan `docker logs <nama>` |
| Port sudah dipakai | Ganti port host, atau `sudo lsof -i :8080` untuk cek proses |

---

## Ringkasan

✅ Docker Engine terinstal  
✅ User ditambahkan ke grup docker  
✅ Docker service berjalan otomatis  
✅ Perintah dasar Docker dikuasai  

**Lanjutkan ke:** [06 — Docker Playground & Aplikasi Nginx](06-docker-playground-nginx.md)
