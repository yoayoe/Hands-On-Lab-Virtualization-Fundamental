# 09 — Skenario Playground Proxmox LXC

**Durasi estimasi:** 60–90 menit  
**Level:** Menengah  
**Prasyarat:** Proxmox VE 9.x, minimal 2 LXC container berjalan ([lihat panduan setup](07-proxmox-lxc-setup.md))

---

## Tujuan Lab

- Mengelola beberapa container sekaligus di Proxmox
- Deploy web server Nginx di LXC container
- Monitoring resource container
- Skenario high availability: snapshot, clone, dan failover manual
- Manajemen jaringan antar container

---

## Topologi Lab

```
┌─────────────────────────────────────────────────────────────┐
│                    Proxmox VE Host                          │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  CT 100      │  │  CT 101      │  │  CT 102      │     │
│  │  Web Server  │  │  Web Backup  │  │  Monitor     │     │
│  │  Nginx       │  │  Nginx       │  │  (tools)     │     │
│  │  192.168.1.10│  │  192.168.1.11│  │  192.168.1.12│     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│          │                 │                 │              │
│  ════════╪═════════════════╪═════════════════╪═══════════  │
│                        vmbr0 Bridge                        │
│                        192.168.1.0/24                      │
└─────────────────────────────────────────────────────────────┘
         │
    ┌────┴────┐
    │  Router │ / Switch
    └────┬────┘
         │
   Laptop peserta
   (Browser, SSH)
```

---

## Persiapan — Buat 3 Container Lab

Jalankan perintah berikut dari **Proxmox host** (SSH ke Proxmox):

```bash
# Template yang digunakan (sesuaikan nama file jika berbeda)
TEMPLATE="local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst"

# Container 100 — Web Server
pct create 100 $TEMPLATE \
  --hostname ct100-webserver \
  --password Lab123! \
  --cores 1 --memory 512 --swap 256 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 1 --features nesting=1

# Container 101 — Web Backup
pct create 101 $TEMPLATE \
  --hostname ct101-web-backup \
  --password Lab123! \
  --cores 1 --memory 512 --swap 256 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 1 --features nesting=1

# Container 102 — Monitor
pct create 102 $TEMPLATE \
  --hostname ct102-monitor \
  --password Lab123! \
  --cores 1 --memory 256 --swap 256 \
  --rootfs local-lvm:4 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 1

# Start semua container
pct start 100
pct start 101
pct start 102

# Cek semua container berjalan
pct list
```

---

## Skenario 1 — Deploy Web Server Nginx di LXC

### 1.1 Install Nginx di CT 100

```bash
# Masuk ke CT 100
pct enter 100

# Update dan install Nginx
apt update && apt install -y nginx curl wget

# Aktifkan Nginx
systemctl enable nginx
systemctl start nginx
systemctl status nginx

# Cek IP container
ip addr show eth0 | grep "inet "
# Catat IP ini, misalnya: 192.168.1.10
```

### 1.2 Buat Halaman Web Custom

```bash
# Masih di dalam CT 100

# Buat halaman web custom
cat > /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>CT 100 - Web Server Utama</title>
    <style>
        body { font-family: Arial; text-align: center; padding: 60px;
               background: #e3f2fd; }
        .box { background: white; padding: 30px; border-radius: 15px;
               display: inline-block; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        h1 { color: #1565c0; }
        .badge { background: #1976d2; color: white; padding: 5px 15px;
                 border-radius: 20px; }
        .info { margin-top: 20px; text-align: left; }
    </style>
</head>
<body>
    <div class="box">
        <h1>🖥️ Web Server Utama</h1>
        <span class="badge">CT 100 | Proxmox LXC</span>
        <div class="info">
            <p>✅ Status: <strong>Online</strong></p>
            <p>🐧 OS: Ubuntu 22.04 LXC</p>
            <p>🌐 Teknologi: Nginx</p>
            <p>📅 Lab: Proxmox Playground</p>
        </div>
    </div>
</body>
</html>
EOF

# Test dari dalam container
curl http://localhost

# Keluar dari container
exit
```

### 1.3 Akses Web dari Laptop

Di browser laptop: `http://192.168.1.10`  
(ganti dengan IP CT 100 yang dicatat tadi)

### 1.4 Ulangi untuk CT 101 (Web Backup)

```bash
pct enter 101

apt update && apt install -y nginx
systemctl enable nginx && systemctl start nginx

cat > /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>CT 101 - Web Backup</title>
    <style>
        body { font-family: Arial; text-align: center; padding: 60px;
               background: #e8f5e9; }
        .box { background: white; padding: 30px; border-radius: 15px;
               display: inline-block; }
        h1 { color: #2e7d32; }
    </style>
</head>
<body>
    <div class="box">
        <h1>🔄 Web Server Backup</h1>
        <p>CT 101 | Berjalan sebagai backup dari CT 100</p>
        <p>✅ Status: <strong>Standby / Active</strong></p>
    </div>
</body>
</html>
EOF

exit
```

---

## Skenario 2 — Monitoring Resource Container

### 2.1 Monitoring dari Proxmox Host

```bash
# Lihat status semua container
pct list

# Lihat penggunaan resource realtime
# (dari Proxmox host terminal)
watch -n 2 'pct list'

# Lihat resource detail per container
pvesh get /nodes/pve/lxc/100/status/current
pvesh get /nodes/pve/lxc/101/status/current
```

### 2.2 Install Tools Monitoring di CT 102

```bash
pct enter 102

# Install tools monitoring
apt update && apt install -y \
  htop \
  nethogs \
  iotop \
  nmap \
  curl \
  wget

# Monitor CPU dan RAM semua proses
htop
# (tekan Q untuk keluar)

exit
```

### 2.3 Monitor dari CT 102 ke Container Lain

```bash
pct enter 102

# Test koneksi ke CT 100 dan CT 101
# (ganti IP sesuai container Anda)
ping -c 4 192.168.1.10
ping -c 4 192.168.1.11

# Test akses HTTP
curl -I http://192.168.1.10
curl -I http://192.168.1.11

# Scan port yang terbuka di CT 100
nmap -p 1-1000 192.168.1.10

exit
```

### 2.4 Monitoring via Web UI Proxmox

1. Di Web UI, klik CT 100
2. Tab **Summary** — lihat CPU, Memory, Network usage
3. Tab **Console** — akses terminal langsung
4. Tab **Snapshots** — lihat daftar snapshot

---

## Skenario 3 — Snapshot dan Restore (Simulasi Backup)

### 3.1 Buat Snapshot Sebelum Perubahan

```bash
# Buat snapshot CT 100 dalam kondisi bersih
pct snapshot 100 snap-nginx-clean \
  --description "Nginx berjalan normal, sebelum perubahan konfigurasi"

# Verifikasi snapshot terbuat
pct listsnapshot 100
```

Output:
```
             PARENT NAME                      DESCRIPTION
                    current
snap-nginx-clean                              Nginx berjalan normal...
```

### 3.2 Simulasi Kerusakan Konfigurasi

```bash
pct enter 100

# "Sengaja" rusak konfigurasi Nginx
echo "KONFIGURASI RUSAK" > /etc/nginx/sites-enabled/default

# Restart Nginx — akan gagal
systemctl restart nginx
# Error: nginx: configuration file /etc/nginx/nginx.conf test failed

# Test dari luar — web tidak bisa diakses
exit
```

### 3.3 Restore dari Snapshot

```bash
# Stop container dulu
pct stop 100

# Rollback ke snapshot
pct rollback 100 snap-nginx-clean

# Start ulang
pct start 100

# Verifikasi Nginx berjalan kembali
pct exec 100 -- systemctl status nginx
```

Akses kembali `http://192.168.1.10` — website kembali normal!

---

## Skenario 4 — Clone Container (Duplikasi Cepat)

### 4.1 Clone CT 100 untuk Lab Baru

```bash
# Stop CT 100 dulu (untuk linked clone, tidak perlu stop)
# Untuk full clone, bisa dilakukan saat running

# Clone CT 100 ke CT 103
pct clone 100 103 \
  --hostname ct103-web-clone \
  --full

# Cek hasil clone
pct list
pct config 103
```

### 4.2 Ubah IP Clone (jika IP Statis)

```bash
pct set 103 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.13/24,gw=192.168.1.1
```

### 4.3 Start Clone dan Verifikasi

```bash
pct start 103
pct exec 103 -- ip addr show eth0
pct exec 103 -- systemctl status nginx
```

Akses `http://192.168.1.13` — clone berjalan identik dengan CT 100!

---

## Skenario 5 — Simulasi Failover Manual

Skenario ini mensimulasikan kondisi di mana web server utama (CT 100) mati, dan traffic dialihkan ke backup (CT 101).

### 5.1 Kondisi Normal

```bash
# CT 100 aktif (server utama)
# Akses: http://192.168.1.10 → "Web Server Utama"
curl http://192.168.1.10
```

### 5.2 Simulasi Server Utama Down

```bash
# Stop CT 100 (simulasi server mati)
pct stop 100

# Verifikasi tidak bisa diakses
curl --connect-timeout 3 http://192.168.1.10
# Timeout — server tidak merespons
```

### 5.3 Switch ke Server Backup

Informasikan tim (atau update DNS/Load Balancer) bahwa traffic beralih ke CT 101:

```bash
# CT 101 tetap aktif
curl http://192.168.1.11
# "Web Server Backup" — tersedia!
```

### 5.4 Recovery — Hidupkan Kembali Server Utama

```bash
pct start 100

# Tunggu beberapa detik
sleep 5

# Verifikasi kembali aktif
curl http://192.168.1.10
```

---

## Skenario 6 — Manajemen Resource Dinamis

### 6.1 Stress Test dan Monitor

```bash
# Install stress tool di CT 100
pct exec 100 -- apt install -y stress

# Jalankan stress test (2 CPU, 30 detik)
pct exec 100 -- stress --cpu 2 --timeout 30 &

# Monitor dari Proxmox host
watch -n 1 'pvesh get /nodes/pve/lxc/100/status/current | grep -E "cpu|mem"'
```

### 6.2 Batasi Resource (CPU Limit)

```bash
# Set CPU limit CT 100 menjadi 50% dari 1 core
pct set 100 --cpulimit 0.5

# Jalankan stress test lagi dan lihat perbedaan
pct exec 100 -- stress --cpu 2 --timeout 30 &
watch -n 1 'pvesh get /nodes/pve/lxc/100/status/current | grep cpu'

# Hapus limit (kembali unlimited)
pct set 100 --cpulimit 0
```

### 6.3 Tambah Memori On-the-fly

```bash
# Cek memory sekarang
pct config 100 | grep memory

# Tambah menjadi 1 GB (tanpa restart!)
pct set 100 --memory 1024

# Verifikasi dari dalam container
pct exec 100 -- free -h
```

---

## Latihan Mandiri

### Latihan 1 — Deploy Multi-Service

Di CT 100, install dan jalankan dua service sekaligus:
- Nginx (port 80)
- SSH server (port 22)

Verifikasi keduanya berjalan dan bisa diakses dari CT 102.

### Latihan 2 — Scheduled Snapshot via Cron

Di Proxmox host, buat cron job untuk snapshot otomatis setiap hari:

```bash
crontab -e

# Tambahkan baris ini (snapshot setiap hari jam 02:00)
0 2 * * * /usr/sbin/pct snapshot 100 snap-$(date +\%Y\%m\%d) --description "Snapshot otomatis $(date)"
```

### Latihan 3 — Test Connectivity Matrix

Dari CT 102, buat script bash untuk test koneksi ke semua container:

```bash
pct enter 102

cat > /root/check-ct.sh << 'EOF'
#!/bin/bash
CONTAINERS=("192.168.1.10" "192.168.1.11" "192.168.1.13")
NAMES=("CT100-WebServer" "CT101-Backup" "CT103-Clone")

echo "=== Connectivity Check $(date) ==="
for i in "${!CONTAINERS[@]}"; do
  if ping -c 1 -W 2 ${CONTAINERS[$i]} > /dev/null 2>&1; then
    echo "✅ ${NAMES[$i]} (${CONTAINERS[$i]}) - ONLINE"
  else
    echo "❌ ${NAMES[$i]} (${CONTAINERS[$i]}) - OFFLINE"
  fi
done
EOF

chmod +x /root/check-ct.sh
bash /root/check-ct.sh

exit
```

### Latihan 4 — Simulasi Migrasi Container

```bash
# Jika Anda memiliki 2 node Proxmox di cluster:
# Migrasi CT 100 dari node pve1 ke pve2
pct migrate 100 pve2 --restart

# Verifikasi container berjalan di node baru
ssh root@pve2 pct list
```

---

## Perintah Manajemen Berguna

```bash
# Ringkasan status semua CT
pct list

# Bulk start semua CT
for ct in 100 101 102; do pct start $ct; done

# Bulk stop semua CT
for ct in 100 101 102; do pct stop $ct; done

# Jalankan perintah di semua CT sekaligus
for ct in 100 101; do
  echo "=== CT $ct ==="
  pct exec $ct -- apt update -y
done

# Lihat penggunaan disk semua CT
pct list | awk 'NR>1 {print $1}' | while read id; do
  echo "CT $id:"; pct config $id | grep rootfs
done

# Export/Backup CT ke file tar
vzdump 100 --compress zstd --storage local --mode snapshot
```

---

## Troubleshooting Playground

| Masalah | Solusi |
|---|---|
| Container tidak bisa ping satu sama lain | Cek bridge vmbr0, pastikan semua CT di bridge yang sama |
| Nginx tidak bisa diakses dari luar | Cek `ufw` di dalam container, jalankan `ufw allow 80` |
| Clone lambat | Gunakan linked clone untuk lab (lebih cepat, hemat storage) |
| Snapshot gagal | Pastikan ada cukup ruang di storage; cek `pvesm status` |
| `pct enter` menampilkan error | Coba `pct exec 100 -- bash` sebagai alternatif |

---

## Ringkasan Skenario

| Skenario | Yang Dipelajari |
|---|---|
| Skenario 1 | Deploy Nginx di LXC, akses dari browser |
| Skenario 2 | Monitoring resource dengan htop dan pvesh |
| Skenario 3 | Snapshot → simulasi kerusakan → restore |
| Skenario 4 | Clone container untuk duplikasi cepat |
| Skenario 5 | Failover manual antar container |
| Skenario 6 | Mengubah resource CPU/RAM secara dinamis |

---

## Selesai! 🎉

Selamat! Anda telah menyelesaikan seluruh rangkaian lab:

1. ✅ [Install VirtualBox Windows](01-install-virtualbox-windows.md)
2. ✅ [Install VirtualBox macOS M-chip](02-install-virtualbox-macos-silicon.md)
3. ✅ [Import VM dari .ova](03-import-vm-ova.md)
4. ✅ [Install Docker](05-install-docker.md)
5. ✅ [Docker Playground & Nginx](06-docker-playground-nginx.md)
6. ✅ [Setup LXC di Proxmox](07-proxmox-lxc-setup.md)
7. ✅ [Playground LXC Proxmox](09-proxmox-lxc-playground.md)
