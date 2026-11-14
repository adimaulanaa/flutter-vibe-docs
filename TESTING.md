# TESTING.md

Strategi dan aturan testing untuk project ini. Semua command test terdokumentasi di **Makefile** — lihat `make help` atau section 6 di dokumen ini.

## 1. Prinsip

- Test adalah bagian dari Definition of Done, bukan pelengkap.
- Test logic, bukan implementasi. Test perilaku yang terlihat dari luar.
- Test harus deterministik: tanpa network sungguhan, tanpa waktu sungguhan (`DateTime.now()` di-inject atau di-mock), tanpa storage sungguhan.
- Satu test menguji satu hal. Nama test menjelaskan skenario dan ekspektasi.

## 2. Tools

| Kebutuhan | Package |
|---|---|
| Unit & widget test | `flutter_test` (bawaan) |
| Mocking | `mocktail` (tanpa codegen) |
| Bloc test | `bloc_test` |
| Integration test | `integration_test` (bawaan) |

## 3. Struktur Folder

`test/` mirror `lib/`. Test integration terpisah di `integration_test/`.

```
test/
├── core/                    # test untuk utils, error mapper, dsb
├── features/
│   └── <feature>/
│       ├── data/            # datasource, repository impl
│       ├── domain/          # usecase
│       └── presentation/    # bloc, widget
└── helpers/                 # mock, fixture, pump helper
integration_test/
└── <feature>_test.dart      # alur end-to-end
```

Helper bersama (mock umum, fixture JSON, `pumpApp` untuk widget test) taruh di `test/helpers/`.

## 4. Apa yang Wajib Dites

### Unit test

| Layer | Yang dites | Mock |
|---|---|---|
| domain | Setiap usecase: hasil sukses, hasil gagal (Failure), edge case | repository |
| data | Repository impl: konversi DTO→entity, mapping exception→Failure, pemilihan datasource | datasource |
| data | Datasource: request benar, parsing response, error dilempar | dio / database |
| core | Utils, formatter, validator | — |

### Bloc test

Setiap Bloc wajib dites dengan `bloc_test`:

- State awal benar.
- Setiap event menghasilkan urutan state yang benar (loading → success / failure).
- Usecase dipanggil dengan parameter yang benar (`verify`).
- Tidak ada state baru setelah Bloc di-close.

### Widget test

- Komponen design system (`App*`): semua varian dan state (normal, disabled, loading, error).
- Alur UI penting per fitur (misal form login: validasi, submit, tampilkan error).
- Ukuran layar: 360x640 (kecil) dan 411x891 (phone) minimal; tambah tablet/lebar untuk halaman utama.
- Mode: light dan dark untuk komponen design system.
- Text scale 1.3x tidak overflow.

### Integration test

- Alur kritikal end-to-end yang tercantum di `PRD.md` (misal: buka app → login → lihat home).
- Berjalan dengan flavor `staging` di device/emulator sungguhan.
- Backend: pakai environment staging atau mock server — didokumentasikan per test.

## 5. Aturan Menulis Test

- Dilarang memanggil API sungguhan di unit/widget test. Selalu mock.
- Fixture JSON taruh di `test/fixtures/` atau sebagai konstanta di `test/helpers/`.
- Nama test: `should <ekspektasi> when <kondisi>` atau deskripsi skenario singkat dalam Bahasa Inggris.
- Setup berulang masuk ke `setUp`, bukan di-copy per test.
- Jangan mengetes kode generated (injectable config, drift, asset gen).
- Jangan mengetes library pihak ketiga (dio, drift) — percaya mereka sudah dites.

## 6. Menjalankan Test (via Makefile)

Semua command dibungkus Makefile supaya mudah diingat dan dijalankan:

| Command | Fungsi |
|---|---|
| `make test` | Jalankan semua unit + widget test |
| `make test-feature f=<feature>` | Test satu fitur saja, misal `make test-feature f=auth` |
| `make test-file f=<path>` | Test satu file, misal `make test-file f=test/features/auth/domain/login_test.dart` |
| `make test-coverage` | Semua test + laporan coverage |
| `make test-integration` | Integration test di device yang sedang terhubung (flavor staging) |

Command asli di baliknya (kalau Makefile belum ada):

```bash
fvm flutter test                                          # semua
fvm flutter test test/features/<feature>/                 # per fitur
fvm flutter test <path_ke_file>                           # per file
fvm flutter test --coverage                               # coverage
fvm flutter test integration_test/ --flavor staging -t lib/main_staging.dart -d <device_id>   # integration
```

## 7. Menambah Test untuk Fitur Baru

1. Buat struktur `test/features/<feature>/{data,domain,presentation}` mirror dengan `lib/`.
2. Tulis test usecase dulu (paling bernilai), lalu repository, lalu bloc.
3. Widget test untuk halaman utama fitur.
4. Mock dependency dengan `mocktail`; registrasi mock umum di `test/helpers/`.
5. Jalankan `make test-feature f=<feature>` sampai hijau, lalu `make test` untuk memastikan tidak ada yang rusak.

## 8. Checklist

- [ ] Usecase, repository, dan bloc baru punya test
- [ ] Komponen `App*` baru punya widget test
- [ ] Tidak ada network/storage/waktu sungguhan di unit test
- [ ] `make test` hijau
- [ ] Alur kritikal PRD punya integration test (atau tercatat kenapa belum)
