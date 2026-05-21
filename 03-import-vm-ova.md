# 03 — Import VM dari File .ova di VirtualBox

**Durasi estimasi:** 15–30 menit  
**Level:** Pemula  
**Prasyarat:** VirtualBox sudah terinstal ([Windows](01-install-virtualbox-windows.md) | [macOS](02-install-virtualbox-macos-silicon.md)), file `.ova` tersedia

---

## Tujuan Lab

- Memahami apa itu file `.ova`
- Mengimpor file `.ova` ke VirtualBox
- Mengkonfigurasi jaringan VM
- Menjalankan dan login ke VM pertama kali

---

## Apa itu File .ova?

File `.ova` (Open Virtualization Archive) adalah format standar untuk mengemas dan mendistribusikan virtual machine. Di dalamnya berisi:

- **Konfigurasi VM** (CPU, RAM, jaringan)
- **Virtual disk** (sistem operasi + data)
- **Metadata** (deskripsi VM)

Format `.ova` kompatibel lintas platform — bisa dibuat di VMware dan diimpor di VirtualBox, atau sebaliknya.

---

## Langkah 1 — Persiapan File .ova

Pastikan file `.ova` sudah tersedia di komputer Anda. Beberapa sumber umum:

- Disediakan oleh instruktur (shared drive / USB)
- Download dari [OSBoxes.org](https://www.osboxes.org/virtualbox-images/) — koleksi VM siap pakai
- Download dari [VirtualBoxes.org](https://virtualboxes.org/images/)

Contoh file yang akan digunakan di lab ini:  
`ubuntu-22.04-server-lab.ova`

---

## Langkah 2 — Import .ova via GUI VirtualBox

### 2.1 Buka Dialog Import

1. Buka **VirtualBox**
2. Menu **File** → **Import Appliance...** (atau tekan `Ctrl + I`)

### 2.2 Pilih File .ova

1. Klik ikon folder di sebelah field **File**
2. Navigasi ke lokasi file `.ova`
3. Pilih file `.ova` → klik **Open**
4. Klik **Next**

### 2.3 Review Konfigurasi Appliance

VirtualBox akan menampilkan konfigurasi VM dari dalam file `.ova`:

```
Virtual System 1
  Name:         ubuntu-22.04-server-lab
  Guest OS:     Ubuntu (64-bit)
  CPU:          2
  RAM:          2048 MB
  Storage:      20 GB (ubuntu-22.04-disk.vmdk)
  Network:      NAT
```

**Yang bisa diubah sebelum import:**

| Setting | Saran |
|---|---|
| Name | Ganti jika ada VM dengan nama sama |
| CPU | Sesuaikan dengan spesifikasi laptop (max 50% core fisik) |
| RAM | Sesuaikan — minimal 1024 MB untuk server Ubuntu |
| Storage | Biarkan default |
| MAC Address Policy | Pilih **Generate new MAC addresses for all network adapters** |

> **Penting:** Selalu pilih **Generate new MAC addresses** jika beberapa peserta mengimpor VM yang sama, agar tidak ada konflik alamat MAC di jaringan.

### 2.4 Tentukan Lokasi Import

1. Di bagian **Machine Base Folder**, arahkan ke folder yang sudah dikonfigurasi sebelumnya (misalnya `D:\VirtualBox VMs`)
2. Klik **Finish**

### 2.5 Tunggu Proses Import

Proses import membutuhkan waktu **5–15 menit** tergantung ukuran file `.ova` dan kecepatan storage. Jangan tutup VirtualBox selama proses ini.

---

## Langkah 3 — Import .ova via Command Line (Opsional)

Cara alternatif menggunakan VBoxManage:

```bash
# Windows
"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" import "D:\ubuntu-22.04-server-lab.ova" --options keepallmacs

# macOS / Linux
VBoxManage import ~/Downloads/ubuntu-22.04-server-lab.ova --options keepallmacs
```

Untuk melihat opsi import sebelum eksekusi:
```bash
VBoxManage import ubuntu-22.04-server-lab.ova --dry-run
```

---

## Langkah 4 — Konfigurasi Jaringan VM

Setelah import berhasil, konfigurasi jaringan sebelum menjalankan VM.

### Mode Jaringan VirtualBox

| Mode | Deskripsi | Gunakan Untuk |
|---|---|---|
| **NAT** | VM bisa akses internet melalui host, tidak bisa diakses dari luar | Default, cocok untuk update & download |
| **NAT Network** | Beberapa VM di jaringan yang sama + akses internet | Lab multi-VM yang perlu saling komunikasi |
| **Bridged Adapter** | VM mendapat IP dari router/DHCP jaringan fisik | VM perlu diakses dari jaringan fisik |
| **Host-only** | VM hanya bisa berkomunikasi dengan host | Isolasi total, cocok untuk lab keamanan |
| **Internal Network** | VM hanya berkomunikasi sesama VM | Lab multi-tier |

**Rekomendasi untuk lab ini:** Gunakan **NAT Network** agar VM bisa akses internet dan saling berkomunikasi.

### 4.1 Buat NAT Network (jika belum ada)

1. Menu **File** → **Tools** → **Network Manager**
2. Tab **NAT Networks** → klik **Create**
3. Atur:
   - **Name:** `LabNetwork`
   - **IPv4 Prefix:** `10.0.2.0/24`
   - **Enable DHCP:** ✅
4. Klik **Apply**

### 4.2 Atur Network Adapter VM

1. Pilih VM di sidebar VirtualBox
2. Klik **Settings** → bagian **Network**
3. **Adapter 1:**
   - **Enable Network Adapter:** ✅
   - **Attached to:** NAT Network
   - **Name:** LabNetwork
4. Klik **OK**

---

## Langkah 5 — Jalankan VM

1. Pilih VM di daftar VirtualBox
2. Klik **Start** (tombol hijau) atau tekan `Ctrl + Enter`
3. Jendela VM akan terbuka — tunggu proses booting

### 5.1 Login ke VM Ubuntu Server

Setelah boot selesai, akan muncul prompt login:

```
ubuntu-lab login: _
```

Kredensial default (sesuaikan dengan file .ova dari instruktur):

| Field | Nilai |
|---|---|
| Username | `ubuntu` atau `pln` |
| Password | `ubuntu` atau `P@ssw0rd123#` (cek dokumentasi .ova) |

Setelah login berhasil:

```bash
# Cek informasi sistem
uname -a

# Cek IP address VM
ip addr show
# atau
hostname -I

# Install network Utils
sudo apt update && sudo apt install iputils-ping -y

# Cek koneksi internet
ping -c 4 google.com
```

---

## Langkah 6 — Matikan VM dengan Benar

Setelah selesai bekerja, matikan VM menggunakan salah satu cara berikut:

```bash
# Di dalam VM (cara yang disarankan — graceful shutdown)
sudo shutdown -h now
# atau
sudo poweroff
```

Atau dari VirtualBox: Menu **Machine** → **ACPI Shutdown**

> **Catatan:** Jangan tutup jendela VM langsung (tombol X) karena bisa menyebabkan filesystem corruption. Selalu shutdown dari dalam VM atau gunakan ACPI Shutdown.

---

## Langkah 7 — Checkpoint: Buat Snapshot Pertama

Sebelum lanjut ke modul berikutnya, buat **snapshot** sebagai titik pemulihan. Jika ada yang salah di modul-modul berikutnya, Anda bisa kembali ke kondisi ini.

Di VirtualBox, saat VM **dalam kondisi mati:**

1. Menu **Machine** → **Take Snapshot...**
2. Nama: `00 - Fresh Import - Sebelum Konfigurasi`
3. Klik **OK**

> Panduan lengkap tentang snapshot, snapshot tree, restore, dan strategi manajemen snapshot ada di modul berikutnya: **[04 — Operasional VM di VirtualBox](04-virtualbox-operasional.md)**

---

## Troubleshooting Umum

| Masalah | Solusi |
|---|---|
| Import gagal — "not enough disk space" | Pastikan ada ruang kosong minimal 2x ukuran file .ova |
| VM stuck di layar hitam saat boot | Tunggu 2–3 menit; coba matikan dan restart VM |
| Tidak bisa login — password tidak diterima | Cek dokumentasi .ova; coba tekan Caps Lock |
| VM sangat lambat | Aktifkan VT-x di BIOS; tambah RAM alokasi VM |
| Network adapter tidak terdeteksi di dalam VM | Install Guest Additions atau cek driver jaringan |
| Layar VM terlalu kecil | Install VirtualBox Guest Additions untuk auto-resize |

---

## Ringkasan

✅ File .ova berhasil diimpor  
✅ Jaringan VM dikonfigurasi (NAT Network)  
✅ VM berjalan dan bisa login  
✅ Snapshot awal dibuat sebagai checkpoint  

**Lanjutkan ke:** [04 — Operasional VM di VirtualBox](04-virtualbox-operasional.md)
