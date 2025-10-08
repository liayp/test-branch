### \#\#\# **Dokumentasi & SOP: Sistem Hak Akses Dinamis**

**Tujuan:** Dokumen ini menjelaskan implementasi dan prosedur standar untuk sistem *Role-Based Access Control* (RBAC) pada proyek ini.

-----

### \#\# 1. Konsep & Arsitektur

Sistem ini menggunakan package `spatie/laravel-permission` untuk mengelola hak akses secara terpusat dari database.

**Aturan Emas (Golden Rule):**

> Semua *route* backend yang perlu dilindungi hak aksesnya **WAJIB** memiliki nama yang diawali dengan `backend.`.

**Alur Kerja Sistem:**

1.  **`routes/web.php`**: Mendefinisikan *route* dengan nama berawalan `backend.`.
2.  **`php artisan permissions:sync`**: Perintah ini membaca nama-nama *route* tersebut dan mendaftarkannya sebagai **Izin** (*Permissions*) di database.
3.  **Halaman Admin**: Antarmuka untuk memberikan **Izin** kepada **Peran** (*Roles*).
4.  **`CheckPermission` Middleware**: "Penjaga" yang memeriksa apakah pengguna yang login memiliki **Izin** yang sesuai dengan *route* yang diakses.

-----

### \#\# 2. Prosedur Operasi Standar (SOP)

Ikuti alur kerja ini untuk tugas-tugas pengembangan umum.

#### **A. SOP: Menambah Fitur/Halaman Backend Baru**

Lakukan 6 langkah ini secara berurutan:

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

### \#\# 3. Perintah Penting (Cheat Sheet)

| Perintah | Fungsi |
| :--- | :--- |
| `php artisan permissions:sync` | Mendaftarkan *route* baru sebagai izin. Dijalankan setelah mengubah `routes/web.php`. |
| `php artisan db:seed --class=...` | Memberikan izin baru ke Admin & mengatur data awal. |
| `php artisan permission:cache:reset`| Membersihkan *cache* jika perubahan hak akses tidak langsung terlihat. |
| `php artisan route:clear`| Membersihkan *cache* *route*. Wajib dijalankan setelah mengubah `routes/web.php`.|

-----

### \#\# 4. Troubleshooting Umum

**Masalah:** Muncul grup izin duplikat di halaman "Atur Izin" (misalnya `produk-lama` dan `produk_baru`) setelah mengubah nama *route*. Ini terjadi karena izin lama yang tidak terpakai tidak terhapus secara otomatis.

**Solusi ("Hard Reset"):**
⚠️ **Peringatan:** Prosedur ini akan **menghapus semua data izin** dan relasinya, lalu membangunnya kembali dari awal.

1.  **Kosongkan Tabel Izin (via Tinker):**
    ```bash
    php artisan tinker
    ```
    Lalu jalankan perintah berikut di dalam Tinker:
    ```php
    DB::statement('SET FOREIGN_KEY_CHECKS=0;');
    DB::table('permissions')->truncate();
    DB::table('role_has_permissions')->truncate();
    DB::table('model_has_permissions')->truncate(); // Jika menggunakan direct permission
    DB::statement('SET FOREIGN_KEY_CHECKS=1;');
    exit
    ```
2.  **Jalankan Ulang Seeder** untuk membangun ulang semua izin & relasi Admin dari awal.
    ```bash
    php artisan db:seed --class=RolesAndPermissionsSeeder
    ```
3.  **Assign Ulang Peran ke User** jika relasinya ikut terhapus (selain Admin yang sudah di-handle oleh Seeder).
