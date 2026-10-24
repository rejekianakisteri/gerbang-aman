# 🚪 Gerbang Aman

<p align="center">
  <img src="https://img.shields.io/badge/License-MIT-green.svg" alt="License: MIT">
  <img src="https://img.shields.io/badge/OWASP-Top%2010%3A2025-red.svg" alt="OWASP Top 10:2025">
  <img src="https://img.shields.io/badge/ASVS-v5-blue.svg" alt="OWASP ASVS v5">
  <img src="https://img.shields.io/badge/Works_with-OpenCode%20%7C%20Antigravity-blueviolet.svg" alt="Works with OpenCode and Antigravity">
</p>

**Skill audit keamanan & penetration test pra-deploy untuk aplikasi web** — selaras **OWASP Top 10:2025** dan **OWASP ASVS v5.0.0**. Sebuah aplikasi tidak boleh lewat ke production sebelum melewati gerbang ini.

> **English summary:** A pre-deployment security audit & pentest Agent Skill for web apps, aligned with OWASP Top 10:2025 and ASVS v5. Covers SAST, supply-chain & dependency scanning, configuration audit, automated DAST (Nuclei, Nikto, OWASP ZAP, sqlmap), and manual authorization/logic testing — ending with a GO / NO-GO deployment verdict. Instructions are in Bahasa Indonesia.

## Isi Skill

Alur audit 6 fase + 1 gerbang putusan:

| Fase | Isi |
|---|---|
| Aturan main | Hanya sistem sendiri/berizin, staging-first, production non-destruktif, backup & cleanup wajib |
| Fase 0 | Ruang lingkup + pemilihan level ASVS (L1/L2/L3) |
| Fase 1 | Lingkungan, secret & supply chain — runtime tertambal, audit dependensi (PHP, Node, Python, Go, dll.) |
| Fase 2 | SAST checklist OWASP Top 10:2025 (A01–A10) + set grep siap-jalan untuk PHP & Node.js |
| Fase 3 | Review konfigurasi — php.ini, Node di balik reverse proxy, security header, eksposur `.env`/`.git` |
| Fase 4 | DAST otomatis — Nuclei, Nikto, OWASP ZAP, sqlmap, cek TLS |
| Fase 5 | Pentest logika manual — IDOR antar-akun selevel, eskalasi role, SSRF, upload, sesi/JWT, prototype pollution |
| Fase 6 | Laporan Bahasa Indonesia (PoC + contoh perbaikan diff) + putusan **GO / NO-GO** |

Stack yang dicakup mendalam: **PHP** (native, Laravel, CodeIgniter) dan **Node.js** (Express). Stack lain dicakup lewat perintah audit resmi masing-masing ekosistem + fase-fase generiknya.

## Contoh Keluaran

Setiap temuan dilaporkan dengan bukti dan contoh perbaikannya:

> ### [HIGH] IDOR — Data Pengguna Lain Terbuka
> - **Lokasi:** `GET /api/peserta/{id}`
> - **Bukti:** login sebagai user A, ubah `{id}` ke milik user B → data B tampil penuh
> - **Dampak:** kebocoran data pribadi lintas akun
> - **Perbaikan:** validasi kepemilikan data di server sebelum data dikembalikan
> - **Status:** ❌ Belum diperbaiki — **menahan GO**

## Instalasi

### Cara cepat (skills CLI)

```bash
npx skills add rejekianakisteri/gerbang-aman
```

### Cara manual

Salin folder `gerbang-aman/` (berisi `SKILL.md`) ke folder skills agent kamu:

| Agent | Path global |
|---|---|
| OpenCode | `~/.config/opencode/skills/gerbang-aman/` |
| Google Antigravity | `~/.gemini/antigravity/skills/gerbang-aman/` |
| Claude Code | `~/.claude/skills/gerbang-aman/` |
| Agent lain (netral) | `~/.agents/skills/gerbang-aman/` |

Atau per-proyek: letakkan di `<proyek>/.agents/skills/gerbang-aman/` — dibaca OpenCode, Antigravity, dan banyak agent lain, dan bisa ikut ter-commit di git.

Di Windows, `~` = `C:\Users\<username>`. Setelah menyalin, restart agent / buka sesi baru.

## Penggunaan

Panggil manual dengan `/gerbang-aman`, atau minta secara natural — skill aktif otomatis dari deskripsinya:

> "Audit keamanan aplikasi ini sebelum deploy."
> "Pentest dulu app ini, cek OWASP-nya."

Tools eksternal (Nuclei, Nikto, sqlmap, ZAP) bersifat opsional — skill tetap berjalan sebagai audit kode & konfigurasi tanpa tools tersebut, dan akan memberi tahu perintah instalasinya bila diperlukan.

## Keselamatan

Skill ini dibuat untuk menguji **aplikasi milik sendiri atau yang berizin tertulis**. Aturan main di dalam skill mewajibkan pengujian intrusif hanya di staging dengan data dummy, dan pengujian production hanya yang non-destruktif.

## Kredit

Gerbang Aman adalah karya kolaborasi dua proses pembuatan yang sama-sama dibantu AI agent: panduan audit berbasis standar **OWASP Top 10:2025** dan **OWASP ASVS v5.0.0** (ditulis dari referensi dokumentasi standar keamanan internasional yang dipublikasikan OWASP) yang digabungkan dengan workflow pentest pra-deploy berbasis eksekusi tools, lalu diselaraskan untuk stack PHP & Node.js. Nama dan kerangka standar OWASP dirujuk sebagaimana mestinya; materi OWASP sendiri diterbitkan di bawah lisensi CC BY-SA.

## Lisensi

MIT — lihat file [LICENSE](LICENSE).
