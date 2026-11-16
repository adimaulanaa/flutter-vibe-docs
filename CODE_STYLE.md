# CODE_STYLE.md

Aturan penulisan kode untuk project ini. Tujuannya: kode konsisten siapa pun (atau AI apa pun) yang menulisnya. Kalau aturan di sini bertentangan dengan lint yang aktif, lint yang menang — laporkan konfliknya.

## 1. Lint & Format

- Lint: `very_good_analysis`. Rule `public_member_api_docs` **dimatikan** di `analysis_options.yaml` (doc comment hanya wajib untuk `core/` dan `shared/` — lihat section 9). Selain itu, jangan mematikan rule tanpa konfirmasi.
- Format: `fvm dart format .` wajib sebelum selesai. Jangan format ulang file yang tidak kamu ubah.
- Selalu pakai **trailing comma** pada argument/parameter list multiline — bikin format rapi dan diff bersih.
- Batas panjang baris: 80 karakter (rule `lines_longer_than_80_chars` aktif).

## 2. Penamaan

| Hal | Aturan | Contoh |
|---|---|---|
| File & folder | `snake_case.dart` | `login_page.dart` |
| Class | `PascalCase` | `LoginPage` |
| Variable, method, parameter | `camelCase` | `isLoading` |
| Private member | prefix `_` | `_authRepository` |
| Konstanta | `camelCase` di dalam class token | `AppSpacing.md` |
| Route | constant di `AppRoutes` | `AppRoutes.login`, name `login` |
| Bloc | `<Feature>Bloc` di `<feature>_bloc.dart` | `auth_bloc.dart` → `AuthBloc` |
| Event | `<Feature><Aksi>`, kata kerja | `AuthLoginSubmitted` |
| State | sealed `<Feature>State` + subclass `<Feature><Sub><Kondisi>` | `AuthState` → `AuthLoginLoading`, `AuthLoginSuccess`, `AuthLoginError` |
| Usecase | kata kerja + objek | `LoginWithEmail`, `GetCartItems` |
| Repository | interface `XRepository`, impl `XRepositoryImpl` | `AuthRepository` / `AuthRepositoryImpl` |
| Datasource | `<Feature><Sumber>DataSource` | `AuthRemoteDataSource`, `AuthLocalDataSource` |
| DTO | `<X>Dto` | `UserDto` |
| Widget design system | prefix `App` | `AppButton` |
| Test file | `<nama_file>_test.dart` | `auth_bloc_test.dart` |

Catatan state: selalu pakai suffix `Error` untuk state gagal (`AuthLoginError`) — **jangan** `*Failure`, supaya tidak tertukar dengan class `Failure` dari `core/error`.

## 3. Import

- **Selalu `package:` import** — rule `always_use_package_imports` aktif, jadi relative import (`import 'widgets/login_form.dart'`) akan error. Ini berlaku bahkan untuk file dalam fitur yang sama.
- Urutan import diatur otomatis oleh rule `directives_ordering` (dart → package → export); biarkan formatter/IDE mengurutkan.
- Dilarang import file dari layer dalam ke layer luar (lihat dependency rule di `ARCHITECTURE.md`).
- Tidak ada import yang tidak terpakai.

```dart
import 'dart:async';

import 'package:dio/dio.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

import 'package:app/core/error/result.dart';
import 'package:app/features/auth/presentation/widgets/login_form.dart';
```

## 4. Class, Entity, dan Equatable

- **Entity** (domain) dan **Bloc state & event** wajib `extends Equatable` — perbandingan value-based, supaya Bloc tidak emit state berulang dan test mudah.
- **DTO** tidak perlu Equatable; tugasnya hanya serialisasi (`fromJson`/`toJson`).
- Semua field `final`. Class data harus immutable.
- Satu public class per file. Helper class kecil yang private (`_X`) boleh satu file dengan pemakainya.
- `late` hanya untuk inisialisasi yang pasti terjadi sebelum dipakai (misal di `initState`), dan wajib diberi komentar alasan.

## 5. Null Safety

- Dilarang `!` (force unwrap) kecuali benar-benar aman dan diberi komentar alasan.
- Hindari `dynamic`. Untuk JSON selalu lewat DTO, bukan `Map<String, dynamic>` yang dioper-oper ke layer lain. (Catatan: `strict-casts` dan `strict-inference` aktif di lint.)
- Prefer `?` + early return atau default value daripada force unwrap.

## 6. Widget

- `const` constructor kapan pun bisa.
- Pecah widget besar jadi widget kecil privat — `build()` tidak boleh panjang dan nested dalam.
- Layout utama: tidak ada `width`/`height` hardcode (aturan lengkap di `DESIGN_SYSTEM.md`).
- Tidak ada logic bisnis di widget — kirim event ke Bloc, render dari state. Widget juga **dilarang** memanggil repository/usecase langsung.
- Gunakan komponen `App*` dari design system, bukan widget Material mentah kalau padanannya ada.
- `super.key` selalu diteruskan di constructor widget.

## 7. Fungsi

- Maksimal ~30 baris. Lebih dari itu, pecah jadi fungsi privat.
- Parameter lebih dari 3 → wajib named parameter.
- Satu fungsi satu tugas. Kalau perlu kata "dan" untuk menjelaskannya, pecah.
- Early return untuk guard condition, hindari nested `if` dalam.

## 8. Error Handling

- Pola resmi: data layer **throw `Failure`** → usecase mengembalikan **`Result<T>`** (`Success`/`Err`) → Bloc `switch` pada `Result`. Detail + contoh kode di `ARCHITECTURE.md` section 10.
- Bloc tidak pakai try-catch. Usecase yang menangkap `Failure` dari repository.
- Dilarang `try-catch` kosong atau menelan exception diam-diam.
- Pesan error untuk user selalu lewat l10n, dipetakan dari tipe/kode `Failure` — bukan string mentah dari exception.

## 9. Komentar & Dokumentasi

- Komentar dalam **Bahasa Inggris**.
- Doc comment (`///`) wajib untuk public API di `core/` dan `shared/`.
- Komentar menjelaskan **kenapa**, bukan apa — kode yang jelas tidak butuh komentar "apa".
- `TODO` harus disertai konteks: `// TODO(nama): apa dan kenapa`.

## 10. Logging

- Semua logging lewat **`AppLogger`** dari `core/logger` (lihat `ARCHITECTURE.md` section 11). Dilarang `print`, `debugPrint`, atau memanggil package logger langsung.
- Pilih level yang tepat: `debug` untuk pengembangan, `info` untuk peristiwa penting, `warning` untuk kondisi aneh yang masih tertangani, `error` untuk kegagalan.
- Jangan log data sensitif — lihat `SECURITY.md`.

## 11. Code Generation

- File generated tidak boleh diedit manual: `*.g.dart`, `*.config.dart` (injectable), file drift, asset gen, dan output `gen-l10n`.
- Setelah mengubah sumber codegen (anotasi injectable, tabel drift, assets, file ARB), jalankan `make gen`, atau secara manual:

```bash
fvm flutter gen-l10n
fvm dart run build_runner build --delete-conflicting-outputs
```

## 12. Checklist Sebelum Selesai

- [ ] `fvm dart format .` sudah dijalankan
- [ ] `fvm flutter analyze` bersih tanpa warning baru
- [ ] Tidak ada `!`, `dynamic`, `print`/`debugPrint` baru
- [ ] Tidak ada relative import
- [ ] Penamaan sesuai tabel section 2
- [ ] Entity & state pakai Equatable
- [ ] Error lewat pola `Failure`/`Result` (bukan exception mentah ke UI)
- [ ] Komentar dalam Bahasa Inggris
- [ ] Codegen di-regenerate bila perlu
