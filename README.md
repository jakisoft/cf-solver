# cf-solver-scraper

`cf-solver-scraper` adalah sebuah server API Node.js yang dirancang sebagai paket eksperimental dan edukasi untuk berinteraksi dengan proteksi Cloudflare. Server ini menggunakan `puppeteer-real-browser` untuk mengotomatiskan tugas-tugas browser dan menyediakan beberapa mode operasi untuk mengekstrak data atau menyelesaikan tantangan.

## Fitur Utama

Project ini menyediakan satu endpoint `POST /solver` yang menerima berbagai mode operasi:

* **`source`**: Mengambil konten HTML penuh (source code) dari URL target setelah halaman selesai dimuat.
* **`waf-session`**: Menavigasi ke URL target dan mengekstrak cookie serta header request yang relevan (seperti `accept-language`) yang diperlukan untuk sesi WAF (Web Application Firewall).
* **`turnstile-min`**: Menyelesaikan tantangan Cloudflare Turnstile dengan mode minimal. Ini bekerja dengan mencegat request ke URL target dan menyajikan halaman HTML kustom (`data/fakePage.html`) yang hanya berisi widget Turnstile, lalu menunggu token respons. Mode ini memerlukan `siteKey`.
* **`turnstile-max`**: Menyelesaikan tantangan Cloudflare Turnstile dengan memuat URL target secara penuh (mode maksimal) dan menunggu widget Turnstile dieksekusi secara alami di halaman tersebut untuk mendapatkan token.

Fitur tambahan:
* **Dukungan Proxy**: Semua mode mendukung penggunaan proxy HTTP, termasuk proxy dengan autentikasi username dan password.
* **Validasi Request**: Menggunakan `ajv` untuk memvalidasi skema body request yang masuk.
* **Manajemen Browser**: Mengelola instance browser secara global menggunakan `puppeteer-real-browser` untuk efisiensi.
* **Autentikasi**: Mendukung token otorisasi (`authToken`) opsional untuk mengamankan endpoint.

## Teknologi yang Digunakan

* **Runtime**: Node.js
* **Framework Server**: Express.js
* **Automasi Browser**: `puppeteer-real-browser`
* **Validasi Skema**: `ajv` dan `ajv-formats`
* **Utility**: `body-parser`, `cors`, `dotenv`
* **Deployment**: Docker, `pm2`

## Prasyarat Instalasi

### Lokal
* Node.js (versi terbaru direkomendasikan)
* NPM (Node Package Manager)

### Untuk Deployment (via Docker)
Berdasarkan `Dockerfile`, dependensi sistem berikut diperlukan:
* `chromium`
* `chromium-driver`
* `xvfb` (X virtual framebuffer, untuk menjalankan browser 'headless' di server)

## Instalasi dan Menjalankan

### Lokal
1.  Clone repositori ini.
2.  Instal dependensi:
    ```bash
    npm install
    ```
3.  Buat file `.env` di root project untuk mengatur variabel lingkungan (opsional):
    ```env
    PORT=3000
    authToken=your_secret_token_here
    browserLimit=20
    timeOut=60000
    ```
    * `PORT`: Port yang akan digunakan server (default: 3000).
    * `authToken`: Token yang harus disertakan klien (default: null/tidak ada).
    * `browserLimit`: Jumlah maksimum koneksi browser bersamaan (default: 20).
    * `timeOut`: Waktu timeout global untuk setiap request (default: 60000ms).

4.  Jalankan server:
    ```bash
    npm start
    ```
    (Skrip ini menjalankan `node src/index.js`. Pastikan file `index.js` Anda berada di lokasi yang benar sesuai `package.json`).

### Menggunakan Docker
1.  Build image Docker:
    ```bash
    docker build -t cf-solver .
    ```
2.  Jalankan kontainer:
    ```bash
    # Ganti 3000 jika Anda menggunakan PORT yang berbeda di .env
    docker run -p 3000:3000 --env-file ./.env -d cf-solver
    ```
    Server akan berjalan di dalam kontainer dan terekspos di port 3000.

## Susunan Project

/

├── index.js # Entry point server Express

├── Dockerfile # Konfigurasi build Docker

├── LICENSE # Lisensi MIT

├── package.json # Dependensi NPM dan skrip

├── endpoints/ # Logika untuk setiap mode 'solver'

│ ├── getSource.js # Implementasi mode 'source'

│ ├── solveTurnstile.max.js # Implementasi mode 'turnstile-max'

│ ├── solveTurnstile.min.js # Implementasi mode 'turnstile-min'

│ └── wafSession.js # Implementasi mode 'waf-session'

├── module/ # Modul helper internal

│ ├── createBrowser.js # Logika untuk inisialisasi puppeteer

│ └── reqValidate.js # Skema dan fungsi validasi request

└── data/ # Aset statis

├── fakePage.html # Template HTML untuk mode 'turnstile-min'

└── sdo.gif # (File gambar)


## Contoh Penggunaan

Kirim request `POST` ke endpoint `/solver` dengan body JSON.

### Mode: `source`
Mengambil kode sumber HTML dari sebuah halaman.

**Request:**
```json
{
  "mode": "source",
  "url": "[https://httpbin.org/get](https://httpbin.org/get)"
}
```
**Respon Sukses (200):**

```json

{
  "source": "<html>... (konten HTML halaman) ...</html>",
  "code": 200
}
```
# Mode: waf-session
Mendapatkan cookie dan header dari sesi WAF.

**Request:**

```json
{
  "mode": "waf-session",
  "url": "[https://example.com/protected](https://example.com/protected)"
}
```

**Respon Sukses (200):**
```json
{
  "cookies": [
    {
      "name": "cf_clearance",
      "value": "...",
      ...
    }
  ],
  "headers": {
    "user-agent": "...",
    "accept-language": "en-US,en;q=0.9",
    ...
  },
  "code": 200
}
```

# Mode: turnstile-min
Menyelesaikan Turnstile menggunakan siteKey.

**Request:**
```json
{
  "mode": "turnstile-min",
  "url": "[https://example.com/login](https://example.com/login)",
  "siteKey": "0x4AAAAAAABcdE_Js_challenge_site_key"
}
```

**Respon Sukses (200):**
```json
{
  "token": "0.T3... (token turnstile) ...-w",
  "code": 200
}
```

# Mode: turnstile-max
Menyelesaikan Turnstile dengan memuat halaman penuh.

**Request:**
```json
{
  "mode": "turnstile-max",
  "url": "[https://example.com/page-with-turnstile](https://example.com/page-with-turnstile)"
}
```

**Respon Sukses (200):**
```json
{
  "token": "0.T3... (token turnstile) ...-w",
  "code": 200
}
```

# Contoh dengan Proxy dan Auth
Semua mode mendukung parameter proxy dan authToken.

**Request:**
```json
{
  "mode": "source",
  "url": "[https://example.com](https://example.com)",
  "authToken": "your_secret_token_here",
  "proxy": {
    "host": "proxy.example.com",
    "port": 8080,
    "username": "proxy_user",
    "password": "proxy_password"
  }
}
```

# Kontribusi
Kontribusi dipersilakan. Jika Anda menemukan masalah atau memiliki ide untuk perbaikan, silakan buka Issue atau kirimkan Pull Request.

# Lisensi
Project ini dilisensikan di bawah Lisensi MIT. Lihat file `LICENSE` untuk detail lengkapnya.
