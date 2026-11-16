# DESIGN_SYSTEM.md

Sumber kebenaran untuk tampilan UI. Semua widget wajib memakai token dan komponen di sini. Agent dilarang menulis nilai visual mentah di kode fitur.

## 1. Prinsip

- **Konsisten**: satu token/komponen untuk satu kebutuhan. Jangan bikin varian baru kalau yang ada cukup.
- **Token dulu**: warna, ukuran, dan gaya selalu lewat token, tidak pernah literal.
- **Aksesibel**: kontras cukup, tap target cukup, mendukung text scaling.
- **Mobile-first**: target utama phone (lihat `PRD.md` section 3); layout tetap tidak boleh rusak di layar lain.

## 2. Lokasi Kode

```
lib/core/design_system/
├── tokens/
│   ├── app_colors.dart
│   ├── app_typography.dart
│   ├── app_spacing.dart
│   ├── app_radius.dart
│   ├── app_elevation.dart
│   └── app_motion.dart
├── theme/
│   ├── app_theme.dart          # ThemeData light + dark
│   └── app_theme_extension.dart
├── components/                 # widget reusable (AppButton, AppTextField, dst.)
└── design_system.dart          # barrel export
```

Akses token **selalu lewat `context`** (mis. `context.colors`, `context.textStyles`, `context.spacing`), diimplementasikan via `ThemeExtension`. Class konstanta (`AppColors`, `AppSpacing`, dst.) hanya dipakai internal oleh token/extension — jangan di-import langsung di widget.

## 3. Design Tokens

> Nilai di bawah adalah contoh. Ganti dengan nilai final brand/desain.

### 3.1 Warna

Gunakan **semantic color**, bukan nama warna mentah.

| Token | Light | Dark | Dipakai untuk |
|---|---|---|---|
| `primary` | `#<hex>` | `#<hex>` | Aksi utama, elemen aktif |
| `onPrimary` | `#<hex>` | `#<hex>` | Teks/ikon di atas primary |
| `secondary` | `#<hex>` | `#<hex>` | Aksi sekunder, aksen |
| `background` | `#<hex>` | `#<hex>` | Latar halaman |
| `surface` | `#<hex>` | `#<hex>` | Card, sheet, dialog |
| `onSurface` | `#<hex>` | `#<hex>` | Teks utama |
| `onSurfaceVariant` | `#<hex>` | `#<hex>` | Teks sekunder, hint |
| `outline` | `#<hex>` | `#<hex>` | Border, divider |
| `error` | `#<hex>` | `#<hex>` | Error, destruktif |
| `success` | `#<hex>` | `#<hex>` | Status berhasil |
| `warning` | `#<hex>` | `#<hex>` | Peringatan |

Aturan:
- Jangan pakai `Colors.xxx` atau `Color(0xFF...)` di luar `app_colors.dart`.
- Kontras teks terhadap latar minimal **4.5:1** (3:1 untuk teks besar).

### 3.2 Tipografi

Font: `<nama font>` (fallback: system). Nama token mengikuti `TextTheme` Material 3.

| Token | Ukuran / Line height | Weight | Dipakai untuk |
|---|---|---|---|
| `displayLarge` | 32 / 40 | Bold | Hero, angka besar |
| `headlineMedium` | 24 / 32 | SemiBold | Judul halaman |
| `titleMedium` | 18 / 24 | SemiBold | Judul section/card |
| `bodyLarge` | 16 / 24 | Regular | Teks utama |
| `bodyMedium` | 14 / 20 | Regular | Teks sekunder |
| `labelLarge` | 14 / 20 | Medium | Tombol, label |
| `bodySmall` | 12 / 16 | Regular | Catatan, metadata |

Aturan:
- Jangan set `fontSize`/`fontWeight` manual. Ambil dari token, lalu `copyWith` hanya untuk warna.
- Layout harus tahan text scale sampai **1.3x** tanpa overflow.

### 3.3 Spacing

Skala berbasis 4:

| Token | Nilai |
|---|---|
| `xs` | 4 |
| `sm` | 8 |
| `md` | 16 |
| `lg` | 24 |
| `xl` | 32 |
| `xxl` | 48 |

Jangan pakai angka literal di `EdgeInsets`/`SizedBox`. Pakai token spacing lewat context (mis. `context.spacing.md`).

### 3.4 Radius, Elevation, Motion

- **Radius**: `sm` 8, `md` 12, `lg` 16, `full` (pill/lingkaran).
- **Elevation**: `none`, `low`, `medium`, `high`. Di dark mode utamakan perbedaan surface daripada shadow.
- **Motion**: durasi `fast` 150ms, `normal` 250ms, `slow` 400ms. Curve default `easeInOut`. Hormati pengaturan "reduce motion" OS.

### 3.5 Responsive dan Adaptive Layout

**Cakupan default: phone-first portrait.** Breakpoint `medium`/`expanded`, landscape, dan foldable hanya wajib kalau `PRD.md` section 3 mencentangnya. Yang selalu wajib di semua project: tidak overflow di layar kecil, aman dari keyboard, tahan text scale 1.3x.

Layout ditentukan oleh **ruang yang tersedia**, bukan jenis perangkat.

**Breakpoint**

| Nama | Lebar | Perangkat umum |
|---|---|---|
| `compact` | < 600 | Phone portrait |
| `medium` | 600 sampai 839 | Phone landscape, tablet kecil, foldable |
| `expanded` | ≥ 840 | Tablet |

Akses lewat helper di `core/design_system` (mis. `context.screenSize` atau `AppResponsiveBuilder`), bukan `MediaQuery` yang tersebar di widget.

**Prinsip**
- Pakai `LayoutBuilder` / constraints untuk keputusan layout. Jangan cek `Platform.isIOS` atau nama device untuk urusan ukuran.
- Jangan hardcode `width`/`height` untuk layout utama. Pakai `Flexible`, `Expanded`, `Wrap`, `AspectRatio`, dan `ConstrainedBox(maxWidth: ...)`.
- Gambar dan media memakai aspect ratio, bukan tinggi tetap.
- Semua konten yang bisa panjang harus scrollable agar tidak overflow (keyboard muncul, text scale besar).

**Pola adaptif per breakpoint** (hanya kalau PRD mencentang tablet/landscape)

| Aspek | compact | medium | expanded |
|---|---|---|---|
| Navigasi utama | `NavigationBar` (bawah) | `NavigationRail` | Rail diperluas / drawer permanen |
| Daftar konten | List 1 kolom | Grid 2 kolom | Grid 3+ kolom atau master-detail |
| Detail item | Halaman baru | Halaman baru / panel | Panel di samping daftar (master-detail) |
| Modal | Bottom sheet | Dialog | Dialog dengan lebar tetap |
| Padding halaman | `md` (16) | `lg` (24) | `xl` (32) |

**Insets dan area aman**
- Bungkus halaman dengan `SafeArea` (notch, status bar, gesture bar).
- Perhatikan `viewInsets` (keyboard): input tidak boleh tertutup, gunakan scroll view.

**Pengujian**
- Widget test minimal di 360x640 (kecil) dan 411x891 (phone) — ukuran tablet/lebar hanya kalau PRD mencentangnya.
- Uji juga dengan text scale 1.3x dan dark mode.

## 4. Theming

- Satu `ThemeData` untuk light dan satu untuk dark, dibuat di `app_theme.dart` (Material 3).
- Token tambahan di luar `ThemeData` lewat `ThemeExtension`.
- Mode tema mengikuti sistem dan bisa dioverride user. Semua komponen wajib dicek di kedua mode.

## 5. Komponen

Semua komponen berada di `components/` dan diawali prefix `App`.

| Komponen | Varian | Catatan |
|---|---|---|
| `AppButton` | primary, secondary, outline, text, destructive | Ada state loading dan disabled |
| `AppTextField` | default, password, search | Label, helper, dan error terintegrasi |
| `AppCard` | default, outlined | Padding dan radius dari token |
| `AppAppBar` | default, transparent | |
| `AppDialog` / `AppBottomSheet` | confirm, info | |
| `AppSnackbar` / `AppToast` | success, error, info | |
| `AppLoading` | inline, fullscreen, skeleton | |
| `AppEmptyState` / `AppErrorState` | | Selalu sediakan aksi retry |
| `AppAvatar`, `AppBadge`, `AppChip` | | |

Aturan membuat komponen:
- Cek dulu apakah komponen yang ada bisa dipakai atau diberi varian baru.
- Komponen hanya menerima data dan callback. Tanpa logic bisnis dan tanpa akses ke state fitur.
- Semua state tercakup: normal, pressed, focused, disabled, loading, error.
- Ada `const` constructor dan parameter yang terdokumentasi.
- Tambahkan widget test untuk komponen baru.

## 6. Ikon dan Aset

- Ikon: `<set ikon, mis. Material Symbols / custom icon font>`. Ukuran standar 16, 20, 24.
- Aset di `assets/` dan diakses lewat class generated (`flutter_gen_runner`), bukan string path manual.
- Prefer SVG untuk ilustrasi dan ikon custom.
- Ikon app & splash per flavor di-generate dengan `flutter_launcher_icons` dan `flutter_native_splash` (config per flavor) — lihat `ARCHITECTURE.md` section 6.1.

## 7. Aksesibilitas

- Tap target minimal **48x48 dp**.
- Semua gambar/ikon bermakna punya `Semantics` / `semanticLabel`.
- Jangan mengandalkan warna saja untuk menyampaikan status (tambahkan ikon atau teks).
- Urutan fokus dan navigasi screen reader masuk akal.
- Teks tidak dipotong saat text scale membesar.

## 8. Aturan untuk Agent

- Sebelum membuat UI, baca bagian token dan komponen di dokumen ini.
- Pakai komponen `App*` yang sudah ada. Jangan pakai widget Material mentah kalau padanannya ada.
- Dilarang: warna literal, `TextStyle` manual, angka spacing/radius literal, string hardcode.
- Butuh token atau komponen baru? **Tanya dulu**, jelaskan alasannya, lalu tambahkan di design system, bukan di kode fitur.
- Setelah selesai, cek tampilan di light dan dark mode, di layar kecil (360x640), dan text scale 1.3x.

## 9. Checklist UI Sebelum Selesai

- [ ] Hanya memakai token dan komponen design system
- [ ] Light dan dark mode sudah dicek
- [ ] Tidak overflow di layar kecil (360x640) dan text scale 1.3x
- [ ] Tidak ada width/height hardcoded untuk layout utama; keyboard tidak menutup input
- [ ] (Kalau PRD mencentang) layout adaptif di medium/expanded dan landscape
- [ ] Tap target dan semantics sesuai
- [ ] State loading, empty, dan error tersedia
- [ ] Widget test ditambahkan untuk komponen baru
