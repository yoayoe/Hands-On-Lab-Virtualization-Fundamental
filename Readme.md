# 📚 Hands-On Lab — Virtualisasi & Docker
### IT Operations Unit

---

## Tentang Lab Ini

Panduan hands-on untuk membangun environment lab virtualisasi menggunakan **VirtualBox**, **Docker**, dan **Proxmox VE**. Disusun secara bertahap dari instalasi hingga skenario operasional nyata.

**Total estimasi waktu:** ± 6–8 jam  
**Level:** Pemula hingga Menengah  
**Prasyarat:** Laptop dengan RAM minimal 8 GB, akses ke server Proxmox VE 9.x

---

## Alur Belajar

```
BAGIAN 1 — FONDASI VIRTUALBOX
┌─────────────────────────────────────────────────────────────┐
│  01  Install VirtualBox        (Windows 10/11)              │
│  02  Install VirtualBox        (macOS Apple Silicon M1/M2)  │
│  03  Import VM dari file .ova                               │
│  04  Operasional VM            (SSH, vCPU, RAM, Snapshot)   │
└───────────────────────────┬─────────────────────────────────┘
                            │
BAGIAN 2 — DOCKER           ▼
┌─────────────────────────────────────────────────────────────┐
│  05  Install Docker Engine    (di dalam VM Linux)           │
│  06  Docker Playground        (Nginx web server)            │
└───────────────────────────┬─────────────────────────────────┘
                            │
BAGIAN 3 — PROXMOX LXC      ▼
┌─────────────────────────────────────────────────────────────┐
│  07  Membuat LXC Container    (Setup Proxmox VE 9.x)        │
│  08  Operasional Proxmox      (vCPU, RAM, Snapshot)         │
│  09  Playground LXC           (Skenario operasional nyata)  │
└─────────────────────────────────────────────────────────────┘
```

---

## Daftar Modul

### 🖥️ Bagian 1 — VirtualBox

| No | Judul | Durasi | Level |
|---|---|---|---|
| [01](01-install-virtualbox-windows.md) | Install VirtualBox di Windows 10/11 | 30–45 mnt | Pemula |
| [02](02-install-virtualbox-macos-silicon.md) | Install VirtualBox di macOS Apple Silicon | 30–45 mnt | Pemula |
| [03](03-import-vm-ova.md) | Import VM dari File .ova | 15–30 mnt | Pemula |
| [04](04-virtualbox-operasional.md) | Operasional VM — Host-Only NIC, SSH, vCPU, RAM, Snapshot | 30–45 mnt | Pemula–Menengah |

### 🐳 Bagian 2 — Docker

| No | Judul | Durasi | Level |
|---|---|---|---|
| [05](05-install-docker.md) | Install Docker Engine di Linux VM | 20–30 mnt | Pemula |
| [06](06-docker-playground-nginx.md) | Docker Playground & Aplikasi Web Nginx | 45–60 mnt | Pemula–Menengah |

### ⚙️ Bagian 3 — Proxmox LXC

| No | Judul | Durasi | Level |
|---|---|---|---|
| [07](07-proxmox-lxc-setup.md) | Membuat LXC Container di Proxmox VE 9.x | 30–45 mnt | Menengah |
| [08](08-proxmox-operasional.md) | Operasional Proxmox — vCPU, RAM, Snapshot, Restore | 45–60 mnt | Menengah |
| [09](09-proxmox-lxc-playground.md) | Skenario Playground Proxmox LXC | 60–90 mnt | Menengah |

---

## Ringkasan Materi per Bagian

### Bagian 1 — VirtualBox (Modul 01–04)

Peserta belajar menyiapkan environment virtualisasi di laptop masing-masing. Dimulai dari mengaktifkan fitur virtualisasi di BIOS, instalasi VirtualBox, import VM siap pakai dari file `.ova`, lalu mengaktifkan koneksi SSH via Host-Only Adapter dan Netplan, hingga mengelola resource VM (tambah vCPU, RAM, disk) dan membuat titik pemulihan dengan snapshot.

### Bagian 2 — Docker (Modul 05–06)

Peserta menginstal Docker Engine di dalam VM Linux, memahami konsep image dan container, lalu langsung praktik deploy web server Nginx. Materi mencakup Docker Volume, port mapping, Dockerfile, dan Docker Compose untuk skenario multi-container.

### Bagian 3 — Proxmox LXC (Modul 07–09)

Peserta bekerja langsung di server Proxmox VE 9.x. Dimulai dari membuat LXC container via Web UI dan CLI, mengelola resource container secara dinamis, membuat dan merestore snapshot, hingga skenario praktis seperti failover manual, clone container, dan batch snapshot.

---

## Prasyarat Teknis

| Komponen | Spesifikasi Minimum |
|---|---|
| **Laptop peserta** | RAM 8 GB, Storage kosong 30 GB, CPU dengan VT-x/AMD-V |
| **OS Laptop** | Windows 10/11 (64-bit) atau macOS Ventura ke atas |
| **VirtualBox** | Versi 7.1 ke atas |
| **VM Guest** | Ubuntu 22.04 LTS (file .ova disediakan instruktur) |
| **Server Proxmox** | Proxmox VE 9.x, akses Web UI `https://IP:8006` |
| **Jaringan** | Akses internet untuk download image Docker |

---

## Tips Belajar

Buat snapshot VM sebelum setiap modul baru dimulai. Dengan snapshot, Anda bisa kembali ke kondisi awal kapan saja tanpa harus mengulang instalasi dari awal.

Gunakan SSH ke VM daripada bekerja langsung di jendela VirtualBox — pengalaman terminal jauh lebih nyaman dan memungkinkan copy-paste perintah.

Jika sebuah langkah gagal, baca pesan error dengan seksama dan cek bagian **Troubleshooting** di setiap modul sebelum meminta bantuan.

---

*Lab ini disusun untuk kebutuhan internal IT Operations Unit.*
