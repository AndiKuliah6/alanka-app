# PRD: Alanka Daily — Personal Productivity & Finance App
> **Versi:** v1 (Production / Deployed)
> **Terakhir Diupdate:** 2026-05-10
> **Status:** ✅ Aktif Digunakan

---

## 📌 Project Overview

**Alanka Daily** adalah aplikasi web personal untuk manajemen harian yang mencakup tiga modul utama:
1. **Keuangan** — Mencatat arus kas masuk & keluar per rekening
2. **Jurnal Harian** — Menulis catatan harian dengan sistem tagging
3. **To-Do List** — Manajemen tugas dengan prioritas dan deadline

Aplikasi ini dirancang untuk **penggunaan harian oleh satu pengguna (personal use)**, dengan data tersimpan secara cloud via Supabase dan dapat diakses dari perangkat manapun.

---

## 🛠 Tech Stack (Implementasi Aktual)

| Layer | Teknologi |
|---|---|
| **Frontend** | Vanilla HTML5 + JavaScript (ES6+) |
| **Styling** | Tailwind CSS (via CDN) |
| **Icons** | Lucide Icons (via CDN) |
| **Charts** | Chart.js (via CDN) |
| **Auth & Database** | Supabase (Auth + PostgreSQL) |
| **Hosting** | File lokal / dapat di-deploy ke Vercel/Netlify |

> ⚠️ **Catatan untuk Next Project:** Tech stack saat ini menggunakan Vanilla JS + CDN (tanpa build tool). Untuk pengembangan lanjutan, disarankan migrasi ke **Vite + React/Vue** agar lebih mudah di-maintain, komponen lebih modular, dan proses deployment lebih optimal.

---

## 🔐 Autentikasi & Keamanan

### Implementasi Saat Ini
- **Provider:** Supabase Auth (Email + Password)
- **Session Storage:** `sessionStorage` (bukan `localStorage`) — sesi mati saat tab/browser ditutup
- **Token Management:** Supabase JWT Token, di-handle otomatis oleh Supabase JS SDK v2

### Fitur Auth yang Sudah Diimplementasi

| Fitur | Status | Catatan |
|---|---|---|
| Login (Email + Password) | ✅ | `signInWithPassword()` |
| Registrasi Akun Baru | ✅ | `signUp()` — mungkin butuh konfirmasi email tergantung setting Supabase |
| Logout | ✅ | Force clear sessionStorage + localStorage token sb- |
| Lupa Password (Kirim Email) | ✅ | `resetPasswordForEmail()` dengan `redirectTo` ke app |
| Reset Password (via Link Email) | ✅ | Deteksi URL hash `#type=recovery`, tampilkan modal ganti password |
| Handle Link Expired (`otp_expired`) | ✅ | Tampilkan banner error merah yang informatif |

### Alur Reset Password (Detail)
```
1. User klik "Lupa Password?" → Modal muncul
2. User masukkan email → Klik "Kirim Link Reset"
3. Supabase kirim email dengan magic link
4. User klik link → Redirect ke app dengan URL hash:
   - Sukses:  #type=recovery&access_token=...
   - Expired: #error=access_denied&error_code=otp_expired
5. App deteksi hash:
   - Sukses  → Modal "Buat Password Baru" muncul
   - Expired → Banner error merah muncul + anjuran kirim ulang
6. User isi password baru → Simpan → sb.auth.updateUser() → Logout otomatis
7. User login ulang dengan password baru
```

### Bug Auth yang Sudah Diperbaiki
- **Race condition logout:** Flag `isLoggingOut` mencegah `onAuthStateChange` bereaksi selama proses logout
- **Double load data:** Flag `isLoadingData` mencegah `loadData()` dipanggil paralel
- **Token sisa di storage:** Saat logout, semua key `sb-*` dihapus dari sessionStorage dan localStorage
- **TOKEN_REFRESHED loop:** Event `TOKEN_REFRESHED` di-skip jika `currentUser` sudah ada

---

## 📦 Struktur File Proyek

```
v1/
├── index.html     # Semua UI (HTML + Tailwind classes)
├── app.js         # Semua logika (Auth, CRUD, Render, Navigation)
├── styles.css     # Override CSS minimal
├── PRD.MD         # Dokumen ini
└── README.md      # Catatan teknis tambahan
```

---

## 🗄 Database Schema (Supabase PostgreSQL)

### Tabel `finances`
| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | uuid (PK) | Auto-generated |
| `user_id` | uuid (FK → auth.users) | Untuk RLS |
| `type` | text | `'income'` atau `'expense'` |
| `bank` | text | `'Tunai'`, `'BCA'`, `'Sea Bank'`, `'Mandiri'` |
| `amount` | numeric | Nominal transaksi (Rp) |
| `category` | text | Kategori transaksi (lihat daftar kategori di bawah) |
| `date` | date | Tanggal transaksi |
| `note` | text | Catatan opsional |

**Kategori Pemasukan:** Gaji, Pasang Wifi, Pasang CCTV, Desain, Penjualan Babi, Komisi Penjualan Bali, Tabungan, Tarik Tunai, Lainnya

**Kategori Pengeluaran:** Sayur, Canang, Galon, Gas, Pulsa Listrik, Paket Data, MCK, Bensin, Jajan, Keluar Pacar, Keluar Temen, Belanja Online, Tiket Seminar, Bayar Sertifikasi, Iuran HIMA, Futsal Rutin, Baju Panitia, Nugas, Kasi Ortu, Ternak Babi, Sea Bank, Pegadaian, BPR, Crypto, Sakit, Melayat, Di Jalan, Kundangan, Bayar Kos, Laundry, Service Motor, Tarik Tunai, Lainnya

### Tabel `activities`
| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | uuid (PK) | Auto-generated |
| `user_id` | uuid (FK → auth.users) | Untuk RLS |
| `content` | text | Isi catatan |
| `tags` | text[] (array) | Tag-tag (misal: `["kuliah", "kerja"]`) |
| `created_at` | timestamptz | Waktu pencatatan (bisa diset manual per tanggal) |

### Tabel `tasks`
| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | uuid (PK) | Auto-generated |
| `user_id` | uuid (FK → auth.users) | Untuk RLS |
| `title` | text | Judul tugas |
| `desc` | text | Deskripsi opsional |
| `deadline` | date | Tanggal deadline |
| `priority` | text | `'High'`, `'Medium'`, atau `'Low'` |
| `is_done` | boolean | Status selesai/belum |

### RLS Policy (Row Level Security)
Setiap tabel harus punya policy agar user hanya bisa akses data miliknya sendiri:
```sql
-- Contoh untuk tabel finances (berlaku sama untuk activities dan tasks)
CREATE POLICY "Users can only access their own data"
ON finances FOR ALL
USING (auth.uid() = user_id);
```

---

## 🧩 Fitur Per Modul

### 1. Dashboard (Beranda)
- **Ringkasan Saldo** per rekening: Tunai, BCA, Sea Bank, Mandiri
- **Pemasukan & Pengeluaran bulan berjalan**
- **4 Tugas tertunda** teratas (urut priority + deadline)
- **3 Aktivitas terbaru**
- **Tombol Export CSV** laporan keuangan bulan berjalan

### 2. Modul Keuangan
- Form input: Tipe, Rekening/Dompet, Nominal, Kategori, Tanggal, Catatan
- Kategori dinamis berubah sesuai pilihan Tipe (Pemasukan/Pengeluaran)
- Pie Chart pengeluaran bulan ini per kategori (Chart.js)
- Tabel riwayat transaksi (semua waktu), urut tanggal terbaru
- Hapus transaksi individual
- Export CSV bulan berjalan

### 3. Modul Jurnal Harian
- Form input: Teks bebas (textarea), Tanggal, Tags (pisah koma)
- Filter pencarian: Keyword search + filter per tanggal
- Card view urut waktu terbaru (descending)
- Hapus catatan individual
- Tags ditampilkan sebagai badge berwarna

### 4. Modul To-Do List
- Form input: Judul, Deskripsi (opsional), Deadline, Prioritas (High/Medium/Low)
- Auto-sort: Prioritas tertinggi → Deadline terdekat
- Checkbox toggle selesai/belum (persisten ke database)
- Dua seksi terpisah: "Belum Dikerjakan" & "Sudah Dikerjakan"
- Hapus tugas individual
- Badge warna sesuai prioritas (Merah/Kuning/Biru)

---

## 🔄 Manajemen Data & Caching

### Strategi Saat Ini
1. Data diambil dari Supabase saat login (`loadData()`)
2. Disimpan sementara di `sessionStorage` sebagai `alanka_cache` (JSON)
3. Setiap navigasi antar halaman → tampil dari cache dulu (cepat) → lalu fetch ulang dari server (sinkronisasi)
4. Setiap CRUD berhasil → cache di-update → UI di-re-render tanpa reload halaman

### Real-time Sync
- Tidak menggunakan Supabase Realtime (subscription)
- Sync dilakukan secara **manual polling** setiap navigasi — cukup untuk single-user

---

## 🎨 UI/UX

### Layout
- **Desktop:** Sidebar tetap di kiri (lebar 256px) + konten di kanan
- **Mobile:** Header atas + Bottom navigation bar (4 tab)

### Design System
- **Font:** System font (sans-serif default Tailwind)
- **Primary Color:** Indigo (`#4f46e5` / indigo-600)
- **Background:** Gray-50
- **Cards:** White + shadow-sm + border border-gray-100 + rounded-2xl
- **Toast Notification:** Muncul kanan atas, 3 detik, untuk konfirmasi aksi berhasil

### Komponen UI yang Ada
| Komponen | ID Element | Keterangan |
|---|---|---|
| Auth Screen | `#auth-screen` | Full-screen overlay saat belum login |
| Modal Reset Password | `#modal-reset` | Kirim email reset |
| Modal Password Baru | `#modal-new-password` | Isi password baru setelah klik link email |
| Banner Expired | `#banner-expired` | Muncul jika link reset sudah kedaluwarsa |
| Toast Notifikasi | `#toast` | Konfirmasi sukses simpan data |
| Loading Spinner | `#data-loading` | Saat fetch pertama kali |

---

## 📋 Changelog v1

| Tanggal | Perubahan |
|---|---|
| Awal | Inisialisasi proyek: 3 modul dasar (Finance, Activity, Tasks) |
| Awal | Integrasi Supabase Auth + Database |
| 2026-05-09 | Fix: Race condition logout (`isLoggingOut` flag) |
| 2026-05-09 | Fix: Double load data (`isLoadingData` flag) |
| 2026-05-09 | Fix: Token auth tidak terhapus saat logout |
| 2026-05-09 | Fix: Auto re-login antar sesi dihilangkan (migrasi ke sessionStorage) |
| 2026-05-09 | Tambah: Favicon SVG di browser tab |
| 2026-05-10 | Fix: Fitur reset password — handle `otp_expired` via URL hash |
| 2026-05-10 | Tambah: Modal "Buat Password Baru" (setelah klik link email valid) |
| 2026-05-10 | Tambah: Banner error informatif saat link reset kedaluwarsa |

---

## 🚀 Rencana Pengembangan (Next Project / v2)

### Prioritas Tinggi
- [ ] **Migrasi ke Vite + React** — lebih modular, build process lebih baik
- [ ] **Multi-user support** — saat ini RLS sudah siap, tinggal UI-nya
- [ ] **Dark Mode** — toggle gelap/terang
- [ ] **PWA (Progressive Web App)** — `manifest.json` + Service Worker agar bisa di-install di HP

### Prioritas Menengah
- [ ] **Filter keuangan per bulan/tahun** — saat ini hanya tampil semua
- [ ] **Edit transaksi/tugas/catatan** — saat ini hanya bisa hapus & tambah baru
- [ ] **Recurring Tasks** — tugas yang berulang tiap minggu/bulan
- [ ] **Budget Limit per Kategori** — alert jika pengeluaran melebihi batas
- [ ] **Grafik tren keuangan** — Line chart pemasukan vs pengeluaran per bulan

### Prioritas Rendah
- [ ] **Export PDF laporan** — laporan bulanan yang bisa dicetak
- [ ] **Notifikasi deadline tugas** — via browser notification API
- [ ] **Supabase Realtime** — sync otomatis antar tab/device tanpa perlu navigasi
- [ ] **Import CSV** — masukkan data keuangan dari file CSV

---

## ⚙️ Setup & Deployment

### Supabase Setup (Wajib untuk v2)
1. Buat project baru di [supabase.com](https://supabase.com)
2. Buat 3 tabel: `finances`, `activities`, `tasks` (schema lihat di atas)
3. Aktifkan RLS pada setiap tabel dan tambahkan policy
4. Salin `SUPABASE_URL` dan `SUPABASE_KEY` (anon key) ke konfigurasi app
5. Di **Authentication > URL Configuration**, set **Site URL** ke URL app yang akan di-deploy (bukan localhost) agar link reset password mengarah ke domain yang benar

### Variabel Konfigurasi (saat ini hardcoded di app.js baris 1-2)
```javascript
const SUPABASE_URL = 'https://[project-ref].supabase.co';
const SUPABASE_KEY = 'eyJ...'; // anon public key
```
> ⚠️ Untuk v2, pindahkan ke `.env` file dan jangan commit ke git!

### Menjalankan Lokal
Karena menggunakan CDN, cukup buka `index.html` via browser atau gunakan Live Server extension di VS Code.

---

## 👤 Informasi Developer

- **Developer:** Lespa
- **Kontak:** (isi sendiri)
- **Repository:** (isi link repo jika ada)
