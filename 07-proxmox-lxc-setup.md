# 07 — Membuat LXC Container di Proxmox VE 9.x

**Durasi estimasi:** 30–45 menit  
**Level:** Menengah  
**Prasyarat:** Akses ke Proxmox VE 9.x (via Web UI atau SSH), hak akses Administrator/Root

---

## Tujuan Lab

- Memahami perbedaan LXC vs Virtual Machine (KVM)
- Download template CT (Container Template) di Proxmox
- Membuat LXC container via Web UI
- Membuat LXC container via CLI (pct)
- Melakukan operasi dasar manajemen LXC

---

## Konsep: LXC vs Virtual Machine

```
┌──────────────────────────────────────────────────────┐
│                  Hardware (Proxmox Host)              │
├──────────────────────────────────────────────────────┤
│                  Proxmox VE Hypervisor               │
├─────────────────────┬────────────────────────────────┤
│   KVM (Full VM)     │      LXC Container             │
│                     │                                │
│  ┌───────────────┐  │  ┌──────────┐ ┌──────────┐    │
│  │ Guest OS      │  │  │ Ubuntu   │ │ Debian   │    │
│  │ (kernel own)  │  │  │ CT 100   │ │ CT 101   │    │
│  └───────────────┘  │  └──────────┘ └──────────┘    │
│  ┌───────────────┐  │         ↕           ↕          │
│  │ Hypervisor    │  │    Shared Proxmox Kernel        │
│  └───────────────┘  │                                │
└─────────────────────┴────────────────────────────────┘
```

| Aspek | LXC Container | KVM (VM) |
|---|---|---|
| **Kernel** | Shared dengan host | Kernel sendiri |
| **Booting** | < 1 detik | 15–60 detik |
| **RAM overhead** | ~2–4 MB | ~128–256 MB |
| **Isolasi** | Namespace (OS-level) | Full hardware virtualization |
| **Cocok untuk** | Service Linux ringan | OS berbeda, Windows, custom kernel |
| **Storage** | Lebih efisien | Full disk image |

---

## Langkah 1 — Akses Proxmox Web UI

1. Buka browser, akses: `https://IP-PROXMOX:8006`
2. Login dengan:
   - **User:** `root`
   - **Password:** password root Proxmox
   - **Realm:** Linux PAM standard authentication
3. Klik **Login**

> **Catatan:** Browser akan menampilkan warning "Not secure" karena Proxmox menggunakan self-signed certificate. Klik **Advanced** → **Proceed to IP (unsafe)** untuk melanjutkan.

---

## Langkah 2 — Download Template Container

Template adalah base image untuk LXC container. Proxmox menyediakan template resmi dari berbagai distro Linux.

### 2.1 Via Web UI

1. Di panel kiri, pilih **node Proxmox** (biasanya bernama `pve` atau nama host)
2. Di bagian **local (pve)** → klik **CT Templates**
3. Klik tombol **Templates** di toolbar atas
4. Cari template yang diinginkan, contoh: `ubuntu-22.04-standard`
5. Klik **Download**
6. Tunggu proses download selesai (progress bar di pojok kiri bawah)

**Template yang disarankan untuk lab:**
- `ubuntu-22.04-standard_xxxxxx_amd64.tar.zst`
- `debian-12-standard_xxxxxx_amd64.tar.zst`

### 2.2 Via CLI (Proxmox Host)

SSH ke Proxmox host, kemudian:

```bash
# Lihat template yang tersedia
pveam available | grep ubuntu
pveam available | grep debian

# Download template Ubuntu 22.04
pveam download local ubuntu-22.04-standard_22.04-1_amd64.tar.zst

# Download template Debian 12
pveam download local debian-12-standard_12.2-1_amd64.tar.zst

# Lihat template yang sudah diunduh
pveam list local
```

---

## Langkah 3 — Buat LXC Container via Web UI

### 3.1 Buka Wizard Create CT

1. Klik tombol **Create CT** di toolbar atas kanan
2. Wizard pembuatan container akan terbuka

### 3.2 Tab: General

| Field | Nilai | Keterangan |
|---|---|---|
| Node | pve | Pilih node Proxmox |
| CT ID | 100 | ID unik container (auto-increment) |
| Hostname | ubuntu-lab-01 | Nama hostname container |
| Unprivileged container | ✅ Enabled | Lebih aman, gunakan selalu |
| Password | \*\*\*\*\* | Password root container |
| SSH public key | (opsional) | Paste public key untuk akses SSH |

Klik **Next**

### 3.3 Tab: Template

1. Storage: **local** (atau storage yang tersedia)
2. Template: Pilih `ubuntu-22.04-standard_xxxxx_amd64.tar.zst`

Klik **Next**

### 3.4 Tab: Disks

| Field | Nilai |
|---|---|
| Storage | local-lvm |
| Disk size (GiB) | 8 |

Klik **Next**

### 3.5 Tab: CPU

| Field | Nilai |
|---|---|
| Cores | 1 |
| CPU limit | (kosong = unlimited) |

Klik **Next**

### 3.6 Tab: Memory

| Field | Nilai |
|---|---|
| Memory (MiB) | 512 |
| Swap (MiB) | 512 |

Klik **Next**

### 3.7 Tab: Network

| Field | Nilai |
|---|---|
| Name | eth0 |
| Bridge | vmbr0 |
| IPv4 | DHCP (atau isi IP statis) |
| IPv6 | DHCP atau none |

> **Untuk IP statis:**  
> IPv4: `192.168.1.100/24`  
> Gateway: `192.168.1.1`

Klik **Next**

### 3.8 Tab: DNS

- Gunakan setting host (biarkan default) atau isi DNS server: `8.8.8.8`

Klik **Next**

### 3.9 Tab: Confirm

Review semua konfigurasi, kemudian:
- Centang **Start after created** jika ingin langsung menyala
- Klik **Finish**

---

## Langkah 4 — Buat LXC Container via CLI

SSH ke Proxmox host, kemudian jalankan perintah `pct create`:

```bash
# Sintaks dasar
pct create <CTID> <template> [OPTIONS]

# Contoh — Buat CT ID 101 dengan Ubuntu 22.04
pct create 101 \
  local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
  --hostname ubuntu-lab-02 \
  --password StrongPassword123! \
  --cores 1 \
  --memory 512 \
  --swap 512 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 1 \
  --features nesting=1 \
  --start 1

# Lihat daftar semua container
pct list

# Cek status container 101
pct status 101
```

**Keterangan flag:**

| Flag | Keterangan |
|---|---|
| `--hostname` | Nama host container |
| `--password` | Password root |
| `--cores` | Jumlah CPU core |
| `--memory` | RAM dalam MiB |
| `--rootfs` | Storage:ukuran disk |
| `--net0` | Konfigurasi jaringan |
| `--unprivileged` | Mode tidak berprevilese (aman) |
| `--features nesting=1` | Izinkan nested container (untuk Docker di LXC) |
| `--start` | Langsung start setelah dibuat |

---

## Langkah 5 — Operasi Dasar LXC Container

### 5.1 Start, Stop, Restart

```bash
# Via Web UI: pilih container → klik tombol Start/Stop/Restart

# Via CLI
pct start 100
pct stop 100
pct restart 100
pct shutdown 100     # graceful shutdown
```

### 5.2 Masuk ke Container

```bash
# Via Web UI: pilih container → klik Console

# Via CLI dari Proxmox host
pct enter 100

# Via SSH (jika sudah ada IP dan SSH service)
ssh root@IP-CONTAINER
```

### 5.3 Konfigurasi di Dalam Container

Setelah masuk ke container:

```bash
# Update sistem
apt update && apt upgrade -y

# Cek info sistem
uname -a
hostname
ip addr

# Cek resource
free -h
df -h
```

### 5.4 Lihat dan Edit Konfigurasi Container

```bash
# Lihat konfigurasi container
pct config 100

# Output contoh:
# arch: amd64
# cores: 1
# hostname: ubuntu-lab-01
# memory: 512
# net0: name=eth0,bridge=vmbr0,hwaddr=...,ip=dhcp,type=veth
# ostype: ubuntu
# rootfs: local-lvm:vm-100-disk-0,size=8G
# swap: 512
# unprivileged: 1
```

---

## Langkah 6 — Checkpoint: Buat Snapshot Pertama

Setelah container berhasil dibuat dan berjalan normal, buat **snapshot** sebagai titik pemulihan sebelum melakukan konfigurasi apapun:

```bash
# Buat snapshot awal
pct snapshot 100 snap-fresh-install \
  --description "Fresh install, sebelum konfigurasi apapun"

# Verifikasi
pct listsnapshot 100
```

> **Panduan lengkap** tentang mengelola resource (vCPU, RAM, disk), snapshot, restore, dan skenario operasional LXC & KVM ada di modul berikutnya:
> **[08 — Operasional Proxmox](08-proxmox-operasional.md)**

---

## Langkah 7 — Clone Container

```bash
# Clone container 100 ke ID baru 102
pct clone 100 102 --hostname ubuntu-lab-03

# Clone dengan full copy (bukan linked clone)
pct clone 100 103 --hostname ubuntu-lab-04 --full

# Start clone baru
pct start 102
```

---

## Perintah pct yang Berguna

```bash
# Daftar semua container dan statusnya
pct list

# Detail lengkap container
pct config <CTID>

# Jalankan perintah di container tanpa masuk
pct exec <CTID> -- <command>
pct exec 100 -- apt update
pct exec 100 -- systemctl status nginx

# Push file dari host ke container
pct push 100 /tmp/file.txt /root/file.txt

# Pull file dari container ke host
pct pull 100 /etc/hosts /tmp/ct100-hosts

# Hapus container (harus stop dulu)
pct stop 100
pct destroy 100
```

---

## Troubleshooting Umum

| Masalah | Solusi |
|---|---|
| Error "CT ID already in use" | Gunakan CT ID yang berbeda |
| Container tidak bisa akses internet | Cek bridge vmbr0, cek IP/gateway, cek DNS |
| Template tidak tersedia | Download template via `pveam download` atau Web UI |
| Error "cannot create a privileged container" | Gunakan `--unprivileged 1` atau login sebagai root di Proxmox |
| Container start tapi langsung stop | Cek log: `pct config <ID>` dan `journalctl -xe` |
| SSH tidak bisa terhubung | Pastikan SSH terinstall: `pct exec <ID> -- apt install -y openssh-server` |

---

## Ringkasan

✅ Template Ubuntu berhasil diunduh  
✅ LXC container berhasil dibuat via Web UI  
✅ LXC container berhasil dibuat via CLI  
✅ Operasi dasar start/stop/enter dikuasai  
✅ Clone container dikuasai  
✅ Snapshot awal dibuat sebagai checkpoint  

**Lanjutkan ke:** [08 — Operasional Proxmox](08-proxmox-operasional.md)
