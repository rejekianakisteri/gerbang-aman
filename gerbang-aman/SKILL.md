---
name: gerbang-aman
description: Audit keamanan dan penetration test pra-deploy untuk aplikasi web, selaras OWASP Top 10:2025 dan OWASP ASVS v5. Gunakan saat user meminta security audit, pentest, vulnerability assessment, code security review, "audit keamanan", "pentest", "cek OWASP", "review celah keamanan", atau mengecek aplikasi sebelum deploy / sebelum production. Mencakup PHP (native, Laravel, CodeIgniter), Node.js (Express), dan pemeriksaan generik untuk stack lain — SAST berdasarkan kategori OWASP, pemindaian dependensi & supply chain, audit konfigurasi, DAST otomatis (Nuclei, Nikto, OWASP ZAP, sqlmap), dan tes manual otorisasi/logika (IDOR, eskalasi privilege, SSRF, file upload, sesi/JWT), diakhiri laporan audit Bahasa Indonesia, findings.json machine-readable, dan putusan deploy GO / NO-GO.
---

# Audit Keamanan & Pentest Pra-Deploy (Aplikasi Web)

### Standar: OWASP Top 10:2025 & OWASP ASVS v5.0.0

Penilaian keamanan lengkap sebelum aplikasi web naik production: SAST (audit kode berdasarkan kategori OWASP), peninjauan supply chain, audit konfigurasi, pemindaian dinamis otomatis, dan pentest logika manual — ditutup dengan laporan dan putusan deploy. Seksi per-stack mencakup PHP (native / Laravel / CodeIgniter) dan Node.js (Express); alur kerjanya sendiri berlaku untuk semua stack.

## Aturan main (rules of engagement) — baca dulu, jangan pernah dilewati

1. **Hanya uji sistem yang dimiliki user atau yang ada izin tertulisnya.** Jika kepemilikan/izin tidak jelas, berhenti dan tanya. Jangan pernah memindai situs pihak ketiga atau situs pemerintah "buat ngecek".
2. **Staging dulu.** Tes intrusif (eksploitasi sqlmap, bypass file upload, brute force) HANYA boleh dijalankan di salinan staging dengan database dummy — tidak pernah di data production.
3. **Production = non-destruktif saja**: cek versi/konfigurasi, cek eksposur file, cek header, Nuclei dengan template aman. Tanpa flag eksploitasi, tanpa `--dump`, tanpa fuzzing di production.
4. **Backup sebelum pengujian** (database + file). Catat lokasi backup di laporan.
5. **Bersih-bersih setelah selesai**: hapus semua file tes yang diunggah, semua akun tes yang dibuat, semua artefak payload — dan daftarkan pembersihannya di laporan.
6. **Berhenti dan lapor segera saat menemukan temuan Critical**, sebelum melanjutkan.

> **Versi 1.1** — menambahkan disiplin status temuan & kalibrasi severity, `findings.json` machine-readable, dan daftar anti-pattern, diadaptasi dari praktik terbaik skill audit keamanan open-source (termasuk security-audit dari Cloudflare). Kerangka OWASP/ASVS dan seluruh fase tidak berubah.

## Disiplin temuan — status, bukti, dan severity

Temuan bukanlah "best practice yang belum diterapkan". Temuan adalah **batas kepercayaan (trust boundary) yang nyata dilanggar dengan hasil konkret**. Tanpa pelanggaran boundary yang bisa ditunjukkan, sesuatu bukan temuan terkonfirmasi — seberapa pun tidak enaknya kelihatannya.

Setiap kandidat temuan wajib berstatus tepat satu:

- **`confirmed`** — terbukti: ada bukti yang bisa diverifikasi ulang (PoC request/response, lokasi kode + jalur eksploitasinya, atau hasil scanner yang sudah diverifikasi manual). **Hanya status ini yang boleh memiliki severity** dan hanya status ini yang memengaruhi putusan GO / NO-GO.
- **`needs_validation`** — hipotesis kuat yang berakar pada kode/konfigurasi, tetapi fakta penentunya berada di luar jangkauan pemeriksaan (perilaku reverse proxy/CDN, setting panel/hosting, kondisi production yang tidak boleh diuji intrusif). Wajib mencantumkan: fakta yang kurang + rencana validasi yang aman oleh pemilik aplikasi. **Tanpa severity** — jangan menakuti dengan label yang belum terbukti.
- **`rejected`** — sudah diperiksa dan ternyata bukan celah (kontrolnya ada di lapisan lain, dampak hanya ke akun sendiri, atau memang perilaku yang dimaksudkan). Catat alasannya agar tidak dilaporkan ulang pada audit berikutnya.

**Syarat `confirmed` (uji boundary)** — sebutkan keenamnya: (1) aktor/principal dengan trust lebih rendah, (2) input atau aksi yang diterima sistem, (3) kontrol yang seharusnya menahan, (4) boundary yang diterobos, (5) resource/pihak yang terdampak, (6) hasil konkret yang teramati. Kurang satu → turunkan ke `needs_validation`.

**Kalibrasi severity** (jangkar untuk aplikasi web — severity tidak boleh melebihi dampak yang didemonstrasikan):
- **Critical** — aktor tanpa login memperoleh eksekusi kode (RCE), dump database, ambil alih akun siapa pun, atau bypass autentikasi total.
- **High** — kontrol keamanan eksplisit runtuh dengan konsekuensi nyata: IDOR baca/tulis data pengguna lain, eskalasi ke admin, stored XSS yang mengenai pengguna lain, SQL injection terkonfirmasi.
- **Medium** — pelanggaran nyata dengan radius terbatas: IDOR satu record non-sensitif, CSRF pada aksi berdampak rendah, rate limit hilang di endpoint non-kritis.
- **Low** — kebocoran informasi non-rahasia (versi software, path internal), atau dampak yang butuh usaha besar untuk hasil kecil.
- **Informational** — terkonfirmasi tetapi dampak minimal; berguna sebagai catatan atau bahan rantai temuan lain.

Pembeda High vs Medium: apakah hasil yang didemonstrasikan **sepenuhnya menumbangkan** kontrol untuk aksi berkonsekuensi nyata, atau hanya **melemahkannya**? Jika dampak konkretnya tidak bisa disebutkan, severity-nya lebih rendah dari yang terasa.

**Perbaikan terkecil**: untuk setiap `confirmed`, rekomendasikan perubahan sumber terkecil yang menegakkan invariant pada titik keputusan terakhir yang tepercaya (contoh diff ada di template Fase 6) — bukan nasihat hardening umum.

## Fase 0 — Ruang lingkup & level ASVS

Kumpulkan dan nyatakan sebelum menyentuh apa pun:

- Nama aplikasi, stack, versi runtime, web server / reverse proxy (Apache/nginx), database (MySQL/MariaDB/PostgreSQL/SQLite/MongoDB).
- URL staging (diutamakan) dan/atau URL production; path codebase; lokasi backup.
- Role pengguna dan **akun tes: dua akun dengan role berbeda + dua akun dengan role yang sama** (diperlukan untuk tes otorisasi di Fase 5).
- Konfirmasi: target milik sendiri/berizin? Backup sudah ada?

Pilih level verifikasi ASVS berdasarkan risiko:
- **Level 1 (Dasar)** — minimum untuk semua aplikasi web; pengujian otomatis + review kode ringan.
- **Level 2 (Standar)** — aplikasi yang menangani data pribadi (PII), data kritis bisnis, kesehatan, atau keuangan. **Default untuk aplikasi internal/kantor yang berisi data pegawai.** Wajib review kode mendalam.
- **Level 3 (Lanjutan)** — sistem jaminan tinggi (infrastruktur kritis, perbankan inti).

## Fase 1 — Lingkungan, secret & supply chain (A02/A03:2025)

1. **Runtime wajib tertambal.** Bandingkan dengan rilis keamanan vendor:
   - PHP (php.net): lantai php2shell — **8.2.35 / 8.3.36 / 8.4.27 / 8.5.12 atau lebih baru** (`php -v`).
   - Node.js: LTS tertambal terkini menurut rilis keamanan nodejs.org (`node --version`) — Node sering merilis perbaikan High; perlakukan sama seperti PHP.
2. **Audit dependensi** — temuan High/Critical adalah penghambat rilis:
   - PHP: `composer audit` — Node: `npm audit` / `pnpm audit` / `yarn npm audit` — Python: `pip-audit` — Ruby: `bundle audit check --update` — Go: `govulncheck ./...` — Java: OWASP Dependency-Check — .NET: `dotnet list package --vulnerable` — Rust: `cargo audit`.
   - Lockfile (`composer.lock`, `package-lock.json`, `poetry.lock`) ter-commit, hash integritas terverifikasi, tidak ada paket unpinned/abandoned/typosquatting.
3. **Secret**: periksa `.env`, `config/`, dan source dari password DB, JWT secret, token bot, API key yang tertanam hardcode. `.gitignore` wajib mengecualikan `.env`, `*.pem`, `*.key`, `*.sql`, `*.bak`, `storage/`, `vendor/`, `node_modules/`. Cek: `git ls-files | grep -E "^\.env$|\.sql$|\.bak$|\.pem$|phpinfo"` tidak boleh mengembalikan apa pun yang layak kirim.
4. **Debug mati**: `APP_DEBUG=false` / `APP_ENV=production` (PHP), `NODE_ENV=production` (Node) — stack trace tidak boleh sampai ke klien.
5. **Isolasi filesystem**: document root menunjuk ke `public/` / `dist/`, tidak pernah ke root proyek. Aturan web menolak akses langsung ke `.env`, `.git/`, `storage/`, `includes/`, `vendor/`, `node_modules/`, folder log, dan file backup (`*.sql`, `*.zip`, `*.tar.gz`, `*.rar`, `*.xlsx`, `*.bak`, `*.log`).

## Fase 2 — SAST: checklist OWASP Top 10:2025

Review kode terhadap setiap kategori. Blok grep siap-jalan setelah daftar ini adalah titik awal; review setiap hasil temuan secara manual.

**A01 Broken Access Control (termasuk IDOR & SSRF)**
- IDOR: ID dari user dipakai query tanpa verifikasi pemilik/penyewa (`SELECT ... WHERE id = ?` tanpa mengecek record itu milik user yang login).
- Route aksi privileged tanpa guard server-side — menyembunyikan menu BUKAN perlindungan.
- SSRF: mengambil URL yang dikendalikan user (`curl`, `file_get_contents($url)`, `fetch`, `axios`) tanpa memblokir range privat (`127.0.0.1`, `10.0.0.0/8`, `192.168.0.0/16`) dan metadata cloud (`169.254.169.254`).
- Mass assignment: binding model tanpa guard memungkinkan penyerang mengisi `role`, `is_admin`, `status` (PHP `$fillable`/request-all; Node `Object.assign(model, req.body)`).

**A02 Security Misconfiguration** — directory listing menyala; detail error terekspos; kredensial admin default/hasil seed; script setup/migrasi terekspos (`setup.php`, `migrate.php`); security header hilang (lihat Fase 3).

**A03 Software Supply Chain Failures** — temuan Fase 1, ditambah script CDN eksternal tanpa Subresource Integrity (`integrity="sha384-..."`).

**A04 Cryptographic Failures**
- Password di-hash dengan `md5()`/`sha1()`/plaintext, bukan `password_hash()` / bcrypt / argon2id (Node: `bcrypt`/`argon2`, jangan pernah MD5/SHA1 polos atau hash tanpa salt).
- Verifikasi TLS dimatikan (`CURLOPT_SSL_VERIFYPEER => false`, `rejectUnauthorized: false`).
- Token mudah ditebak dari `rand()`/`mt_rand()`/`Math.random()`, bukan `random_bytes()`/`crypto.randomBytes`/`crypto.randomUUID()`.
- Cookie sesi tanpa `HttpOnly`, `Secure`, `SameSite`.

**A05 Injection**
- SQL: query hasil gabungan string, bukan prepared statement / parameter binding — berlaku di PHP (PDO `execute([$val])`) maupun Node.
- Command injection: `exec/shell_exec/system/passthru` atau `child_process.exec` di Node yang dibangun dari input user tanpa escaping/array argumen.
- XSS: rendering tanpa escape — `echo $_GET[...]`, Blade `{!! !!}`, template literal Node ke HTML, `res.send` berisi data user, assignment `innerHTML`.
- NoSQL injection (Node/Mongo): objek request diteruskan ke query — injeksi operator ala `{"$ne": null}`.

**A06 Insecure Design** — bypass alur bisnis (transisi status tanpa prasyarat), anti-otomasi hilang (rate limit/CAPTCHA/honeypot) di form publik, race condition pada transaksi keuangan/status tanpa penguncian DB (`FOR UPDATE`, transaction).

**A07 Authentication Failures** — kebijakan password lemah (<8 karakter, tanpa entropi); endpoint login/OTP/reset tanpa throttling persisten; fiksasi sesi (PHP: tidak memanggil `session_regenerate_id(true)` setelah login/logout; Node: sesi tidak dirotasi); memercayai `X-Forwarded-For`/`Client-IP` yang tidak divalidasi untuk keputusan keamanan.

**A08 Software & Data Integrity Failures** — deserialisasi data tak tepercaya yang tidak aman (`unserialize()`, `pickle.loads()`, `yaml.load()`); upload file memercayai MIME dari klien (`$_FILES['file']['type']`) alih-alih pemeriksaan server-side `finfo` magic bytes + whitelist ekstensi; upload disimpan di path yang bisa dieksekusi web dengan nama yang mudah ditebak; **prototype pollution** (Node): menggabungkan JSON dari user dengan kunci `__proto__` tanpa guard.

**A09 Logging & Alerting Failures** — aksi sensitif tidak tercatat (login, logout, perubahan role, penghapusan, persetujuan); secret ikut tercatat di log (password, token, nomor kartu lengkap); file log bisa diakses lewat URL web.

**A10 Mishandling of Exceptional Conditions** — exception tak tertangani membocorkan string koneksi, path, query; respons error yang tidak konsisten memungkinkan enumerasi user/record; exception kritis ditelan diam-diam tanpa log audit.

**Grep siap-jalan — PHP:**
```bash
grep -rnE "\beval\s*\(|unserialize\s*\(\s*\\\$_(GET|POST|COOKIE|REQUEST)" --include="*.php" .
grep -rnE "\\\$_(GET|POST|REQUEST|COOKIE)\s*\]" --include="*.php" . | grep -iE "query|sql|SELECT|INSERT|UPDATE|DELETE"
grep -rnE "md5\s*\(|sha1\s*\(|rand\s*\(|mt_rand" --include="*.php" .
grep -rnE "echo\s+\\\$_(GET|POST|REQUEST)" --include="*.php" .
grep -rn "{!!" --include="*.blade.php" .
grep -rnE "CURLOPT_SSL_VERIFYPEER.*false|file_get_contents\s*\(\s*\\\$" --include="*.php" .
```

**Grep siap-jalan — Node.js:**
```bash
grep -rnE "eval\s*\(|new Function|child_process" --include="*.js" --include="*.ts" src/
grep -rnE "exec\s*\(|execSync|spawn\s*\(" --include="*.js" src/ | grep -E "\+|\\\$\{"
grep -rnE "\\\$where|\\\$ne|\\\$gt|\\\$regex" --include="*.js" src/
grep -rnE "jwt\.decode|algorithms\s*:\s*\[.?'none'|verify\s*=\s*false" --include="*.js" src/
grep -rnE "Math\.random" --include="*.js" src/
grep -rnE "cors\s*\(\s*\{\s*origin\s*:\s*true" --include="*.js" src/
grep -rnE "rejectUnauthorized\s*:\s*false|__proto__" --include="*.js" src/
grep -rniE "secret|password|api[_-]?key" --include="*.js" src/ | grep -E "=\s*['\"][^'\"]{8,}['\"]"
```

## Fase 3 — Review konfigurasi

**php.ini (production):** `display_errors = Off`, `log_errors = On`, `expose_php = Off`, `allow_url_include = Off`; cookie sesi `httponly=1`, `samesite=Lax`, `secure=1`; `disable_functions` mencakup `exec, shell_exec, system, passthru, popen, proc_open` (hapus hanya yang terbukti dibutuhkan aplikasi). Laravel: docroot wajib `public/` saja. PHP native di nginx: `.htaccess` TIDAK berpengaruh apa pun di nginx — aturan deny/rewrite wajib ada di vhost nginx.

**Node.js:** aplikasi hanya boleh di balik reverse proxy (jangan pernah mengekspos port 3000+ langsung ke internet); process manager (PM2/systemd) menjalankannya sebagai user non-root dengan `NODE_ENV=production` terverifikasi di env prosesnya; header dari `helmet` atau setara; `trust proxy` disetel benar di balik nginx (kalau tidak, rate limiting dan log IP jadi salah); batas ukuran body disetel.

**Semua stack:** hanya 80/443 yang publik — port DB (3306/5432/27017), Redis, dan port panel tidak boleh menghadap internet.

**Cek live:**
```bash
curl -sI <url>            # Strict-Transport-Security, X-Content-Type-Options: nosniff,
                          # X-Frame-Options atau CSP frame-ancestors, Content-Security-Policy, Referrer-Policy
curl -s <url>/.env | head -3        # TIDAK BOLEH menampilkan isi env (harapkan 403/404)
curl -s <url>/.git/HEAD             # TIDAK BOLEH menampilkan ref git
curl -s <url>/phpinfo.php | head -3 # khusus PHP: TIDAK BOLEH ada
```

## Fase 4 — Pemindaian dinamis otomatis (DAST)

Pasang bila belum ada: Nuclei, Nikto, sqlmap, OWASP ZAP.

```bash
# Sapuan CVE dikenal / eksposur / miskonfigurasi (aman untuk production)
nuclei -u <url> -tags cve,exposure,misconfig -exclude-tags dos,intrusive,fuzz \
  -severity critical,high,medium -o nuclei-report.txt

# Cek klasik web server
nikto -h <url> -o nikto-report.txt

# Baseline OWASP ZAP (pasif + aktif ringan)
docker run --rm -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t <url> -r zap-report.html

# TLS: https://www.ssllabs.com/ssltest/ atau docker run --rm drwetter/testssl.sh <host>
```

**Tes injeksi — STAGING SAJA**, deteksi dulu:
```bash
sqlmap -u "<staging-url>/page?id=1" --batch --level=3 --risk=1
sqlmap -u "<staging-url>/login" --forms --batch --level=3 --risk=1
# Eksploitasi (--dbs, --dump): hanya staging + data dummy + persetujuan eksplisit user.
# NoSQL (Node/Mongo): coba body {"user":{"$ne":null}} dan parameter user[$ne]= di endpoint auth/pencarian — bypass apa pun = Critical.
```

Area yang butuh login: sertakan sesi yang sudah login (`nuclei -header "Cookie: ..."`, `sqlmap --cookie="..."`, konteks ZAP dengan user tes). Scan tanpa autentikasi melewatkan sebagian besar permukaan aplikasi yang bergerbang login.

## Fase 5 — Pentest logika manual (staging, akun tes dari Fase 0)

Scanner tidak bisa menemukan celah logika. Jalankan setiap tes; catat bukti request/response:

1. **IDOR / horizontal** — sebagai User A, buka URL record milik A; ganti ID-nya (angka, UUID, atau nomor pegawai/NIP) ke milik User B. Data B yang terlihat/teredit/terhapus apa pun = temuan `confirmed`; severity mengikuti kalibrasi di bagian Disiplin Temuan (umumnya **High**; **Critical** bila tanpa login, massal, atau bisa tulis/hapus).
2. **Eskalasi vertikal** — sebagai role terendah, minta langsung URL admin (manajemen user, hapus, ekspor). Wajib ditolak di server-side.
3. **Manipulasi parameter privilege** — tambah/ubah field `role`, `is_admin`, `status`, harga/jumlah pada request; server wajib mengabaikan atau menolak (cek mass assignment, A01/A06).
4. **Uji throttling** — rate limit wajib bertahan terhadap rotasi `X-Forwarded-For` dan manipulasi header; termasuk endpoint OTP/reset. (Staging saja.)
5. **CSRF** — kirim request pengubah state dari origin asing tanpa token valid. Wajib ditolak.
6. **File upload** — coba tipe executable (`.php`, `.js`, `.html`), ekstensi ganda `x.php.jpg`, kode script di dalam `.jpg`, file oversize. Upload tidak boleh bisa dieksekusi, tidak boleh berada di path executable, tidak boleh memakai nama yang mudah ditebak. **Hapus semua upload tes setelahnya.**
7. **Sesi/JWT** — ID sesi berotasi saat login; logout menonaktifkan sesi; token reset kedaluwarsa dan sekali pakai. JWT: token kedaluwarsa ditolak; token `alg:none`/tanpa tanda tangan dan token bertanda tangan secret salah ditolak; signature diverifikasi di setiap request.
8. **XSS** — simpan `<script>alert(1)</script>` dan `"><svg onload=alert(1)>` di setiap field teks; wajib tampil ter-escape di mana pun, termasuk tampilan admin atas data user lain.
9. **Kebocoran informasi** — paksa error 400/404/500; respons tidak boleh membocorkan stack trace, nama skema, path internal, atau versi; pesan error tidak boleh memungkinkan enumerasi user/record (respons sama untuk "user tidak ditemukan" vs "password salah").
10. **Prototype pollution (Node)** — kirim `{"__proto__": {"polluted": true}}` ke endpoint yang menggabungkan input; verifikasi tidak ada properti global yang muncul setelahnya.
11. **SSRF** — jika aplikasi mengambil URL (webhook, impor via URL, ambil gambar), arahkan ke `http://169.254.169.254/` dan alamat LAN internal di staging. Wajib diblokir.

## Fase 6 — Laporan & gerbang GO / NO-GO

Tulis laporan dalam format ini (Bahasa Indonesia):

```markdown
# 🛡️ Laporan Audit Keamanan Aplikasi
### Standar: OWASP Top 10:2025 & OWASP ASVS v5.0.0

## Ringkasan Eksekutif
- **Target Aplikasi**: [nama]
- **Stack & Versi Runtime**: [PHP/Node + versi, framework, web server]
- **Target ASVS Level**: [1 / 2 / 3]
- **Tools**: [versi nuclei/sqlmap/zap]
- **Backup sebelum tes**: [lokasi]
- **Total Temuan (confirmed)**: [X] ([n] Critical, [n] High, [n] Medium, [n] Low)
- **Perlu Validasi (needs_validation, tanpa severity)**: [n]
- **Putusan Deploy**: GO / NO-GO — [alasan satu baris]

## Temuan Detail
### [SEVERITY] Judul Celah
- **Kategori OWASP**: [A01:2025 – Broken Access Control / dst.]
- **Referensi ASVS**: [V4 Access Control / dst.]
- **Lokasi**: `path/file.php:L45` atau URL endpoint
- **Deskripsi**: mengapa ini rentan
- **Skenario Eksploitasi (PoC)**: langkah penyerang, bukti request/response
- **Perbaikan**:
  (diff: - kode rentan / + kode aman)

## Perlu Validasi (needs_validation — tanpa severity)
### Judul Hipotesis
- **Hipotesis**: boundary yang diduga dilanggar dan mengapa
- **Fakta yang kurang**: fakta penentu yang belum bisa diperiksa
- **Rencana validasi aman**: cara pemilik aplikasi membuktikannya tanpa risiko

## Pembersihan (Cleanup)
- File/akun/payload tes yang dibuat & sudah dihapus: [daftar]

## Hasil Re-Test
- [temuan] → [lolos/gagal setelah perbaikan]

## Action Plan
1. Critical & High — wajib beres sebelum deploy
2. Medium — perbaiki atau dokumentasikan penerimaan risiko oleh pemilik aplikasi
3. Low — perbaikan oportunistik
```

**`findings.json` (wajib, machine-readable)** — selain laporan Markdown, tulis file `findings.json` berisi seluruh temuan (`confirmed`, `needs_validation`, `rejected`) agar hasil audit bisa dilacak, dibandingkan antar-rilis, dan diproses alat lain:

```json
{
  "target": "nama-aplikasi",
  "date": "2026-01-31",
  "asvs_level": 2,
  "verdict": "NO-GO",
  "findings": [
    {
      "id": "GA-001",
      "title": "IDOR pada endpoint data pegawai",
      "status": "confirmed",
      "severity": "High",
      "owasp": "A01:2025",
      "asvs": "V4",
      "location": "app/detail.php:L45",
      "evidence": "User A membuka /detail.php?nip=<milik B> → data B tampil penuh",
      "impact": "Kebocoran data pribadi antarpegawai",
      "fix": "Tambahkan cek kepemilikan record terhadap sesi login sebelum menampilkan",
      "retest": "pending"
    },
    {
      "id": "GA-002",
      "title": "Dugaan header keamanan ditelan reverse proxy",
      "status": "needs_validation",
      "severity": null,
      "missing_fact": "Konfigurasi proxy di depan aplikasi tidak tersedia di repo",
      "validation_plan": "Pemilik memeriksa respons dari origin langsung vs via proxy"
    }
  ]
}
```

Aturan: field `severity` hanya terisi untuk `confirmed` (selain itu `null`); setiap `confirmed` wajib punya `evidence`; setiap `needs_validation` wajib punya `missing_fact` + `validation_plan`; `rejected` wajib punya alasan di `evidence`/catatan. Angka di Ringkasan Eksekutif wajib sama dengan isi `findings.json` dan daftar Temuan Detail — jika berbeda, laporan belum selesai.

**Aturan putusan:**
- Ada **Critical atau High** yang masih terbuka → **NO-GO**. Perbaiki, jalankan ulang fase terkait, tes ulang.
- Medium → perbaiki bila murah, selain itu dokumentasikan penerimaan risikonya oleh pemilik aplikasi.
- `needs_validation` tidak otomatis memblokir GO, tetapi wajib tercantum di laporan dan diterima risikonya secara eksplisit oleh pemilik aplikasi — **kecuali** hipotesisnya berdampak setara Critical (mis. dugaan SQL injection di production yang tidak boleh diuji intrusif): itu **NO-GO** sampai tervalidasi di staging.
- Jangan pernah mengklaim "100% aman" — klaim yang benar adalah "tidak ada temuan Critical/High setelah checklist ini, ASVS Level [n]".

## Anti-pattern — jangan lakukan ini

1. Melaporkan deviasi checklist / best practice yang belum diterapkan sebagai kerentanan, tanpa pelanggaran boundary yang terbukti.
2. Memberi severity pada temuan yang belum tervalidasi (`needs_validation` tidak punya severity).
3. Menebak perilaku deployment (proxy, CDN, panel hosting, provider) yang tidak ada di kode/konfigurasi yang diperiksa — jadikan `needs_validation`, bukan kesimpulan.
4. Menganggap dampak ke diri sendiri (self-XSS, membaca data sendiri) sebagai temuan lintas-pengguna.
5. Mengklaim dampak lebih kuat dari yang teramati (crash parser ≠ RCE; input terefleksi ≠ stored XSS).
6. Melaporkan ulang temuan lama yang sudah diperbaiki tanpa re-test pada kode terkini.
7. Laporan prosa saja tanpa bukti yang bisa diverifikasi ulang (lokasi kode, request/response, versi tool).
8. Mengubah kode target selama audit — audit mendeskripsikan perbaikan; perubahan kode adalah pekerjaan terpisah atas persetujuan pemilik aplikasi.

## Referensi cepat — pemeriksaan bernilai tertinggi

Jika waktu sempit, kelima ini menangkap sebagian besar kebobolan nyata aplikasi web internal/kantor:

1. Audit dependensi bersih + runtime tertambal (Fase 1).
2. `.env` / `.git` tidak bisa diakses web; mode debug mati (Fase 1, 3).
3. Tes IDOR antar dua akun selevel (Fase 5.1).
4. Tes injeksi di staging (Fase 4).
5. Security header + TLS (Fase 3).
