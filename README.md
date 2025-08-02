# 🛰️ Panduan Deploy Laravel ke InfinityFree (Anti-Gagal)

> **Panduan lengkap untuk deploy aplikasi Laravel ke hosting gratis InfinityFree dengan metode yang telah terbukti stabil dan aman.**

[![Laravel](https://img.shields.io/badge/Laravel-8%2B-red.svg)](https://laravel.com)
[![InfinityFree](https://img.shields.io/badge/Hosting-InfinityFree-blue.svg)](https://www.infinityfree.com)
[![PHP](https://img.shields.io/badge/PHP-7.4%2B-purple.svg)](https://php.net)

---

## 📋 Daftar Isi

- [Prasyarat](#-prasyarat)
- [Struktur Direktori](#-struktur-direktori)
- [Langkah Deploy](#-langkah-deploy)
- [Troubleshooting](#-troubleshooting)
- [Tips & Optimasi](#-tips--optimasi)
- [FAQ](#-faq)

---

## 🧰 Prasyarat

Pastikan kamu sudah memiliki:

- ✅ **Laravel** versi 8, 9, atau 10
- ✅ **Aplikasi** berjalan normal di lokal
- ✅ **Akun InfinityFree** aktif dengan domain/subdomain
- ✅ **FTP Client** (FileZilla) atau akses File Manager
- ✅ **Composer** terinstall di lokal

---

## 📁 Struktur Direktori

Struktur ini memastikan keamanan optimal di shared hosting dengan memisahkan file publik dan privat:

```
htdocs/
├── index.php         ← Entry point (dari public/)
├── .htaccess         ← URL rewriting rules
├── assets/           ← Isi dari folder public/ Laravel (css, js, images)
│   ├── css/
│   ├── js/
│   └── images/
└── laravel/          ← Core Laravel application
    ├── app/
    ├── bootstrap/
    ├── config/
    ├── database/
    ├── resources/
    ├── routes/
    ├── storage/
    ├── vendor/       ← Dependencies (file terbesar)
    ├── artisan
    ├── .env
    └── composer.json
```

---

## 🚀 Langkah Deploy

### 1. Persiapan Lokal

```bash
# Optimasi aplikasi untuk production
composer install --no-dev --optimize-autoloader
php artisan config:clear
php artisan route:clear
php artisan view:clear
php artisan key:generate --show
```

> **⚠️ Penting:** Simpan `APP_KEY` yang dihasilkan untuk file `.env` hosting.

### 2. Konfigurasi File Entry Point

#### Edit `index.php` (dari `public/index.php`)

```php
<?php

use Illuminate\Contracts\Http\Kernel;
use Illuminate\Http\Request;

define('LARAVEL_START', microtime(true));

// Maintenance mode check
if (file_exists($maintenance = __DIR__.'/laravel/storage/framework/maintenance.php')) {
    require $maintenance;
}

// Autoloader
require __DIR__.'/laravel/vendor/autoload.php';

// Bootstrap application
$app = require_once __DIR__.'/laravel/bootstrap/app.php';

$kernel = $app->make(Kernel::class);

$response = $kernel->handle(
    $request = Request::capture()
)->send();

$kernel->terminate($request, $response);
```

#### Edit `AppServiceProvider.php`

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register()
    {
        // Fix public path untuk InfinityFree
        $this->app->bind('path.public', function() {
            return base_path('../');
        });
    }

    public function boot()
    {
        // Force HTTPS jika diperlukan
        if (env('FORCE_HTTPS', false)) {
            \URL::forceScheme('https');
        }
    }
}
```

#### Buat Custom Asset Helper

Karena folder `public/` Laravel dipindah ke `htdocs/assets/`, kita perlu helper untuk path yang benar.

Buat file `app/helpers.php`:

```php
<?php

if (!function_exists('public_asset')) {
    function public_asset($path)
    {
        // Untuk akses file di htdocs/assets/
        return url('/assets') . '/' . ltrim($path, '/');
    }
}
```

Daftarkan di `composer.json`:

```json
{
    "autoload": {
        "files": [
            "app/helpers.php"
        ],
        "psr-4": {
            "App\\": "app/"
        }
    }
}
```

### 3. Upload ke Hosting

> **⚠️ Batasan:** InfinityFree memiliki limit upload 8MB per file. Folder `vendor/` harus diupload bertahap.

#### Strategi Upload Bertahap

**Step 1: Upload File Utama Dulu**

1. **Isi `htdocs/`:** Upload `index.php` dan `.htaccess` (yang sudah diedit) dari folder `public/` lokalmu ke `htdocs/`
2. **Upload assets:** Pindahkan isi folder `public/` (css, js, images) ke folder `assets/` di `htdocs/`
3. **Buat folder `laravel/`:** Di dalam `htdocs/`, buat folder baru bernama `laravel`
4. **Upload folder Laravel:**

```
htdocs/laravel/
├── app/              ← Upload pertama
├── bootstrap/        ← Upload kedua  
├── config/           ← Upload ketiga
├── database/         ← Upload keempat
├── resources/        ← Upload kelima
├── routes/           ← Upload keenam
├── storage/          ← Upload ketujuh
├── .env              ← Upload kedelapan
├── artisan           ← Upload kesembilan
├── composer.json     ← Upload kesepuluh
└── file lainnya...   ← Upload kesebelas
```

**Step 2: Upload Vendor (Pilih Metode)**

##### 🔥 Metode A: Upload Bertahap
```bash
# Upload satu per satu folder vendor/
vendor/
├── symfony/     ← Upload pertama (terbesar)
├── laravel/     ← Upload kedua
├── composer/    ← Upload ketiga
└── ...          ← Lanjutkan bertahap
```

##### 💡 Metode B: Kompresi
```bash
# Di lokal, split vendor menjadi bagian kecil
cd vendor/
tar -czf ../vendor-part1.tar.gz symfony/ laravel/
tar -czf ../vendor-part2.tar.gz composer/ psr/ monolog/
```

Upload file `.tar.gz` lalu extract via File Manager cPanel.

#### ⚙️ Pengaturan FileZilla

```
Transfer Settings:
✓ Concurrent transfers: 1
✓ Maximum transfers: 1
✓ Timeout: 60 seconds
✓ Passive mode: enabled
✓ Transfer mode: Binary
```

### 4. Konfigurasi Environment

Edit `.env` di `htdocs/laravel/`:

```env
APP_NAME="Your App Name"
APP_ENV=production
APP_DEBUG=false
APP_URL=http://yourdomain.page.gd
APP_KEY=base64:your_generated_key_here

# Database credentials dari InfinityFree
DB_CONNECTION=mysql
DB_HOST=sqlXXX.page.gd
DB_PORT=3306
DB_DATABASE=epiz_xxxxxxxx_dbname
DB_USERNAME=epiz_xxxxxxxx
DB_PASSWORD=your_db_password

# Session & Cache
SESSION_DRIVER=file
CACHE_DRIVER=file
QUEUE_CONNECTION=database

# Disable untuk hosting gratis
BROADCAST_DRIVER=log
MAIL_MAILER=log

# Timezone
APP_TIMEZONE=Asia/Makassar

# Optional
FORCE_HTTPS=false
```

### 5. Setup .htaccess

File `.htaccess` di `htdocs/`:

```apache
<IfModule mod_rewrite.c>
    <IfModule mod_negotiation.c>
        Options -MultiViews -Indexes
    </IfModule>

    RewriteEngine On

    # Handle Authorization Header
    RewriteCond %{HTTP:Authorization} .
    RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]

    # Redirect Trailing Slashes
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_URI} (.+)/$
    RewriteRule ^ %1 [L,R=301]

    # Front Controller
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

# Protect Sensitive Files
<FilesMatch "\.(env|log|htaccess)$">
    Order Allow,Deny
    Deny from all
</FilesMatch>
```

### 6. Database Setup

1. **Buat database** di cPanel InfinityFree
2. **Import** file SQL via phpMyAdmin
3. **Jalankan migrasi** jika menggunakan session database:

```bash
# Di lokal
php artisan session:table
php artisan cache:table
php artisan migrate
```

### 7. Testing

Buat route test di `routes/web.php`:

```php
Route::get('/test-db', function () {
    try {
        \DB::connection()->getPdo();
        return "✅ Database connection successful!";
    } catch (\Exception $e) {
        return "❌ Database failed: " . $e->getMessage();
    }
});
```

Akses: `http://yourdomain.page.gd/test-db`

---

## ⚠️ Troubleshooting

### 🔴 Kendala Umum & Solusi

| **Masalah** | **Gejala** | **Solusi** |
|-------------|------------|------------|
| **Error 500 / Halaman Putih** | Server Error, halaman kosong | • Aktifkan `APP_DEBUG=true` sementara<br>• Periksa path di `index.php`<br>• Cek permission `storage/` (755)<br>• Validasi syntax `.env` |
| **Asset (CSS/JS) Tidak Load** | Style tidak muncul, 404 error | • Gunakan helper `public_asset()`<br>• Pastikan folder `assets/` di `htdocs/`<br>• Cek console browser untuk error |
| **Session Error** | Login logout terus, session hilang | • Pastikan tabel `sessions` ada<br>• Set `SESSION_DRIVER=database`<br>• Cek permission `storage/framework/sessions/` |
| **Database Connection Failed** | Error koneksi database | • Verifikasi kredensial di `.env`<br>• Pastikan database sudah dibuat<br>• Test dengan route `/test-db` |
| **403 Forbidden** | Akses ditolak | • Periksa permission folder<br>• Cek file `.htaccess`<br>• Pastikan `index.php` ada |
| **Mixed Content Error** | HTTPS/HTTP conflict | • Set `FORCE_HTTPS=true`<br>• Update semua URL ke HTTPS |
| **Upload Gagal/Timeout** | Upload terputus, file rusak | • Upload bertahap untuk `vendor/`<br>• Set FileZilla 1 concurrent transfer<br>• Upload di jam sepi (malam) |
| **Vendor Tidak Lengkap** | Class not found error | • Cek semua package di `vendor/`<br>• Gunakan metode kompresi<br>• Verifikasi `autoload.php` |

### 🔴 Kendala Tambahan (Studi Kasus Real)

#### ❌ 1. Error: `Cannot resolve public path` saat Generate PDF

**Penyebab:** InfinityFree tidak mengenali folder `public/` Laravel karena struktur direktori diubah.

**Gejala:**
```
ErrorException: file_get_contents(public/css/app.css): failed to open stream
```

**Solusi:**
```php
// Di AppServiceProvider.php method register()
$this->app->bind('path.public', function () {
    return base_path('../');
});
```

**Checklist tambahan:**
- ✅ File `composer.json` ada di `htdocs/laravel/`
- ✅ Hapus semua cache di `bootstrap/cache/`
- ✅ Package `barryvdh/laravel-dompdf` terupload lengkap

#### ❌ 2. Error: `file_get_contents(...composer.json): Failed to open stream`

**Penyebab:** File `composer.json` belum terupload ke hosting.

**Solusi:** Upload file `composer.json` ke folder `htdocs/laravel/`

#### ❌ 3. PDF Generator Error saat `Pdf::loadView()`

**Penyebab:** Error resolve asset saat path public tidak cocok.

**Solusi Alternatif:**
```php
// Ganti dari:
$pdf = Pdf::loadView('tabungan.pdf', compact(...));

// Menjadi:
$view = view('tabungan.pdf', compact(...))->render();
$pdf = Pdf::loadHtml($view);
```

#### ❌ 4. TailwindCSS Tidak Ter-render

**Penyebab:** Helper `asset()` mengarah ke path Laravel asli.

**Solusi:** Gunakan helper kustom:
```blade
<!-- Ganti dari: -->
<link href="{{ asset('css/app.css') }}" rel="stylesheet">

<!-- Menjadi: -->
<link href="{{ public_asset('assets/css/app.css') }}" rel="stylesheet">
```

#### ❌ 5. Laravel Mix / Vite Build Assets Error

**Penyebab:** File hasil build tidak dikenali struktur hosting.

**Solusi:**
1. Pastikan file hasil `npm run build` terupload ke `htdocs/build/`
2. Gunakan path manual:
```blade
<link rel="stylesheet" href="{{ asset('build/assets/app-Bu5eBVKL.css') }}">
<script src="{{ asset('build/assets/app-CLAht3ih.js') }}" defer></script>
```

> **💡 Tip:** Nama file hash berubah setiap build. Cek nama file yang tepat di folder `build/assets/`

---

## 📦 Source Code Penting

### 🔧 `config/app.php` - Provider & Alias

Daftarkan package PDF generator:

```php
<?php

return [
    'providers' => [
        // Provider lainnya...
        Barryvdh\DomPDF\ServiceProvider::class,
    ],

    'aliases' => [
        // Alias lainnya...
        'Pdf' => Barryvdh\DomPDF\Facade\Pdf::class,
    ],
];
```

### 🔧 `app/Providers/AppServiceProvider.php` - Path Configuration

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register()
    {
        // Perbaiki path public untuk InfinityFree
        $this->app->bind('path.public', function () {
            return base_path('../');
        });
    }

    /**
     * Bootstrap any application services.
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

### 🔧 `app/Http/Controllers/ExampleController.php` - PDF Export

Contoh controller untuk generate PDF tanpa error path:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Barryvdh\DomPDF\Facade\Pdf;

class ExampleController extends Controller
{
    public function exportPdf(Request $request)
    {
        $data = $this->getFilteredQuery($request)->get();
        
        $totalPemasukan = $data->where('kategoriJenis.jenis', 'Pemasukan')->sum('nominal');
        $totalPengeluaran = $data->where('kategoriJenis.jenis', 'Pengeluaran')->sum('nominal');
        $saldoAkhir = $totalPemasukan - $totalPengeluaran;
        
        $filters = $request->only(['tanggal_mulai', 'tanggal_selesai']);

        // Render manual untuk menghindari error path
        $view = view('laporan.pdf', compact(
            'data',
            'totalPemasukan', 
            'totalPengeluaran',
            'saldoAkhir',
            'filters'
        ))->render();

        $pdf = Pdf::loadHtml($view);
        
        return $pdf->download('laporan-' . date('d-m-Y') . '.pdf');
    }
}
```

### 🔧 `app/helpers.php` - Custom Asset Helper

```php
<?php

/**
 * Generate asset URL untuk InfinityFree hosting
 */
if (!function_exists('public_asset')) {
    function public_asset($path)
    {
        return url('/') . '/' . ltrim($path, '/');
    }
}

/**
 * Check if app in production
 */
if (!function_exists('is_production')) {
    function is_production()
    {
        return app()->environment('production');
    }
}

/**
 * Generate versioned asset untuk cache busting
 */
if (!function_exists('versioned_asset')) {
    function versioned_asset($path)
    {
        $timestamp = filemtime(public_path($path));
        return public_asset($path) . '?v=' . $timestamp;
    }
}
```

### 🔧 `composer.json` - Autoload Configuration

```json
{
    "name": "laravel/laravel",
    "type": "project",
    "description": "The Laravel Framework.",
    "keywords": ["framework", "laravel"],
    "license": "MIT",
    "require": {
        "php": "^7.4|^8.0",
        "laravel/framework": "^8.75"
    },
    "autoload": {
        "files": [
            "app/helpers.php"
        ],
        "psr-4": {
            "App\\": "app/",
            "Database\\Factories\\": "database/factories/",
            "Database\\Seeders\\": "database/seeders/"
        }
    },
    "scripts": {
        "post-autoload-dump": [
            "@php artisan package:discover --ansi"
        ]
    }
}
```

### 🔧 `resources/views/layouts/app.blade.php` - Asset Loading

```blade
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="csrf-token" content="{{ csrf_token() }}">
    
    <title>{{ config('app.name', 'Laravel') }}</title>

    <!-- Styles -->
    @if(is_production())
        <!-- Production: gunakan public_asset helper -->
        <link href="{{ public_asset('assets/css/app.css') }}" rel="stylesheet">
        <link href="{{ public_asset('assets/css/bootstrap.min.css') }}" rel="stylesheet">
    @else
        <!-- Development: gunakan asset helper biasa -->
        <link href="{{ asset('css/app.css') }}" rel="stylesheet">
        <link href="{{ asset('css/bootstrap.min.css') }}" rel="stylesheet">
    @endif
</head>
<body>
    <div id="app">
        @yield('content')
    </div>

    <!-- Scripts -->
    @if(is_production())
        <script src="{{ public_asset('assets/js/app.js') }}"></script>
        <script src="{{ public_asset('assets/js/bootstrap.bundle.min.js') }}"></script>
    @else
        <script src="{{ asset('js/app.js') }}"></script>
        <script src="{{ asset('js/bootstrap.bundle.min.js') }}"></script>
    @endif
    
    @stack('scripts')
</body>
</html>
```

### 🔧 `routes/web.php` - Testing Routes

```php
<?php

use Illuminate\Support\Facades\Route;

/*
|--------------------------------------------------------------------------
| Web Routes
|--------------------------------------------------------------------------
*/

Route::get('/', function () {
    return view('welcome');
});

// Route untuk testing database connection
Route::get('/test-db', function () {
    try {
        \DB::connection()->getPdo();
        $dbName = \DB::connection()->getDatabaseName();
        return "✅ Database connection successful!<br>Database: {$dbName}";
    } catch (\Exception $e) {
        return "❌ Database connection failed: " . $e->getMessage();
    }
});

// Route untuk testing file permissions
Route::get('/test-storage', function () {
    $storagePath = storage_path('app');
    $isWritable = is_writable($storagePath);
    
    return $isWritable 
        ? "✅ Storage directory is writable" 
        : "❌ Storage directory is not writable";
});

// Route untuk testing helper functions
Route::get('/test-helpers', function () {
    return [
        'public_asset' => public_asset('css/app.css'),
        'is_production' => is_production(),
        'app_url' => config('app.url'),
        'public_path' => public_path(),
    ];
});
```

---

## 🎯 Tips & Optimasi

### Performance
```bash
# Di lokal sebelum upload
php artisan config:cache
php artisan route:cache
php artisan view:cache
npm run production  # Jika pakai Laravel Mix
```

### Security
- 🔒 Selalu gunakan `APP_DEBUG=false` di production
- 🔒 Jangan expose file `.env`
- 🔒 Update Laravel ke versi terbaru
- 🔒 Gunakan HTTPS jika tersedia

### Upload Strategy
- ⏰ Upload di jam 00:00-06:00 WIB
- 📡 Gunakan koneksi kabel untuk stabilitas
- 📦 Compress asset sebelum upload
- 💾 Backup sebelum deploy

---

## ❓ FAQ

**Q: Berapa lama waktu upload ke InfinityFree?**
A: File Laravel dasar: 5-10 menit, Vendor lengkap: 30-60 menit

**Q: Apakah bisa pakai Laravel 11?**
A: Ya, tapi perlu penyesuaian struktur folder bootstrap yang baru

**Q: Bagaimana jika upload vendor gagal terus?**
A: Coba upload di jam sepi atau gunakan metode kompresi

**Q: Bisa pakai database selain MySQL?**
A: InfinityFree hanya support MySQL/MariaDB

**Q: Bagaimana cara update aplikasi?**
A: Upload file yang berubah saja, jangan upload ulang vendor jika tidak perlu

---

## 📞 Dukungan

Jika mengalami kendala:
1. Periksa [Troubleshooting](#-troubleshooting) di atas
2. Cek error log di cPanel InfinityFree
3. Buat issue di repository ini
4. Join komunitas Laravel Indonesia

---

## 📄 Lisensi

Panduan ini gratis digunakan untuk keperluan edukasi dan komersial.

---

**💡 Pro Tip:** Bookmark panduan ini untuk referensi deploy selanjutnya!

---

*Dibuat dengan ❤️ untuk komunitas Laravel Indonesia*
