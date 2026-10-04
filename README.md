# Starlabs AM Preset Finder Scraper

Scraper JavaScript untuk mencari link preset **Alight Motion** dari komentar video TikTok melalui endpoint publik Starlabs/Bintang API.

Tool ini menerima URL video TikTok, mengirim request `GET` ke endpoint API, lalu mengembalikan informasi video, pembuat konten, thumbnail, jumlah komentar yang dipindai, dan daftar preset Alight Motion yang ditemukan.

> Endpoint tidak memerlukan API key, login, atau CAPTCHA. Namun, ketersediaan endpoint dan hasil pencarian tetap bergantung pada server pihak ketiga serta akses komentar TikTok.

---

## Informasi Proyek

| Properti | Detail |
|---|---|
| Nama | Starlabs AM Preset Finder Scraper |
| Creator GitHub | SCRIPETEREN |
| Base Web | [starlabs.biz.id/ampresetfind](https://starlabs.biz.id/ampresetfind) |
| Tipe | Scraper / API Wrapper |
| Fungsi Utama | Mencari link preset Alight Motion dari komentar video TikTok |
| Metode | HTTP `GET` ke endpoint JSON publik |
| API Endpoint | `https://bintangapi.my.id/api/amfind/?url=` |
| API Key | Tidak diperlukan |
| Login | Tidak diperlukan |
| CAPTCHA | Tidak diperlukan |

---

## Fitur

- Validasi dasar URL sebelum request dikirim.
- Mendukung URL TikTok dari domain umum:
  - `www.tiktok.com`
  - `vt.tiktok.com`
  - `vm.tiktok.com`
- Mengambil data JSON dari endpoint publik.
- Mengembalikan informasi video TikTok.
- Mengembalikan informasi akun pembuat video.
- Mengambil preset Alight Motion yang ditemukan pada komentar.
- Mengembalikan URL thumbnail video jika tersedia.
- Menampilkan total komentar yang dipindai oleh server.
- Memiliki timeout request hingga 110 detik.
- Memiliki mekanisme retry maksimal 2 kali untuk kegagalan jaringan atau timeout.
- Tidak melakukan retry buta apabila server sudah mengembalikan respons final `success: false`.
- Pesan error dibatasi agar aman ditampilkan ke pengguna.

---

## Instalasi

Modul ini menggunakan sintaks **ES Module** dan membutuhkan runtime JavaScript modern yang mendukung:

- `fetch`
- `AbortSignal.timeout`
- `async/await`
- ES Module `import` dan `export`

Direkomendasikan menggunakan **Node.js 18+**, dengan Node.js 20+ sebagai pilihan yang lebih aman untuk kompatibilitas API bawaan.

Simpan kode scraper sebagai:

```text
am-preset-find.js
```

Kemudian impor ke proyek:

```js
import { amPresetFind } from './am-preset-find.js'
```

Atau gunakan default export:

```js
import amPresetFinder from './am-preset-find.js'
```

---

## Kode Utama

```js
const API = '[https://bintangapi.my.id/api/amfind/?url=](https://bintangapi.my.id/api/amfind/?url=)'
const UA = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36'
const REFERER = '[https://starlabs.biz.id/ampresetfind/](https://starlabs.biz.id/ampresetfind/)'

const TIMEOUT_MS = 110_000
const MAX_RETRIES = 2

const sleep = (ms) => new Promise((r) => setTimeout(r, ms))

/**
 * Cari preset AM dari URL video TikTok.
 * @param {string} tiktokUrl URL video TikTok (vt.tiktok.com / www.tiktok.com / vm.tiktok.com)
 * @returns {Promise<{ok:true, video, user, preset, thumbnail, totalComments} | {ok:false, error:string}>}
 */
export async function amPresetFind(tiktokUrl) {
  const u = String(tiktokUrl || '').trim()

  if (!/^https?:///i.test(u)) {
    return {
      ok: false,
      error: 'Link harus diawali http:// atau https://',
    }
  }

  if (!/tiktok.com//i.test(u)) {
    return {
      ok: false,
      error: 'Link harus dari TikTok.',
    }
  }

  let lastError = 'Gagal menghubungi server.'

  for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
    try {
      const res = await fetch(API + encodeURIComponent(u), {
        headers: {
          'User-Agent': UA,
          Accept: 'application/json',
          Referer: REFERER,
        },
        signal: AbortSignal.timeout(TIMEOUT_MS),
      })

      const json = await res.json().catch(() => null)

      if (!json || typeof json !== 'object') {
        lastError = `Server balas bukan JSON (HTTP ${res.status}).`
        continue
      }

      if (json.success === true && json.data) {
        const d = json.data

        return {
          ok: true,
          video: d.video || {},
          user: d.user || {},
          preset: d.preset || null,
          thumbnail: d.thumbnail || null,
          totalComments: d.total_comments_scanned ?? null,
        }
      }

      // success:false merupakan respons final dari server.
      // Tidak perlu retry karena umumnya bukan error jaringan sementara.
      return {
        ok: false,
        error: String(
          json.error ||
          json.message ||
          'Data tidak ditemukan.'
        ).slice(0, 200),
      }
    } catch (e) {
      lastError = /abort|timeout/i.test(e?.message || '')
        ? 'Server timeout (>110 dtk). Coba lagi.'
        : `Gagal terhubung: ${String(e?.message || e).slice(0, 120)}`

      if (attempt < MAX_RETRIES) {
        await sleep(2000 * attempt)
      }
    }
  }

  return {
    ok: false,
    error: lastError,
  }
}

export default { amPresetFind }
```

---

## Cara Penggunaan

### Penggunaan Dasar

```js
import { amPresetFind } from './am-preset-find.js'

const result = await amPresetFind(
  '[https://www.tiktok.com/@username/video/1234567890123456789](https://www.tiktok.com/@username/video/1234567890123456789)'
)

if (!result.ok) {
  console.error('Gagal:', result.error)
} else {
  console.log('Video:', result.video)
  console.log('User:', result.user)
  console.log('Preset:', result.preset)
  console.log('Thumbnail:', result.thumbnail)
  console.log('Komentar dipindai:', result.totalComments)
}
```

### Menggunakan URL Pendek TikTok

```js
import { amPresetFind } from './am-preset-find.js'

const result = await amPresetFind(
  '[https://vt.tiktok.com/ZSxxxxxxxx/](https://vt.tiktok.com/ZSxxxxxxxx/)'
)

console.log(result)
```

### Menampilkan Link Preset

Struktur `preset` dapat berbeda tergantung respons endpoint. Oleh karena itu, lakukan pengecekan tipe data sebelum menampilkan hasil.

```js
import { amPresetFind } from './am-preset-find.js'

const result = await amPresetFind(
  '[https://www.tiktok.com/@username/video/1234567890123456789](https://www.tiktok.com/@username/video/1234567890123456789)'
)

if (!result.ok) {
  console.error(result.error)
  process.exit(1)
}

console.log('Informasi video:')
console.log(result.video)

console.log('
Informasi user:')
console.log(result.user)

console.log('
Jumlah komentar yang dipindai:')
console.log(result.totalComments)

console.log('
Preset ditemukan:')
console.dir(result.preset, { depth: null })
```

---

## Bentuk Respons

Fungsi `amPresetFind()` selalu mengembalikan object dengan dua kemungkinan hasil: sukses atau gagal.

### Respons Sukses

```js
{
  ok: true,
  video: {},
  user: {},
  preset: null,
  thumbnail: null,
  totalComments: 0
}
```

Contoh ilustratif respons sukses:

```js
{
  ok: true,
  video: {
    id: '1234567890123456789',
    title: 'Tutorial edit Alight Motion',
    url: '[https://www.tiktok.com/@username/video/1234567890123456789](https://www.tiktok.com/@username/video/1234567890123456789)'
  },
  user: {
    username: 'username',
    nickname: 'Nama Kreator'
  },
  preset: [
    {
      label: 'Preset AM XML',
      url: '[https://alight.link/xxxxxxxx](https://alight.link/xxxxxxxx)'
    }
  ],
  thumbnail: '[https://example.com/thumbnail.jpg](https://example.com/thumbnail.jpg)',
  totalComments: 250
}
```

> Struktur detail properti `video`, `user`, dan `preset` mengikuti data yang dikembalikan oleh endpoint. Jangan mengasumsikan setiap field selalu tersedia.

### Respons Gagal

```js
{
  ok: false,
  error: 'Pesan error dari server atau jaringan.'
}
```

Contoh:

```js
{
  ok: false,
  error: 'Link harus dari TikTok.'
}
```

Contoh lain:

```js
{
  ok: false,
  error: 'Server timeout (>110 dtk). Coba lagi.'
}
```

---

## Parameter

### `amPresetFind(tiktokUrl)`

Mencari preset Alight Motion dari komentar sebuah video TikTok.

| Parameter | Tipe | Wajib | Keterangan |
|---|---:|---:|---|
| `tiktokUrl` | `string` | Ya | URL video TikTok yang ingin dipindai |

Contoh URL yang dapat digunakan:

```text
[https://www.tiktok.com/@username/video/1234567890123456789](https://www.tiktok.com/@username/video/1234567890123456789)
[https://vt.tiktok.com/ZSxxxxxxxx/](https://vt.tiktok.com/ZSxxxxxxxx/)
[https://vm.tiktok.com/ZSxxxxxxxx/](https://vm.tiktok.com/ZSxxxxxxxx/)
```

---

## Output

| Properti | Tipe | Keterangan |
|---|---|---|
| `ok` | `boolean` | Status keberhasilan proses |
| `video` | `object` | Informasi video TikTok dari API |
| `user` | `object` | Informasi pengguna atau kreator video |
| `preset` | `array | object | null` | Daftar atau data preset Alight Motion yang ditemukan |
| `thumbnail` | `string | null` | URL thumbnail video apabila tersedia |
| `totalComments` | `number | null` | Jumlah komentar yang dipindai oleh server |
| `error` | `string` | Pesan error apabila `ok` bernilai `false` |

---

## Alur Kerja

1. Fungsi menerima URL TikTok dari pengguna.
2. URL dibersihkan menggunakan `trim()`.
3. Fungsi memeriksa apakah URL diawali `http://` atau `https://`.
4. Fungsi memeriksa apakah URL berasal dari domain TikTok.
5. URL dikodekan menggunakan `encodeURIComponent()`.
6. Request `GET` dikirim ke endpoint:

```text
[https://bintangapi.my.id/api/amfind/?url=](https://bintangapi.my.id/api/amfind/?url=)<URL_TIKTOK_TERENCODE>
```

7. Request menggunakan header berikut:

```http
Accept: application/json
Referer: [https://starlabs.biz.id/ampresetfind/](https://starlabs.biz.id/ampresetfind/)
User-Agent: Mozilla/5.0 ...
```

8. Apabila server mengembalikan `success: true`, data video, user, preset, thumbnail, dan total komentar dikembalikan ke pemanggil fungsi.
9. Apabila server mengembalikan `success: false`, error dari server langsung dikembalikan tanpa retry tambahan.
10. Apabila terjadi timeout atau gangguan jaringan, fungsi melakukan retry hingga maksimum 2 percobaan.
11. Jika semua percobaan gagal, fungsi mengembalikan error terakhir.

---

## Konfigurasi

```js
const API = '[https://bintangapi.my.id/api/amfind/?url=](https://bintangapi.my.id/api/amfind/?url=)'
```

Endpoint utama untuk melakukan pencarian preset.

```js
const UA = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36'
```

User-Agent yang dikirim bersama request. Header ini berguna untuk menyerupai request browser biasa jika server menerapkan validasi header dasar.

```js
const REFERER = '[https://starlabs.biz.id/ampresetfind/](https://starlabs.biz.id/ampresetfind/)'
```

Referer yang dikirim ke endpoint. Endpoint kemungkinan mengharapkan request berasal dari halaman Starlabs AM Preset Finder.

```js
const TIMEOUT_MS = 110_000
```

Batas maksimum waktu tunggu setiap request, yaitu 110.000 milidetik atau 110 detik.

```js
const MAX_RETRIES = 2
```

Jumlah maksimum percobaan request ketika terjadi kegagalan jaringan, koneksi, atau timeout.

---

## Penanganan Error

| Kondisi | Pesan Error |
|---|---|
| Input kosong atau bukan URL | `Link harus diawali http:// atau https://` |
| URL bukan dari TikTok | `Link harus dari TikTok.` |
| Respons server bukan JSON | `Server balas bukan JSON (HTTP <status>).` |
| Request melebihi timeout | `Server timeout (>110 dtk). Coba lagi.` |
| Koneksi gagal | `Gagal terhubung: <detail error>` |
| API mengembalikan kegagalan | Pesan dari properti `error` atau `message` respons API |
| Data tidak tersedia | `Data tidak ditemukan.` |

### Mengapa `success: false` Tidak Di-retry?

Apabila API merespons seperti berikut:

```json
{
  "success": false,
  "error": "Data tidak ditemukan."
}
```

Maka request dianggap sudah berhasil sampai ke server. Masalahnya bukan koneksi sementara, tetapi data tidak tersedia, video tidak dapat diproses, komentar tidak ditemukan, atau terdapat pembatasan dari sumber data.

Karena itu, melakukan retry berulang kali biasanya tidak menghasilkan data baru dan hanya menambah beban server.

---

## Catatan Penting

- Endpoint ini merupakan layanan pihak ketiga dan dapat berubah, dibatasi, dipindahkan, atau tidak tersedia tanpa pemberitahuan.
- Hasil preset sepenuhnya bergantung pada komentar yang dapat dibaca oleh server.
- Tidak semua video TikTok memiliki komentar yang memuat link preset Alight Motion.
- Video privat, dihapus, dibatasi wilayah, dibatasi usia, atau memiliki komentar nonaktif mungkin tidak dapat diproses.
- URL pendek TikTok seperti `vt.tiktok.com` dan `vm.tiktok.com` dapat bergantung pada kemampuan server API untuk melakukan redirect atau resolusi URL.
- Jangan mengirim request dalam jumlah besar atau agresif. Terapkan rate limit di aplikasi Anda.
- Jangan menyimpan, mendistribusikan, atau menggunakan konten pihak lain dengan cara yang melanggar hak cipta, ketentuan TikTok, Alight Motion, atau layanan terkait.
- Gunakan tool ini hanya untuk tujuan yang sah, wajar, dan sesuai dengan ketentuan layanan platform.

---

## Contoh Integrasi Express.js

```js
import express from 'express'
import { amPresetFind } from './am-preset-find.js'

const app = express()

app.get('/api/am-preset', async (req, res) => {
  const tiktokUrl = req.query.url

  const result = await amPresetFind(tiktokUrl)

  if (!result.ok) {
    return res.status(400).json(result)
  }

  return res.json(result)
})

app.listen(3000, () => {
  console.log('Server berjalan di http://localhost:3000')
})
```

Contoh request:

```text
GET /api/am-preset?url=https%3A%2F%2Fwww.tiktok.com%2F%40username%2Fvideo%2F1234567890123456789
```

---

## Contoh Integrasi Bot

```js
import { amPresetFind } from './am-preset-find.js'

async function handleAmPresetCommand(message, args) {
  const url = args

  if (!url) {
    return message.reply(
      'Masukkan URL video TikTok.

Contoh:
.am [https://vt.tiktok.com/ZSxxxxxxxx/](https://vt.tiktok.com/ZSxxxxxxxx/)'
    )
  }

  const result = await amPresetFind(url)

  if (!result.ok) {
    return message.reply(`Gagal mencari preset.

Alasan: ${result.error}`)
  }

  let text = 'Preset Alight Motion berhasil dicari.

'

  if (result.video?.title) {
    text += `Video: ${result.video.title}
`
  }

  if (result.user?.username) {
    text += `User: @${result.user.username}
`
  }

  if (result.totalComments !== null) {
    text += `Komentar dipindai: ${result.totalComments}
`
  }

  text += '
Preset:
'

  if (!result.preset) {
    text += 'Tidak ada preset yang ditemukan.'
  } else {
    text += JSON.stringify(result.preset, null, 2)
  }

  return message.reply(text)
}
```

---

## Struktur Proyek

```text
project-root/
├── am-preset-find.js
├── index.js
├── package.json
└── README.md
```

Contoh `package.json` untuk proyek ES Module:

```json
{
  "name": "starlabs-am-preset-finder",
  "version": "1.0.0",
  "type": "module",
  "description": "Scraper pencari preset Alight Motion dari komentar TikTok",
  "main": "am-preset-find.js",
  "scripts": {
    "start": "node index.js"
  }
}
```

---

## Lisensi dan Penafian

Kode ini disediakan untuk kebutuhan edukasi, eksperimen, dan integrasi pribadi. Creator GitHub tidak menjamin endpoint pihak ketiga akan selalu aktif, stabil, akurat, atau kompatibel di masa mendatang.

Pengguna bertanggung jawab penuh atas penggunaan kode ini, termasuk kepatuhan terhadap:

- Ketentuan layanan TikTok.
- Ketentuan layanan Alight Motion.
- Kebijakan endpoint atau situs pihak ketiga.
- Hak cipta dan hak kekayaan intelektual.
- Peraturan yang berlaku di wilayah masing-masing.

Jangan gunakan tool ini untuk spam, pengambilan data masif, pelanggaran privasi, atau aktivitas yang dapat merugikan pihak lain.

---

## Kredit

| Peran | Nama / Link |
|---|---|
| Creator GitHub | SCRIPETEREN |
| API Endpoint | `https://bintangapi.my.id/api/amfind/?url=` |