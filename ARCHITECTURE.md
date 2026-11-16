# ARCHITECTURE.md

Dokumen ini mendefinisikan struktur dan aturan arsitektur project. Semua kode harus mengikuti dokumen ini. Kalau ada kebutuhan yang memaksa penyimpangan, diskusikan dulu, lalu update dokumen ini.

## 1. Prinsip Utama

- **Clean Architecture 3 layer** per fitur: `data` → `domain` → `presentation`.
- **Dependency rule**: dependency hanya mengarah ke dalam. `presentation → domain ← data`. Layer `domain` hanya boleh import Dart murni, `equatable`, anotasi `injectable`, serta `core/error` dan `core/usecase` — tidak boleh import Flutter, dio, drift, atau package data mana pun.
- **Feature-first**: kode dikelompokkan per fitur, bukan per jenis file.
- **Satu arah data**: UI mengirim event ke Bloc, Bloc memanggil usecase (yang mengembalikan `Result`), state baru di-emit, UI render ulang. Tidak ada jalan pintas (widget langsung panggil repository/usecase = dilarang).
- **Error eksplisit**: data layer melempar `Failure`, usecase membungkus hasil dalam `Result<T>`, Bloc `switch` pada `Result`. Detail di section 10.

## 2. Struktur Folder Lengkap

```
lib/
├── core/
│   ├── config/       # AppConfig per flavor, pembacaan env (dart-define)
│   ├── design_system/# tokens, theme, komponen App* (detail di DESIGN_SYSTEM.md)
│   ├── di/           # setup get_it + injectable (injection.dart, injection.config.dart)
│   ├── error/        # Failure (sealed), Result<T>, exception, error mapper
│   ├── l10n/         # ARB files sumber (app_id.arb, app_en.arb)
│   ├── logger/       # AppLogger: abstraksi logging (console / Crashlytics / remote API)
│   ├── network/      # dio client, interceptor (auth, logging, error)
│   ├── router/       # go_router setup — SEMUA route terpusat di sini
│   ├── usecase/      # UseCase base class (params → Result<T>)
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
│       │   ├── entities/      # objek bisnis murni (extends Equatable)
│       │   ├── repositories/  # interface (abstract class) repository
│       │   └── usecases/      # satu class = satu aksi bisnis
│       └── presentation/
│           ├── bloc/          # <feature>_bloc.dart, <feature>_event.dart, <feature>_state.dart
│           ├── pages/         # screen full-page
│           └── widgets/       # widget khusus fitur ini
├── app.dart         # root widget (MaterialApp.router: theme, l10n, router)
├── bootstrap.dart   # init bersama: error handler, AppConfig, DI, logger sinks
├── main_staging.dart
└── main_prod.dart
test/                # mirror struktur lib/
integration_test/    # alur end-to-end (TESTING.md)
l10n.yaml            # konfigurasi gen-l10n (arb-dir, output)
```

## 3. Tanggung Jawab Tiap Layer

### data
- Berkomunikasi dengan dunia luar: REST API (dio), database (drift), prefs, secure storage.
- DTO (`models/`) punya `fromJson`/`toJson`. DTO **bukan** entity — selalu dikonversi.
- Repository impl menggabungkan datasource, konversi DTO → entity, dan mapping exception → `Failure` lalu **melemparnya** (`throw`).

### domain
- Murni Dart. Import yang diizinkan: `dart:*`, `equatable`, anotasi `injectable`, `core/error`, `core/usecase`.
- `entities`: objek bisnis yang dipakai UI, wajib `extends Equatable`.
- `repositories`: abstract class — kontrak yang diimplementasi di layer data.
- `usecases`: satu class satu aksi, misal `LoginWithEmail`, `GetCartItems`. Meng-extend `UseCase<T, Params>` dan **mengembalikan `Result<T>`** — catch `Failure` dari repository di sini.

### presentation
- `bloc/`: menerima event, memanggil usecase, `switch` pada `Result`, emit state. **Bloc murni — dilarang pakai Cubit.** Tanpa try-catch.
- `pages/` dan `widgets/`: hanya render state dan mengirim event. Tanpa logic bisnis. **Dilarang** memanggil repository/usecase langsung — semua lewat event Bloc.
- State harus immutable dan `extends Equatable`. Gunakan `sealed class` untuk state agar `switch` di UI exhaustive.

## 4. core/ Modules

| Modul | Isi | Aturan |
|---|---|---|
| `config` | `AppConfig` per flavor (baseUrl, nama app, flag fitur) | Nilai config WAJIB dari `.env` via `--dart-define-from-file`, lihat section 6 |
| `di` | Registrasi get_it via injectable | File `*.config.dart` adalah generated — jangan diedit |
| `error` | `Failure` (sealed), `Result<T>` (sealed), exception, mapper | Pola lengkap di section 10 |
| `l10n` | File ARB sumber (`app_id.arb`, `app_en.arb`) | Output `gen-l10n` adalah generated — jangan diedit, lihat section 13 |
| `logger` | `AppLogger`: abstraksi logging | Semua log lewat `AppLogger`, lihat section 11 |
| `network` | Dio client + interceptor (auth token, logging, error) | Satu instance dio dari DI, jangan `Dio()` baru di tempat lain |
| `router` | go_router, semua route terpusat | Route tidak boleh tersebar di file fitur, lihat section 9 |
| `design_system` | Tokens, `ThemeData` light/dark, komponen `App*` | Struktur dan aturan lengkap di `DESIGN_SYSTEM.md` |
| `usecase` | Base class `UseCase<Type, Params>` + `NoParams` | Semua usecase meng-extend ini |
| `utils` | Helper murni | Tidak boleh ada logic bisnis atau state |

`core/` tidak boleh berisi logic fitur. Kalau ragu ini core atau fitur → hampir pasti fitur.

## 5. shared/

- `shared/widgets/`: widget komposit lintas fitur yang **bukan** bagian design system (misal kartu gabungan beberapa komponen). Komponen dasar `App*` (button, text field, dialog, dsb) tinggal di `core/design_system/components/` — lihat `DESIGN_SYSTEM.md`.
- `shared/models/`: model yang dipakai 2+ fitur.
- Aturan promosi: kode baru mulai di fitur. Baru pindah ke `shared/` kalau benar-benar dipakai 2+ fitur. Jangan preemptive.

## 6. Flavor & Environment Config

Flavor tersedia: `staging` dan `prod`. Development sehari-hari pakai `staging`.

- Entry point: `lib/main_staging.dart` dan `lib/main_prod.dart`. Isinya sangat tipis — hanya memanggil `bootstrap()` dengan flavor yang sesuai. Semua init bersama ada di `bootstrap.dart`: setup error handler, load `AppConfig`, init DI, daftarkan logger sink, lalu `runApp(App())`.
- Config penting (baseUrl, API key, dsb) **wajib lewat `.env`**, tidak boleh hardcode di Dart:
  - `.env.staging` dan `.env.prod` — **tidak di-commit** (masuk `.gitignore`).
  - `.env.example` — berisi daftar key tanpa value asli, **di-commit** sebagai panduan.
- Env dimuat lewat flag **`--dart-define-from-file=.env.<flavor>`** pada setiap `run`/`build`/integration test. Command pasti ada di Makefile/README — jangan mengetik flag panjang manual.
- `AppConfig` (di `core/config`) adalah **satu-satunya** class yang membaca nilai dart-define (`String.fromEnvironment`). Ia memvalidasi key wajib saat startup dan gagal cepat dengan pesan jelas kalau ada yang kurang. Nilai disediakan ke seluruh app lewat DI. **Kode lain dilarang** membaca env langsung.
- Catatan keamanan: nilai dart-define ikut ter-compile ke binary — jangan taruh secret server di `.env`, hanya konfigurasi client. Detail di `SECURITY.md`.

### 6.1 Setup Flavor Native

Perubahan file native wajib konfirmasi user dulu (lihat `AGENTS.md` section 6). Kebutuhan flavor per platform:

**Android** (`android/app/build.gradle.kts`):
- `productFlavors` dengan dua flavor: `staging` dan `prod`.
- `applicationId` per flavor sesuai `PRD.md` section 3 (staging dengan suffix `.staging`).
- Nama app per flavor: `strings.xml` di `android/app/src/<flavor>/res/values/` (staging misal `[STG] NamaApp`).

**iOS**:
- Scheme per flavor di Xcode (duplicate `Runner` → `Runner staging`, `Runner prod`) dengan konfigurasi build terpisah (`Debug-staging`, `Release-staging`, dst. via "Duplicate Configuration").
- `ios/Flutter/<flavor>.xcconfig` per flavor yang `#include "Generated.xcconfig"` dan men-set `FLUTTER_TARGET` plus variabel bundle id / nama app.
- Bundle id per flavor lewat variabel xcconfig di build settings (misal `PRODUCT_BUNDLE_IDENTIFIER = $(APP_BUNDLE_ID)`).
- Mengedit `project.pbxproj` raw berisiko — **utamakan memberi instruksi langkah di Xcode ke user** daripada mengedit file project langsung.

**Ikon & splash per flavor**: `flutter_launcher_icons` dan `flutter_native_splash` dengan config per flavor (`flutter_launcher_icons-staging.yaml`, dst). Ikon staging diberi penanda (badge/warna) — lihat `PRD.md` section 13.

## 7. Alur Data (Contoh: Login)

```
UI (page) ──event──> Bloc ──> Usecase ──> Repository (interface)
                                              │ (throws Failure)
                                              v
UI <──state── Bloc <──Result<User>── Usecase <────── RepositoryImpl <──DTO── Datasource (dio/drift)
```

1. User tap tombol → UI `bloc.add(AuthLoginSubmitted(email: ..., password: ...))`. Event ini memakai transformer `droppable()` agar tap berulang tidak mengirim request ganda (section 12).
2. Bloc emit `AuthLoginLoading`, panggil usecase `LoginWithEmail`.
3. Usecase panggil `AuthRepository.login()`, menangkap `Failure`, mengembalikan `Result<User>`.
4. RepositoryImpl panggil remote datasource, konversi DTO → `User` entity; exception → map → `throw AuthFailure(...)`.
5. Bloc `switch` pada `Result`: `Success` → emit `AuthLoginSuccess(user)`; `Err` → emit `AuthLoginError(failure)`.
6. UI render sesuai state (`BlocBuilder` dengan `switch` exhaustive pada sealed state; pesan error dipetakan ke l10n).

## 8. Dependency Injection

- Semua dependency (datasource, repository, usecase, bloc, dio, database) diregistrasi lewat injectable.
- Scope:
  - datasource, repository impl, usecase = **lazy singleton**.
  - bloc = **factory** (instance baru per pemakaian).
- Bloc dibuat oleh `BlocProvider` di **route builder** `core/router` (`create: (_) => getIt<AuthBloc>()`), bukan di dalam page. Page hanya mengonsumsi lewat `context.read` / `BlocBuilder`.
- Setelah menambah/mengubah anotasi injectable, wajib regenerate: `make gen` atau `fvm dart run build_runner build --delete-conflicting-outputs`.

## 9. Navigasi

- Semua route didefinisikan terpusat di `core/router/` memakai go_router, sesuai tabel layar di `PRD.md` section 5.
- Path dan nama route didefinisikan sebagai constant (`AppRoutes`), tidak ada string path tersebar di fitur.
- **Auth guard**: go_router `redirect` membaca status auth. Agar redirect reaktif, pasang `refreshListenable` ke stream status auth (misal `GoRouterRefreshStream` yang membungkus `AuthBloc.stream`). Jangan pakai polling/timer.
- Deep link divalidasi di router — parameter yang gagal validasi tidak boleh sampai ke page. Lihat `SECURITY.md` section 7.

## 10. Error Handling

Pola resmi project: **`Failure` (throw di data) + `Result<T>` (return dari usecase)** — tanpa package FP tambahan.

```dart
// core/error/failure.dart
sealed class Failure implements Exception {
  const Failure({this.code, this.debugMessage});
  final String? code;
  final String? debugMessage; // Technical detail — for logs only, never UI.
}

final class NetworkFailure extends Failure { ... }
final class ServerFailure extends Failure { ... }    // carries server error code
final class AuthFailure extends Failure { ... }      // invalid credentials, session expired, dll
final class CacheFailure extends Failure { ... }
final class ValidationFailure extends Failure { ... }

// core/error/result.dart
sealed class Result<T> {
  const Result();
}
final class Success<T> extends Result<T> {
  const Success(this.data);
  final T data;
}
final class Err<T> extends Result<T> {
  const Err(this.failure);
  final Failure failure;
}

// core/usecase/usecase.dart
abstract class UseCase<Type, Params> {
  Future<Result<Type>> call(Params params);
}
```

Aturan per layer:

- **Data**: repository impl menangkap exception (dio, drift, platform) → map ke `Failure` → **`throw`**. Mapping exception→Failure terpusat di satu mapper di `core/error`.
- **Domain**: usecase membungkus pemanggilan repository:

```dart
@lazySingleton
class LoginWithEmail extends UseCase<User, LoginParams> {
  LoginWithEmail(this._repository);
  final AuthRepository _repository;

  @override
  Future<Result<User>> call(LoginParams params) async {
    try {
      final user = await _repository.login(
        email: params.email,
        password: params.password,
      );
      return Success(user);
    } on Failure catch (failure) {
      return Err(failure);
    }
  }
}
```

- **Presentation**: Bloc tidak pakai try-catch — cukup `switch` pada `Result`:

```dart
switch (result) {
  case Success(:final data):
    emit(AuthLoginSuccess(data));
  case Err(:final failure):
    emit(AuthLoginError(failure));
}
```

- Pesan yang tampil ke user **selalu lewat l10n**, dipetakan dari tipe/kode `Failure` di presentation layer. `debugMessage` hanya untuk log. Tidak ada exception yang lolos mentah ke UI. Tidak ada `try-catch` kosong.

## 11. Logging (`AppLogger`)

- Semua logging lewat **`AppLogger`** dari `core/logger` — dilarang `print`, `debugPrint`, atau memanggil package logger langsung di kode fitur.
- `AppLogger` adalah abstraksi (interface) dengan level: `debug`, `info`, `warning`, `error`. Implementasi konkret dipasang sebagai **sink** di belakangnya.
- Sink yang tersedia dan kapan aktifnya:
  - `ConsoleSink` — flavor `staging` / build debug.
  - `CrashlyticsSink` (Firebase Crashlytics) — flavor `prod`, hanya level `warning` ke atas. **Hanya dibuat kalau Firebase disetujui** (`PRD.md` section 12).
  - `RemoteApiSink` (kirim log ke API sendiri) — opsional per project, diaktifkan lewat config.
- Sink didaftarkan saat bootstrap lewat DI. Mengganti backend log = ganti sink, tanpa menyentuh kode fitur.
- `AppLogger` bertugas menyaring data sensitif sebelum log keluar — lihat `SECURITY.md` section 9.
- Uncaught error: `FlutterError.onError` dan `PlatformDispatcher.onError` di `bootstrap.dart` meneruskan ke `AppLogger` (dan Crashlytics bila aktif).

## 12. Concurrency di Bloc

Event handler yang memicu request wajib memakai transformer dari `bloc_concurrency` agar tidak ada request ganda/race:

- `droppable()` — submit, retry, pull-to-refresh (event baru di-drop saat handler berjalan).
- `restartable()` — search, filter (event baru membatalkan handler lama).
- Default (concurrent) hanya untuk event yang aman dijalankan paralel.

## 13. Localization (l10n)

- Sumber ARB di `lib/core/l10n/` (`app_id.arb` sebagai template, `app_en.arb`). Locale default `id`, tambahan `en`.
- Konfigurasi di `l10n.yaml` (root project): `arb-dir: lib/core/l10n`, `template-arb-file`, `output-dir: lib/core/l10n/generated` (output generated — jangan diedit).
- Generate lewat `make gen` (`fvm flutter gen-l10n` + build_runner).
- Akses di kode: `context.l10n.<key>` (extension tipis di atas `AppLocalizations`). Pesan error dari `Failure` dipetakan ke key l10n di presentation layer.
- Setiap string baru wajib ditambahkan ke ARB `id` **dan** `en` sekaligus.

## 14. Testing

Struktur `test/` mirror `lib/`. Aturan lengkap di `TESTING.md`.

- `domain`: usecase dites dengan repository mock (sukses → `Success`, gagal → `Err` dengan `Failure` yang benar).
- `data`: repository dites dengan datasource mock — memastikan exception termapping ke `Failure` yang benar; datasource dites dengan dio/storage mock.
- `presentation`: bloc dites dengan `bloc_test`; widget penting dites dengan widget test.
- Database drift dites dengan `NativeDatabase.memory()` — in-memory, bukan file sungguhan.

## 15. Menambah Fitur Baru (Checklist)

1. Pastikan fitur ada di `PRD.md` (punya `F-xx` + acceptance criteria).
2. Buat struktur `features/<feature>/{data,domain,presentation}`.
3. Definisikan entity dan repository interface di `domain`.
4. Implementasi usecase (extend `UseCase`, return `Result`).
5. Implementasi datasource + repository impl + DTO di `data`.
6. Buat bloc (event + state sealed) di `presentation/bloc` — pasang transformer concurrency yang sesuai.
7. Buat page/widget memakai komponen `App*`.
8. Daftarkan semua dependency di injectable, lalu regenerate codegen.
9. Daftarkan route di `core/router/` (dengan `BlocProvider` di route builder).
10. Tambahkan string l10n (`id` + `en`).
11. Tulis test (lihat `TESTING.md`).
12. Jalankan `make check` — lihat Definition of Done di `AGENTS.md`.
