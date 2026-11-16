# AGENTS.md

Panduan untuk AI agent yang bekerja di repo ini. Baca file ini dulu, lalu baca dokumen rujukan sesuai tugas.

## 1. Tentang Proyek

Flutter enterprise starter kit untuk app mobile (Android & iOS). Tujuannya jadi fondasi app production-ready (arsitektur rapi, security, testing, design system) sekaligus portfolio demo.

- **Flutter / Dart**: `3.47.9` (Dart `3.13.7`), wajib via **FVM**, dikunci di `.fvmrc`
- **State management**: `flutter_bloc` (Bloc murni, tanpa Cubit) + `bloc_concurrency` (event transformer)
- **Routing**: `go_router`
- **DI**: `get_it` + `injectable` (codegen via `build_runner`)
- **Networking**: `dio`
- **Error handling**: `Failure` (sealed) + `Result<T>` buatan sendiri di `core/error` — tanpa package FP
- **Local storage**:
  - `drift` (SQLite, type-safe) untuk data terstruktur/relasional
  - `shared_preferences` untuk setting/flag kecil non-sensitif
  - `flutter_secure_storage` untuk token dan data sensitif
- **Config**: `.env.<flavor>` dimuat lewat `--dart-define-from-file`, dibaca hanya oleh `AppConfig`
- **Localization**: `flutter_localizations` + `gen-l10n` — default `id`, tambahan `en`
- **Asset & native**: `flutter_gen_runner`, `flutter_launcher_icons`, `flutter_native_splash` (per flavor)
- **Firebase**: **belum diputuskan** — ditanyakan per project. Jangan tambahkan SDK Firebase tanpa persetujuan.

> Jangan pernah memanggil `flutter`/`dart` global langsung. Selalu lewat `fvm flutter` / `fvm dart`, atau lewat Makefile kalau tersedia.

## 2. Dokumen Rujukan

Jangan duplikasi isi dokumen ini. Baca sesuai kebutuhan:

| File | Baca kalau... |
|---|---|
| `PRD.md` | Perlu tahu fitur (`F-xx`), layar (`S-xx`), acceptance criteria, scope, dan prioritas |
| `ARCHITECTURE.md` | Mau menambah/mengubah struktur, layer, DI, routing, flavor, atau error handling |
| `DESIGN_SYSTEM.md` | Mengerjakan UI: warna, tipografi, spacing, komponen, responsive |
| `CODE_STYLE.md` | Menulis atau me-review kode (penamaan, lint, pola) |
| `TESTING.md` | Menulis test atau menentukan apa yang wajib dites |
| `SECURITY.md` | Menyentuh auth, storage, networking, secret, permission, atau data sensitif |
| `README.md` | Setup lingkungan dan daftar lengkap command (Make + raw) |

Kalau dokumen bertentangan dengan AGENTS.md, **dokumen spesifik yang menang**, lalu laporkan konfliknya.

## 3. Perintah Penting

Gunakan Makefile kalau tersedia (`make <target>`). Daftar lengkap target + raw command ada di `README.md`. Yang paling sering:

```bash
fvm flutter pub get                                   # install dependency
fvm flutter gen-l10n                                  # generate localization
fvm dart run build_runner build --delete-conflicting-outputs   # codegen (injectable, drift, assets)
fvm dart format .                                     # format
fvm flutter analyze                                   # static analysis
fvm flutter test                                      # unit + widget test
fvm flutter run --flavor staging -t lib/main_staging.dart --dart-define-from-file=.env.staging   # run staging
```

- Flavor yang tersedia: `staging` dan `prod`. Development sehari-hari pakai `staging`.
- **Setiap** `run`/`build`/integration test wajib menyertakan `--flavor <f>` dan `--dart-define-from-file=.env.<f>` yang sama. Tanpa env, `AppConfig` gagal saat startup.
- Jalankan `make check` (format + analyze + test) sebelum menyatakan tugas selesai.

## 4. Struktur Folder

Ringkasan saja. Detail lengkap ada di `ARCHITECTURE.md`.

```
lib/
├── core/          # config, design_system, di, error, l10n, logger, network, router, usecase, utils
├── shared/        # widget komposit dan model yang dipakai lintas fitur
├── features/
│   └── <feature>/
│       ├── data/          # datasource, DTO, repository impl
│       ├── domain/        # entity, repository interface, usecase
│       └── presentation/  # page, widget, bloc (event, state)
├── app.dart       # root widget (MaterialApp.router)
├── bootstrap.dart # init bersama: config, DI, logger, error handler
├── main_staging.dart
└── main_prod.dart
test/              # mirror struktur lib/
integration_test/  # alur end-to-end
```

Aturan menaruh file baru:
- Kode milik satu fitur → di dalam `features/<feature>/`.
- Dipakai 2+ fitur → pindah ke `shared/` atau `core/`.
- Dependency searah: `presentation → domain ← data`. `domain` hanya boleh import Dart murni, `equatable`, anotasi `injectable`, serta `core/error` dan `core/usecase`.

## 5. Aturan Kode (Ringkas)

Detail di `CODE_STYLE.md`. Yang paling sering dilanggar:

- Pakai `const` constructor kapan pun bisa.
- Pecah widget besar. Hindari `build()` panjang dan nested dalam.
- **Tidak ada hardcode**: string (pakai l10n), warna/ukuran/font (pakai token lewat `context`, lihat `DESIGN_SYSTEM.md`), URL/key (pakai `AppConfig`).
- Jangan taruh logic bisnis di widget. Taruh di bloc atau usecase.
- Error: data layer **throw `Failure`** → usecase mengembalikan **`Result<T>`** → Bloc `switch` pada `Result`. Bloc tidak pakai try-catch. Lihat `ARCHITECTURE.md` section 10.
- Event yang memicu request (submit, refresh) pakai transformer `droppable()`; search/filter pakai `restartable()`.
- Hindari `dynamic` dan `!` (force unwrap) kecuali benar-benar aman dan diberi komentar.
- Semua import pakai `package:` (bukan relative).
- Semua logging lewat `AppLogger` dari `core/logger` — dilarang `print`/`debugPrint`.
- Komentar kode ditulis dalam **Bahasa Inggris**.

## 6. Aturan "Jangan"

- Jangan menambah package baru tanpa konfirmasi. Sebutkan alasan dan alternatifnya.
- Jangan mengubah file native tanpa konfirmasi: `android/**` (`AndroidManifest.xml`, `build.gradle.kts`, `proguard-rules.pro`), `ios/**` (`Info.plist`, `Podfile`, `*.xcconfig`, `project.pbxproj`, scheme). Untuk perubahan Xcode project, utamakan memberi instruksi langkah di Xcode ke user daripada mengedit `project.pbxproj` langsung.
- Jangan menambah permission yang tidak tercantum di `PRD.md` section 10.
- Jangan edit file generated (`*.g.dart`, `*.config.dart`, `*.gr.dart`, output `gen-l10n`, asset gen). Ubah sumbernya lalu regenerate.
- Jangan mengubah arsitektur atau struktur folder di luar scope tugas.
- Jangan refactor, rename, atau hapus kode yang tidak terkait tugas.
- Jangan commit secret, API key, keystore, atau file `.env.*`. Lihat `SECURITY.md`.
- Jangan mematikan lint rule atau memakai `// ignore:` hanya supaya warning hilang.
- Jangan menghapus atau melemahkan test supaya build hijau.
- Jangan mengarang API, field, atau perilaku backend. Kalau tidak ada di `PRD.md`, tanya.
- Jangan menjalankan perintah destruktif tanpa izin: `git push --force`, `git reset --hard`, hapus branch, `rm -rf ios/Pods` / `pod deintegrate`, hapus database lokal, ubah `.fvmrc`.
- Jangan upgrade versi Flutter/Dart atau major version package tanpa konfirmasi.

## 7. Alur Kerja

1. **Pahami dulu.** Baca dokumen relevan (minimal `PRD.md` untuk fitur terkait) dan kode sekitar sebelum menulis.
2. **Rencanakan.** Tulis rencana singkat: fitur/AC yang dikerjakan (`F-xx`), daftar file yang akan dibuat/diubah, dan pendekatannya. **Tunggu konfirmasi user sekali** sebelum mulai.
3. **Kerjakan sampai selesai.** Setelah rencana disetujui, kerjakan tanpa perlu minta konfirmasi per langkah.
4. **Berhenti dan tanya** kalau di tengah jalan muncul salah satu dari ini:
   - Perlu mengubah **lebih dari 3 file di luar rencana**.
   - Perlu menyentuh file native (`android/`, `ios/`).
   - Perlu package baru atau upgrade versi.
   - Perlu mengubah arsitektur, struktur folder, atau kontrak antar layer.
   - Requirement di `PRD.md` ambigu, kosong, atau tercantum di "Pertanyaan Terbuka".
5. **Verifikasi.** Jalankan `make check` (atau format + analyze + test raw).
6. **Laporkan.** Ringkas apa yang berubah, AC mana yang terpenuhi, apa yang belum, dan asumsi yang dipakai.

Kalau ada beberapa pendekatan yang masuk akal, ajukan satu pertanyaan spesifik beserta rekomendasimu daripada menebak.

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

Branch utama: `main` (stabil, production-ready) dan `development` (integrasi). Semua pekerjaan dilakukan di branch tipe di atas, merge ke `development` dulu. `hotfix/` dibuat dari `main` dan di-merge ke `main` + `development`.

**Commit** — pakai Conventional Commits, Bahasa Inggris:

```
<type>(<scope>): <deskripsi singkat>

feat(auth): add email login usecase
fix(cart): prevent duplicate item on rapid tap
chore(deps): bump dio to 5.x
test(auth): add login bloc tests
docs: update architecture diagram
```

Type: `feat`, `fix`, `chore`, `refactor`, `test`, `docs`, `style`, `perf`, `build`, `ci`. Scope opsional tapi disarankan (pakai nama fitur). Satu commit = satu perubahan logis. Agent hanya commit/push kalau diminta.

## 9. Definition of Done

Tugas dianggap selesai kalau:

- [ ] Acceptance criteria fitur terkait di `PRD.md` terpenuhi
- [ ] Sesuai `ARCHITECTURE.md` (layer, DI, `Result`/`Failure`, routing)
- [ ] Codegen sudah di-regenerate kalau ada perubahan DI/database/l10n/assets (`make gen`)
- [ ] `make check` lolos: format, analyze bersih (tanpa warning baru), test hijau
- [ ] Ada test baru untuk logic yang ditambah (usecase, repository, bloc)
- [ ] Tidak ada hardcoded string/warna/ukuran/secret; string baru ada di ARB `id` **dan** `en`
- [ ] Tidak ada file di luar scope yang berubah
- [ ] Untuk UI: sesuai `DESIGN_SYSTEM.md`, light & dark, tidak overflow di layar kecil dan text scale 1.3x, state loading/empty/error ada

## 10. Testing (Ringkas)

Detail di `TESTING.md`.

- Struktur `test/` mirror `lib/`.
- Wajib dites: usecase, repository, bloc, dan util.
- Widget test untuk komponen design system dan alur UI penting.
- Mock dependency eksternal dengan `mocktail`. Jangan memanggil API sungguhan di test.

## 11. Keamanan (Ringkas)

Detail di `SECURITY.md`. Minimal:

- Token dan data sensitif hanya di `flutter_secure_storage`.
- Jangan log data sensitif (token, PII, password).
- Semua input dari luar (API, deep link, form) divalidasi.
- Komunikasi hanya lewat HTTPS.
- Nilai di `.env` ikut ter-compile ke binary — jangan taruh secret server di sana.

## 12. Memperbarui Dokumen

Kalau perubahan kamu mengubah arsitektur, konvensi, atau perintah, update dokumen terkait di commit yang sama dan sebutkan di laporan akhir. Kalau tugas menambah/menyelesaikan fitur, centang AC terkait di `PRD.md`.
