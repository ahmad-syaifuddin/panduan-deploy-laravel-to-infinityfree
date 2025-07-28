# 🛰️ Panduan Revisi: Deploy Laravel ke InfinityFree (Anti-Gagal)

Panduan ini adalah versi yang disempurnakan untuk memastikan aplikasi Laravel kamu berjalan lancar di **InfinityFree** dengan metode yang paling stabil.

---

## 🧰 Prasyarat

- Laravel versi 8, 9, atau 10.
- Aplikasi sudah berjalan normal di lokal.
- Akun InfinityFree aktif dengan domain/subdomain siap pakai.
- FTP client (seperti FileZilla) atau akses File Manager.

---

## 📁 Struktur Direktori

Struktur ini adalah yang terbaik untuk keamanan di *shared hosting*. Tujuannya agar hanya folder `public` yang bisa diakses dari luar.

```
htdocs/
├── index.php         ← dari folder public/ (sudah diedit)
├── .htaccess         ← dari folder public/ (sudah diedit)
├── assets/           ← (Opsional) Folder untuk CSS/JS/Gambar
└── laravel/          ← Semua sisa file & folder Laravel
    ├── app/
    ├── bootstrap/
    ├── config/
    ├── database/
    ├── resources/
    ├── routes/
    ├── storage/
    ├── vendor/
    ├── artisan
    ├── .env
    └── ... (file lainnya)
```

---

## 🔧 Langkah-langkah Deploy

### 1. ✅ Siapkan di Lokal

Di terminal lokal kamu, jalankan perintah ini untuk optimasi:

```bash
composer install --no-dev --optimize-autoloader
php artisan config:clear
php artisan route:clear
php artisan view:clear
php artisan key:generate --show
```

**Penting:**
- Gunakan `config:clear`, `route:clear`, dan `view:clear` (bukan `cache`). *Caching* konfigurasi kadang menyebabkan masalah path di InfinityFree.
- Salin `APP_KEY` yang muncul dari perintah `key:generate --show` untuk dimasukkan ke file `.env` nanti.

---

### 2. 🛠️ (REVISI) Edit `index.php` dan Path Configuration

Ini adalah langkah paling krusial yang sering gagal.

**a. Edit `htdocs/index.php` (sebelumnya `public/index.php`)**

Ubah path agar menunjuk ke folder `laravel/`:

```php
<?php

use Illuminate\Contracts\Http\Kernel;
use Illuminate\Http\Request;

define('LARAVEL_START', microtime(true));

/*
|--------------------------------------------------------------------------
| Check If The Application Is Under Maintenance
|--------------------------------------------------------------------------
|
| If the application is in maintenance / demo mode via the "down" command
| we will load this file so that any pre-rendered content can be shown
| instead of starting the framework, which could cause an exception.
|
*/

if (file_exists($maintenance = __DIR__.'/laravel/storage/framework/maintenance.php')) {
    require $maintenance;
}

/*
|--------------------------------------------------------------------------
| Register The Auto Loader
|--------------------------------------------------------------------------
|
| Composer provides a convenient, automatically generated class loader for
| this application. We just need to utilize it! We'll simply require it
| into the script here so we don't need to manually load our classes.
|
*/

require __DIR__.'/laravel/vendor/autoload.php';

/*
|--------------------------------------------------------------------------
| Run The Application
|--------------------------------------------------------------------------
|
| Once we have the application, we can handle the incoming request using
| the application's HTTP kernel. Then, we will send the response back
| to this client's browser, allowing them to enjoy our application.
|
*/

$app = require_once __DIR__.'/laravel/bootstrap/app.php';

$kernel = $app->make(Kernel::class);

$response = $kernel->handle(
    $request = Request::capture()
)->send();

$kernel->terminate($request, $response);
```

**b. Edit `laravel/app/Providers/AppServiceProvider.php`**

Beberapa fitur seperti *asset linking* mungkin error. Atasi dengan menambahkan kode berikut di dalam method `register()` untuk memperbaiki path `public`.

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     *
     * @return void
     */
    public function register()
    {
        // Perbaiki path public untuk InfinityFree
        $this->app->bind('path.public', function() {
            return base_path('../');
        });
    }

    /**
     * Bootstrap any application services.
     *
     * @return void
     */
    public function boot()
    {
        // Paksa HTTPS jika diperlukan (opsional)
        if (env('FORCE_HTTPS', false)) {
            \URL::forceScheme('https');
        }
    }
}
```

**c. Buat Custom Helper untuk Asset (Opsional tapi Disarankan)**

Buat file `laravel/app/helpers.php`:

```php
<?php

if (!function_exists('public_asset')) {
    function public_asset($path)
    {
        return url('/') . '/' . ltrim($path, '/');
    }
}
```

Kemudian daftarkan di `laravel/composer.json`:

```json
{
    "autoload": {
        "files": [
            "app/helpers.php"
        ],
        "psr-4": {
            "App\\": "app/",
            "Database\\Factories\\": "database/factories/",
            "Database\\Seeders\\": "database/seeders/"
        }
    }
}
```

---

### 3. 📂 Upload File ke Hosting

Gunakan **FileZilla** untuk proses yang lebih andal daripada File Manager web.

1. **Isi `htdocs/`**: Upload `index.php` dan `.htaccess` (yang sudah diedit) dari folder `public/` lokalmu ke `htdocs/`.
2. **Buat folder `laravel/`**: Di dalam `htdocs/`, buat folder baru bernama `laravel`.
3. **Isi `htdocs/laravel/`**: Upload **semua sisa folder dan file Laravel** (termasuk `vendor/` dan `.env`) ke dalam folder `htdocs/laravel/`.
4. **Set Permissions**: Pastikan folder `laravel/storage/` dan `laravel/bootstrap/cache/` memiliki permission 755 atau 775.

---

### 4. 🔑 (REVISI) Sesuaikan File `.env`

Edit file `.env` di `htdocs/laravel/` dengan kredensial dari InfinityFree.

```env
APP_NAME="Nama Aplikasi Kamu"
APP_ENV=production
APP_DEBUG=false
APP_URL=http://domainkamu.epizy.com

# Kunci Aplikasi dari langkah 1
APP_KEY=base64:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=

# Kredensial Database dari Control Panel InfinityFree
DB_CONNECTION=mysql
DB_HOST=sqlXXX.epizy.com
DB_PORT=3306
DB_DATABASE=epiz_xxxxxxxx_namadatabase
DB_USERNAME=epiz_xxxxxxxx
DB_PASSWORD=password_database_kamu

# Pengaturan Session & Cache (Penting untuk InfinityFree)
SESSION_DRIVER=database
SESSION_LIFETIME=120
SESSION_ENCRYPT=false
SESSION_PATH=/
SESSION_DOMAIN=null

CACHE_DRIVER=database
QUEUE_CONNECTION=database

# Disable Broadcasting & Mail untuk menghindari error
BROADCAST_DRIVER=log
MAIL_MAILER=log

# Timezone
APP_TIMEZONE=Asia/Makassar

# Optional: Force HTTPS
FORCE_HTTPS=false
```

**Penting:** Menggunakan `database` sebagai `SESSION_DRIVER` dan `CACHE_DRIVER` lebih disarankan karena seringkali izin tulis ke folder `storage` di hosting gratis tidak stabil.

---

### 5. 🧷 (REVISI) Edit File `.htaccess` di `htdocs/`

File `.htaccess` di root `htdocs/` sangat penting untuk mengarahkan semua permintaan ke `index.php`. Ganti seluruh isinya dengan kode di bawah agar lebih stabil dan aman.

```apache
<IfModule mod_rewrite.c>
    <IfModule mod_negotiation.c>
        Options -MultiViews -Indexes
    </IfModule>

    RewriteEngine On

    # Handle Authorization Header
    RewriteCond %{HTTP:Authorization} .
    RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]

    # Redirect Trailing Slashes If Not A Folder...
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_URI} (.+)/$
    RewriteRule ^ %1 [L,R=301]

    # Send Requests To Front Controller...
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^ index.php [L]
</IfModule>

# Security Headers
<IfModule mod_headers.c>
    Header always set X-Content-Type-Options nosniff
    Header always set X-Frame-Options DENY
    Header always set X-XSS-Protection "1; mode=block"
</IfModule>

# Disable Directory Browsing
Options -Indexes

# Protect sensitive files
<FilesMatch "\.(env|log|htaccess|htpasswd)$">
    Order Allow,Deny
    Deny from all
</FilesMatch>
```

---

### 6. 🧪 Import Database & Jalankan Migrasi

1. **Buat Database**: Di cPanel InfinityFree, buat database baru.
2. **Import SQL**: Masuk ke **phpMyAdmin**, pilih databasenya, lalu *import* file `.sql` dari lokal.
3. **Migrasi Tambahan**: Jika kamu mengikuti saran untuk `SESSION_DRIVER=database`, kamu perlu menambahkan tabel `sessions` dan `cache`.

**Di lokal**, jalankan:
```bash
php artisan session:table
php artisan cache:table
php artisan migrate
```

Kemudian ekspor database yang sudah lengkap dan import ke InfinityFree.

---

### 7. 🔄 Optimasi Tambahan

**a. Buat file `laravel/bootstrap/cache/config.php` secara manual:**

Di lokal, jalankan:
```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

Upload file cache yang dihasilkan ke server.

**b. Compress Assets (Opsional):**

Jika menggunakan Laravel Mix, pastikan untuk build production:
```bash
npm run production
```

---

### 8. ✅ Testing & Debugging

1. **Test Basic Route**: Akses `http://domainkamu.epizy.com`
2. **Test Database**: Buat route sederhana untuk test koneksi database
3. **Check Error Logs**: Periksa error log di cPanel jika ada masalah

**Route Test Database** (tambahkan di `routes/web.php`):
```php
Route::get('/test-db', function () {
    try {
        \DB::connection()->getPdo();
        return "Database connection successful!";
    } catch (\Exception $e) {
        return "Database connection failed: " . $e->getMessage();
    }
});
```

---

## ⚠️ Kendala Umum & Solusi

| Masalah | Solusi |
|---------|--------|
| **Error 500 / Halaman Putih** | Aktifkan `APP_DEBUG=true` sementara. Periksa path di `index.php`, syntax `.env`, dan permission `storage/`. |
| **Asset (CSS/JS) Tidak Load** | Gunakan helper `public_asset()` atau path manual. Pastikan folder `assets/` di `htdocs/`. |
| **Session Error** | Pastikan tabel `sessions` sudah ada dan `SESSION_DRIVER=database`. |
| **Database Connection Failed** | Periksa kredensial database di `.env` dan pastikan database sudah dibuat. |
| **403 Forbidden** | Periksa permission folder dan file `.htaccess`. |
| **Mixed Content Error** | Set `FORCE_HTTPS=true` jika menggunakan HTTPS. |

---

## 📝 Tips Tambahan

1. **Backup Selalu**: Selalu backup database dan file sebelum deploy.
2. **Version Control**: Gunakan Git untuk tracking perubahan.
3. **Environment Specific**: Pisahkan konfigurasi untuk development dan production.
4. **Monitor Error Logs**: Selalu cek error log di cPanel untuk debugging.
5. **Optimize Images**: Compress gambar sebelum upload untuk menghemat storage.

---

## 🚀 Selesai!

Jika semua langkah diikuti dengan benar, aplikasi Laravel kamu seharusnya sudah berjalan lancar di InfinityFree. Ingat untuk selalu test setiap functionality setelah deploy.
