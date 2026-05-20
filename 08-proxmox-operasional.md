# 08 — Operasional Proxmox: vCPU, RAM, Disk, Snapshot & Restore

**Durasi estimasi:** 45–60 menit  
**Level:** Menengah  
**Prasyarat:** Proxmox VE 9.x, akses Web UI dan SSH ke host, minimal 1 LXC Container dan/atau 1 KVM VM

---

## Tujuan Lab

- Menambah/mengubah vCPU, RAM, dan Disk pada **LXC Container**
- Menambah/mengubah vCPU, RAM, dan Disk pada **KVM Virtual Machine**
- Membuat snapshot untuk LXC dan KVM
- Restore dari snapshot
- Mengelola snapshot tree di Proxmox

---

## Perbedaan Operasional: LXC vs KVM

| Operasi | LXC Container | KVM Virtual Machine |
|---|---|---|
| Ubah vCPU | ✅ Bisa saat **running** | ⚠️ Harus **shut down** |
| Ubah RAM | ✅ Bisa saat **running** | ⚠️ Harus **shut down** |
| Perbesar disk | ✅ Bisa saat **running** | ✅ Bisa saat **running** (virtio) |
| Snapshot | ✅ Didukung (via `pct snapshot`) | ✅ Didukung (via `qm snapshot`) |
| Restore snapshot | ✅ CT harus **stop** | ✅ VM harus **stop** |
| Live migration | ✅ Didukung (cluster) | ✅ Didukung (cluster) |

---

## BAGIAN A — Operasional LXC Container

---

### A.1 Tambah / Ubah vCPU pada LXC

#### Via Web UI

1. Klik LXC container di panel kiri
2. Pilih tab **Resources**
3. Klik baris **Cores** → klik **Edit**
4. Ubah nilai **Cores** sesuai kebutuhan → klik **OK**

> LXC tidak memerlukan restart untuk perubahan CPU — berlaku langsung.

#### Via CLI

```bash
# Lihat konfigurasi cores saat ini
pct config 100 | grep cores

# Set menjadi 2 core
pct set 100 --cores 2

# Verifikasi
pct config 100 | grep cores

# Verifikasi dari dalam container
pct exec 100 -- nproc
pct exec 100 -- lscpu | grep "^CPU(s):"
```

**Panduan alokasi vCPU LXC:**

```
Total Core Host: 8

Rekomendasi alokasi:
  ├── Proxmox OS        : 2 core (reserved)
  ├── CT 100 (web)      : 2 core
  ├── CT 101 (backup)   : 1 core
  └── CT 102 (monitor)  : 1 core
                          ────────
  Total digunakan       : 6 core  ✅ Aman
```

#### CPU Limit (Throttle)

CPU limit membatasi persentase penggunaan CPU maksimum:

```bash
# Batasi CT 100 hanya boleh pakai 50% dari 2 core
pct set 100 --cpulimit 1.0

# Hapus limit (0 = unlimited)
pct set 100 --cpulimit 0
```

---

### A.2 Tambah / Ubah RAM pada LXC

#### Via Web UI

1. Klik LXC → tab **Resources**
2. Klik baris **Memory** → **Edit**
3. Ubah nilai **Memory (MiB)** dan **Swap (MiB)** → **OK**

> Perubahan RAM pada LXC berlaku **tanpa restart**.

#### Via CLI

```bash
# Lihat konfigurasi memory saat ini
pct config 100 | grep -E "memory|swap"

# Set RAM menjadi 1 GB dan swap 1 GB
pct set 100 --memory 1024 --swap 1024

# Set RAM menjadi 2 GB
pct set 100 --memory 2048

# Verifikasi dari dalam container
pct exec 100 -- free -h

# Output contoh:
#               total   used   free
# Mem:           2.0G   120M   1.9G
# Swap:          1.0G     0B   1.0G
```

**Panduan alokasi RAM LXC:**

| Fungsi Container | RAM Minimum | RAM Rekomendasi |
|---|---|---|
| Web server (Nginx) | 128 MB | 256–512 MB |
| Web server + PHP | 256 MB | 512 MB–1 GB |
| Database (MariaDB) | 512 MB | 1–2 GB |
| App server (Node/Python) | 256 MB | 512 MB–1 GB |
| Monitoring (Zabbix agent) | 64 MB | 128 MB |

---

### A.3 Perbesar Disk LXC

#### Via Web UI

1. Klik LXC → tab **Resources**
2. Klik baris **Root Disk** → klik **Resize disk** (ikon panah)
3. Masukkan jumlah yang ingin **ditambahkan** (bukan total)

   ```
   ┌─────────────────────────────────┐
   │ Resize disk: rootfs (local-lvm) │
   │                                 │
   │ Size Increment (GiB): [+4    ]  │
   │                                 │
   │           [Cancel] [  Resize ]  │
   └─────────────────────────────────┘
   ```
   
   > Jika disk sebelumnya 8 GB dan diisi +4, maka total menjadi 12 GB.

#### Via CLI

```bash
# Lihat ukuran disk saat ini
pct config 100 | grep rootfs

# Tambah 4 GB ke disk root
pct resize 100 rootfs +4G

# Tambah 10 GB
pct resize 100 rootfs +10G

# Verifikasi (dari dalam container)
pct exec 100 -- df -h /
```

> ✅ Untuk LXC, filesystem otomatis diperluas — **tidak perlu** `resize2fs` manual seperti di VirtualBox.

---

### A.4 Snapshot LXC Container

#### Via Web UI

1. Klik LXC → tab **Snapshots**
2. Klik tombol **Take Snapshot**

   ```
   ┌──────────────────────────────────────────┐
   │ Take Snapshot                            │
   │                                          │
   │ Name:         [snap-nginx-clean        ] │
   │ Description:  [Nginx berjalan, fresh   ] │
   │               [install sebelum config  ] │
   │                                          │
   │ ☐ Include RAM  (hanya untuk VM yang     │
   │                 sedang berjalan)         │
   │                                          │
   │             [Cancel]  [ Take Snapshot ] │
   └──────────────────────────────────────────┘
   ```

3. Klik **Take Snapshot**

#### Via CLI

```bash
# Buat snapshot LXC container 100
pct snapshot 100 snap-nginx-clean \
  --description "Nginx berjalan normal, sebelum modifikasi konfigurasi"

# Buat snapshot dengan timestamp otomatis
pct snapshot 100 snap-$(date +%Y%m%d-%H%M) \
  --description "Auto snapshot $(date)"

# Lihat semua snapshot
pct listsnapshot 100

# Output contoh:
#              PARENT NAME                      DESCRIPTION
#                     current
# snap-nginx-clean                              Nginx berjalan normal...
# snap-20250601-1430                            Auto snapshot...
```

#### Konvensi Nama Snapshot

```
Format yang disarankan:
  snap-[kondisi]-[tanggal]

Contoh:
  snap-fresh-install
  snap-post-nginx-install
  snap-pre-config-change
  snap-working-20250601
  snap-before-update
```

---

### A.5 Restore Snapshot LXC

#### Via Web UI

1. Klik LXC → tab **Snapshots**
2. Pilih snapshot yang dituju dari daftar
3. Klik **Rollback**

   ```
   ┌──────────────────────────────────────────────┐
   │ ⚠️  WARNING                                   │
   │                                              │
   │ Are you sure you want to rollback to         │
   │ snapshot 'snap-nginx-clean'?                 │
   │                                              │
   │ This will STOP the container and revert      │
   │ all changes made after the snapshot.         │
   │                                              │
   │              [Cancel]  [  Rollback  ]        │
   └──────────────────────────────────────────────┘
   ```

4. Klik **Rollback** → container akan di-stop otomatis, lalu di-restore

#### Via CLI

```bash
# Stop container dulu
pct stop 100

# Rollback ke snapshot
pct rollback 100 snap-nginx-clean

# Start container setelah restore
pct start 100

# Verifikasi kondisi setelah restore
pct exec 100 -- systemctl status nginx
```

#### Hapus Snapshot

```bash
# Via Web UI: tab Snapshots → pilih snapshot → klik Delete

# Via CLI
pct delsnapshot 100 snap-20250601-1430

# Lihat snapshot yang tersisa
pct listsnapshot 100
```

---

## BAGIAN B — Operasional KVM Virtual Machine

> Bagian ini untuk Anda yang mengelola **KVM VM** di Proxmox (bukan LXC container). KVM VM dikelola dengan perintah `qm` (QEMU Manager).

---

### B.1 Buat KVM VM (Ringkasan Cepat)

Jika belum punya KVM VM, buat melalui Web UI:

1. Klik **Create VM** di toolbar
2. Isi: VM ID (mis. 200), Name (mis. `kvm-ubuntu`)
3. Tab **OS**: upload/pilih ISO Ubuntu
4. Tab **System**: Machine Q35, BIOS OVMF (UEFI) atau SeaBIOS
5. Tab **Disks**: VirtIO Block, 20 GB, storage `local-lvm`
6. Tab **CPU**: 2 sockets, 1 core, Type host
7. Tab **Memory**: 2048 MiB
8. Tab **Network**: Model VirtIO, Bridge vmbr0
9. Klik **Finish**

---

### B.2 Tambah / Ubah vCPU pada KVM VM

#### Via Web UI

1. **Shutdown VM** terlebih dahulu
2. Klik VM → tab **Hardware**
3. Klik baris **Processors** → klik **Edit**

   ```
   ┌────────────────────────────────────┐
   │ Edit: Processors                   │
   │                                    │
   │ Sockets:   [ 1  ▼]                │
   │ Cores:     [ 2  ▼]                │
   │ VCPUs:     [ 2  ] (sockets×cores) │
   │                                    │
   │ Type:      [ host          ▼]     │
   │ ☑ Enable NUMA                      │
   │                                    │
   │          [Cancel]  [  OK  ]        │
   └────────────────────────────────────┘
   ```

4. Klik **OK** → start VM kembali

#### Via CLI

```bash
# Lihat konfigurasi CPU VM saat ini
qm config 200 | grep -E "cores|sockets|vcpus"

# Set menjadi 2 core
qm set 200 --cores 2

# Set menjadi 2 socket × 2 core = 4 vCPU
qm set 200 --sockets 2 --cores 2

# Verifikasi
qm config 200 | grep -E "cores|sockets"
```

**Verifikasi di dalam VM:**

```bash
# SSH ke VM atau buka console
nproc
lscpu
```

---

### B.3 Tambah / Ubah RAM pada KVM VM

#### Via Web UI

1. **Shutdown VM**
2. Klik VM → tab **Hardware**
3. Klik baris **Memory** → **Edit**
4. Ubah nilai **Memory (MiB)** → **OK**
5. Start VM kembali

#### Via CLI

```bash
# Lihat konfigurasi memory saat ini
qm config 200 | grep memory

# Set RAM menjadi 4 GB
qm set 200 --memory 4096

# Set RAM minimum dan maksimum (dengan ballooning)
qm set 200 --memory 4096 --balloon 1024
# memory = maksimum, balloon = minimum yang digunakan

# Verifikasi
qm config 200 | grep memory
```

> **Memory Ballooning:** Fitur Proxmox yang memungkinkan VM melepas RAM yang tidak dipakai kembali ke host. Berguna di lingkungan dengan banyak VM.

---

### B.4 Perbesar Disk KVM VM

#### Via Web UI

1. Klik VM → tab **Hardware**
2. Klik baris disk (mis. `scsi0`) → klik **Resize** di toolbar
3. Masukkan jumlah yang **ditambahkan**

   ```
   ┌─────────────────────────────────────┐
   │ Resize disk: scsi0 (local-lvm)      │
   │                                     │
   │ Size Increment (GiB): [+10       ]  │
   │                                     │
   │            [Cancel]  [  Resize  ]   │
   └─────────────────────────────────────┘
   ```

#### Via CLI

```bash
# Lihat disk VM
qm config 200 | grep -E "scsi|virtio|ide"

# Tambah 10 GB ke disk scsi0
qm resize 200 scsi0 +10G

# Verifikasi
qm config 200 | grep scsi0
```

#### Extend Partisi di Dalam VM (KVM)

Berbeda dengan LXC, KVM VM memerlukan langkah manual untuk extend partisi:

```bash
# SSH atau masuk console VM

# Cek partisi saat ini
lsblk
df -h

# Install growpart jika belum ada
sudo apt install -y cloud-guest-utils

# Extend partisi 1 pada disk sda
sudo growpart /dev/sda 1

# Resize filesystem ext4
sudo resize2fs /dev/sda1

# Untuk LVM (jika menggunakan LVM di dalam VM)
sudo pvresize /dev/sda3
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv

# Verifikasi
df -h
```

---

### B.5 Snapshot KVM VM

#### Via Web UI

1. Klik VM → tab **Snapshots**
2. Klik **Take Snapshot**
3. Isi nama dan deskripsi

   > Untuk KVM, Anda bisa memilih **Include RAM** — snapshot akan menyimpan isi memory sehingga VM bisa di-resume persis seperti sebelum snapshot (seperti hibernate).

4. Klik **Take Snapshot**

#### Via CLI

```bash
# Buat snapshot VM 200
qm snapshot 200 snap-fresh-install \
  --description "VM baru install, sebelum konfigurasi"

# Snapshot dengan memory state (VM harus running)
qm snapshot 200 snap-with-ram \
  --description "Snapshot dengan kondisi RAM" \
  --vmstate 1

# Lihat daftar snapshot
qm listsnapshot 200

# Output contoh:
#              PARENT NAME           DESCRIPTION
#                     current
# snap-fresh-install                 VM baru install...
#    snap-post-nginx                 Setelah install Nginx
```

---

### B.6 Restore Snapshot KVM VM

#### Via Web UI

1. Klik VM → tab **Snapshots**
2. Pilih snapshot → klik **Rollback**
3. Konfirmasi rollback

#### Via CLI

```bash
# Stop VM dulu
qm stop 200

# Rollback ke snapshot
qm rollback 200 snap-fresh-install

# Start VM kembali
qm start 200

# Verifikasi VM berjalan normal
qm status 200
```

#### Hapus Snapshot KVM

```bash
# Hapus snapshot tertentu
qm delsnapshot 200 snap-dengan-ram

# Lihat snapshot yang tersisa
qm listsnapshot 200
```

---

## BAGIAN C — Praktik Gabungan: Skenario Operasional Harian

### Skenario 1 — Upgrade Resource Sebelum Instalasi Software Berat

```bash
# Langkah 1: Buat snapshot "sebelum upgrade"
pct snapshot 100 snap-pre-upgrade \
  --description "Sebelum tambah resource dan install software"

# Langkah 2: Tambah RAM dari 512 MB ke 1 GB
pct set 100 --memory 1024

# Langkah 3: Tambah CPU dari 1 ke 2 core
pct set 100 --cores 2

# Langkah 4: Perbesar disk +4 GB
pct resize 100 rootfs +4G

# Langkah 5: Update dan install software di container
pct exec 100 -- apt update
pct exec 100 -- apt install -y nginx php-fpm

# Langkah 6: Buat snapshot setelah instalasi berhasil
pct snapshot 100 snap-post-nginx-php \
  --description "Nginx + PHP-FPM terinstall dan berjalan"

# Verifikasi
pct listsnapshot 100
```

### Skenario 2 — Simulasi Disaster Recovery

```bash
# Kondisi awal: snapshot "snap-nginx-clean" sudah ada

# Langkah 1: Simulasikan kerusakan konfigurasi
pct exec 100 -- bash -c "echo 'ERROR CONFIG' > /etc/nginx/sites-enabled/default"
pct exec 100 -- systemctl restart nginx
# Nginx gagal start — simulasi masalah produksi

# Langkah 2: Cek waktu yang dibutuhkan untuk restore
time pct rollback 100 snap-nginx-clean
pct start 100

# Langkah 3: Verifikasi layanan kembali normal
pct exec 100 -- systemctl status nginx
curl http://$(pct exec 100 -- hostname -I | awk '{print $1}')
```

### Skenario 3 — Batch Snapshot Semua Container

```bash
# Script untuk snapshot semua container sekaligus
SNAP_NAME="snap-$(date +%Y%m%d)"
SNAP_DESC="Snapshot rutin $(date '+%d %b %Y')"

for CT_ID in $(pct list | awk 'NR>1 {print $1}'); do
  echo "Membuat snapshot untuk CT $CT_ID..."
  pct snapshot $CT_ID $SNAP_NAME --description "$SNAP_DESC" 2>&1
done

echo "Selesai! Daftar snapshot:"
for CT_ID in $(pct list | awk 'NR>1 {print $1}'); do
  echo "--- CT $CT_ID ---"
  pct listsnapshot $CT_ID
done
```

### Skenario 4 — Lihat Ringkasan Resource Semua Container

```bash
# Script monitoring resource semua container
echo "╔══════════════════════════════════════════════════════════╗"
echo "║           RESOURCE SUMMARY — $(date '+%d/%m/%Y %H:%M')           ║"
echo "╠═══════╦══════════╦════════════╦═════════╦══════════════╣"
printf "║ %-5s ║ %-8s ║ %-10s ║ %-7s ║ %-12s ║\n" "CTID" "STATUS" "NAME" "CPU" "RAM"
echo "╠═══════╬══════════╬════════════╬═════════╬══════════════╣"

pct list | awk 'NR>1' | while read ctid status name; do
  CORES=$(pct config $ctid 2>/dev/null | grep "^cores:" | awk '{print $2}')
  MEM=$(pct config $ctid 2>/dev/null | grep "^memory:" | awk '{print $2}')
  CORES=${CORES:-"-"}
  MEM=${MEM:-"-"}
  printf "║ %-5s ║ %-8s ║ %-10s ║ %-7s ║ %-9s MiB ║\n" \
    "$ctid" "$status" "${name:0:10}" "${CORES}c" "$MEM"
done

echo "╚═══════╩══════════╩════════════╩═════════╩══════════════╝"
```

---

## Referensi Perintah Lengkap

### Perintah `pct` (LXC)

```bash
pct list                              # Daftar semua container
pct config <ID>                       # Lihat konfigurasi
pct set <ID> --cores <N>              # Set vCPU
pct set <ID> --memory <MiB>           # Set RAM
pct set <ID> --cpulimit <N>           # Batasi CPU (0=unlimited)
pct resize <ID> rootfs +<N>G          # Perbesar disk
pct snapshot <ID> <name>              # Buat snapshot
pct listsnapshot <ID>                 # Lihat daftar snapshot
pct rollback <ID> <name>              # Restore snapshot
pct delsnapshot <ID> <name>           # Hapus snapshot
pct start/stop/restart <ID>           # Kontrol container
pct exec <ID> -- <command>            # Jalankan command
```

### Perintah `qm` (KVM VM)

```bash
qm list                               # Daftar semua VM
qm config <ID>                        # Lihat konfigurasi
qm set <ID> --cores <N>               # Set vCPU
qm set <ID> --memory <MiB>            # Set RAM
qm resize <ID> <disk> +<N>G           # Perbesar disk
qm snapshot <ID> <name>               # Buat snapshot
qm listsnapshot <ID>                  # Lihat daftar snapshot
qm rollback <ID> <name>               # Restore snapshot
qm delsnapshot <ID> <name>            # Hapus snapshot
qm start/stop/shutdown/reset <ID>     # Kontrol VM
qm status <ID>                        # Cek status
```

---

## Troubleshooting Umum

| Masalah | Solusi |
|---|---|
| Snapshot gagal dengan error "lock" | Tunggu operasi lain selesai; cek `pct unlock <ID>` |
| Rollback LXC lambat | Normal untuk disk besar; tunggu hingga selesai |
| KVM VM tidak bisa resize saat running | Gunakan disk type VirtIO; pastikan guest agent terinstall |
| Disk di dalam VM tidak bertambah setelah resize | Perlu `growpart` dan `resize2fs` manual di dalam VM |
| Snapshot KVM gagal (storage penuh) | Cek `pvesm status`; hapus snapshot lama yang tidak perlu |
| Error "CT is locked" saat snapshot | Jalankan `pct unlock <ID>` lalu coba lagi |
| RAM LXC tidak berubah setelah `pct set` | Cek dengan `pct exec <ID> -- free -h`; mungkin perlu reboot CT |

---

## Ringkasan

✅ Ubah vCPU LXC dan KVM (online dan offline)  
✅ Ubah RAM LXC dan KVM  
✅ Perbesar disk LXC (otomatis) dan KVM (perlu growpart)  
✅ Buat dan kelola snapshot LXC via UI dan CLI  
✅ Buat dan kelola snapshot KVM via UI dan CLI  
✅ Restore snapshot LXC dan KVM  
✅ Skenario praktis: upgrade resource, disaster recovery, batch snapshot  

**Lanjutkan ke:** [09 — Skenario Playground Proxmox LXC](09-proxmox-lxc-playground.md)

**Lihat juga:** [04 — Operasional VirtualBox](04-virtualbox-operasional.md)
