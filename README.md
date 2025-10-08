Tentu. Anda benar, dokumentasi yang baik harus menjelaskan "kenapa" dan "apa dampaknya" agar tim bisa bekerja dengan percaya diri.

Berikut adalah versi dokumentasi yang lebih rinci, fokus menjelaskan dampak implementasi Spatie pada database dan alur logika aplikasi Anda.

-----

### **Dokumentasi Arsitektur & SOP: Sistem Hak Akses Dinamis (RBAC)**

**Tujuan:** Dokumen ini menjelaskan arsitektur, dampak perubahan, dan prosedur operasi standar untuk sistem *Role-Based Access Control* (RBAC) yang baru diimplementasikan dalam proyek ini.

-----

### **1. Perubahan Fundamental: Sebelum vs. Sesudah Implementasi Spatie**

Implementasi package `spatie/laravel-permission` mengubah cara kerja hak akses secara fundamental, baik di sisi database maupun logika kode.

#### **A. Sebelum Implementasi (Sistem Lama)**

  * **Struktur Database:** Hak akses hanya bergantung pada satu kolom `role` di dalam tabel `users`.
  * **Logika Kode:** Pengecekan hak akses dilakukan secara manual dan tersebar di banyak tempat (Controller, Blade) menggunakan `if (Auth::user()->role === 'admin')`.
  * **Keterbatasan:** Sistem ini kaku, sulit dikelola, dan tidak fleksibel untuk peran-peran baru.

#### **B. Sesudah Implementasi (Sistem Baru)**

Sistem yang baru memisahkan data hak akses dari tabel `users` dan memindahkannya ke tabel-tabel khusus.

**Dampak pada Database:**
Implementasi ini menambahkan **5 tabel baru** yang menjadi pusat dari sistem keamanan. Kolom `role` di tabel `users` **TIDAK LAGI DIGUNAKAN** untuk otorisasi.

| Tabel Baru | Fungsinya Apa? |
| :--- | :--- |
| `roles` | Menyimpan daftar nama peran (e.g., 'Admin', 'User'). |
| `permissions`| Menyimpan daftar semua izin yang ada. Di sistem kita, ini diisi otomatis dari nama *route*. |
| `role_has_permissions`| **Tabel Relasi Kunci:** Menghubungkan **Peran** dengan **Izin**. Di sinilah data dari halaman "Atur Izin" disimpan. |
| `model_has_roles`| **Tabel Relasi Kunci:** Menghubungkan **Pengguna** (`user_id`) dengan **Peran**. Di sinilah data dari halaman "Ubah Peran" disimpan. |
| `model_has_permissions`| Menyimpan izin yang diberikan langsung ke pengguna (jarang digunakan dalam sistem kita). |

**Dampak pada Logika Kode:**
Pengecekan hak akses sekarang terpusat dan otomatis.

  * **Cara Lama (Ditinggalkan):** `@if (Auth::user()->role === 'admin')`
  * **Cara Baru (Wajib Digunakan):**
      * **Di Backend:** `CheckPermission` Middleware secara otomatis memeriksa izin berdasarkan nama *route*.
      * **Di Frontend (Blade):** Gunakan direktif `@can('nama.izin')` untuk menyembunyikan/menampilkan tombol atau menu.

-----

### **2. Komponen Kunci Sistem**

  * **`CheckPermission` Middleware:** "Penjaga" otomatis untuk semua *route* backend. Manfaatnya, logika keamanan terpusat di satu file.
  * **`permissions:sync` Command:** "Pendeteksi" fitur baru. Manfaatnya, Anda tidak perlu manual menambahkan izin ke database setiap kali membuat *route* baru.
  * **`RolesAndPermissionsSeeder`:** "Instalatur & Pembaru". Manfaatnya, penyiapan data awal menjadi otomatis dan memastikan peran 'Admin' selalu memiliki semua akses, termasuk izin-izin baru.
  * **`RoleController` & Views:** "Ruang Kontrol". Manfaatnya, menyediakan antarmuka grafis (UI) untuk mengelola sistem yang kompleks ini dengan mudah.

-----

### **3. Prosedur Operasi Standar (SOP)**

Ikuti alur kerja ini untuk tugas-tugas pengembangan umum.

#### **A. SOP: Menambah Fitur/Halaman Backend Baru**

1.  **Buat Route**: Di `routes/web.php`, tambahkan `Route` baru di dalam grup `middleware(['auth', 'permission'])`.
2.  **Beri Nama Standar**: Pastikan `Route` tersebut diberi nama dengan awalan `backend.`.
    ```php
    // Contoh:
    Route::get('/laporan-tahunan', [LaporanController::class, 'index'])->name('backend.laporan_tahunan.index');
    ```
3.  **Update Tampilan**: Jika Anda memanggil *route* ini dari file Blade (misal: sidebar), pastikan menggunakan nama yang baru: `route('backend.laporan_tahunan.index')`.
4.  **Sinkronisasi Izin**: Jalankan perintah ini di terminal untuk mendaftarkan *route* baru sebagai izin.
    ```bash
    php artisan permissions:sync
    ```
5.  **Update Peran Admin**: Jalankan *seeder* untuk memberikan izin baru ini ke peran 'Admin' secara otomatis.
    ```bash
    php artisan db:seed --class=RolesAndPermissionsSeeder
    ```
6.  **Bersihkan Cache**: Jalankan perintah ini agar perubahan segera diterapkan.
    ```bash
    php artisan permission:cache:reset
    ```

#### **B. SOP: Mengelola Hak Akses**

  * **Mengubah Izin Peran**: Login sebagai Admin, buka halaman **Manajemen Hak Akses**, klik **"Atur Izin"** pada peran yang diinginkan, centang/hapus centang izin, lalu simpan. Pengguna dengan peran tersebut harus **login ulang** untuk merasakan perubahan.
  * **Mengubah Peran User**: Di halaman yang sama, cari pengguna di tabel "Daftar Pengguna", klik **"Ubah Peran"**, pilih peran baru dari *modal*, lalu simpan.

-----

### **4. Troubleshooting & Perintah Penting**

**Masalah Umum:** Hak akses tidak berubah setelah disimpan, atau muncul grup izin duplikat setelah mengubah nama *route*.

**Solusi:**

1.  **"Soft Reset" (Hapus Cache):** Selalu coba ini terlebih dahulu.
    ```bash
    php artisan permission:cache:reset
    php artisan route:clear
    ```
2.  **"Hard Reset" Izin:** Lakukan jika terjadi inkonsistensi data izin. **Peringatan:** Ini akan menghapus dan membangun ulang semua data izin.
      * Buka Tinker: `php artisan tinker`
      * Jalankan:
        ```php
        DB::statement('SET FOREIGN_KEY_CHECKS=0;');
        DB::table('permissions')->truncate();
        DB::table('role_has_permissions')->truncate();
        DB::table('model_has_permissions')->truncate();
        DB::statement('SET FOREIGN_KEY_CHECKS=1;');
        exit
        ```
      * Jalankan ulang Seeder: `php artisan db:seed --class=RolesAndPermissionsSeeder`.
      * Assign ulang peran ke user yang relevan.
