# TESTING.md

Strategi dan aturan testing untuk project ini. Semua command test terdokumentasi di **Makefile** — lihat `README.md` (tabel Common Commands) atau `make help`.

## 1. Prinsip

- Test adalah bagian dari Definition of Done, bukan pelengkap.
- Test logic, bukan implementasi. Test perilaku yang terlihat dari luar.
- Test harus deterministik: tanpa network sungguhan, tanpa waktu sungguhan (`DateTime.now()` di-inject atau di-mock). Storage eksternal selalu di-mock/in-memory — pengecualian: drift dites dengan `NativeDatabase.memory()` (in-memory, bukan file).
- Satu test menguji satu hal. Nama test menjelaskan skenario dan ekspektasi.
- Acuan test adalah **acceptance criteria** di `PRD.md` — setiap AC punya test yang membuktikannya.

## 2. Tools

| Kebutuhan | Package |
|---|---|
| Unit & widget test | `flutter_test` (bawaan) |
| Mocking | `mocktail` (tanpa codegen) |
| Bloc test | `bloc_test` |
| Integration test | `integration_test` (bawaan) |

Tidak ada golden test. Tidak ada target coverage minimum — coverage hanya alat bantu melihat bagian yang belum dites.

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
├── fixtures/                # fixture JSON (boleh juga konstanta di helpers/)
└── helpers/                 # mock umum, pump helper
integration_test/
└── <feature>_test.dart      # alur end-to-end
```

Helper bersama (mock umum, fixture JSON, `pumpApp` untuk widget test) taruh di `test/helpers/`.

## 4. Apa yang Wajib Dites

### Unit test

| Layer | Yang dites | Mock |
|---|---|---|
| domain | Setiap usecase: sukses → `Success`, gagal → `Err` dengan `Failure` yang benar, edge case | repository |
| data | Repository impl: konversi DTO→entity, exception→`Failure` yang benar, pemilihan datasource | datasource |
| data | Datasource: request benar, parsing response, error dilempar | dio / storage |
| core | Utils, formatter, validator, error mapper | — |

### Bloc test

Setiap Bloc wajib dites dengan `bloc_test`:

- State awal benar.
- Setiap event menghasilkan urutan state yang benar (loading → success / error).
- Usecase dipanggil dengan parameter yang benar (`verify`).
- Transformer concurrency teruji: event ganda saat loading tidak memanggil usecase dua kali (untuk `droppable()`).

### Widget test

- Komponen design system (`App*`): semua varian dan state (normal, disabled, loading, error).
- Alur UI penting per fitur (misal form login: validasi, submit, tampilkan error).
- Ukuran layar: 360x640 (kecil) dan 411x891 (phone). Ukuran tablet/lebar hanya wajib kalau `PRD.md` section 3 mencentang tablet.
- Mode: light dan dark untuk komponen design system.
- Text scale 1.3x tidak overflow.

### Integration test

- Alur bertanda **[kritikal]** di `PRD.md` section 6 (misal: buka app → login → lihat home).
- Berjalan dengan flavor `staging` di device/emulator sungguhan.
- Backend: pakai environment staging atau mock server — didokumentasikan per test.

## 5. Aturan Menulis Test

- Dilarang memanggil API sungguhan di unit/widget test. Selalu mock.
- Fixture JSON taruh di `test/fixtures/` atau sebagai konstanta di `test/helpers/`.
- Nama test: `should <ekspektasi> when <kondisi>` atau deskripsi skenario singkat dalam Bahasa Inggris.
- Setup berulang masuk ke `setUp`, bukan di-copy per test.
- Jangan mengetes kode generated (injectable config, drift, asset gen, output l10n).
- Jangan mengetes library pihak ketiga (dio, drift) — percaya mereka sudah dites.

## 6. Menjalankan Test (via Makefile)

Semua command dibungkus Makefile supaya mudah diingat dan dijalankan:

| Command | Fungsi |
|---|---|
| `make test` | Jalankan semua unit + widget test |
| `make test-feature f=<feature>` | Test satu fitur saja, misal `make test-feature f=auth` |
| `make test-file f=<path>` | Test satu file, misal `make test-file f=test/features/auth/domain/login_with_email_test.dart` |
| `make test-coverage` | Semua test + laporan coverage |
| `make test-integration` | Integration test di device yang sedang terhubung (flavor staging) |

Command asli di baliknya (kalau Makefile belum ada):

```bash
fvm flutter test                                          # semua
fvm flutter test test/features/<feature>/                 # per fitur
fvm flutter test <path_ke_file>                           # per file
fvm flutter test --coverage                               # coverage
# integration (catatan: `flutter test` TIDAK punya flag -t; pilih target lewat path):
fvm flutter test integration_test/<feature>_test.dart \
  --flavor staging --dart-define-from-file=.env.staging -d <device_id>
```

## 7. Menambah Test untuk Fitur Baru

1. Buat struktur `test/features/<feature>/{data,domain,presentation}` mirror dengan `lib/`.
2. Tulis test usecase dulu (paling bernilai), lalu repository, lalu bloc.
3. Widget test untuk halaman utama fitur.
4. Mock dependency dengan `mocktail`; registrasi mock umum di `test/helpers/`.
5. Jalankan `make test-feature f=<feature>` sampai hijau, lalu `make test` untuk memastikan tidak ada yang rusak.

## 8. Checklist

- [ ] Usecase, repository, dan bloc baru punya test
- [ ] Setiap acceptance criteria terkait punya test
- [ ] Komponen `App*` baru punya widget test
- [ ] Tidak ada network/waktu sungguhan di unit test; storage di-mock atau in-memory
- [ ] `make test` hijau
- [ ] Alur [kritikal] PRD punya integration test (atau tercatat kenapa belum)
