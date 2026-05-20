# 01 — Instalasi VirtualBox di Windows 10/11

**Durasi estimasi:** 30–45 menit  
**Level:** Pemula  
**Prasyarat:** Laptop/PC dengan Windows 10/11 (64-bit), RAM minimal 8 GB, storage kosong minimal 20 GB

---

## Tujuan Lab

- Mengaktifkan fitur virtualisasi di BIOS/UEFI
- Menginstal VirtualBox versi terbaru di Windows
- Memverifikasi instalasi berjalan dengan benar

---

## Langkah 1 — Aktifkan Virtualisasi di BIOS/UEFI

Sebelum menginstal VirtualBox, pastikan fitur **Intel VT-x** atau **AMD-V** sudah aktif di BIOS.

### 1.1 Cek Status Virtualisasi

1. Tekan `Ctrl + Shift + Esc` → buka **Task Manager**
2. Klik tab **Performance** → pilih **CPU**
3. Lihat bagian bawah — cari baris **Virtualization: Enabled**

Jika sudah **Enabled**, lanjut ke Langkah 2.  
Jika **Disabled**, ikuti langkah berikut untuk masuk BIOS.

### 1.2 Masuk ke BIOS/UEFI

| Merek Laptop | Tombol Masuk BIOS |
|---|---|
| Dell | F2 atau F12 |
| HP | F10 atau Esc |
| Lenovo | F1 atau F2 |
| ASUS | F2 atau Del |
| Acer | F2 atau Del |

**Cara masuk:**
1. Restart laptop
2. Segera tekan tombol BIOS sesuai merek (sebelum Windows loading)
3. Di dalam BIOS, cari menu **Advanced** → **CPU Configuration** atau **System Configuration**
4. Temukan opsi **Intel Virtualization Technology (VT-x)** atau **SVM Mode** (AMD)
5. Ubah ke **Enabled**
6. Simpan dengan **F10** → **Yes** → komputer restart

---

## Langkah 2 — Nonaktifkan Hyper-V (Jika Aktif)

Hyper-V dari Windows dapat berkonflik dengan VirtualBox. Cek dan nonaktifkan jika perlu.

### 2.1 Cek apakah Hyper-V aktif

Buka **PowerShell sebagai Administrator**, jalankan:

```powershell
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V
```

Jika output menunjukkan `State : Enabled`, lakukan langkah berikut.

### 2.2 Nonaktifkan Hyper-V

```powershell
Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All
```

Atau melalui GUI:
1. Tekan `Win + R` → ketik `optionalfeatures` → Enter
2. Hilangkan centang pada **Hyper-V**
3. Hilangkan centang pada **Virtual Machine Platform** dan **Windows Hypervisor Platform**
4. Klik **OK** → restart komputer

> **Catatan:** Di Windows 11, VirtualBox 7.x sudah lebih kompatibel dengan Hyper-V. Namun untuk performa optimal di lab ini, nonaktifkan Hyper-V.

---

## Langkah 3 — Download VirtualBox

1. Buka browser, kunjungi: **https://www.virtualbox.org/wiki/Downloads**
2. Klik **Windows hosts** di bagian **VirtualBox platform packages**
3. File `.exe` akan terunduh (sekitar 100–110 MB)
4. Unduh juga **VirtualBox Extension Pack** (klik link "All supported platforms" di bagian Extension Pack) — untuk fitur USB 2.0/3.0 dan Remote Desktop

---

## Langkah 4 — Install VirtualBox

1. Jalankan file installer `.exe` yang sudah diunduh (klik kanan → **Run as administrator**)
2. Klik **Next** pada halaman Welcome

3. **Custom Setup** — biarkan default, klik **Next**

4. **Warning: Network Interfaces** — klik **Yes** (internet akan terputus sesaat, normal)

5. **Missing Dependencies** — jika muncul peringatan Python/Win32, klik **Yes** untuk install

6. Klik **Install** → tunggu proses selesai (2–5 menit)

7. Klik **Finish** — VirtualBox akan terbuka otomatis

### 4.1 Install Extension Pack

1. Buka VirtualBox
2. Menu **File** → **Tools** → **Extension Pack Manager**
3. Klik ikon **Install** (tanda +)
4. Arahkan ke file `.vbox-extpack` yang sudah diunduh
5. Klik **Install** → setujui lisensi → **I Agree**

---

## Langkah 5 — Verifikasi Instalasi

### 5.1 Cek versi via Command Prompt

Buka **Command Prompt** atau **PowerShell**, jalankan:

```cmd
"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" --version
```

Output yang diharapkan (versi bisa berbeda):
```
7.1.x
```

### 5.2 Cek VirtualBox terbuka normal

1. Buka VirtualBox dari Start Menu
2. Pastikan tampil jendela **Oracle VirtualBox Manager** tanpa error
3. Di bagian bawah jendela, pastikan tidak ada pesan merah (error)

---

## Langkah 6 — Konfigurasi Folder Default VM

Atur lokasi penyimpanan VM agar tidak memenuhi drive C:

1. Menu **File** → **Preferences** (atau `Ctrl + G`)
2. Bagian **General** → **Default Machine Folder**
3. Ubah ke drive dengan ruang kosong cukup, misalnya `D:\VirtualBox VMs`
4. Klik **OK**

---

## Troubleshooting Umum

| Masalah | Solusi |
|---|---|
| Error "VT-x is not available" | Aktifkan VT-x di BIOS (lihat Langkah 1) |
| Error "VT-x is being used by another hypervisor" | Nonaktifkan Hyper-V (lihat Langkah 2) |
| VM sangat lambat / lag | Pastikan RAM di VM tidak melebihi 50% RAM fisik |
| VirtualBox tidak bisa install | Jalankan installer sebagai Administrator |
| USB device tidak terdeteksi | Install Extension Pack (lihat Langkah 4.1) |

---

## Ringkasan

✅ Virtualisasi aktif di BIOS  
✅ Hyper-V dinonaktifkan (jika perlu)  
✅ VirtualBox terinstal dan berjalan  
✅ Extension Pack terinstal  
✅ Folder default VM dikonfigurasi  

**Lanjutkan ke:** [03 — Import VM dari file .ova](03-import-vm-ova.md)
