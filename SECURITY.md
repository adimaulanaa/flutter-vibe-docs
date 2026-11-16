# SECURITY.md

Standar keamanan untuk project ini. Aturan di sini **wajib** kecuali ditandai opsional. Kalau ada kebutuhan yang bertentangan, diskusikan dulu dan dokumentasikan keputusannya.

## 1. Prinsip

- Keamanan adalah default, bukan fitur tambahan.
- Data sensitif tidak pernah: di-hardcode, di-commit, di-log, atau disimpan tanpa enkripsi.
- Asumsikan semua input dari luar (API, form, deep link, file) berbahaya sampai divalidasi.
- Asumsikan device bisa di-root/jailbreak — jangan simpan rahasia penting hanya di client.

## 2. Secret & Environment Config

- Config penting (baseUrl, API key) lewat `.env` per flavor — lihat `ARCHITECTURE.md` section 6.
- `.env.staging` dan `.env.prod` **dilarang di-commit**. Yang di-commit hanya `.env.example` berisi daftar key tanpa value.
- **Batasan dart-define**: nilai di `.env` ikut ter-compile ke binary dan bisa diekstrak dari APK/IPA. Jangan taruh secret server (API key privat, credential database) di `.env` — hanya konfigurasi client (baseUrl, public API key, flag). Kalau butuh rahasia sungguhan, ambil dari backend, jangan di-bundle.
- Keystore, sertifikat, dan credential CI/CD juga dilarang masuk repo.
- `.gitignore` wajib mencakup: `.env*` (kecuali `.env.example`), `*.keystore`, `*.jks`, `key.properties`, `GoogleService-Info.plist` / `google-services.json` bila berisi credential produksi.
- Kode membaca config hanya lewat `AppConfig` — tidak ada `String.fromEnvironment` atau baca env langsung tersebar di kode.

### Rotasi Secret

Kalau secret terlanjur bocor (ter-commit, ter-log, atau dicurigai):

1. Anggap secret sudah publik — langsung revoke/regenerate di sisi provider.
2. Ganti dengan secret baru, update semua environment.
3. Hapus dari history git (`git filter-repo` atau buat repo baru jika perlu) — menghapus di commit terakhir saja **tidak cukup**.
4. Catat kejadiannya: apa yang bocor, kapan, dan langkah perbaikannya.

## 3. Autentikasi & Token

Pola default: **JWT access token + refresh token**. Ini pola dasar starter kit — kalau project memakai mekanisme lain (session cookie, OAuth provider, dll), sesuaikan section ini dan dokumentasikan di project tersebut.

- Access token disimpan di `flutter_secure_storage` (Keychain di iOS, Keystore terenkripsi di Android). **Dilarang** di `shared_preferences` atau memory global yang longgar.
- Refresh token juga di secure storage, terpisah dari access token.
- Refresh token otomatis lewat dio **interceptor** di `core/network`:
  - Response 401 → coba refresh token sekali → ulangi request asli.
  - Request paralel saat refresh harus di-queue, bukan refresh berkali-kali.
  - Refresh gagal → hapus token, kembalikan user ke halaman login (redirect lewat router, `ARCHITECTURE.md` section 9).
- Token tidak pernah muncul di log, crash report, atau analytics.
- **Keychain iOS tetap ada setelah uninstall**: saat first launch setelah install (flag di `shared_preferences`), hapus dulu seluruh isi secure storage — jangan andalkan data sisa instalasi lama.
- Logout: hapus token + clear semua data lokal user (lihat section 6).

## 4. Network

- **HTTPS only.** Tidak ada HTTP cleartext. Di Android pastikan `usesCleartextTraffic=false`, di iOS ATS aktif (jangan pakai `NSAllowsArbitraryLoads`).
- Timeout wajib diset di dio (connect, receive) — jangan biarkan request menggantung tanpa batas.
- Error dari server tidak boleh menampilkan detail teknis (stack trace, query, info server) ke UI — tampilkan pesan generik via l10n, detail teknis cukup di `debugMessage` untuk log internal.
- **Certificate pinning** (via dio): opsional saat development, direkomendasikan aktif di flavor `prod` terutama untuk app yang menangani data sensitif/finansial. Keputusan dicatat di `PRD.md` section 14. Kalau diaktifkan, dokumentasikan sertifikat yang di-pin dan jadwal rotasinya.

## 5. Build & Distribusi

- Build release **wajib** obfuscation:

  ```bash
  fvm flutter build appbundle --release --flavor prod \
    --dart-define-from-file=.env.prod \
    --obfuscate --split-debug-info=build/symbols
  ```

- Simpan folder `build/symbols` aman (jangan di-commit) — dibutuhkan untuk membaca stack trace crash production.
- Debug logging dan flag development wajib mati di build `prod`.
- **Android backup**: set `android:allowBackup="false"` di `AndroidManifest.xml` supaya data app tidak ikut backup otomatis ke cloud.
- **Root/jailbreak detection**: opsional per project (keputusan blokir vs peringatan dicatat di `PRD.md` section 14).
- **Screen capture protection** (mencegah screenshot di halaman sensitif): opsional per project, terapkan per halaman yang membutuhkan (dicatat di `PRD.md`).

## 6. Data Lokal & Privacy

- Data sensitif (token, credential) hanya di `flutter_secure_storage`.
- Data non-sensitif tapi pribadi (cache profil, riwayat) boleh di drift/shared_preferences — prinsipnya: simpan sesedikit mungkin, selama dibutuhkan saja.
- **Saat logout wajib**: hapus token dari secure storage, clear cache database/prefs milik user, dan reset state Bloc/DI yang menyimpan data user. Tidak ada data user sebelumnya yang tertinggal untuk user berikutnya di device yang sama.
- Database drift tidak boleh berisi data sensitif tanpa enkripsi — kalau perlu simpan data sensitif di database, bahas dulu (pertimbangkan SQLCipher).

## 7. Validasi Input

- Semua input user (form) divalidasi di sisi client sebelum dikirim — dan selalu diasumsikan akan divalidasi ulang oleh server.
- Data dari API divalidasi lewat DTO: field wajib, tipe benar, nilai masuk akal. Jangan percaya bentuk response mentah.
- Deep link: validasi semua parameter di router (bukan di page) sebelum dipakai navigasi atau query. Jangan pernah mengeksekusi aksi sensitif hanya karena deep link memintanya.

## 8. Permission

- Hanya permission yang tercantum di `PRD.md` section 10 yang boleh ditambahkan — beserta teks alasan yang ditampilkan ke user (NSCameraUsageDescription dsb / manifest Android).
- Minta permission secara **runtime, tepat saat fitur dipakai** — jangan saat app start.
- Tangani penolakan dengan baik: tampilkan penjelasan + cara mengaktifkan lewat Settings, jangan loop meminta.

## 9. Logging & Monitoring

- Dilarang log: token, password, PIN, data PII (email, nomor telepon, alamat), isi request/response yang memuat data tersebut.
- Gunakan `AppLogger` dari `core/logger`; di build release, log sensitif otomatis tidak tercetak (penyaringan di `AppLogger`, bukan di kode fitur).
- Crash reporting (kalau ada — diputuskan di `PRD.md` section 12) dikonfigurasi untuk menyaring data sensitif.

## 10. Dependency

- Tambah package baru harus konfirmasi dulu (aturan `AGENTS.md`) — sekalian cek: maintainer aktif, lisensi cocok, tidak ada isu keamanan terbuka.
- Jalankan audit dependency berkala (`fvm flutter pub outdated`, cek advisory di pub.dev).
- Pin versi dependency penting (networking, storage, security) supaya update tidak diam-diam mengubah perilaku.

## 11. Checklist Sebelum Rilis

- [ ] Tidak ada secret/key di repo (cek dengan `grep`/scan)
- [ ] `.env` hanya berisi konfigurasi client, bukan secret server
- [ ] `.gitignore` mencakup semua file di section 2
- [ ] Token hanya di secure storage; flow refresh token berfungsi; keychain lama dibersihkan saat first launch
- [ ] HTTPS only, tanpa pengecualian; `allowBackup=false` di Android
- [ ] Build release dengan obfuscation; symbols disimpan aman
- [ ] Logout membersihkan semua data lokal user
- [ ] Log tidak memuat data sensitif
- [ ] Error UI tidak membocorkan detail teknis
- [ ] Permission sesuai daftar di PRD, diminta runtime dengan teks alasan
- [ ] (Jika aktif) cert pinning teruji di staging sebelum prod
