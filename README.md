# 🖧 MySharing Server

Aplikasi desktop Windows untuk **berbagi folder dan printer di jaringan lokal (LAN)** tanpa memakai fitur *File and Printer Sharing* (SMB) bawaan Windows. Folder dibagikan lewat **WebDAV** dan printer lewat **IPP (Internet Printing Protocol)**, keduanya dilayani oleh **satu server HTTP/HTTPS di satu port**.

Client cukup memakai fitur bawaan Windows (*Map network drive* dan *Add printer*), tidak perlu install apa pun di sisi client.

> **Versi:** 1.0.0  
> **Developer:** Muhammad Rizki (reztdev) · ryzkireztdev@gmail.com  
> © 2026 All Rights Reserved

---

## 📑 Daftar Isi

1. [Konsep Dasar](#-konsep-dasar)
2. [Fitur](#-fitur)
3. [Arsitektur & Struktur](#-arsitektur--struktur)
4. [Cara Kerja Detail](#-cara-kerja-detail)
5. [Protokol yang Dipakai](#-protokol-yang-dipakai)
6. [Keamanan](#-keamanan)
7. [Kelebihan](#-kelebihan)
8. [Kekurangan & Batasan](#-kekurangan--batasan)
9. [Perbandingan dengan Windows Sharing (SMB)](#-perbandingan-dengan-windows-sharing-smb)
10. [Panduan Pemakaian](#-panduan-pemakaian)
11. [Penjelasan Tab Aplikasi](#-penjelasan-tab-aplikasi)
12. [Troubleshooting](#-troubleshooting)
13. [FAQ](#-faq)
14. [Nilai Bawaan](#-nilai-bawaan)

---

## 💡 Konsep Dasar

Windows punya fitur berbagi file bawaan berbasis **SMB**. Fitur itu kuat, tapi sering merepotkan: harus mengatur profil jaringan, akun, izin NTFS, dan *share permission*, ditambah kerap muncul masalah seperti popup login berulang, folder tidak terlihat, atau port 445 yang sebaiknya tidak dibuka sembarangan.

MySharing Server memakai pendekatan berbeda:

| Aspek | Pendekatan |
|---|---|
| **Transport** | HTTP biasa atau HTTPS, bukan SMB |
| **File** | WebDAV, ekstensi HTTP untuk mengelola file jarak jauh |
| **Printer** | IPP, protokol cetak berbasis HTTP |
| **Setup** | Jalankan aplikasi, pilih folder, klik Start |
| **Akses client** | Alamat URL (misalnya `http://192.168.1.10:8040/`) |
| **Satu pintu** | File dan printer sama-sama lewat satu port, dibedakan dari alamatnya |

Konsep intinya: **satu aplikasi portable, satu port, satu alamat**, dengan lapisan proteksi tambahan (filter IP, filter jaringan LAN, penguncian login, batas client) yang bisa diatur dari antarmuka grafis.

---

## ✨ Fitur

**Berbagi file**
- Banyak folder dibagikan sekaligus, semuanya muncul di **satu drive** pada client
- Centang/hilangkan centang tiap folder tanpa menghapusnya dari daftar
- Folder yang hilang ditandai merah

**Berbagi printer**
- Printer USB di komputer server bisa dipakai komputer lain lewat jaringan
- Tanpa perlu install driver di client (*driverless*, memakai IPP)
- Antrian cetak terlihat langsung di aplikasi
- **Mode uji**: dokumen hanya disimpan sebagai file, tidak benar-benar dicetak, untuk mencoba koneksi tanpa membuang kertas

**Keamanan**
- Login (Digest dan Basic) dengan username yang bisa diganti
- Pembuat password acak (*Generate*)
- HTTPS dengan sertifikat dibuat otomatis (TLS 1.2+)
- Whitelist & blacklist IP/CIDR
- Hanya jaringan lokal (LAN) secara bawaan
- Penguncian IP otomatis setelah beberapa kali gagal login
- Filter "Hanya Windows Explorer"
- Batas jumlah client aktif

**Operasional**
- Monitor client real-time (jumlah request, data terkirim/diterima, aktivitas terakhir)
- Log aktivitas berlabel
- Pembuat skrip `.bat` agar client konek sekali klik
- Konfigurasi tersimpan otomatis
- Pengaturan proteksi dan filter IP bisa diubah **tanpa menghentikan server**

---

## 🏗️ Arsitektur & Struktur

### Komponen utama

| Komponen | Peran |
|---|---|
| **Antarmuka (GUI)** | Jendela dengan 7 tab untuk mengatur dan memantau server |
| **Server HTTP/HTTPS** | Menerima koneksi dari client, mendukung beberapa koneksi paralel |
| **Penjaga akses (middleware)** | Lapisan pertama yang memeriksa **setiap** request sebelum diteruskan |
| **Router** | Memisahkan jalur printer (`/ipp/...`) dari jalur file |
| **Mesin WebDAV** | Melayani operasi file: lihat, baca, tulis, buat folder, hapus, pindah, salin |
| **Root virtual** | Menampilkan daftar semua folder sebagai satu "drive" saat folder dibagikan lebih dari satu |
| **Mesin IPP (printer)** | Implementasi sendiri protokol IPP: menerima dokumen, mengelola antrian |
| **Auth printer** | Login Digest/Basic khusus jalur printer |
| **Backend cetak** | Meneruskan dokumen ke printer fisik di komputer server |
| **Pengelola sertifikat** | Membuat dan memperbarui sertifikat HTTPS self-signed |
| **Penyimpan konfigurasi** | Membaca/menulis `config.json` |

### Diagram alur

```
        Komputer Client (Windows)
   ┌────────────────────────────────────┐
   │  Map Network Drive   Add Printer   │
   │      (WebDAV)           (IPP)      │
   └───────────┬──────────────┬─────────┘
               │  HTTP / HTTPS │     satu port (bawaan 8040)
               ▼               ▼
   ┌────────────────────────────────────────────────┐
   │              Server HTTP / HTTPS               │
   │  ┌──────────────────────────────────────────┐  │
   │  │ PENJAGA AKSES                            │  │
   │  │ 1. Filter IP (blacklist/whitelist/LAN)   │  │
   │  │ 2. Cek penguncian login                  │  │
   │  │ 3. Filter "Hanya Windows Explorer"       │  │
   │  │ 4. Batas jumlah client                   │  │
   │  │ 5. Catat aktivitas & hitung trafik       │  │
   │  └───────────────────┬──────────────────────┘  │
   │                      ▼                         │
   │              ┌──── Router ────┐                │
   │       alamat /ipp/...    alamat lainnya        │
   │              ▼                ▼                │
   │      ┌──────────────┐  ┌──────────────┐        │
   │      │  Mesin IPP   │  │ Mesin WebDAV │        │
   │      │ (+auth sendiri)│ │ (+login)     │       │
   │      └──────┬───────┘  └──────┬───────┘        │
   └─────────────┼─────────────────┼────────────────┘
                 ▼                 ▼
          Folder spool        Folder yang dibagikan
          → Printer USB       (baca & tulis)
```

### Struktur file saat aplikasi berjalan

Semua data disimpan **di folder yang sama dengan file `.exe`**:

```
📁 (folder aplikasi)
 ├── MySharing.exe        ← aplikasi
 ├── config.json          ← pengaturan (dibuat otomatis, password TIDAK disimpan)
 ├── 📁 certs/            ← sertifikat HTTPS (muncul bila HTTPS pernah dipakai)
 │    ├── server.crt      ← sertifikat (boleh dibagikan ke client)
 │    └── server.key      ← kunci privat (RAHASIA)
 └── 📁 ipp-spool/        ← file dokumen cetak sementara (muncul bila printer aktif)
```

### Model berjalan (thread)

- **Thread antarmuka**: menggambar jendela dan menangani klik.
- **Thread server**: melayani koneksi client secara paralel.
- **Thread pencetakan**: tiap dokumen cetak diproses di thread sendiri supaya server tidak tersendat.
- Komunikasi antar-thread memakai antrian dan kunci (*lock*), sehingga log dan data client aman dari tabrakan. Antarmuka memperbarui tabel client tiap **1 detik** dan log tiap **0,15 detik**.

---

## ⚙️ Cara Kerja Detail

### 1. Saat klik "Start Sharing"

Aplikasi memvalidasi dulu sebelum server menyala:

1. Minimal ada **satu folder tercentang** *atau* **sharing printer aktif** (boleh hanya salah satu).
2. Semua folder yang dicentang **masih ada**.
3. Port dan Max Clients valid.
4. Jika password aktif: tidak boleh kosong, username harus **3–32 karakter** (huruf, angka, titik, minus, underscore), dan password di bawah 8 karakter memunculkan **peringatan** (boleh dilanjutkan).
5. Whitelist yang aktif tidak boleh kosong.
6. Port belum dipakai aplikasi lain.
7. Jika HTTPS aktif: sertifikat disiapkan.

Setelah lolos, server menyala, pengaturan dikunci, daftar URL ditampilkan, dan indikator berubah hijau.

### 2. Perjalanan satu request (penjaga akses)

Setiap request dari client melewati pemeriksaan berurutan. Gagal di tahap mana pun berarti langsung ditolak dengan **403 Forbidden**:

| # | Pemeriksaan | Ditolak jika |
|---|---|---|
| 1 | **Filter IP** | IP ada di blacklist (bila aktif); atau tidak ada di whitelist (bila aktif); atau, bila whitelist tidak aktif dan filter LAN aktif, IP bukan alamat lokal |
| 2 | **Penguncian login** | IP sedang terkunci karena terlalu sering gagal login |
| 3 | **User-Agent** | Filter "Hanya Windows Explorer" aktif dan klien bukan Windows WebDAV (**jalur printer dikecualikan**) |
| 4 | **Batas client** | Client ini belum aktif, sementara jumlah client aktif lain sudah mencapai batas |
| 5 | **Catat & layani** | Lolos semua: request diteruskan, trafik dihitung, hasilnya dicatat di log |

Yang dianggap *alamat lokal* oleh filter LAN: jaringan privat (misalnya `192.168.x.x`, `10.x.x.x`, `172.16–31.x.x`), *loopback* (`127.x.x.x`), dan *link-local*.

### 3. Definisi "client aktif"

Client diidentifikasi lewat **alamat IP**. Sebuah IP dianggap **Active** jika ada transfer yang sedang berjalan **atau** request terakhirnya kurang dari **60 detik** lalu. Setelah itu statusnya **Idle** dan tidak lagi memakai slot. Satu PC yang membuka banyak koneksi tetap dihitung **satu** client.

### 4. Berbagi file (WebDAV)

- Tiap folder dipetakan ke alamat `/namafolder/`. Nama dibuat huruf kecil otomatis dan dibuat unik bila kembar.
- **Satu folder**: alamat utama (`/`) langsung menampilkan isi folder itu.
- **Banyak folder**: alamat utama (`/`) menampilkan **daftar folder** (hanya-baca, tidak bisa tambah/hapus folder dari situ). Client cukup memetakan **satu drive** dan semua folder ada di dalamnya.
- Nama **`ipp`** dicadangkan untuk jalur printer, sehingga folder bernama `ipp` otomatis diganti menjadi `ipp_2`.
- Isi folder yang dibagikan bersifat **baca dan tulis**.
- Tampilan daftar file di browser **dimatikan**: akses ditujukan lewat Windows Explorer.

### 5. Login file & printer

Saat password aktif:

- Server menawarkan **Digest** (diutamakan) dan **Basic**.
- Seluruh folder memakai **satu realm** ("File Sharing"), sehingga Windows tidak terus-menerus memunculkan popup login saat pindah folder.
- Format username ala Windows diterima: `PC\username`, `.\username`, dan `username@host` dianggap sama dengan `username`.
- Username bisa diganti (bawaan: `user`). Aplikasi menyarankan **jangan memakai `user`** karena mudah ditebak.
- Tombol **Generate** membuat password acak 16 karakter.
- Password **hanya ada di memori** selama server berjalan. Tidak ditulis ke disk, harus diisi ulang tiap Start.

### 6. Penguncian login (anti tebak password)

- Setiap login yang ditolak (401, disertai kredensial) dicatat per IP.
- Jika sebuah IP gagal **5 kali dalam 10 menit** (bisa diatur 1–99 kali), IP itu **dikunci 15 menit** (bisa diatur 1–1440 menit).
- Login yang berhasil **mereset** hitungan gagal untuk IP tersebut.
- Tombol **Unlock All** (tab Clients) membuka semua kunci seketika.
- Jumlah IP yang sedang terkunci ditampilkan di tab Clients.

### 7. Berbagi printer (IPP)

Alur dari sisi client ke kertas:

```
Client "Print"  →  dokumen (PDF) dikirim lewat IPP  →  server memeriksa login
   →  disimpan di folder ipp-spool  →  masuk antrian  →  diproses
   →  dikirim ke printer USB  →  status: Selesai / Gagal
```

**Operasi IPP yang didukung:** Print-Job, Validate-Job, Cancel-Job, Get-Job-Attributes, Get-Jobs, Get-Printer-Attributes. Versi IPP 1.1 dan 2.0.

**Format dokumen yang diterima:** **PDF**, serta gambar **JPEG** dan **PNG** (dikenali dari isi file jika dikirim sebagai data biner umum). Format lain (misalnya PostScript, XPS, atau file Office langsung) **ditolak**.

**Cara dokumen dicetak ke printer fisik (urutan prioritas di Windows):**
1. **SumatraPDF** bila terpasang → cetak senyap (disarankan, paling andal)
2. Khusus gambar, tanpa Sumatra → **Paint** bawaan Windows
3. PDF tanpa Sumatra → perintah *print* milik aplikasi PDF yang terpasang (misalnya Adobe Reader)

Jika tidak ada satu pun yang tersedia, pekerjaan ditandai **Gagal** dan alasan dicatat di log.

**Mode uji:** bila printer tidak dipilih, dokumen hanya disimpan di `ipp-spool` dan dianggap selesai, cocok untuk menguji koneksi.

**Manajemen antrian:**
- Ukuran maksimal satu dokumen: **256 MB**.
- Aplikasi menyimpan **50 pekerjaan terakhir**; yang lama dihapus beserta file-nya.
- File spool yang berumur lebih dari **1 hari** dibersihkan saat server dijalankan.
- Pekerjaan bisa **dibatalkan** selama belum selesai.

### 8. HTTPS & sertifikat

- Sertifikat dibuat otomatis: RSA 2048-bit, SHA-256, berlaku sekitar **825 hari**, mencakup `localhost`, nama komputer, IP lokal, dan `127.0.0.1`.
- Dibuat ulang otomatis bila IP komputer berubah (tidak tercakup lagi) atau masa berlaku **kurang dari 30 hari**.
- TLS minimal **versi 1.2**.
- Karena **self-signed**, client harus mempercayainya: klik **Export Cert...**, kirim file `.crt` ke client, lalu install di client.
- Bila sertifikat dibuat ulang, sertifikat lama di client tidak berlaku lagi dan harus di-install ulang.

### 9. Konfigurasi

Disimpan otomatis ke `config.json` setiap ada perubahan, dengan cara **tulis atomik** (file sementara dulu, lalu diganti) agar config lama tidak rusak bila terjadi kegagalan di tengah jalan. Yang disimpan: port, max clients, HTTPS, username, filter User-Agent, filter LAN, pengaturan penguncian, status password (aktif/tidak), daftar folder, whitelist/blacklist, dan pengaturan printer. **Password tidak pernah disimpan.** Bila config rusak, aplikasi memakai pengaturan bawaan dan menampilkan peringatan di Log.

### 10. Perubahan saat server berjalan

Tab **Access** (filter IP) dan bagian **Proteksi** di Settings berlaku **langsung** tanpa Stop/Start. Bila aturan berubah dan sebuah client sedang mengunduh, pemeriksaan ulang dilakukan dan koneksi yang tidak lagi diizinkan **diputus di tengah transfer**.

---

## 📡 Protokol yang Dipakai

| Lapisan | Protokol | Fungsi di aplikasi |
|---|---|---|
| **Transport** | TCP (IPv4) | Server mendengarkan di semua antarmuka jaringan (`0.0.0.0`), satu port |
| **Aplikasi** | **HTTP/1.1** | Dasar bagi WebDAV dan IPP |
| **Enkripsi** (opsional) | **TLS 1.2+** (HTTPS) | Mengenkripsi seluruh lalu lintas, termasuk login |
| **File** | **WebDAV** (RFC 4918) | Operasi file lewat HTTP: melihat isi folder, membaca, menulis, membuat folder, menghapus, memindah, menyalin, dan penguncian dokumen |
| **Printer** | **IPP** 1.1/2.0 (RFC 8010/8011) | Pengiriman dokumen cetak lewat HTTP `POST` bertipe `application/ipp` |
| **Autentikasi** | **HTTP Digest** & **Basic** | Login username/password |

**Alamat penting:**

| Tujuan | Format alamat |
|---|---|
| Semua folder (bila >1 folder) | `http://IP:PORT/` |
| Satu folder tertentu | `http://IP:PORT/namafolder/` |
| Printer | `http://IP:PORT/ipp/print` |

(Ganti `http` dengan `https` bila HTTPS aktif.)

**Sisi client:** Windows memakai layanan **WebClient** (*WebDAV Mini-Redirector*) untuk drive WebDAV, dan fitur **IPP** bawaan untuk printer.

> ℹ️ Windows secara bawaan **menolak login Basic lewat HTTP biasa** (hanya mengizinkannya lewat HTTPS). Karena itu aplikasi menawarkan **Digest** terlebih dahulu, yang diterima Windows di HTTP biasa. Digest memakai MD5 sesuai standar lama demi kompatibilitas.

---

## 🔐 Keamanan

### Model ancaman

MySharing Server dirancang untuk **jaringan lokal yang dipercaya sebagian** (rumah, kantor kecil, lab). Aplikasi **tidak dirancang untuk diekspos langsung ke internet**.

### Lapisan proteksi

| Lapisan | Fungsi | Bawaan |
|---|---|---|
| **Filter LAN** | Tolak IP di luar jaringan lokal | ✅ Aktif |
| **Whitelist** | Izinkan hanya IP/CIDR tertentu (menggantikan filter LAN) | Nonaktif |
| **Blacklist** | Blokir IP/CIDR tertentu (dicek paling dulu) | Nonaktif |
| **Login** | Username + password (Digest/Basic) | Nonaktif |
| **Penguncian login** | Kunci IP setelah 5× gagal selama 15 menit | ✅ Aktif |
| **Filter Windows Explorer** | Tolak browser, curl, dan klien non-Windows-WebDAV | ✅ Aktif |
| **Batas client** | Maksimal client aktif bersamaan | 3 |
| **HTTPS** | Enkripsi lalu lintas | Nonaktif |
| **Printer minta password** | Login juga untuk jalur printer | ✅ Aktif (efektif bila password share aktif) |

### Perilaku keamanan yang perlu dipahami

- 🔓 **Tanpa password, tidak ada login sama sekali.** Siapa pun yang lolos filter IP/LAN bisa mengakses.
- ✏️ **Folder bersifat baca-tulis.** Pengguna yang berhasil login dapat mengunggah, mengubah, dan menghapus file.
- 👤 **Satu kredensial untuk semua.** Tidak ada akun per orang dan tidak ada izin per folder atau per file.
- 🧾 **Password tidak dicatat di log.** Yang dicatat hanya username yang dicoba dan alamat IP.
- 🕵️ **Filter "Hanya Windows Explorer" bukan pengaman sungguhan.** Ia hanya memeriksa identitas klien (*User-Agent*) yang bisa dipalsukan dengan mudah. Anggap ia sebagai pengurang gangguan, bukan penjaga. Filter ini juga memblokir klien WebDAV non-Windows (Mac, Linux, aplikasi Android) sehingga **matikan** bila perlu mengakses dari perangkat lain.
- 🌐 **Filter LAN membaca alamat sumber koneksi.** Bila ada *tunnel*/*proxy*/VPN di komputer server yang meneruskan koneksi dari luar, koneksi bisa tampak berasal dari `127.0.0.1` dan lolos filter. Jangan jalankan aplikasi ini di belakang tunnel ke internet.
- 🎟️ **Slot client dipakai sebelum login.** Request yang gagal login pun menandai IP sebagai aktif selama 60 detik, jadi perangkat asing di LAN bisa menghabiskan slot (bawaan hanya 3). Gunakan whitelist bila ini jadi masalah.
- 🔑 **Kunci privat HTTPS tersimpan tanpa enkripsi** di `certs/server.key`. Lindungi folder aplikasi dan jangan bagikan file `.key`.
- 🖨️ **Login jalur printer memakai implementasi tersendiri**, lebih sederhana daripada mesin WebDAV: nonce Digest berlaku 1 jam dan aplikasi tidak melacak pemakaian ulang request. Secara teori request Digest yang tersadap bisa diputar ulang selama nonce berlaku. **Aktifkan HTTPS** bila ini penting.
- 🗂️ **Dokumen cetak tersimpan sebagai file biasa** di `ipp-spool` (hingga 50 pekerjaan terakhir) dan tidak dienkripsi.
- 📋 **Log hanya ada di memori** (±500 baris terakhir) dan hilang saat aplikasi ditutup. Tidak ada jejak audit permanen.
- 🌍 **Hanya IPv4.**

### Rekomendasi pengaturan

| Situasi | Saran |
|---|---|
| Jaringan rumah yang dipercaya | Password + Generate, pertahankan filter LAN dan penguncian login |
| Kantor / jaringan bersama | Password kuat + **HTTPS** + **whitelist** IP perangkat yang dikenal |
| Dokumen sensitif | HTTPS wajib, bagikan hanya folder yang perlu, hentikan server setelah selesai |
| Akses dari Mac/Linux/HP | Matikan filter "Hanya Windows Explorer" |
| Jangan | Membuka port ke internet, atau memakai tanpa password di jaringan umum |

---

## 👍 Kelebihan

- **Portable & ringan**: satu `.exe`, tanpa instalasi, tanpa mengubah konfigurasi sistem Windows.
- **Cepat disiapkan**: pilih folder → Start. Tidak perlu mengatur profil jaringan, akun lokal, atau izin NTFS.
- **Client tanpa software tambahan**: cukup fitur bawaan Windows.
- **Satu port untuk semuanya**: file dan printer lewat satu port; mudah diatur di firewall.
- **Tidak memakai port 445**: tidak bersinggungan dengan SMB bawaan Windows dan jauh dari permukaan serangan SMB.
- **Opsi enkripsi mudah**: HTTPS dibuat otomatis, tanpa perlu tahu OpenSSL.
- **Kontrol akses visual**: whitelist, blacklist, blokir cepat dari daftar client, penguncian login otomatis.
- **Printer tanpa driver di client**: IPP bersifat *driverless*, cukup tempel alamat.
- **Pemantauan langsung**: terlihat siapa yang terhubung, berapa data yang berpindah, dan apa yang diakses.
- **Login ramah Windows**: satu realm dan format username Windows diterima, sehingga popup login tidak berulang.
- **Mode uji printer**: bisa mencoba alur cetak tanpa mencetak.
- **Config aman**: penulisan atomik, validasi saat dibaca, dan password tidak disimpan.
- **Mudah dihentikan**: Stop Sharing langsung menutup akses dan membersihkan sesi.

---

## 👎 Kekurangan & Batasan

**Performa & kompatibilitas**
- **Lebih lambat daripada SMB**, terutama untuk banyak file kecil atau transfer besar, karena overhead HTTP dan klien WebDAV Windows.
- **Batas ukuran file di klien Windows**: secara bawaan WebDAV Windows membatasi file sekitar **50 MB** per file (bisa dinaikkan lewat pengaturan Windows di sisi client, maksimal sekitar 4 GB).
- Bergantung pada layanan **WebClient** di client. Pada Windows Server, fitur *WebDAV Redirector* harus dipasang dahulu.
- **Hanya IPv4.**
- Tampilan di browser **dimatikan**; tidak ada antarmuka web.

**Fitur akses & izin**
- **Satu akun bersama** dan **tanpa izin per pengguna/per folder**: semua yang berhasil login punya hak sama (baca-tulis penuh).
- Tidak ada mode **hanya-baca** untuk folder tertentu.
- Tidak ada integrasi dengan **Active Directory**, Kerberos, atau akun Windows.
- Tidak ada **kuota**, *previous versions* (VSS), atau audit permanen.

**Printer**
- Hanya menerima **PDF, JPEG, PNG**.
- **Opsi cetak dari client tidak diteruskan** (jumlah salinan, duplex, ukuran kertas, dst.): hasil mengikuti pengaturan bawaan printer di server.
- Pencetakan andal membutuhkan **SumatraPDF** (atau aplikasi PDF lain yang mendukung cetak otomatis).
- Pembatalan tidak menghentikan pekerjaan yang **sudah mulai dicetak**.
- Dokumen dimuat penuh ke memori saat diterima (batas 256 MB).
- Pembuatan daftar printer di Windows memakai PowerShell.

**Operasional**
- **Penemuan otomatis tidak ada**: client harus mengetik/menempel alamat (atau memakai skrip `.bat`).
- **Alamat IP dideteksi saat Start**; bila berpindah jaringan, Stop lalu Start lagi.
- Pada komputer dengan banyak adapter/VPN, IP yang terdeteksi bisa bukan yang diinginkan.
- **Log tidak disimpan ke disk.**
- Server berjalan **hanya selama aplikasi terbuka** (bukan layanan Windows yang otomatis hidup).
- Sertifikat self-signed harus **dipasang manual** di tiap client.
- Pengaturan utama dikunci saat server berjalan.

---

## ⚖️ Perbandingan dengan Windows Sharing (SMB)

> SMB (*Server Message Block*) adalah protokol berbagi file dan printer bawaan Windows (File and Printer Sharing), juga didukung Samba di Linux/macOS. Pembahasan di bawah membandingkan **MySharing Server** dengan **SMB bawaan Windows** secara umum.

### Tabel perbandingan

| Aspek | MySharing Server (WebDAV + IPP) | Windows Sharing (SMB) |
|---|---|---|
| **Protokol file** | WebDAV di atas HTTP/HTTPS | SMB 2/3 |
| **Port** | Bebas (bawaan 8040) | TCP 445 (warisan: NetBIOS 137–139) |
| **Instalasi** | Satu `.exe` portable | Bawaan sistem Windows |
| **Kemudahan setup** | Sangat mudah: pilih folder lalu Start | Sedang: profil jaringan, akun, izin share + NTFS |
| **Akses client** | Alamat URL / Map drive ke URL | Jalur UNC (`\\server\share`) |
| **Penemuan di jaringan** | Manual (URL / skrip `.bat`) | Otomatis di "Network" (bergantung pengaturan) |
| **Autentikasi** | Satu username + password (Digest/Basic) | Akun Windows/domain (NTLM, Kerberos) |
| **Izin akses** | Semua-atau-tidak, baca-tulis penuh | Granular per pengguna/grup (ACL NTFS + izin share) |
| **Enkripsi** | HTTPS/TLS 1.2+ (opsional, self-signed) | Enkripsi SMB 3 (AES), signing |
| **Kecepatan** | Cukup; lebih lambat untuk banyak file kecil | Tinggi; dioptimalkan untuk LAN (multichannel, dll.) |
| **File besar** | Terbatas default 50 MB di klien Windows (bisa diubah) | Tanpa batas praktis |
| **Penguncian file** | Ada (WebDAV lock), kurang andal untuk edit bersama | Matang (oplock/lease) untuk kerja bersama |
| **Berbagi printer** | IPP *driverless*, PDF/JPEG/PNG saja | Via spooler Windows, mendukung semua format & opsi driver |
| **Opsi cetak** | Tidak diteruskan (salinan, duplex, dll.) | Penuh lewat driver |
| **Manajemen & audit** | GUI sederhana, log di memori | Event log, kuota, VSS, FSRM, dll. |
| **Integrasi domain/AD** | Tidak ada | Penuh |
| **Batas koneksi** | Bisa diatur (1–50 client aktif) | Windows 10/11 klien: ±20 koneksi masuk bersamaan |
| **Lintas jaringan / firewall** | Satu port HTTP(S) bebas pilih | Port 445 sering diblokir antar jaringan |
| **Permukaan serangan** | Server HTTP kecil + filter aplikasi | Layanan sistem besar; riwayat kerentanan SMB1 dan lainnya |
| **Dependensi client** | Layanan WebClient aktif | Bawaan |
| **Perlu hak admin** | Umumnya tidak di server (kecuali izin firewall); client mungkin untuk menyalakan WebClient | Ya, untuk mengatur berbagi |

### Perbedaan konsep

- **SMB** adalah sistem berbagi **terintegrasi dengan identitas Windows**: izin mengikuti akun, domain, dan ACL.
- **MySharing Server** adalah **server berbagi mandiri**: tidak menyentuh sistem izin Windows. Keamanannya berada di **lapisan aplikasi** (filter IP, login tunggal, penguncian), bukan di tingkat sistem berkas.

### Perbandingan berbagi printer

- **SMB**: printer dibagikan lewat *print spooler* Windows. Client biasanya mengunduh driver dari server (*Point and Print*). Fiturnya lengkap, tetapi konfigurasinya lebih rumit dan spooler pernah menjadi sasaran kerentanan serius.
- **IPP di aplikasi ini**: tanpa driver di client, cukup tempel alamat. Cocok untuk cetak dokumen biasa (PDF/gambar), tetapi **tidak** cocok bila butuh opsi cetak lanjutan atau format khusus.

### Kapan memilih yang mana?

**Pilih MySharing Server bila:**
- Butuh berbagi cepat tanpa mengubah pengaturan Windows atau tanpa hak admin penuh
- Menggunakan komputer yang tidak bisa/tidak boleh mengaktifkan SMB
- Ingin menghindari port 445
- Hanya perlu satu kelompok pengguna dengan akses yang sama
- Ingin berbagi printer USB sederhana (PDF/gambar) tanpa driver di client
- Butuh solusi portabel dan sementara (acara, presentasi, transfer sesekali)

**Pilih SMB bawaan Windows bila:**
- Butuh izin berbeda untuk pengguna berbeda
- Memindahkan file sangat besar atau sangat banyak secara rutin
- Banyak orang mengedit file yang sama secara bersamaan
- Lingkungan memakai domain/Active Directory
- Butuh opsi cetak lengkap (duplex, ukuran kertas, dll.) dan semua format dokumen
- Butuh audit, kuota, atau versi sebelumnya (*previous versions*)

---

## 🚀 Panduan Pemakaian

### A. Di komputer server

1. Jalankan aplikasi.
2. **Folders** → **Add...** → pilih folder (bisa berkali-kali).
3. *(Opsional)* **Settings** → aktifkan password (username dan password; klik **Generate** untuk password acak) dan/atau HTTPS.
4. *(Opsional)* **Printer** → aktifkan sharing printer, pilih printer USB, isi nama untuk client.
5. Klik **Start Sharing**. Indikator hijau menandakan server berjalan.
6. Salin alamat dari kotak **Client URL** (file) atau **Alamat untuk client** (printer).

### B. Di komputer client: file

**Cara termudah**: di server, klik **Script...** lalu simpan `.bat`; kirim ke client dan jalankan. Skrip akan menyalakan layanan WebClient, memilih huruf drive kosong, menyambungkan drive (memintakan password bila perlu), lalu membukanya.

**Cara manual**:
1. File Explorer → klik kanan **This PC** → **Map network drive**.
2. Isi alamat server, misalnya `http://192.168.1.10:8040/`.
3. Centang *Connect using different credentials* bila memakai password.

> Koneksi dari skrip bersifat **tidak permanen** (hilang setelah restart/logoff).

### C. Di komputer client: printer

1. Windows: **Settings → Printers & scanners → Add device → Add manually**.
2. Pilih **Select a shared printer by name**, lalu tempel alamat printer (`http://IP:PORT/ipp/print`).
3. Selesaikan wizard. Bila diminta, masukkan username dan password.
4. Cetak dokumen. **Disarankan** dari PDF.
5. Untuk uji coba tanpa kertas, biarkan server di **mode uji** dan cek hasilnya di folder `ipp-spool`.

### D. Bila memakai HTTPS

1. Di server: **Settings → Export Cert...** lalu simpan file `.crt`.
2. Kirim ke client dan **install sebagai sertifikat tepercaya** (Trusted Root).
3. Setelah itu gunakan alamat `https://...`.

---

## 🖥️ Penjelasan Tab Aplikasi

| Tab | Isi |
|---|---|
| **Folders** | Daftar folder (centang/hapus/tambah), kotak Client URL, tombol Copy dan Script |
| **Settings** | Port, Max Clients, HTTPS, Export Cert, username/password, Generate, Show password, serta bagian **Proteksi** (filter Windows Explorer, filter LAN, aturan penguncian) |
| **Access** | Whitelist dan blacklist IP/CIDR, masing-masing dengan tombol Enable |
| **Clients** | Tabel client (IP, status, request, data terkirim/diterima, aktivitas terakhir), tombol **Block IP**, **Clear Idle**, **Unlock All**, serta penghitung client aktif dan IP terkunci |
| **Printer** | Aktifkan sharing, pilih printer USB atau mode uji, nama di client, opsi password printer, alamat client, dan antrian cetak (20 pekerjaan terbaru) |
| **Log** | Catatan aktivitas berjam, ±500 baris terakhir |
| **About** | Informasi versi dan pembuat |

**Label di Log:**

| Label | Arti |
|---|---|
| `[INFO]` | Informasi umum |
| `[ACCESS]` | Request file dari client beserta hasilnya |
| `[AUTH]` | Login gagal (dengan username yang dicoba) |
| `[LOCKOUT]` | IP dikunci karena terlalu sering gagal login |
| `[DENIED]` | Client ditolak (diblokir, bukan LAN, bukan klien Windows, batas client, dll.) |
| `[IPP]` | Aktivitas printer (pekerjaan masuk, gagal, dll.) |
| `[WARN]` / `[ERROR]` | Peringatan / kesalahan |

> Pengaturan **port, max clients, HTTPS, password, folder, dan printer** terkunci saat server berjalan. **Access** dan **Proteksi** tetap bisa diubah.

---

## 🧰 Troubleshooting

| Masalah | Kemungkinan penyebab & solusi |
|---|---|
| **Port dipakai** saat Start | Ganti port di tab Settings |
| **Client tidak bisa konek** | Cek server menyala, alamat/port benar, satu jaringan, dan **Windows Firewall** mengizinkan aplikasi/port |
| **`Access denied` / `bukan jaringan lokal`** | IP client bukan alamat lokal (filter LAN), diblokir, atau tidak ada di whitelist. Cek tab Access dan Log |
| **`Akses hanya lewat Windows Explorer`** | Filter User-Agent aktif. Akses lewat Map network drive, atau matikan filter di Settings → Proteksi |
| **`Max clients reached`** | Batas client penuh. Naikkan Max Clients atau tunggu client lain idle (±60 detik) |
| **`Terlalu banyak percobaan login`** | IP terkunci. Tunggu masa kunci habis atau klik **Unlock All** |
| **Login terus ditolak** | Cek username (sesuai yang diatur di Settings), password, dan lihat Log untuk `[AUTH]` |
| **Browser menampilkan halaman kosong/ditolak** | Normal; tampilan browser dimatikan dan filter Windows Explorer aktif |
| **Peringatan sertifikat / HTTPS gagal** | Sertifikat belum di-install di client atau sudah dibuat ulang; export dan install lagi |
| **Printer: pekerjaan "Gagal"** | Cek Log `[IPP]`. Install **SumatraPDF**, pastikan printer terpilih dan menyala |
| **Printer: tidak ada yang tercetak** | Mungkin masih di **mode uji**. Pilih printer di tab Printer lalu Start ulang |
| **Printer tidak muncul di daftar** | Klik **Refresh**; pastikan printer terpasang di Windows server |
| **Dokumen cetak ditolak** | Format bukan PDF/JPEG/PNG. Kirim sebagai PDF |
| **Folder merah** | Folder asli hilang; hapus dari daftar atau kembalikan |
| **Alamat IP salah** | Stop lalu Start lagi setelah berpindah jaringan |
| **Transfer file besar gagal di Windows** | Batas ukuran file WebDAV bawaan Windows di sisi client |
| **Tombol Start tidak bisa diklik sebentar setelah Stop** | Normal, server sedang berhenti di latar belakang |

---

## ❓ FAQ

**Apakah client perlu install aplikasi ini?**  
Tidak. Cukup fitur bawaan Windows.

**Apakah ini menggantikan SMB sepenuhnya?**  
Tidak. Ini alternatif ringan untuk kebutuhan sederhana. Untuk izin per pengguna, performa tinggi, atau lingkungan domain, SMB tetap lebih cocok.

**Apakah aman dipakai lewat internet?**  
Tidak disarankan. Aplikasi dirancang untuk LAN.

**Bisa berbagi printer saja tanpa folder?**  
Bisa. Aktifkan sharing printer di tab Printer; folder tidak wajib dipilih.

**Kenapa harus isi password lagi tiap Start?**  
Password sengaja tidak disimpan di disk. Username tetap tersimpan.

**Bisa diakses dari HP / Mac / Linux?**  
Bisa secara teknis (klien WebDAV), tetapi **matikan** filter "Hanya Windows Explorer" lebih dulu.

**Kenapa nama share jadi huruf kecil semua?**  
Mesin WebDAV tidak membedakan huruf besar/kecil pada nama share, jadi nama dinormalkan untuk menghindari bentrok.

**Apakah log tersimpan?**  
Tidak, hanya ada di memori selama aplikasi terbuka.

**Apakah dokumen cetak dihapus otomatis?**  
Sebagian. Aplikasi menyimpan 50 pekerjaan terakhir, menghapus yang lebih lama, dan membersihkan file berumur lebih dari 1 hari saat server dijalankan. Hapus manual folder `ipp-spool` bila perlu.

---

## 🧾 Nilai Bawaan

| Pengaturan | Nilai |
|---|---|
| Port | `8040` |
| Max Clients | `3` |
| Username | `user` (bisa diganti) |
| Password | Nonaktif |
| HTTPS | Nonaktif |
| Filter "Hanya Windows Explorer" | Aktif |
| Filter "Hanya LAN" | Aktif |
| Penguncian login | 5× gagal dalam 10 menit → kunci 15 menit |
| Whitelist / Blacklist | Nonaktif & kosong |
| Client dianggap aktif | Request < 60 detik |
| Sharing printer | Nonaktif (mode uji) |
| Nama printer di client | `MySharing Printer` |
| Printer minta password | Aktif (berlaku bila password share aktif) |
| Ukuran maksimal dokumen cetak | 256 MB |
| Riwayat pekerjaan cetak | 50 terakhir |
| Masa berlaku sertifikat | ±825 hari |
| Log di layar | ±500 baris terakhir |

---

*MySharing Server · Dibuat oleh Muhammad Rizki (reztdev) · © 2026 All Rights Reserved*
