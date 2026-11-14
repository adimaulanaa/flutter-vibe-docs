# ARCHITECTURE.md

Dokumen ini mendefinisikan struktur dan aturan arsitektur project. Semua kode harus mengikuti dokumen ini. Kalau ada kebutuhan yang memaksa penyimpangan, diskusikan dulu, lalu update dokumen ini.

## 1. Prinsip Utama

- **Clean Architecture 3 layer** per fitur: `data` → `domain` → `presentation`.
- **Dependency rule**: dependency hanya mengarah ke dalam. `presentation → domain ← data`. Layer `domain` tidak boleh import Flutter, dio, drift, atau package data mana pun.
- **Feature-first**: kode dikelompokkan per fitur, bukan per jenis file.
- **Satu arah data**: UI mengirim event ke Bloc, Bloc memanggil usecase/repository, state baru di-emit, UI render ulang. Tidak ada jalan pintas (widget langsung panggil repository = dilarang).

## 2. Struktur Folder Lengkap

```
lib/
├── core/
│   ├── config/       # AppConfig per flavor, pembacaan env
│   ├── design_system/# tokens, theme, komponen App* (detail di DESIGN_SYSTEM.md)
│   ├── di/           # setup get_it + injectable (injection.dart, injection.config.dart)
│   ├── error/        # Failure, exception, error mapper
│   ├── l10n/         # localization (arb files, delegate)
│   ├── logger/       # AppLogger: abstraksi logging (console / Firebase / remote API)
│   ├── network/      # dio client, interceptor (auth, logging, retry)
│   ├── router/       # go_router setup — SEMUA route terpusat di sini
│   └── utils/        # helper murni (formatter, validator, extension)
├── shared/
│   ├── widgets/      # widget komposit lintas fitur yang BUKAN bagian design system
│   └── models/       # model yang dipakai lintas fitur
├── features/
│   └── <feature>/
│       ├── data/
│       │   ├── datasources/   # remote (dio) & local (drift/prefs/secure)
│       │   ├── models/        # DTO: serialisasi JSON/database
│       │   └── repositories/  # implementasi repository interface dari domain
│       ├── domain/
│       │   ├── entities/      # objek bisnis murni
│       │   ├── repositories/  # interface (abstract class) repository
│       │   └── usecases/      # satu class = satu aksi bisnis
│       └── presentation/
│           ├── bloc/          # <feature>_bloc.dart, <feature>_event.dart, <feature>_state.dart
│           ├── pages/         # screen full-page
│           └── widgets/       # widget khusus fitur ini
├── main_staging.dart
└── main_prod.dart
test/                         # mirror struktur lib/
```

## 3. Tanggung Jawab Tiap Layer

### data
- Berkomunikasi dengan dunia luar: REST API (dio), database (drift), prefs, secure storage.
- DTO (`models/`) punya `fromJson`/`toJson`. DTO **bukan** entity — selalu dikonversi.
- Repository impl menggabungkan datasource, konversi DTO → entity, dan mapping exception → `Failure`.

### domain
- Murni Dart. Tanpa Flutter, tanpa package eksternal.
- `entities`: objek bisnis yang dipakai UI.
- `repositories`: abstract class — kontrak yang diimplementasi di layer data.
- `usecases`: satu class satu aksi, misal `LoginWithEmail`, `GetCartItems`. Dipanggil dari Bloc.

### presentation
- `bloc/`: menerima event, memanggil usecase, emit state. **Bloc murni — dilarang pakai Cubit.**
- `pages/` dan `widgets/`: hanya render state dan mengirim event. Tanpa logic bisnis.
- State harus immutable. Gunakan `sealed class` untuk state agar `switch` di UI exhaustive.

## 4. core/ Modules

| Modul | Isi | Aturan |
|---|---|---|
| `config` | `AppConfig` per flavor (baseUrl, nama app, flag fitur) | Nilai sensitif/config penting WAJIB dari `.env`, lihat section 6 |
| `di` | Registrasi get_it via injectable | File `*.config.dart` adalah generated — jangan diedit |
| `error` | `Failure` (sealed), exception, mapper exception→Failure | Semua layer data melempar `Failure`, bukan exception mentah |
| `l10n` | Localization | Semua string UI lewat l10n, tidak ada hardcode |
| `logger` | `AppLogger`: abstraksi logging | Semua log lewat `AppLogger`, lihat section 11 |
| `network` | Dio client + interceptor (auth token, logging, error) | Satu instance dio dari DI, jangan `Dio()` baru di tempat lain |
| `router` | go_router, semua route terpusat | Route tidak boleh tersebar di file fitur |
| `design_system` | Tokens, `ThemeData` light/dark, komponen `App*` | Struktur dan aturan lengkap di `DESIGN_SYSTEM.md` |
| `utils` | Helper murni | Tidak boleh ada logic bisnis atau state |

`core/` tidak boleh berisi logic fitur. Kalau ragu ini core atau fitur → hampir pasti fitur.

## 5. shared/

- `shared/widgets/`: widget komposit lintas fitur yang **bukan** bagian design system (misal kartu gabungan beberapa komponen). Komponen dasar `App*` (button, text field, dialog, dsb) tinggal di `core/design_system/components/` — lihat `DESIGN_SYSTEM.md`.
- `shared/models/`: model yang dipakai 2+ fitur.
- Aturan promosi: kode baru mulai di fitur. Baru pindah ke `shared/` kalau benar-benar dipakai 2+ fitur. Jangan preemptive.

## 6. Flavor & Environment Config

Flavor tersedia: `staging` dan `prod`. Development sehari-hari pakai `staging`.

- Entry point: `lib/main_staging.dart` dan `lib/main_prod.dart`. Isinya tipis — load env, init DI, run app.
- Config penting (baseUrl, API key, dsb) **wajib lewat `.env`**, tidak boleh hardcode di Dart:
  - `.env.staging` dan `.env.prod` — **tidak di-commit** (masuk `.gitignore`).
  - `.env.example` — berisi daftar key tanpa value asli, **di-commit** sebagai panduan.
- `AppConfig` membaca env saat startup dan menyediakan nilai ke seluruh app lewat DI. Kode lain tidak boleh baca env langsung.
- Detail penanganan secret ada di `SECURITY.md`.

## 7. Alur Data (Contoh: Login)

```
UI (page) ──event──> Bloc ──> Usecase ──> Repository (interface)
                                              │
                                              v
UI <──state── Bloc <──entity── RepositoryImpl <──DTO── Datasource (dio/drift)
```

1. User tap tombol → UI `bloc.add(LoginSubmitted(email, password))`.
2. Bloc emit `LoginLoading`, panggil `LoginWithEmail`.
3. Usecase panggil `AuthRepository.login()`.
4. RepositoryImpl panggil remote datasource, konversi DTO → `User` entity, exception → `Failure`.
5. Bloc emit `LoginSuccess(user)` atau `LoginFailure(message)`.
6. UI render sesuai state (pakai `BlocBuilder` dengan `switch` exhaustive pada sealed state).

## 8. Dependency Injection

- Semua dependency (datasource, repository, usecase, bloc, dio, database) diregistrasi lewat injectable.
- Scope: datasource/repository = lazy singleton; bloc = factory (instance baru per pemakaian).
- Setelah menambah/mengubah anotasi injectable, wajib regenerate: `make gen` atau `fvm dart run build_runner build --delete-conflicting-outputs`.

## 9. Navigasi

- Semua route didefinisikan terpusat di `core/router/` memakai go_router.
- Path dan nama route didefinisikan sebagai constant, tidak ada string path tersebar di fitur.
- Deep link dan redirect guard (misal cek auth) diatur di router, bukan di dalam page.

## 10. Error Handling

- Layer data melempar/mengembalikan `Failure` terdefinisi di `core/error` (misal `NetworkFailure`, `ServerFailure`, `CacheFailure`, `AuthFailure`).
- Bloc menerjemahkan `Failure` → state error dengan pesan yang siap tampil (lewat l10n).
- Tidak ada exception yang lolos mentah ke UI. Tidak ada `try-catch` kosong.

## 11. Logging (`AppLogger`)

- Semua logging lewat **`AppLogger`** dari `core/logger` — dilarang `print`, `debugPrint`, atau memanggil package logger langsung di kode fitur.
- `AppLogger` adalah abstraksi (interface) dengan level: `debug`, `info`, `warning`, `error`. Implementasi konkret dipasang sebagai **sink** di belakangnya.
- Sink yang tersedia dan kapan aktifnya:
  - `ConsoleSink` — flavor `staging` / build debug.
  - `CrashlyticsSink` (Firebase Crashlytics) — flavor `prod`, hanya level `warning` ke atas.
  - `RemoteApiSink` (kirim log ke API sendiri) — opsional per project, diaktifkan lewat config.
- Sink didaftarkan saat bootstrap (di `main_<flavor>.dart`) lewat DI. Mengganti backend log = ganti sink, tanpa menyentuh kode fitur.
- `AppLogger` bertugas menyaring data sensitif sebelum log keluar — lihat `SECURITY.md` section 8.

## 12. Testing

Struktur `test/` mirror `lib/`. Aturan lengkap di `TESTING.md`.

- `domain`: usecase dites dengan repository mock.
- `data`: repository dites dengan datasource mock; datasource dites dengan dio/storage mock.
- `presentation`: bloc dites dengan `bloc_test`; widget penting dites dengan widget test.

## 13. Menambah Fitur Baru (Checklist)

1. Buat struktur `features/<feature>/{data,domain,presentation}`.
2. Definisikan entity dan repository interface di `domain`.
3. Implementasi usecase.
4. Implementasi datasource + repository impl + DTO di `data`.
5. Buat bloc (event + state) di `presentation/bloc`.
6. Buat page/widget.
7. Daftarkan semua dependency di injectable, lalu regenerate codegen.
8. Daftarkan route di `core/router/`.
9. Tulis test (lihat `TESTING.md`).
10. Jalankan format, analyze, test — lihat Definition of Done di `AGENTS.md`.
