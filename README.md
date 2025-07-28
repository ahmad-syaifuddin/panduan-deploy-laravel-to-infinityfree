# 🛰️ Panduan Deploy Laravel ke InfinityFree (Survival Mode)

Panduan lengkap ini akan membantu kamu mendeploy aplikasi Laravel ke hosting gratis **InfinityFree** dengan cara yang kompatibel, aman, dan tested.

---

## 🧰 Prasyarat

- Laravel versi 8 sampai 10 (pakai struktur standar Laravel)
- Sudah build aplikasi di lokal (Composer, Artisan, Key, dsb)
- Akun InfinityFree aktif dan domain/subdomain sudah disiapkan
- Punya akses ke File Manager / FTP

---

## 📁 Struktur Direktori

Kita akan pakai **struktur survival mode**:

```
htdocs/
├── index.php          ← dari folder public/
├── .htaccess          ← dari folder public/
├── laravel/           ← semua folder Laravel lainnya (app, routes, vendor, dsb)
│   ├── app/
│   ├── bootstrap/
│   ├── config/
│   ├── database/
│   ├── public/
│   ├── resources/
│   ├── routes/
│   ├── storage/
│   ├── tests/
│   ├── vendor/
│   ├── artisan
│   ├── composer.json
│   └── .env
```

---

## 🔧 Langkah-langkah Deploy

### 1. ✅ Siapkan di Lokal

Jalankan di terminal lokal kamu (bukan di hosting):

```bash
composer install --no-dev --optimize-autoloader
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan key:generate
```

> Simpan `.env` yang udah kamu pakai! Karena nanti akan diedit untuk koneksi ke database InfinityFree.

---

### 2. 🛠️ Edit `index.php` di folder `public/`

Ganti path di file `index.php` ini:

```php
// from
require __DIR__.'/../vendor/autoload.php';
$app = require_once __DIR__.'/../bootstrap/app.php';

// to (karena file Laravel kamu pindah ke /laravel)
require __DIR__.'/laravel/vendor/autoload.php';
$app = require_once __DIR__.'/laravel/bootstrap/app.php';
```

---

### 3. 📂 Pindahkan Folder ke Struktur Hosting

Upload file/folder seperti ini ke **File Manager InfinityFree**:

| Folder/File         | Tujuan                |
|---------------------|------------------------|
| Isi folder `public/` | ke `htdocs/`          |
| Folder Laravel lain | ke `htdocs/laravel/`   |

> Jangan lupa: file `.env` dan folder `vendor/` harus ikut diupload ke `htdocs/laravel/`

---

### 4. 🔑 Sesuaikan File `.env`

Edit `.env` kamu:

```env
APP_NAME="Laravel App"
APP_ENV=production
APP_KEY=base64:xxxxxx  ← biarkan dari hasil artisan key:generate
APP_DEBUG=false
APP_URL=http://yourdomain.epizy.com

DB_CONNECTION=mysql
DB_HOST=sqlXXX.epizy.com
DB_PORT=3306
DB_DATABASE=epiz_12345678_laravel
DB_USERNAME=epiz_12345678
DB_PASSWORD=yourpassword
```

> Ganti `sqlXXX.epizy.com` dan kredensialnya sesuai yang kamu dapat dari **InfinityFree MySQL Panel**

---

### 5. 🧷 Edit File `.htaccess` di `htdocs/`

Tambahkan baris ini di awal file `.htaccess`:

```apache
# Fix path to Laravel subfolder
RewriteEngine On
RewriteRule ^(.*)$ laravel/public/$1 [L]
```

> Jangan hapus aturan `.htaccess` asli dari Laravel `public/`, tapi tambahkan rewrite rule di atas **paling awal**.

---

### 6. 🧪 Import Database

- Masuk ke cPanel → **phpMyAdmin**
- Buat database sesuai nama di `.env`
- Klik database → Import → Upload file `.sql` dari lokal

---

### 7. 📸 Permission Folder

Pastikan folder `storage/` dan `bootstrap/cache/` sudah:

- Ada di `htdocs/laravel/`
- Memiliki permission 755 (bisa atur dari File Manager jika error log)

---

## ✅ Selesai! Cek Website Kamu

Akses `http://yourdomain.epizy.com`

Kalau masih error:
- Cek `htdocs/laravel/storage/logs/laravel.log`
- Cek `APP_DEBUG=true` sementara waktu untuk melihat error
- Pastikan tidak ada folder/file yang hilang saat upload (vendor, env, index.php)

---

## ⚠️ Kendala Umum

| Masalah                          | Solusi                                               |
|----------------------------------|-------------------------------------------------------|
| Error 500 Internal Server Error | Cek `storage/logs/laravel.log`, index.php, .env      |
| File not found                   | Pastikan semua file Laravel lengkap & path benar     |
| Composer not available           | Wajib build di lokal, gak bisa di server             |
| Artisan command gak jalan       | Karena gak ada SSH, jalankan artisan dari lokal saja |

---

## 🧠 Tips Lanjutan

- Jangan pakai fitur artisan scheduler, queue, atau storage symlink (gak akan jalan di InfinityFree)
- Gunakan hanya fitur dasar Laravel: routing, controller, model, view, blade, auth
- Bisa pakai CDN buat asset (JS, CSS, gambar) agar lebih ringan

---

## 🧼 Penutup

InfinityFree cocok buat **project demo, portfolio, testing**. Kalau untuk production, mending pindah ke shared/VPS hosting yang support Laravel full features.
