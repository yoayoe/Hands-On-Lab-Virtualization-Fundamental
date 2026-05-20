# 02 — Instalasi VirtualBox di macOS Apple Silicon (M1/M2/M3)

**Durasi estimasi:** 30–45 menit  
**Level:** Pemula–Menengah  
**Prasyarat:** MacBook/Mac dengan chip Apple M1/M2/M3, macOS Ventura 13 ke atas, RAM minimal 8 GB

---

## ⚠️ Penting: Keterbatasan VirtualBox di Apple Silicon

VirtualBox di chip Apple Silicon (M1/M2/M3) **hanya dapat menjalankan VM ARM 64-bit**.  
VM berbasis x86/x64 (seperti Ubuntu x64, Windows x64 standar) **tidak bisa berjalan** di atas VirtualBox Apple Silicon.

**Pilihan yang tersedia:**

| Opsi | Keterangan | Rekomendasi Lab |
|---|---|---|
| **VirtualBox 7.1+ (ARM)** | Dukungan resmi Apple Silicon, butuh ISO ARM64 | ✅ Pilihan utama |
| **UTM** | Gratis, support emulasi x86 di M-chip | ✅ Alternatif jika VirtualBox gagal |
| **Parallels Desktop** | Berbayar, performa terbaik | Opsional |

Panduan ini mencakup **VirtualBox 7.1+ (ARM)** dan **UTM** sebagai alternatif.

---

## Opsi A — Install VirtualBox 7.1+ untuk Apple Silicon

### A.1 Download VirtualBox

1. Buka browser, kunjungi: **https://www.virtualbox.org/wiki/Downloads**
2. Di bagian **VirtualBox platform packages**, klik **macOS / Apple hosts (arm64)**
3. Unduh file `.dmg` (sekitar 100 MB)
4. Unduh juga **VirtualBox Extension Pack** (All supported platforms)

### A.2 Install VirtualBox

1. Buka file `.dmg` yang diunduh
2. Double-click **VirtualBox.pkg**
3. Ikuti wizard instalasi → klik **Continue** → **Install**
4. Masukkan password administrator Mac jika diminta
5. Klik **Close** setelah selesai

### A.3 Izinkan System Extension

Setelah instalasi, macOS akan memblokir system extension VirtualBox:

1. Buka **System Settings** → **Privacy & Security**
2. Scroll ke bawah — cari notifikasi **"System software from Oracle America, Inc. was blocked"**
3. Klik **Allow**
4. Masukkan password admin → **Modify Settings**
5. Restart Mac

### A.4 Install Extension Pack

1. Buka VirtualBox
2. Menu **VirtualBox** → **Settings** → **Extensions** (atau File → Tools → Extension Pack Manager)
3. Klik tombol **+** → pilih file `.vbox-extpack`
4. Klik **Install** → setujui lisensi → **I Agree**

### A.5 Download ISO ARM64 untuk VM

Karena Apple Silicon = ARM, butuh ISO versi ARM:

| OS | Link Download |
|---|---|
| Ubuntu Server 22.04 ARM64 | https://ubuntu.com/download/server/arm |
| Ubuntu Desktop 22.04 ARM64 | https://ubuntu.com/download/desktop (pilih ARM) |
| Debian ARM64 | https://www.debian.org/distrib/netinst (pilih arm64) |

> **Catatan:** File `.ova` yang dibuat di mesin x86 **tidak kompatibel** dengan VirtualBox ARM. Gunakan ISO ARM64 untuk membuat VM baru, atau gunakan file `.ova` yang memang di-export dari mesin ARM.

---

## Opsi B — Install UTM (Alternatif Rekomendasi)

UTM adalah solusi virtualisasi gratis untuk macOS yang mendukung Apple Silicon dengan lebih baik, dan dapat menjalankan VM x86/x64 melalui emulasi QEMU.

### B.1 Download UTM

1. Kunjungi: **https://mac.getutm.app**
2. Klik **Download** → unduh file `.dmg`
3. Atau install via Mac App Store (berbayar $9.99, mendukung developer)

### B.2 Install UTM

1. Buka file `.dmg`
2. Drag **UTM.app** ke folder **Applications**
3. Buka UTM dari Applications
4. Jika ada peringatan "unidentified developer": **System Settings** → **Privacy & Security** → **Open Anyway**

### B.3 Buat VM Baru di UTM

#### Untuk VM ARM64 (Virtualisasi — performa penuh):

1. Klik **+** (Create a New Virtual Machine)
2. Pilih **Virtualize**
3. Pilih **Linux**
4. Klik **Browse** → pilih file ISO ARM64 Ubuntu/Debian
5. Atur spesifikasi:
   - **Memory:** 2048 MB (2 GB) atau lebih
   - **CPU Cores:** 2 atau lebih
   - **Storage:** 20 GB
6. Klik **Save** → VM siap dijalankan

#### Untuk VM x86_64 (Emulasi — lebih lambat):

1. Klik **+** → pilih **Emulate**
2. Pilih **Linux** → pilih arsitektur **x86_64**
3. Upload ISO Ubuntu/Debian x86_64 seperti biasa
4. Atur memory 2 GB, storage 20 GB
5. Klik **Save**

> **Catatan performa:** VM emulasi x86 di UTM berjalan lebih lambat dibanding VM ARM native. Untuk lab Docker, disarankan gunakan VM ARM64.

### B.4 Import .ova di UTM

UTM tidak mendukung format `.ova` secara langsung. Gunakan cara berikut:

```bash
# Extract .ova (sebenarnya adalah file .tar)
cd ~/Downloads
tar -xvf nama-vm.ova

# Konversi .vmdk ke .qcow2 (install qemu-img via Homebrew dulu)
brew install qemu

qemu-img convert -f vmdk -O qcow2 nama-vm-disk.vmdk nama-vm-disk.qcow2
```

Kemudian di UTM:
1. Buat VM baru → **Virtualize** → **Linux** → **Import VHDX/VMDK/QCOW2 image...**
2. Pilih file `.qcow2` hasil konversi

---

## Verifikasi Instalasi VirtualBox (Opsi A)

Buka **Terminal** dan jalankan:

```bash
VBoxManage --version
```

Output yang diharapkan:
```
7.1.x
```

Atau cek via path lengkap:
```bash
/Applications/VirtualBox.app/Contents/MacOS/VBoxManage --version
```

---

## Konfigurasi Folder Default VM (VirtualBox)

1. Buka VirtualBox
2. Menu **VirtualBox** → **Preferences**
3. Tab **General** → **Default Machine Folder**
4. Ubah ke folder dengan ruang cukup, misalnya `~/Documents/VMs`
5. Klik **OK**

---

## Troubleshooting Umum

| Masalah | Solusi |
|---|---|
| VirtualBox tidak bisa dibuka setelah install | Izinkan System Extension di Privacy & Security |
| Error "kernel driver not installed" | Restart Mac setelah mengizinkan extension |
| VM x86 tidak bisa berjalan di VirtualBox | Gunakan UTM dengan mode Emulate |
| File .ova tidak bisa diimport | Konversi ke .qcow2 menggunakan qemu-img (lihat bagian UTM) |
| UTM crash saat menjalankan VM | Kurangi alokasi CPU/RAM, update macOS |

---

## Ringkasan

| Kondisi | Rekomendasi |
|---|---|
| Butuh VM ARM64 (Ubuntu Server/Desktop ARM) | ✅ VirtualBox 7.1+ ARM |
| Butuh VM x86/x64 atau import .ova x86 | ✅ UTM (mode Emulate) |
| Performa terbaik untuk lab Docker | ✅ UTM (Virtualize ARM64) |

**Lanjutkan ke:** [03 — Import VM dari file .ova](03-import-vm-ova.md)
