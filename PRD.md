# PRD — [Nama Project]

> Template PRD untuk vibe coding dengan AI agent.
> Isi setiap section sesuai project. Hapus section yang tidak relevan.
> File ini adalah **sumber kebenaran utama** — semua dokumen lain (ARCHITECTURE, DESIGN_SYSTEM, dll) harus selaras dengan isi PRD ini.
> Agent merujuk fitur lewat **ID fitur** (`F-01`) dan layar lewat **ID layar** (`S-01`). Jangan ubah ID yang sudah dipakai; tambah ID baru di akhir.

---

## 1. Overview

<!-- Deskripsi singkat app: apa ini, masalah apa yang dipecahkan, kenapa dibuat.
     2-5 kalimat cukup. Contoh: "Aplikasi kasir untuk UMKM F&B yang bisa
     mencatat transaksi offline dan sync saat online." -->

_(isi di sini)_

## 2. Target Users

<!-- Siapa pengguna utama? Karakteristik mereka?
     Contoh: "Pemilik warung kopi, 30-50 tahun, tidak tech-savvy,
     pakai Android low-end." -->

_(isi di sini)_

## 3. Platform & Identitas App

<!-- Centang yang berlaku, hapus yang tidak. -->

- [ ] Android (min SDK: <!-- isi, mis. 24 -->)
- [ ] iOS (min version: <!-- isi, mis. 13.0 -->)

**Form factor & orientasi** — default: **phone, portrait saja**. Centang kalau perlu lebih:

- [ ] Dukung tablet (layout `medium`/`expanded` di `DESIGN_SYSTEM.md` jadi wajib)
- [ ] Dukung landscape

**Identitas per flavor:**

| | staging | prod |
|---|---|---|
| Nama app | <!-- mis. [STG] Kasirku --> | <!-- mis. Kasirku --> |
| Android applicationId | <!-- mis. com.company.kasirku.staging --> | <!-- mis. com.company.kasirku --> |
| iOS bundle id | <!-- mis. com.company.kasirku.staging --> | <!-- mis. com.company.kasirku --> |

## 4. Fitur

<!-- Daftar fitur dengan prioritas dan acceptance criteria.
     P0 = MVP wajib, P1 = rilis berikutnya, P2 = nice to have.
     Acceptance criteria = kondisi yang bisa dicek. Agent memakainya
     sebagai dasar test dan Definition of Done. -->

### P0 — MVP

#### F-01 — <!-- nama fitur, mis. Login dengan email -->

- **Deskripsi**: <!-- 1-2 kalimat -->
- **Layar terkait**: <!-- mis. S-02 -->
- **Acceptance criteria**:
  - [ ] <!-- mis. Email tidak valid → pesan error di field, request tidak dikirim -->
  - [ ] <!-- mis. Login sukses → token tersimpan, diarahkan ke S-03 -->
  - [ ] <!-- mis. Tanpa koneksi → pesan error jaringan + tombol retry -->

#### F-02 — _(isi di sini)_

### P1 — Rilis Berikutnya

- **F-xx** — _(nama + ringkasan; acceptance criteria ditulis saat masuk pengerjaan)_

### P2 — Nice to Have

- **F-xx** — _(isi di sini)_

## 5. Layar & Navigasi

<!-- Daftar semua layar. Route path jadi acuan di core/router.
     Auth "ya" = hanya bisa diakses setelah login. -->

| ID | Layar | Route | Auth | Fitur |
|---|---|---|---|---|
| S-01 | <!-- Splash --> | `/` | tidak | — |
| S-02 | <!-- Login --> | `/login` | tidak | F-01 |
| S-03 | <!-- Home --> | `/home` | ya | <!-- --> |

**Navigasi utama**: <!-- mis. bottom navigation 3 tab: Home, Riwayat, Profil -->

**Deep link** (kalau ada): <!-- mis. https://app.example.com/order/:id → S-05 -->

## 6. User Flow

<!-- Alur utama pengguna, ditulis dengan ID layar.
     Alur bertanda [kritikal] wajib punya integration test. -->

1. [kritikal] _(mis. Buka app → S-01 → belum login → S-02 → login sukses → S-03)_

## 7. Model Data Inti

<!-- Entitas bisnis utama dan field-nya — dasar entity di domain layer.
     Kalau belum pasti tulis "TBD". Agent dilarang mengarang field. -->

| Entity | Field utama | Catatan |
|---|---|---|
| <!-- User --> | <!-- id, name, email --> | <!-- --> |

## 8. API / Backend

<!-- Base URL TIDAK ditulis di kode; masuk ke .env per flavor. -->

- Backend: <!-- REST API / GraphQL / Firebase / Supabase / lainnya -->
- Base URL: staging `<!-- isi -->`, prod `<!-- isi -->`
- Auth method: <!-- JWT access + refresh (default, lihat SECURITY.md) / OAuth2 / lainnya -->
- Format response error: <!-- contoh body, mis. { "code": "...", "message": "..." } -->
- Dokumentasi API: <!-- link Swagger / Postman -->
- Endpoint utama:

  | Method | Endpoint | Deskripsi | Fitur |
  | ------ | -------- | --------- | ----- |
  |        |          |           |       |

## 9. Offline & Sinkronisasi

<!-- Pilih satu. -->

- [ ] **Online only** — tanpa koneksi tampilkan error + retry
- [ ] **Cache read** — data terakhir tampil saat offline, aksi tulis butuh koneksi
- [ ] **Offline-first** — aksi tulis disimpan lokal (drift) lalu disinkronkan

Aturan sync/konflik: _(isi kalau offline-first)_

## 10. Permission & Integrasi Native

<!-- Hanya permission/SDK yang tercantum di sini yang boleh ditambahkan agent. -->

| Permission / SDK | Fitur | Alasan (teks yang ditampilkan ke user) |
|---|---|---|
| <!-- Kamera --> | <!-- F-04 --> | <!-- Untuk memindai barcode produk --> |

Push notification: <!-- ya / tidak, provider -->

## 11. Bahasa & Lokalisasi

- Bahasa default: **Indonesia (`id`)**
- Bahasa tambahan: **Inggris (`en`)**
- Format angka/mata uang/tanggal: <!-- mis. Rupiah "Rp 10.000", dd MMM yyyy -->

## 12. Analytics & Monitoring

<!-- Firebase diputuskan per project. Selama belum diputuskan, agent tidak boleh menambah SDK Firebase. -->

- Crash reporting: <!-- Firebase Crashlytics / Sentry / belum diputuskan -->
- Analytics: <!-- Firebase Analytics / tidak ada / belum diputuskan -->
- Event penting: <!-- mis. login_success, checkout_completed -->

## 13. Desain

- Figma: <!-- link -->
- Referensi app sejenis: <!-- link/nama app -->
- Brand: <!-- warna utama, font, logo — token lengkap di DESIGN_SYSTEM.md -->
- Ikon app & splash: <!-- link aset; versi staging diberi penanda (badge/warna) -->

## 14. Kebutuhan Non-Fungsional

- Performa: <!-- mis. cold start < 3 detik di device low-end -->
- Ukuran app: <!-- mis. APK < 30MB -->
- Keamanan khusus: <!-- cert pinning: ya/tidak; root detection: blokir / peringatan / tidak; screen protection: layar mana -->
- Device minimum: <!-- mis. RAM 2GB -->

## 15. Rilis

- Distribusi: <!-- Play Store / App Store / Firebase App Distribution / TestFlight -->
- Versioning: `MAJOR.MINOR.PATCH+BUILD` di `pubspec.yaml`
- Target rilis MVP: <!-- tanggal -->

## 16. Out of Scope

<!-- Yang JELAS TIDAK dikerjakan. Penting untuk mencegah scope creep saat vibe coding. -->

- _(isi di sini)_

## 17. Pertanyaan Terbuka

<!-- Hal yang belum diputuskan. Kalau tugas menyentuh item di sini, agent wajib bertanya — bukan menebak. -->

- _(isi di sini)_

---

_Terakhir diupdate: <!-- tanggal -->_
