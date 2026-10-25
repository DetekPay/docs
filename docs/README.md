# Dokumentasi API — DetekPay

Panduan integrasi API, QRIS, dan webhook DetekPay.

> Catatan: ini dokumentasi ringkas. Dokumentasi lengkap — panduan setup tiap merchant, semua parameter API, dan contoh payload penuh — tersedia di dashboard setelah Anda login.

- [Cara Kerja](#cara-kerja)
- [Cara Integrasi](#cara-integrasi)
- [Mulai Integrasi](#mulai-integrasi)
- [Buat Invoice](#buat-invoice)
- [Tampilkan QR ke Customer](#tampilkan-qr-ke-customer)
- [Cek Status](#cek-status)
- [Verifikasi Webhook](#verifikasi-webhook)
- [Test Webhook](#test-webhook)
- [Recovery Otomatis](#recovery-otomatis)
- [Monitoring Kesehatan](#monitoring-kesehatan)
- [Penyimpanan Data](#penyimpanan-data)
- [Bot di Panel (Pterodactyl)](#bot-di-panel-pterodactyl)
- [Referensi API](#referensi-api)

## Cara Kerja

1. **Kode unik otomatis.** Setiap invoice ditambah kode unik Rp1–Rp99 ke nominal Anda. Contoh: Rp10.000 jadi Rp10.042. Kode ini yang dipakai untuk mencocokkan pembayaran masuk dengan invoice yang benar.
2. **Deteksi tiap 30 detik.** Server kami memantau transaksi masuk di akun merchant Anda. Kalau ada pembayaran dengan nominal yang cocok, invoice otomatis jadi `paid`.
3. **Non-custodial.** Dana masuk langsung ke rekening merchant Anda — tidak lewat DetekPay, tidak ada potongan, tidak ada delay settlement.
4. **Status invoice:** `pending` (menunggu bayar) → `paid` (terdeteksi) / `expired` (lewat batas waktu).

## Cara Integrasi

1. **Render QR dari field `qris`.** Response invoice berisi string QRIS dinamis — tampilkan sebagai gambar QR di halaman checkout atau chat bot Anda, pakai library QR apa pun. Cocok untuk website, bot Telegram, atau bot WhatsApp.
2. **Pantau statusnya.** Pembeli membayar QR itu seperti QRIS biasa. Anda menerima webhook saat pembayaran terdeteksi, atau polling `GET /api/payments/:id/status` sampai status `paid`.

## Mulai Integrasi

1. **Daftar & hubungkan merchant** di dashboard DetekPay.
2. **Buat API Key** di halaman merchant (Shopee Partner / GoPay Merchant) — maks 3 key per merchant. Key pertama otomatis dibuat saat merchant terhubung; tambah manual lewat tombol + Buat API Key. Key diawali `dp_...`
3. **Buat invoice** via `POST /api/payments`.
4. **Tampilkan QR** dari field `qris` ke customer.
5. **Terima webhook** saat invoice menjadi `paid`. Selalu verifikasi signature!

## Buat Invoice

cURL:

```bash
curl -X POST https://detekpay.com/api/payments \
  -H "X-API-Key: dp_xxxx" \
  -H "Content-Type: application/json" \
  -d '{"amount":10000,"webhook_url":"https://anda.com/hook"}'
```

JavaScript:

```javascript
const res = await fetch('https://detekpay.com/api/payments', {
  method: 'POST',
  headers: {'X-API-Key': 'dp_xxxx', 'Content-Type': 'application/json'},
  body: JSON.stringify({amount: 10000, webhook_url: 'https://anda.com/hook'})
});
const inv = await res.json();
console.log(inv.qris);
```

PHP:

```php
<?php
$ch = curl_init('https://detekpay.com/api/payments');
curl_setopt_array($ch, [
  CURLOPT_POST => true,
  CURLOPT_HTTPHEADER => ['X-API-Key: dp_xxxx', 'Content-Type: application/json'],
  CURLOPT_POSTFIELDS => json_encode(['amount' => 10000, 'webhook_url' => 'https://anda.com/hook']),
  CURLOPT_RETURNTRANSFER => true
]);
$inv = json_decode(curl_exec($ch), true);
echo $inv['qris'];
```

Python:

```python
import requests
r = requests.post('https://detekpay.com/api/payments',
  headers={'X-API-Key': 'dp_xxxx'},
  json={'amount': 10000, 'webhook_url': 'https://anda.com/hook'})
inv = r.json()
print(inv['qris'])
```

**Response:** `invoice_id`, `amount` (unik), `qris`, `webhook_secret`, `expires_at`

**Parameter:** `amount` (wajib, Rp500 – Rp10.000.000), `webhook_url` (opsional). Rate limit: 30 request/menit.

## Tampilkan QR ke Customer

Tampilkan string `qris` dari response sebagai gambar QR — di checkout website Anda, atau sebagai gambar di chat bot. QRIS dinamis dibuat dari QRIS merchant Anda sendiri; tidak ada halaman perantara.

```
POST /api/payments → response.qris (string QRIS dinamis)
```

Pantau statusnya lewat webhook, atau polling endpoint publik berikut tiap 10–30 detik sampai `paid`:

```
GET /api/payments/:id/status
(tanpa API key — hanya field publik)
```

## Cek Status

```
GET /api/payments/:id
X-API-Key: dp_xxxx...
```

**Response:** `invoice_id`, `amount`, `status` (pending / paid / expired), `created_at`, `paid_at`

## Verifikasi Webhook

```
Header: X-DetekPay-Signature: sha256=...
HMAC-SHA256(body, webhook_secret)
```

Isi JSON yang dikirim saat pembayaran terdeteksi:

```json
{
  "event": "payment.confirmed",
  "invoice_id": "inv_xxxx",
  "client_id": "...",
  "amount": 10042,
  "shopee_tx_id": "...",
  "paid_at": "...",
  "issuer": "DANA"
}
```

`webhook_secret` dari response buat invoice. Selalu verifikasi signature sebelum memproses! Webhook yang gagal di-retry otomatis dengan backoff.

**Idempotency:** webhook dikirim *at-least-once* — satu invoice bisa diterima lebih dari sekali. Selalu dedup berdasarkan `invoice_id`.

Pembayaran yang masuk setelah invoice kedaluwarsa (maks 24 jam) tetap dikirim dengan tambahan flag `"is_late": true`. Webhook dari Test Invoice di dashboard membawa flag `"is_test": true`.

## Test Webhook

Uji handler webhook tanpa transaksi sungguhan. Payload ada flag `is_test: true`.

```
POST /api/webhook/test
X-API-Key: dp_xxxx...
```

**Body:** `webhook_url` (opsional), `scenario`: `valid` | `invalid_signature` | `duplicate`.

Response berisi `webhook_secret`, `payload` yang dikirim, `signature_sent`, dan `results`.

## Recovery Otomatis

Jika server sempat mati, poller otomatis mengambil ulang mutasi yang terlewat (maksimal 24 jam) saat kembali aktif. Pencocokan tetap idempoten.

## Monitoring Kesehatan

Di dashboard, setiap koneksi merchant menampilkan status poll terakhir (mis. "Poll terakhir: 12 detik yang lalu") dengan indikator hijau/merah. Kalau poll gagal, pesan error-nya ditampilkan langsung.

Transaksi yang masuk tapi tidak cocok dengan invoice mana pun tercatat di halaman Ringkasan tab **Rekonsiliasi** — bisa ditandai sudah dicek manual.

Riwayat transaksi asli merchant Anda (di luar invoice DetekPay) bisa dilihat di halaman Ringkasan tab **Riwayat** — lengkap dengan aplikasi pembayarnya (DANA, GoPay, m-banking, dll) dan nomor referensinya.

## Penyimpanan Data

Invoice **paid** disimpan permanen. Invoice **pending/expired** dihapus otomatis setelah 90 hari. QRIS dinamis dihapus setelah invoice tidak lagi pending. Log webhook disimpan 30 hari.

## Bot di Panel (Pterodactyl)

Bot/aplikasi Anda berjalan di panel sewaan? Tidak masalah — deteksi berjalan di server DetekPay, bot Anda cukup memanggil API. Tidak ada yang perlu diinstal dari DetekPay.

1. **Buat invoice seperti biasa** via `POST /api/payments`. Tampilkan field `qris` sebagai gambar QR di chat bot Anda.
2. **Webhook — khusus alamat HTTPS.** Webhook hanya dikirim ke URL `https://` publik, jadi alamat alokasi panel (`http://IP:port`) tidak bisa menerimanya langsung. Kalau bot Anda bisa dijangkau lewat domain/tunnel HTTPS, daftarkan URL itu sebagai webhook (tiap kiriman ditandatangani HMAC-SHA256, wajib diverifikasi; yang gagal di-retry otomatis sampai 25x). Kalau tidak, pakai polling di langkah 3 — untuk bot panel itu cara yang paling praktis.
3. **Atau polling status** — tanpa port masuk sama sekali. Cek `GET /api/payments/:id/status` tiap 10–30 detik sampai status `paid`. Tidak ada setting khusus untuk cara ini.

Deteksi tetap berjalan 24/7 di server DetekPay walau container panel Anda restart. Jika penyedia panel memindahkan server Anda (IP node berubah), perbarui URL webhook di dashboard.

## Referensi API

**Autentikasi.** Semua endpoint (kecuali `/status` publik) memakai header `X-API-Key: dp_xxxx...`.

**Endpoint:**

```
POST /api/payments              → buat invoice
GET  /api/payments/:id          → cek status (butuh API key)
GET  /api/payments/:id/status  → status publik (tanpa API key)
POST /api/webhook/test         → test webhook
```

**Error:** `400` parameter salah · `401` API key salah · `403` lisensi expired · `404` invoice tidak ditemukan · `429` rate limit

---

DetekPay — QRIS Automation · [Syarat & Ketentuan](https://detekpay.com/syarat-ketentuan)
