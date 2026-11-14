# AGENTS.md

Panduan untuk AI agent yang bekerja di repo ini. Baca file ini dulu, lalu baca dokumen rujukan sesuai tugas.

## 1. Tentang Proyek

Flutter enterprise starter kit. Tujuannya jadi fondasi app production-ready (arsitektur rapi, security, testing, design system) sekaligus portfolio demo.

- **Flutter / Dart**: `3.47.9` (Dart `3.13.7`), wajib via **FVM**, dikunci di `.fvmrc`
- **State management**: `flutter_bloc` (Bloc murni, tanpa Cubit)
- **Routing**: `go_router`
- **DI**: `get_it` + `injectable` (codegen via `build_runner`)
- **Networking**: `dio`
- **Local storage**:
  - `drift` (SQLite, type-safe) untuk data terstruktur/relasional
  - `shared_preferences` untuk setting/flag kecil non-sensitif
  - `flutter_secure_storage` untuk token dan data sensitif

> Jangan pernah memanggil `flutter`/`dart` global langsung. Selalu lewat `fvm flutter` / `fvm dart`, atau lewat Makefile kalau tersedia.

## 2. Dokumen Rujukan

Jangan duplikasi isi dokumen ini. Baca sesuai kebutuhan:

| File | Baca kalau... |
|---|---|
| `PRD.md` | Perlu tahu fitur apa yang dibangun, scope, dan prioritas |
| `ARCHITECTURE.md` | Mau menambah/mengubah struktur, layer, atau dependency |
| `DESIGN_SYSTEM.md` | Mengerjakan UI: warna, tipografi, spacing, komponen |
| `CODE_STYLE.md` | Menulis atau me-review kode (penamaan, format, pola) |
| `TESTING.md` | Menulis test atau menentukan apa yang wajib dites |
| `SECURITY.md` | Menyentuh auth, storage, networking, secret, atau data sensitif |
| `README.md` | Setup lingkungan dan cara menjalankan proyek |

Kalau dokumen bertentangan dengan AGENTS.md, **dokumen spesifik yang menang**, lalu laporkan konfliknya.

## 3. Perintah Penting

Gunakan Makefile kalau tersedia (`make <target>`). Kalau tidak, ini perintah aslinya:

```bash
fvm flutter pub get                                            # install dependency
fvm dart run build_runner build --delete-conflicting-outputs   # code generation (injectable, drift)
fvm dart format .                                              # format
fvm flutter analyze                                            # static analysis
fvm flutter test                                               # unit + widget test
fvm flutter test --coverage                                    # dengan coverage
fvm flutter run --flavor staging -t lib/main_staging.dart      # run staging
fvm flutter run --flavor prod -t lib/main_prod.dart            # run production
```

Flavor yang tersedia: `staging` dan `prod`. Development sehari-hari pakai `staging`.

Jalankan `fvm flutter analyze` dan `fvm flutter test` sebelum menyatakan tugas selesai.

## 4. Struktur Folder

Ringkasan saja. Detail lengkap ada di `ARCHITECTURE.md`.

```
lib/
├── core/          # config, theme, network, error, utils (tanpa logic fitur)
├── shared/        # widget dan model yang dipakai lintas fitur
├── features/
│   └── <feature>/
│       ├── data/          # datasource, DTO, repository impl
│       ├── domain/        # entity, repository interface, usecase
│       └── presentation/  # page, widget, bloc (event, state)
├── main_staging.dart
└── main_prod.dart
test/              # mirror struktur lib/
```

Aturan menaruh file baru:
- Kode milik satu fitur → di dalam `features/<feature>/`.
- Dipakai 2+ fitur → pindah ke `shared/` atau `core/`.
- Dependency searah: `presentation → domain ← data`. `domain` tidak boleh import Flutter atau package data.

## 5. Aturan Kode (Ringkas)

Detail di `CODE_STYLE.md`. Yang paling sering dilanggar:

- Pakai `const` constructor kapan pun bisa.
- Pecah widget besar. Hindari `build()` panjang dan nested dalam.
- **Tidak ada hardcode**: string (pakai l10n), warna/ukuran/font (pakai theme dan design tokens dari `DESIGN_SYSTEM.md`), URL/key (pakai config/env).
- Jangan taruh logic bisnis di widget. Taruh di bloc atau usecase.
- Handle error secara eksplisit (pakai pola error di `core/error`). Jangan telan exception diam-diam.
- Hindari `dynamic` dan `!` (force unwrap) kecuali benar-benar aman dan diberi komentar.
- Semua logging lewat `AppLogger` dari `core/logger` — dilarang `print`.
- Komentar kode ditulis dalam **Bahasa Inggris**.

## 6. Aturan "Jangan"

- Jangan menambah package baru tanpa konfirmasi. Sebutkan alasan dan alternatifnya.
- Jangan edit file generated (`*.g.dart`, `*.config.dart`, file drift). Ubah sumbernya lalu regenerate.
- Jangan mengubah arsitektur atau struktur folder di luar scope tugas.
- Jangan refactor, rename, atau hapus kode yang tidak terkait tugas.
- Jangan commit secret, API key, keystore, atau file `.env`. Lihat `SECURITY.md`.
- Jangan mematikan lint rule atau memakai `// ignore:` hanya supaya warning hilang.
- Jangan menghapus atau melemahkan test supaya build hijau.
- Jangan mengarang API, field, atau perilaku backend. Kalau tidak jelas, tanya.

## 7. Alur Kerja

1. **Pahami dulu.** Baca dokumen relevan dan kode sekitar sebelum menulis.
2. **Rencanakan singkat.** Sebutkan file yang akan diubah dan pendekatannya.
3. **Kerjakan bertahap.** Satu langkah kecil per giliran, jelaskan perubahan, lalu **tunggu konfirmasi sebelum lanjut**.
4. **Verifikasi.** Jalankan analyze dan test.
5. **Laporkan.** Ringkas apa yang berubah, apa yang belum, dan asumsi yang dipakai.

Kalau requirement ambigu atau ada beberapa pendekatan yang masuk akal, tanya satu pertanyaan yang spesifik daripada menebak.

## 8. Aturan Git

**Branch** — format: `<tipe>/<deskripsi-singkat-kebab-case>`

| Tipe | Untuk |
|---|---|
| `feature/` | Fitur baru, misal `feature/login-email` |
| `fix/` | Perbaikan bug, misal `fix/token-refresh-loop` |
| `chore/` | Maintenance: dependency, config, tooling |
| `refactor/` | Ubah struktur tanpa ubah perilaku |
| `docs/` | Dokumentasi saja |
| `hotfix/` | Fix mendesak dari production |

Branch utama: `main` (stabil, production-ready) dan `development` (integrasi). Semua pekerjaan dilakukan di branch tipe di atas, merge ke `development` dulu.

**Commit** — pakai Conventional Commits, Bahasa Inggris:

```
<type>(<scope>): <deskripsi singkat>

feat(auth): add email login usecase
fix(cart): prevent duplicate item on rapid tap
chore(deps): bump dio to 5.x
test(auth): add login bloc tests
docs: update architecture diagram
```

Type: `feat`, `fix`, `chore`, `refactor`, `test`, `docs`, `style`, `perf`. Scope opsional tapi disarankan. Satu commit = satu perubahan logis.

## 9. Definition of Done

Tugas dianggap selesai kalau:

- [ ] Sesuai `PRD.md` dan `ARCHITECTURE.md`
- [ ] `fvm dart format .` sudah dijalankan
- [ ] `fvm flutter analyze` bersih (tanpa warning baru)
- [ ] `fvm flutter test` lolos, dan ada test baru untuk logic yang ditambah
- [ ] Codegen sudah di-regenerate kalau ada perubahan DI/database (`build_runner build`)
- [ ] Tidak ada hardcoded string/warna/secret
- [ ] Tidak ada file di luar scope yang berubah
- [ ] Untuk UI: sesuai `DESIGN_SYSTEM.md` dan aman di layar kecil/besar

## 10. Testing (Ringkas)

Detail di `TESTING.md`.

- Struktur `test/` mirror `lib/`.
- Wajib dites: usecase, repository, bloc, dan util.
- Widget test untuk komponen shared dan alur UI penting.
- Mock dependency eksternal (network, storage). Jangan memanggil API sungguhan di test.

## 11. Keamanan (Ringkas)

Detail di `SECURITY.md`. Minimal:

- Token dan data sensitif hanya di `flutter_secure_storage`.
- Jangan log data sensitif (token, PII, password).
- Semua input dari luar (API, deep link, form) divalidasi.
- Komunikasi hanya lewat HTTPS.

## 12. Memperbarui Dokumen

Kalau perubahan kamu mengubah arsitektur, konvensi, atau perintah, update dokumen terkait di commit yang sama dan sebutkan di laporan akhir.
