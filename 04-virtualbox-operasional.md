# 04 — Operasional VM di VirtualBox

**Durasi estimasi:** 30–45 menit  
**Level:** Pemula–Menengah  
**Prasyarat:** VirtualBox terinstal, minimal satu VM sudah dibuat/diimport

---

## Tujuan Lab

- Menambah adapter jaringan baru (Host-Only Adapter) agar VM bisa diakses via SSH
- Mengkonfigurasi Netplan di Ubuntu agar adapter baru aktif dengan DHCP
- Menambah dan mengurangi vCPU pada VM
- Menambah dan mengurangi alokasi RAM pada VM
- Memperbesar ukuran disk VM
- Membuat snapshot VM (titik pemulihan)
- Restore VM dari snapshot
- Mengelola snapshot tree (bercabang)

---

## Aturan Dasar Sebelum Mulai

> ⚠️ **Sebagian besar perubahan hardware VM (vCPU, RAM, disk) hanya bisa dilakukan saat VM dalam kondisi MATI (Powered Off).**

| Operasi | VM Harus Mati? |
|---|---|
| Tambah adapter Host-Only | ✅ Ya |
| Konfigurasi Netplan | ❌ Bisa saat berjalan |
| Ubah jumlah vCPU | ✅ Ya |
| Ubah alokasi RAM | ✅ Ya |
| Perbesar disk VM | ✅ Ya |
| Buat snapshot | ❌ Bisa saat berjalan |
| Restore snapshot | ✅ Ya (VM akan di-stop otomatis) |
| Hapus snapshot | ❌ Bisa saat berjalan |

---

## Bagian 1 — Menambah Adapter Jaringan Baru (Host-Only Adapter)

Host-Only Adapter membuat jaringan terisolasi antara VM dan host (laptop). VM tidak bisa mengakses internet melalui adapter ini, tapi bisa berkomunikasi langsung dengan host — cocok untuk akses SSH yang stabil tanpa bergantung pada konfigurasi jaringan luar.

### Skenario Umum: VM dengan 2 Adapter

```
┌──────────────────────────────────────────────────────┐
│                    VM Ubuntu                          │
│                                                       │
│  eth0 / enp0s3 ──── NAT         → Internet           │
│  eth1 / enp0s8 ──── Host-Only   → Host (laptop)      │
└──────────────────────────────────────────────────────┘
         ↕ SSH via IP statis Host-Only
       Laptop (192.168.56.1)
```

> Dengan konfigurasi ini, SSH ke VM bisa menggunakan IP Host-Only yang tetap (tidak berubah), sementara VM tetap bisa mengakses internet via NAT untuk update dan download.

---

### 1.1 Buat Host-Only Network di VirtualBox

Sebelum menambahkan adapter ke VM, pastikan Host-Only Network sudah ada.

**Via GUI:**

1. Menu **File** → **Tools** → **Network Manager**
2. Tab **Host-only Networks**
3. Klik **Create** — VirtualBox akan membuat `vboxnet0` (Linux/macOS) atau `VirtualBox Host-Only Ethernet Adapter` (Windows)
4. Konfigurasi default:
   - **IPv4 Address:** `192.168.56.1` (IP host/laptop)
   - **IPv4 Network Mask:** `255.255.255.0`
   - Tab **DHCP Server** → centang **Enable Server**
   - **Server Address:** `192.168.56.100`
   - **Lower Address Bound:** `192.168.56.101`
   - **Upper Address Bound:** `192.168.56.254`
5. Klik **Apply**

**Via CLI:**

```bash
# Windows
"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" hostonlyif create
"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" hostonlyif ipconfig "VirtualBox Host-Only Ethernet Adapter" --ip 192.168.56.1 --netmask 255.255.255.0

# macOS / Linux
VBoxManage hostonlyif create
VBoxManage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1 --netmask 255.255.255.0

# Aktifkan DHCP server pada Host-Only network
VBoxManage dhcpserver add --ifname vboxnet0 \
  --ip 192.168.56.100 --netmask 255.255.255.0 \
  --lowerip 192.168.56.101 --upperip 192.168.56.254 \
  --enable
```

---

### 1.2 Tambah Adapter Host-Only ke VM

> ⚠️ VM harus dalam kondisi **Powered Off**.

**Via GUI:**

1. Pilih VM → klik **Settings** → menu **Network**
2. Klik tab **Adapter 2**
3. Centang **Enable Network Adapter**
4. **Attached to:** `Host-only Adapter`
5. **Name:** pilih `vboxnet0` (Linux/macOS) atau `VirtualBox Host-Only Ethernet Adapter` (Windows)
6. Klik **OK**

**Via CLI:**

```bash
# Tambah Adapter 2 sebagai Host-Only pada VM bernama "ubuntu-lab"
VBoxManage modifyvm "ubuntu-lab" \
  --nic2 hostonly \
  --hostonlyadapter2 vboxnet0

# Windows — nama adapter berbeda
VBoxManage modifyvm "ubuntu-lab" \
  --nic2 hostonly \
  --hostonlyadapter2 "VirtualBox Host-Only Ethernet Adapter"

# Verifikasi konfigurasi NIC
VBoxManage showvminfo "ubuntu-lab" | grep -E "NIC|Adapter"
```

---

### 1.3 Verifikasi Adapter Terdeteksi di Dalam VM

Jalankan VM, login, kemudian:

```bash
# Lihat semua interface jaringan
ip addr show

# Output yang diharapkan — ada dua interface:
# 1: lo       → loopback
# 2: enp0s3   → Adapter 1 (NAT)
# 3: enp0s8   → Adapter 2 (Host-Only) ← baru

# Nama interface bisa berbeda tergantung VM, cek dengan:
ip link show
```

Jika `enp0s8` (atau nama lain) muncul tapi belum punya IP, lanjut ke Bagian 2 untuk konfigurasi Netplan.

---

## Bagian 2 — Konfigurasi Netplan di Ubuntu (DHCP pada Adapter Baru)

Ubuntu 18.04 ke atas menggunakan **Netplan** sebagai sistem konfigurasi jaringan. File konfigurasi berbasis YAML dan terletak di `/etc/netplan/`.

### Memahami Struktur Netplan

```
/etc/netplan/
├── 00-installer-config.yaml   ← dibuat otomatis saat install Ubuntu
└── 50-cloud-init.yaml         ← atau nama ini jika dari cloud image
```

> Nama file bisa berbeda. Cek dengan `ls /etc/netplan/` terlebih dahulu.

---

### 2.1 Cek Konfigurasi Netplan Saat Ini

```bash
# Lihat file yang ada
ls -la /etc/netplan/

# Lihat isi file konfigurasi
cat /etc/netplan/00-installer-config.yaml
```

Contoh isi default (hanya enp0s3/NAT):

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
```

---

### 2.2 Tambahkan Konfigurasi Adapter Kedua (DHCP)

**Metode A — Edit file yang sudah ada:**

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Ubah isi menjadi:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: true
```

> **Perhatian format YAML:**
> - Gunakan spasi (bukan tab)
> - Indentasi harus konsisten (2 spasi per level)
> - Nama interface harus persis sesuai output `ip link show`

**Metode B — Buat file konfigurasi baru (lebih rapi):**

```bash
sudo nano /etc/netplan/99-host-only.yaml
```

Isi file:

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: true
      dhcp4-overrides:
        use-routes: false   # jangan pakai default route dari adapter ini
```

> Opsi `use-routes: false` penting agar routing default tetap melalui `enp0s3` (NAT), bukan `enp0s8` (Host-Only). Tanpa ini, akses internet bisa terganggu.

---

### 2.3 Terapkan Konfigurasi Netplan

```bash
# Validasi sintaks YAML dulu sebelum diterapkan
sudo netplan generate

# Jika tidak ada error, terapkan konfigurasi
sudo netplan apply

# Verifikasi IP pada adapter baru
ip addr show enp0s8
```

Output yang diharapkan:

```
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP>
    link/ether 08:00:27:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.101/24 brd 192.168.56.255 scope global dynamic enp0s8
       valid_lft 600sec preferred_lft 600sec
```

IP `192.168.56.101` berasal dari DHCP Server Host-Only yang sudah dikonfigurasi di VirtualBox.

---

### 2.4 Test Koneksi SSH dari Laptop ke VM

```bash
# Dari terminal laptop (Windows PowerShell / macOS Terminal)
ssh ubuntu@192.168.56.101

# Atau jika IP berbeda, cek dulu dengan:
# (di dalam VM) hostname -I
```

Setelah berhasil terhubung via SSH Host-Only, koneksi ini lebih stabil dibanding port forwarding karena IP tidak berubah selama VirtualBox Host-Only network aktif.

---

### 2.5 Konfigurasi IP Statis pada Host-Only (Opsional)

Jika ingin IP yang selalu tetap (tidak bergantung DHCP):

```yaml
# /etc/netplan/99-host-only.yaml
network:
  version: 2
  ethernets:
    enp0s8:
      addresses:
        - 192.168.56.10/24   # IP statis pilihan Anda
      dhcp4: false
      routes: []              # jangan tambah default route
```

```bash
sudo netplan apply

# Verifikasi
ip addr show enp0s8
# Harus menampilkan 192.168.56.10
```

---

## Bagian 3 — Menambah / Mengubah vCPU

### 3.1 Via GUI VirtualBox

1. Pastikan VM dalam kondisi **Powered Off**  
   (klik kanan VM → **Close** → **Power Off**)

2. Klik VM di sidebar → klik tombol **Settings** (ikon gear)

3. Pilih menu **System** → tab **Processor**

4. Geser slider **Processor(s)** ke jumlah yang diinginkan

   ```
   ┌─────────────────────────────────────────┐
   │ Processor(s):  [===●════] 2             │
   │                                         │
   │ Execution Cap: [=========●] 100%        │
   │                                         │
   │ ☑ Enable PAE/NX                         │
   │ ☑ Enable Nested VT-x/AMD-V              │
   └─────────────────────────────────────────┘
   ```

   **Panduan alokasi vCPU:**

   | RAM Laptop | Max vCPU untuk VM |
   |---|---|
   | 4 core | Maks 2 vCPU per VM |
   | 6 core | Maks 3 vCPU per VM |
   | 8 core | Maks 4 vCPU per VM |

   > Jangan melebihi 50% dari total core fisik untuk satu VM agar performa host tetap stabil.

5. Klik **OK** → jalankan VM kembali

### 3.2 Via Command Line (VBoxManage)

```bash
# Windows
"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" modifyvm "NamaVM" --cpus 2

# macOS / Linux
VBoxManage modifyvm "NamaVM" --cpus 2

# Verifikasi perubahan
VBoxManage showvminfo "NamaVM" | grep "Number of CPUs"
```

### 3.3 Verifikasi di Dalam VM

Setelah VM dinyalakan kembali, login dan cek:

```bash
# Lihat jumlah CPU yang terdeteksi
nproc
# atau
lscpu | grep "^CPU(s):"

# Detail lebih lengkap
cat /proc/cpuinfo | grep "processor" | wc -l
```

---

## Bagian 4 — Menambah / Mengubah RAM

### 4.1 Via GUI VirtualBox

1. Pastikan VM **Powered Off**

2. Buka **Settings** → **System** → tab **Motherboard**

3. Geser slider **Base Memory** ke nilai yang diinginkan

   ```
   ┌──────────────────────────────────────────────────┐
   │ Base Memory:                                     │
   │                                                  │
   │  4 MB ──────────────●──────────── 16384 MB      │
   │                   2048 MB                        │
   │                                                  │
   │  ████████████████████░░░░░░░░░░░░░░░░░░░         │
   │  ↑ Zona Hijau (Aman)  ↑ Zona Merah (Terlalu banyak)│
   └──────────────────────────────────────────────────┘
   ```

   **Panduan alokasi RAM:**

   | RAM Laptop | RAM VM (Rekomendasi) | Maksimum |
   |---|---|---|
   | 8 GB | 2–3 GB | 4 GB |
   | 16 GB | 4–6 GB | 8 GB |
   | 32 GB | 8–12 GB | 16 GB |

   > Slider akan berubah warna merah jika alokasi terlalu besar. Jaga agar tetap di zona hijau/kuning.

4. Klik **OK** → jalankan VM

### 4.2 Via Command Line (VBoxManage)

```bash
# Set RAM menjadi 2048 MB (2 GB)
VBoxManage modifyvm "NamaVM" --memory 2048

# Set RAM menjadi 4096 MB (4 GB)
VBoxManage modifyvm "NamaVM" --memory 4096

# Cek konfigurasi
VBoxManage showvminfo "NamaVM" | grep "Memory size"
```

### 4.3 Verifikasi di Dalam VM

```bash
# Lihat total RAM yang terdeteksi
free -h

# Output contoh:
#               total   used   free
# Mem:           1.9G   856M   1.1G
# Swap:          2.0G     0B   2.0G

# Atau lebih detail
cat /proc/meminfo | head -5
```

---

## Bagian 5 — Memperbesar Disk VM

> Memperkecil disk **tidak didukung** oleh VirtualBox. Hanya bisa diperbesar.

---

### ⚠️ Wajib Dilakukan Pertama: Hapus Snapshot Sebelum Resize

**Jika VM memiliki snapshot aktif, resize disk kemungkinan besar tidak akan bekerja — VirtualBox menampilkan ukuran baru, tapi OS di dalam VM tetap melihat ukuran lama.**

#### Mengapa Snapshot Menghalangi Resize?

Saat VM memiliki snapshot, VirtualBox tidak lagi menulis langsung ke disk asli (base VDI). Sebagai gantinya, VirtualBox membuat **differencing disk** — file disk terpisah yang hanya menyimpan **selisih perubahan (delta)** dari kondisi saat snapshot diambil.

```
Base VDI (10 GB) ──── dibuat saat import VM
        │
        └── [Snapshot: "Fresh Import - Sebelum Konfigurasi"]
                │
                └── Differencing Disk (delta) ──── kondisi VM saat ini
```

Ketika Anda resize di VirtualBox:
- `VBoxManage modifyhd --resize` **tidak bisa mengubah base VDI** selama masih ada differencing disk (child disk) yang bergantung padanya
- Resize via GUI mungkin terlihat berhasil, tapi sebenarnya hanya memperbarui metadata pada differencing disk
- OS di dalam VM membaca block device yang berakar dari **base VDI yang belum berubah**
- Hasilnya: VirtualBox menampilkan 12 GB, tetapi `lsblk` di dalam VM tetap menunjukkan 10 GB

Anda bisa memverifikasi kondisi ini dari **Settings → Storage → pilih disk**. Jika kolom **Storage details** tertulis:

```
Dynamically allocated differencing storage   ← ada snapshot aktif, resize akan gagal
Dynamically allocated storage                ← tidak ada snapshot, aman untuk resize
```

---

#### Langkah: Hapus Snapshot Sebelum Resize

> ⚠️ VM harus dalam kondisi **Powered Off** sebelum menghapus snapshot.

**Via GUI:**

1. Pastikan VM **Powered Off**
2. Pilih VM di sidebar → klik ikon **Snapshots** (tab di kanan atas panel VM)
3. Klik kanan snapshot yang ada → **Delete Snapshot**

   ```
   ┌──────────────────────────────────────────────────────────┐
   │  Snapshots                                               │
   │                                                          │
   │  📷 Fresh Import - Sebelum Konfigurasi   ← klik kanan   │
   │       └── [Current State]                               │
   │                                                          │
   │  [ Take ]  [ Restore ]  [ Delete ]                      │
   └──────────────────────────────────────────────────────────┘
   ```

4. Konfirmasi penghapusan → VirtualBox akan melakukan **merge** (menggabungkan differencing disk ke base VDI)
5. **Tunggu hingga proses merge selesai** — bisa 2–10 menit tergantung ukuran disk. Jangan tutup VirtualBox selama proses ini
6. Setelah selesai, cek kembali **Storage details** — harus sudah berubah menjadi **"Dynamically allocated storage"** (tanpa kata "differencing")
7. Baru lanjutkan ke langkah 5.1 untuk resize

**Via CLI:**

```bash
# Lihat semua snapshot yang ada
VBoxManage snapshot "NamaVM" list

# Output contoh:
#   Name: Fresh Import - Sebelum Konfigurasi (UUID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx)
#      Name: Current State (UUID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx)

# Hapus snapshot berdasarkan nama
VBoxManage snapshot "NamaVM" delete "Fresh Import - Sebelum Konfigurasi"

# Verifikasi — setelah selesai, cek storage details
VBoxManage showmediuminfo "path/ke/disk.vdi" | grep "Storage format"
# Hasilnya tidak lagi menampilkan "differencing"
```

> **Apakah data VM hilang saat snapshot dihapus?**  
> **Tidak.** Menghapus snapshot hanya menggabungkan (merge) perubahan dari differencing disk ke base VDI. Semua data dan kondisi VM saat ini tetap utuh. Yang hilang hanya kemampuan untuk kembali ke kondisi snapshot tersebut.

> **Jika memiliki beberapa snapshot bercabang:** Hapus dari snapshot paling baru (paling bawah pohon) menuju ke yang paling lama. Menghapus snapshot tengah akan menyebabkan merge yang lebih kompleks dan memakan waktu lebih lama.

---

### 5.1 Perbesar VDI/VMDK dari VirtualBox Manager

1. Pastikan VM **Powered Off**
2. Menu **File** → **Tools** → **Virtual Media Manager** (atau `Ctrl+D`)
3. Pilih disk VM yang ingin diperbesar
4. Di bagian bawah, geser slider **Size** ke ukuran baru

   ```
   ┌─────────────────────────────────────────┐
   │ Size:                                   │
   │ [━━━━━━━━━━━━━━━━━●━━━━━━━━━━] 30.00 GB│
   └─────────────────────────────────────────┘
   ```

5. Klik **Apply** → **Close**

### 5.2 Via Command Line (VBoxManage)

```bash
# Lihat daftar virtual disk
VBoxManage list hdds

# Perbesar disk ke 30 GB (nilai dalam MB)
# Gunakan UUID disk dari output list hdds
VBoxManage modifyhd "C:\VMs\ubuntu-lab\ubuntu-lab.vdi" --resize 30720

# Atau gunakan path lengkap
VBoxManage modifyhd /home/user/VirtualBox\ VMs/ubuntu-lab/ubuntu-lab.vdi --resize 30720
```

### 5.3 Buat Snapshot Sebelum Extend LVM (Checkpoint Keamanan)

Setelah resize disk di VirtualBox selesai, VM masih dalam kondisi **Powered Off** dan LVM belum disentuh sama sekali. Ini adalah momen terbaik untuk membuat snapshot.

#### Mengapa Snapshot di Titik Ini?

Proses extend LVM di Bagian 5.4 melibatkan beberapa perintah berurutan (`growpart` → `pvresize` → `lvextend` → `resize2fs`). Jika salah satu perintah dieksekusi dengan parameter yang keliru — misalnya nomor partisi salah pada `growpart`, atau nama LV yang tidak tepat pada `lvextend` — bisa menyebabkan partisi tidak terbaca atau filesystem rusak.

Dengan snapshot di titik ini, Anda punya jaring pengaman: jika terjadi kesalahan di tengah proses extend, cukup **restore snapshot ini** dan mulai ulang dari kondisi bersih (disk sudah 12 GB, LVM belum dimodifikasi).

```
[Disk di-resize ke 12 GB] ──── snapshot di sini ────► [Extend LVM di dalam VM]
        ↑                                                        ↓
        └────────────── revert ke sini jika ada error ──────────┘
```

> Snapshot diambil saat VM **Powered Off** karena: tidak ada memory state yang perlu disimpan, proses lebih cepat, dan ukuran file snapshot lebih kecil dibanding snapshot saat VM berjalan.

**Via GUI:**

1. Pastikan VM masih **Powered Off**
2. Buka tab **Snapshots** di VirtualBox
3. Klik tombol **Take**
4. Isi nama:

   ```
   Snapshot Name:  02 - Disk Resized - Pre LVM Extend
   Description:    Disk sudah diperbesar ke [X] GB di VirtualBox.
                   LVM belum dimodifikasi. Revert ke sini jika extend gagal.
   ```

5. Klik **OK** → snapshot dibuat dalam hitungan detik

**Via CLI:**

```bash
VBoxManage snapshot "NamaVM" take "02 - Disk Resized - Pre LVM Extend" \
  --description "Disk diperbesar di VirtualBox, LVM belum disentuh. Safety checkpoint."

# Verifikasi
VBoxManage snapshot "NamaVM" list
```

Setelah snapshot dibuat, nyalakan VM dan lanjutkan ke Bagian 5.4.

---

### 5.4 Extend Partisi di Dalam VM (LVM)

Ubuntu 22.04 yang diinstal dengan opsi default menggunakan **LVM (Logical Volume Manager)**. Setelah disk diperbesar di VirtualBox, ada 4 tahap yang harus dilakukan di dalam VM:

```
Disk fisik diperbesar
        ↓
1. Perluas partisi LVM  (growpart)
        ↓
2. Perluas Physical Volume  (pvresize)
        ↓
3. Perluas Logical Volume  (lvextend)
        ↓
4. Perluas filesystem  (resize2fs)
```

```bash
# ── LANGKAH 0: Cek layout saat ini ──────────────────────────
lsblk
# Contoh output — perhatikan nama disk dan partisi:
# NAME                      MAJ:MIN  SIZE
# sda                         8:0    30G   ← disk sudah diperbesar
#   sda1                      8:1     1G   ← /boot
#   sda2                      8:2     1G   ← /boot/efi (jika UEFI)
#   sda3                      8:3    28G   ← partisi LVM ← yang akan diperluas

df -h
# Pastikan / masih terlihat ukuran lama (belum otomatis bertambah)

# ── LANGKAH 1: Perluas partisi LVM ──────────────────────────
sudo apt install -y cloud-guest-utils   # install growpart jika belum ada

# Angka "3" = nomor partisi LVM (sesuaikan dengan output lsblk di atas)
sudo growpart /dev/sda 3

# Verifikasi — sda3 sekarang harus menunjukkan ukuran baru
lsblk

# ── LANGKAH 2: Perluas Physical Volume ──────────────────────
sudo pvresize /dev/sda3

# Cek hasilnya
sudo pvs
# Kolom PFree harus menunjukkan ruang kosong baru

# ── LANGKAH 3: Perluas Logical Volume ───────────────────────
# Lihat nama VG dan LV yang aktif
sudo vgs
sudo lvs

# Perluas LV hingga memenuhi semua ruang kosong di VG
# Nama default Ubuntu: ubuntu-vg / ubuntu-lv — sesuaikan jika berbeda
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv

# ── LANGKAH 4: Perluas filesystem (ext4) ────────────────────
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv

# ── VERIFIKASI ───────────────────────────────────────────────
df -h
# Kolom Size pada / harus menunjukkan ukuran baru
```

> **Catatan nama LV:** Jika nama VG/LV berbeda dari `ubuntu-vg/ubuntu-lv`, cek dengan `sudo lvdisplay` dan sesuaikan perintah `lvextend` dan `resize2fs`.

---

### 5.5 Buat Snapshot Baru Setelah Extend Berhasil

Di Bagian 5.3 kita sudah membuat snapshot "Pre LVM Extend" sebagai jaring pengaman. Sekarang setelah extend berhasil dan `df -h` menunjukkan ukuran yang benar, **hapus snapshot lama itu dan ganti dengan snapshot baru** yang mencerminkan kondisi VM yang sudah selesai dikonfigurasi.

Segera setelah `df -h` menunjukkan ukuran disk yang benar, **matikan VM dan buat snapshot baru** sebagai checkpoint.

#### Mengapa Harus Setelah VM Mati?

Snapshot yang diambil saat VM **mati (Powered Off)** tidak menyimpan memory state — proses ini lebih cepat, ukuran file snapshot lebih kecil, dan tidak ada risiko data yang sedang ditulis ke disk tertangkap dalam kondisi tidak konsisten. Untuk checkpoint pasca-konfigurasi hardware seperti ini, snapshot dari kondisi mati adalah pilihan yang lebih baik.

```bash
# ── Di dalam VM: shutdown dengan benar ───────────────────────
sudo shutdown -h now
```

Setelah VM mati, buat snapshot dari host:

**Via GUI:**

1. Pastikan VM sudah **Powered Off** (status di sidebar VirtualBox: *Powered Off*)
2. Buka tab **Snapshots**
3. Klik tombol **Take** (ikon kamera)
4. Isi nama yang deskriptif:

   ```
   Snapshot Name:  03 - Post Disk Extend - [ukuran GB]
   Description:    Disk diperbesar dan LVM berhasil di-extend.
                   Snapshot 02 (Pre LVM) bisa dihapus.
   ```

5. Klik **OK**

**Via CLI (dari terminal host):**

```bash
# Buat snapshot saat VM dalam kondisi Powered Off
VBoxManage snapshot "NamaVM" take "03 - Post Disk Extend" \
  --description "Disk diperbesar dan LVM berhasil di-extend. Snapshot 02 bisa dihapus."

# Verifikasi snapshot berhasil dibuat
VBoxManage snapshot "NamaVM" list
```

> **Setelah snapshot 03 dibuat, snapshot 02 ("Pre LVM Extend") sudah tidak diperlukan** — tujuannya hanya sebagai jaring pengaman selama proses extend, dan extend sudah berhasil. Hapus snapshot 02 untuk menjaga pohon snapshot tetap rapi dan tidak membebani performa disk.

---

## Bagian 6 — Snapshot: Membuat Titik Pemulihan

Snapshot adalah "foto" kondisi VM pada waktu tertentu. Gunakan snapshot sebelum melakukan perubahan berisiko.

### Kapan Membuat Snapshot?

- Sebelum install software baru
- Sebelum update sistem operasi
- Sebelum mengubah konfigurasi penting
- Sebelum praktek lab yang berisiko

### 6.1 Buat Snapshot via GUI

**Cara 1 — Dari menu saat VM berjalan:**

1. VM sedang berjalan
2. Menu **Machine** → **Take Snapshot...**
3. Isi nama snapshot yang deskriptif

   ```
   ┌──────────────────────────────────────────┐
   │ Take Snapshot of Virtual Machine         │
   │                                          │
   │ Snapshot Name:                           │
   │ [Fresh Install - Sebelum Docker        ] │
   │                                          │
   │ Snapshot Description (optional):         │
   │ [Ubuntu 22.04 baru install, belum ada   ]│
   │ [konfigurasi apapun                     ]│
   │                                          │
   │              [Cancel]  [  OK  ]          │
   └──────────────────────────────────────────┘
   ```

4. Klik **OK**

**Cara 2 — Dari panel Snapshots:**

1. Pilih VM di sidebar VirtualBox
2. Klik ikon **Snapshots** (ikon kalender di kanan atas panel VM)
   — atau tekan `Ctrl+Shift+S` (Windows/Linux) / `⌘+Shift+S` (macOS)
3. Klik tombol **Take** (ikon kamera)
4. Isi nama dan deskripsi → klik **OK**

### 6.2 Buat Snapshot via Command Line

```bash
# Buat snapshot saat VM berjalan atau mati
VBoxManage snapshot "NamaVM" take "Fresh-Install" \
  --description "Ubuntu 22.04 fresh install, sebelum konfigurasi"

# Buat snapshot saat VM mati (lebih cepat, tidak ada memory state)
VBoxManage snapshot "NamaVM" take "Snap-Sebelum-Update" \
  --description "Sebelum apt upgrade" \
  --live   # <-- hapus flag ini jika VM sedang mati

# Lihat daftar snapshot
VBoxManage snapshot "NamaVM" list

# Output contoh:
#   Name: Fresh-Install (UUID: xxxxxxxx)
#      Name: Snap-Sebelum-Update (UUID: xxxxxxxx)
#         Name: current state (UUID: xxxxxxxx)
```

### 6.3 Konvensi Penamaan Snapshot

Gunakan nama yang jelas dan konsisten:

```
Format: [Tanggal] - [Kondisi] - [Keterangan]

Contoh yang baik:
  2025-06-01 - Fresh Install - Ubuntu 22.04
  2025-06-02 - Post Docker Install - Docker 26.x
  2025-06-03 - Pre Config - Sebelum ubah nginx.conf
  2025-06-03 - Working State - Nginx + Docker berjalan normal

Contoh yang buruk:
  snap1
  test
  backup
```

---

## Bagian 7 — Snapshot Tree (Pohon Snapshot)

VirtualBox mendukung snapshot bercabang. Ini berguna untuk mencoba beberapa konfigurasi berbeda dari satu titik yang sama.

```
Fresh Install
    │
    ├─── Cabang A: Docker Setup
    │         │
    │         └─── Docker + Nginx
    │                   │
    │                   └─── [Current State - Cabang A]
    │
    └─── Cabang B: Kubernetes Setup
              │
              └─── [Current State - Cabang B]
```

### Cara Membuat Cabang Snapshot

1. Buka panel **Snapshots**
2. Klik snapshot yang ingin dijadikan titik cabang (misalnya "Fresh Install")
3. Klik **Restore** untuk kembali ke titik itu
4. Mulai konfigurasi berbeda dari titik tersebut
5. Ambil snapshot baru → cabang baru terbentuk otomatis

---

## Bagian 8 — Restore Snapshot

### 8.1 Restore via GUI

1. Pilih VM → buka panel **Snapshots**
2. Klik kanan snapshot yang dituju → pilih **Restore**

   ```
   ┌──────────────────────────────────────────────────┐
   │ Are you sure you want to restore snapshot        │
   │ 'Fresh Install - Sebelum Docker'?                │
   │                                                  │
   │ ☑ Create a snapshot of the current machine state │
   │   Snapshot name: [Restored Snapshot 1          ] │
   │                                                  │
   │              [Cancel]  [  Restore  ]             │
   └──────────────────────────────────────────────────┘
   ```

   > **Rekomendasi:** Centang opsi **"Create a snapshot of the current machine state"** sebelum restore. Ini membuat snapshot dari kondisi sekarang sehingga Anda bisa kembali ke sini jika perlu.

3. Klik **Restore** → VM akan di-stop otomatis jika sedang berjalan
4. Jalankan kembali VM

### 8.2 Restore via Command Line

```bash
# Lihat daftar snapshot dan UUID-nya
VBoxManage snapshot "NamaVM" list --machinereadable

# Restore berdasarkan nama snapshot
VBoxManage snapshot "NamaVM" restore "Fresh-Install"

# Restore berdasarkan UUID
VBoxManage snapshot "NamaVM" restorecurrent
# (restorecurrent = restore ke snapshot yang sedang aktif/terpilih)

# Start VM setelah restore
VBoxManage startvm "NamaVM" --type headless
# atau buka VirtualBox GUI dan klik Start
```

---

## Bagian 9 — Mengelola Snapshot

### 9.1 Hapus Snapshot

Menghapus snapshot **tidak akan menghapus data** di VM. Snapshot yang dihapus akan di-merge ke snapshot induknya atau ke current state.

```bash
# Via GUI: klik kanan snapshot → Delete Snapshot

# Via CLI
VBoxManage snapshot "NamaVM" delete "Snapshot-Lama"
```

> ⚠️ Menghapus snapshot dari tengah pohon bisa memakan waktu karena VirtualBox harus melakukan merge disk. Proses ini bisa berlangsung beberapa menit.

### 9.2 Lihat Detail Snapshot

```bash
# Lihat semua snapshot dengan detail
VBoxManage snapshot "NamaVM" list

# Lihat info snapshot tertentu
VBoxManage snapshot "NamaVM" showvminfo "Fresh-Install"
```

### 9.3 Strategi Manajemen Snapshot

```
✅ DO:
  - Buat snapshot sebelum perubahan berisiko
  - Beri nama deskriptif dengan tanggal
  - Hapus snapshot lama yang tidak diperlukan
  - Batasi jumlah snapshot (maks 5–7 per VM)

❌ DON'T:
  - Jangan andalkan snapshot sebagai backup jangka panjang
  - Jangan biarkan snapshot terlalu banyak (berdampak ke performa)
  - Jangan hapus snapshot saat VM sedang berjalan proses berat
```

---

## Bagian 10 — Export VM ke .ova (Backup Portabel)

Untuk backup yang bisa dipindah ke komputer lain:

```bash
# Via GUI: File → Export Appliance → pilih VM → ikuti wizard

# Via CLI
VBoxManage export "NamaVM" \
  --output "D:\Backup\ubuntu-lab-backup.ova" \
  --options manifest,nomacs

# Flag --options:
#   manifest  = buat file checksum untuk verifikasi integritas
#   nomacs    = generate MAC address baru saat import (hindari konflik)
```

---

## Ringkasan Perintah VBoxManage

```
╔═══════════════════════════════════════════════════════════════╗
║              VBOXMANAGE — OPERASIONAL VM                     ║
╠═══════════════════════════════════════════════════════════════╣
║  HARDWARE                                                     ║
║  modifyvm "VM" --cpus 2          → Set vCPU                  ║
║  modifyvm "VM" --memory 2048     → Set RAM (MB)              ║
║  modifyhd "disk.vdi" --resize N  → Perbesar disk (MB)        ║
║  modifyvm "VM" --nic2 hostonly   → Tambah Host-Only adapter  ║
╠═══════════════════════════════════════════════════════════════╣
║  SNAPSHOT                                                     ║
║  snapshot "VM" take "Nama"       → Buat snapshot             ║
║  snapshot "VM" list              → Lihat daftar              ║
║  snapshot "VM" restore "Nama"    → Restore snapshot          ║
║  snapshot "VM" delete "Nama"     → Hapus snapshot            ║
╠═══════════════════════════════════════════════════════════════╣
║  KONTROL VM                                                   ║
║  startvm "VM"                    → Jalankan VM               ║
║  controlvm "VM" poweroff         → Matikan paksa             ║
║  controlvm "VM" acpipowerbutton  → Shutdown normal           ║
║  showvminfo "VM"                 → Info detail VM            ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## Troubleshooting Umum

| Masalah | Solusi |
|---|---|
| Slider vCPU/RAM tidak bisa digeser | Pastikan VM dalam kondisi Powered Off |
| Error saat modifyhd resize | Pastikan VM off; jika ada snapshot, hapus dulu — VirtualBox tidak bisa resize base VDI selama masih ada differencing disk |
| Resize berhasil di VirtualBox tapi `lsblk` masih ukuran lama | VM masih punya snapshot aktif saat resize dilakukan — hapus snapshot, resize ulang, lalu extend partisi |
| Restore snapshot memakan waktu lama | Normal jika ada banyak perubahan disk; tunggu hingga selesai |
| Snapshot tidak muncul di daftar | Refresh panel dengan klik VM lain lalu klik kembali |
| Disk di VM tidak bertambah setelah resize | Ada dua kemungkinan: (1) masih ada snapshot saat resize — lihat catatan di Bagian 5; (2) perlu extend partisi LVM di dalam VM — lihat Bagian 5.3 |
| VBoxManage: VM not found | Pastikan nama VM persis sama (case-sensitive di Linux/macOS) |
| `sudo netplan apply` error "invalid YAML" | Cek indentasi — gunakan spasi bukan tab; jalankan `cat -A file.yaml` untuk lihat karakter tersembunyi |
| Interface tidak dapat IP setelah apply | Pastikan nama interface benar: jalankan `ip link show` dan sesuaikan nama di YAML |
| Internet mati setelah tambah adapter kedua | Tambahkan `use-routes: false` di konfigurasi `enp0s8` |
| IP berubah setiap restart | Gunakan konfigurasi IP statis (lihat Bagian 2.5) atau tambahkan MAC binding di DHCP server VirtualBox |
| `netplan generate` tidak ada error tapi `apply` gagal | Cek `journalctl -xe` atau `systemctl status systemd-networkd` untuk detail error |

---

## Ringkasan

✅ Menambah adapter Host-Only Adapter ke VM  
✅ Konfigurasi Netplan di Ubuntu untuk DHCP pada adapter baru  
✅ Bisa mengubah vCPU dan RAM VM  
✅ Bisa memperbesar disk VM dan extend partisi  
✅ Bisa membuat snapshot sebelum perubahan  
✅ Bisa restore VM ke kondisi snapshot sebelumnya  
✅ Memahami snapshot tree (cabang)  

**Lanjutkan ke:** [05 — Instalasi Docker di Linux VM](05-install-docker.md)

**Lihat juga:** [08 — Operasional Proxmox (vCPU, RAM, Snapshot)](08-proxmox-operasional.md)
