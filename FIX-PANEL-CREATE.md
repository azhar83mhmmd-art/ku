# Kairoo Panel Manager — Panel Create Fix

Perbaikan utama pada versi ini:

- Create panel tidak lagi memakai `readAll + writeAll` untuk tabel `panels` pada alur kritis. Setiap panel diinsert/update satu row langsung ke Supabase.
- Log aktivitas memakai insert langsung sehingga request Vercel yang bersamaan tidak saling menimpa snapshot database.
- Pencarian Nest/Egg/Location Pterodactyl dipercepat dengan request paralel dan cache 5 menit.
- Egg terpilih diambil ulang dengan `include=variables` agar environment Egg tidak kosong pada panel Pterodactyl tertentu.
- Timeout Pterodactyl dibuat jelas dan dapat diatur lewat `PTERODACTYL_TIMEOUT_MS`.
- ID user Pterodactyl disimpan segera setelah user berhasil dibuat.
- Timeout setelah create server tidak otomatis di-retry untuk mencegah server ganda.
- Ditambahkan endpoint rekonsiliasi untuk menemukan server yang sebenarnya sudah dibuat tetapi response-nya hilang/timeout.
- Frontend mencoba rekonsiliasi otomatis saat koneksi Vercel terputus atau backend mengembalikan status `processing`.
- Vercel Function diberi `maxDuration: 60`.
- Error Pterodactyl 401/403/409/422 dan network timeout diterjemahkan menjadi pesan yang lebih jelas.

## Deploy

1. Upload/deploy project ini ke Vercel.
2. Pastikan environment variables Supabase dan `PANEL_SECRET` sudah benar.
3. Pastikan `PTERODACTYL_URL`/pengaturan Domain dan PTLA benar.
4. Di halaman Admin → Pengaturan, gunakan tombol test koneksi.
5. Buat satu panel percobaan.
6. Jika request timeout, **jangan klik Buat Panel berulang kali**. Buka Riwayat Panel dan tunggu proses rekonsiliasi.

`PTLC` tidak dipakai untuk create server Application API. Simpan PTLC tetap aman untuk fitur panel/client API yang memang membutuhkannya.

## Update — perbaikan "Tidak dapat terhubung ke server" (Panel gagal dibuat)

Pesan "Tidak dapat terhubung ke server" di frontend HANYA muncul kalau `fetch()`
browser ke `/api/panels/create` gagal total (tidak ada response HTTP sama
sekali) — beda dengan error Pterodactyl (domain salah, PTLA ditolak, dsb) yang
sudah punya pesan spesifik sendiri di app ini. Penyebab paling umum: proses
`createUser` → `resolveResources` (nests/locations/eggs) → `createServer` ke
Pterodactyl berjalan berurutan dan totalnya mendekati/melewati `maxDuration`
Function di Vercel (60 detik), sehingga Vercel MEMATIKAN function di tengah
jalan — browser menerima connection reset tanpa response JSON apa pun.

Perbaikan di versi ini:

1. **Budget waktu internal** (`withBudget` di `server.js`) — setiap panggilan
   Pterodactyl sekarang dibatasi oleh sisa waktu function ini sendiri (default
   45 detik, bisa diubah lewat env `PANEL_CREATE_BUDGET_MS`), dengan margin 4
   detik supaya masih sempat menulis status ke Supabase dan mengirim response
   JSON sebelum Vercel mematikan function secara paksa. Hasilnya: user akan
   selalu menerima pesan error yang jelas, bukan silent connection failure.
2. **Frontend sekarang menampilkan detail error browser asli** (mis. "Failed
   to fetch" / "Load failed") di belakang pesan "Tidak dapat terhubung ke
   server", supaya kalau ini muncul lagi, penyebab sebenarnya langsung
   terlihat tanpa perlu buka DevTools.

### Cara memastikan penyebabnya benar-benar Pterodactyl (bukan Vercel)

Sebelum menyalahkan konfigurasi PTLA/PTLC/domain, cek dulu:

1. **Vercel → Project → Deployments → (deployment aktif) → Functions/Logs** —
   lihat log persis pada waktu percobaan create panel gagal. Kalau ada baris
   error dari `services/pterodactyl.js`, itu penyebab aslinya, dan pesannya
   jauh lebih spesifik dari "Tidak dapat terhubung ke server".
2. Buka `GET /api/admin/pterodactyl/diagnostic` (login sebagai admin_web) —
   endpoint ini sudah ada di project ini, langsung tes koneksi + resolve
   nest/egg/location tanpa membuat user/server sungguhan.
3. Kalau logs Vercel menunjukkan function benar-benar timeout/terbunuh
   (bukan error Pterodactyl), naikkan `PTERODACTYL_TIMEOUT_MS`/
   `PANEL_CREATE_BUDGET_MS` hanya kalau panel-nya memang lambat merespons,
   atau pertimbangkan upgrade plan Vercel (Hobby dibatasi 60 detik keras untuk
   Serverless Function biasa).
