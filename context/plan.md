# TaskFlow Go API — CI/CD Pipeline Implementation Plan

## 1. Ringkasan Proyek

**Nama Proyek:** TaskFlow Go API  
**Tujuan:** Merancang dan mengimplementasikan sistem CI/CD otomatis untuk aplikasi manajemen proyek TaskFlow Inc.  
**Tech Stack:** Go 1.22, PostgreSQL 16, Docker (multi-stage), net/http  
**Deliverables:** Pipeline CI/CD end-to-end, perbaikan bug, testing, Docker image registry, smoke test, rollback strategy, security audit, laporan, dan demo live.

---

## 2. Struktur Tim & Pembagian Tugas

Berdasarkan `context/pembagian.md`:

| Orang | Role | Fokus Utama | Tanggung Jawab Utama |
|-------|------|-------------|----------------------|
| **1** | Application Quality Engineer | S1 — Bug Fix, Test, Coverage | Temukan & perbaiki 3 bug, tambah ≥2 test case, pastikan semua unit test & race test lulus, coverage ≥75%. |
| **2** | CI Pipeline Engineer | S2 — CI Pipeline Otomatis | Buat pipeline CI (sesuai tool kelompok), trigger push & PR, jalankan vet, unit test, race test, integration test PostgreSQL, coverage gate, build binary, simpan coverage artifact. |
| **3** | Docker & Registry Engineer | S3 — Docker Image & Container Registry | Docker build multi-stage, tag image berbasis commit SHA, push ke registry, bukti image bisa di-pull, perbandingan ukuran image multi-stage vs single-stage. |
| **4** | Deployment & Smoke Test Engineer | S4 — Smoke Test Post-Deploy | Jalankan aplikasi hasil deploy, smoke test otomatis `/health` & `/api/v1/stats`, pipeline gagal jika smoke test gagal, bukti sukses & gagal. |
| **5** | Notification & Release Evidence Engineer | S4 — Notifikasi & Bukti Release | Notifikasi pipeline sukses/gagal (Slack/Telegram/Email), kumpulkan bukti: branch, commit SHA, waktu, link pipeline, screenshot. |
| **6** | Rollback & Release Management Engineer | S5 — Rollback Strategy | Strategi rollback, tag `stable` hanya update saat pipeline sukses, prosedur rollback, demo rollback, bukti normal setelah rollback. |
| **7** | Security, Report & Presentation Lead | S6 + Laporan + Presentasi | Integrasikan ≥2 kategori security scan (SCA, SAST, Secret Scanning, Container Image Scan), gabung laporan akhir, susun presentasi, koordinasi demo live. |

---

## 3. Deliverables Per Skenario

### S1 — Bug Fix, Test & Coverage (Orang 1)

| # | Task | Detail | Output |
|---|------|--------|--------|
| 1.1 | Identifikasi 3 bug | Cari komentar `// BUG` di source code. | Daftar bug: file, baris, deskripsi. |
| 1.2 | Perbaiki 3 bug | Fix bug yang ditemukan. | Kode terupdate, test mulai hijau. |
| 1.3 | Unit test | Jalankan `go test ./...` | Semua PASS. |
| 1.4 | Race condition test | Jalankan `go test -race ./...` | Semua PASS, tidak ada race condition. |
| 1.5 | Tambah test case | Minimal 2 test case baru yang relevan. | File test terupdate, coverage naik. |
| 1.6 | Coverage gate | `go test ./... -coverprofile=cov.out` | Coverage ≥ 75%. |

> **Kunci Sukses:** Semua test PASS, race test bersih, coverage ≥75%.

---

### S2 — CI Pipeline Otomatis (Orang 2)

| # | Task | Detail | Output |
|---|------|--------|--------|
| 2.1 | Konfigurasi trigger | Pipeline jalan saat `push` ke `main`/`develop` dan `pull request`. | File konfigurasi CI terisi trigger. |
| 2.2 | Stage `go vet` | Jalankan `go vet ./...`, pipeline gagal jika error. | Pipeline merah jika ada vet error. |
| 2.3 | Stage unit test | Jalankan `go test -race ./...` | Pipeline gagal jika test gagal. |
| 2.4 | Stage integration test | Service container PostgreSQL, jalankan `go test -tags=integration -race ./...` | Integration test PASS dengan DB nyata. |
| 2.5 | Stage coverage | Cek coverage, pipeline gagal jika < 75%. | Coverage artifact tersimpan. |
| 2.6 | Stage build binary | `go build` harus berhasil. | Binary tergenerate. |
| 2.7 | Simpan artifact | Upload laporan coverage ke pipeline. | Artifact bisa diunduh. |
| 2.8 | Proof | Revert salah satu bug S1 → push → pipeline merah. | Screenshot merah + hijau. |

> **Kunci Sukses:** Pipeline otomatis, memblokir bug, artifact tersedia.

---

### S3 — Docker Image & Registry (Orang 3)

| # | Task | Detail | Output |
|---|------|--------|--------|
| 3.1 | Multi-stage Dockerfile | Pastikan builder → scratch, image kecil. | Dockerfile valid. |
| 3.2 | Build image | Pipeline stage build Docker image. | Image berhasil dibuild. |
| 3.3 | Tag SHA | Format: `<registry>/taskflow-api:sha-<7-char>`. | Image tertag per commit. |
| 3.4 | Push ke registry | Push ke registry sesuai tool (GHCR/GitLab/Docker Hub). | Image terlihat di registry. |
| 3.5 | Proof pull | Bukti bisa `docker pull` image dari registry. | Screenshot/command sukses. |
| 3.6 | Perbandingan ukuran | Dokumentasikan multi-stage vs `FROM golang:1.22` langsung. | Laporan perbandingan ukuran. |
| 3.7 | Depends on CI | CD hanya jalan jika S2 CI sukses. | Dependency eksplisit di pipeline. |

> **Kunci Sukses:** Image ter-push dengan tag SHA, bisa di-pull, ukuran ≤15 MB, CD bergantung CI.

---

### S4 — Smoke Test, Notifikasi & Bukti Release (Orang 4 & 5)

#### Orang 4 — Smoke Test

| # | Task | Detail | Output |
|---|------|--------|--------|
| 4.1 | Smoke test script | Setelah container jalan, tunggu 5 detik, curl `/health` dan `/api/v1/stats`. | Script otomatis. |
| 4.2 | Pipeline integration | Smoke test jadi stage di pipeline, gagal → pipeline merah. | Pipeline gagal jika smoke test gagal. |
| 4.3 | Proof sukses | Screenshot/screen recording smoke test berhasil. | Bukti sukses. |
| 4.4 | Proof gagal | Simulasi smoke test gagal (misal: salah port). | Bukti gagal. |

#### Orang 5 — Notifikasi & Bukti Release

| # | Task | Detail | Output |
|---|------|--------|--------|
| 4.5 | Setup notifikasi | Pilih Slack/Telegram/Email, buat webhook/bot. | Notifikasi terkonfigurasi di pipeline. |
| 4.6 | Notifikasi sukses | Kirim pesan dengan ✅, sertakan branch, commit SHA, waktu, link pipeline. | Screenshot notifikasi hijau. |
| 4.7 | Notifikasi gagal | Kirim pesan dengan ❌, sertakan error, branch, commit SHA, link pipeline. | Screenshot notifikasi merah. |
| 4.8 | Kumpulkan bukti release | Dokumentasikan: branch, commit SHA, waktu pipeline, link run, status. | Dokumen/CSV/screenshot bukti release. |

> **Kunci Sukses:** Smoke test memblokir pipeline, notifikasi beda isi untuk sukses/gagal.

---

### S5 — Rollback Strategy (Orang 6)

| # | Task | Detail | Output |
|---|------|--------|--------|
| 5.1 | Tag `stable` kondisional | Tag `stable` hanya diperbarui jika semua stage + smoke test PASS. | Pipeline stage push stable tag. |
| 5.2 | Makefile target `rollback` | `make rollback ROLLBACK_TAG=sha-xxxxx` → pull, stop, run, verify health. | Target `rollback` di Makefile. |
| 5.3 | Simulasi bug | Commit kode dengan bug (integer division), push, pipeline hijau. | Bukti bug ter-deploy. |
| 5.4 | Demo rollback | Jalankan `make rollback ROLLBACK_TAG=sha-<lama>`, verifikasi `/api/v1/stats` benar. | Demo live rollback berhasil. |
| 5.5 | Prosedur rollback | Dokumen 1 halaman: deteksi → rollback → verifikasi → selesai. | `ROLLBACK_PROCEDURE.md`. |

> **Kunci Sukses:** Tag `stable` kondisional, rollback bisa dijalankan siapa saja, aplikasi normal setelah rollback.

---

### S6 — Audit Keamanan Pipeline (Orang 7)

| # | Task | Detail | Output |
|---|------|--------|--------|
| 6.1 | Pilih ≥2 kategori | Pilih dari: A (SCA), B (SAST), C (Secret Scanning), D (Container Image Scan). | Keputusan kategori. |
| 6.2 | Integrasi scan ke pipeline | Setiap scan jadi stage di CI, gagal jika HIGH/CRITICAL. | Stage scan di pipeline config. |
| 6.3 | Generate artifact | Setiap scan output JSON/HTML sebagai artifact. | Artifact tersedia di pipeline. |
| 6.4 | Analisis temuan | Bedakan false positive vs true positive, rekomendasi perbaikan. | Laporan 1 halaman per kategori. |
| 6.5 | Pre-commit hook (jika Secret Scanning) | `gitleaks protect --staged` sebelum commit. | `.git/hooks/pre-commit` terkonfigurasi. |
| 6.6 | Dokumentasi scratch image | Jelaskan kenapa image scratch hampir tidak punya vulnerability OS. | Penjelasan di laporan. |

> **Kunci Sukses:** Minimal 2 kategori, pipeline blokir temuan kritis, artifact laporan tersedia.

---

## 4. Timeline & Milestone

| Fase | Durasi | Milestone | PIC |
|------|--------|-----------|-----|
| **Persiapan** | Hari 1 | Setup repo, setup tool CI/CD masing-masing, semua bisa push & pull. | Semua |
| **S1: Bug Fix & Test** | Hari 1–2 | 3 bug diperbaiki, semua test PASS, coverage ≥75%. | Orang 1 |
| **S2: CI Pipeline** | Hari 2–3 | Pipeline otomatis jalan: vet → test → coverage gate → build. | Orang 2 |
| **S3: Docker & Registry** | Hari 3–4 | Image ter-build, ter-tag SHA, ter-push, bisa di-pull. | Orang 3 |
| **S4: Smoke Test + Notifikasi** | Hari 4–5 | Smoke test otomatis, notifikasi sukses & gagal terkirim. | Orang 4 & 5 |
| **S5: Rollback** | Hari 5–6 | Tag stable kondisional, `make rollback` berfungsi, prosedur tertulis. | Orang 6 |
| **S6: Security Audit** | Hari 5–6 | ≥2 kategori scan terintegrasi, pipeline blokir kritis, laporan jadi. | Orang 7 |
| **Integrasi & Testing** | Hari 6–7 | Full pipeline end-to-end diuji: push → CI → CD → smoke → notifikasi. | Semua |
| **Laporan & Presentasi** | Hari 7–8 | Laporan final, slide presentasi, latihan demo live. | Orang 7 (lead), Semua (kontribusi) |

---

## 5. Alur Pipeline End-to-End

```
Developer Push ke main/develop atau buat Pull Request
              │
              ▼
    ┌─────────────────────┐
    │  1. Trigger Pipeline  │
    └─────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │  2. go vet            │  ──► Gagal? Pipeline merah + notifikasi gagal
    └─────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │  3. Unit Test + Race  │  ──► Gagal? Pipeline merah + notifikasi gagal
    └─────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │  4. Integration Test  │  ──► Gagal? Pipeline merah + notifikasi gagal
    │     (PostgreSQL)      │
    └─────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │  5. Coverage Gate     │  ──► < 75%? Pipeline merah + notifikasi gagal
    └─────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │  6. Security Scan     │  ──► HIGH/CRITICAL? Pipeline merah + notifikasi gagal
    │     (S6: SCA/SAST/Secret/Image)
    └─────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │  7. Build Binary      │  ──► Gagal? Pipeline merah + notifikasi gagal
    └─────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │  8. Build Docker Image│  ──► Multi-stage build
    └─────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │  9. Tag Image         │  ──► sha-<7-char>
    └─────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │ 10. Push ke Registry  │
    └─────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │ 11. Smoke Test        │  ──► Gagal? Pipeline merah + notifikasi gagal
    │     /health + /stats  │
    └─────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │ 12. Tag stable        │  ──► Hanya jika semua PASS
    └─────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │ 13. Notifikasi Sukses │  ──► ✅ Pipeline hijau + detail release
    └─────────────────────┘
```

---

## 6. Struktur Laporan Akhir

Disediakan oleh Orang 7, dengan kontribusi dari semua anggota:

```
1. Identitas kelompok & tool CI/CD yang digunakan
2. Diagram alur pipeline lengkap (push → vet → test → build → docker → deploy)
3. Tabel 3 bug: file, baris, kode salah, kode benar, nama test yang mendeteksi
4. Screenshot pipeline MERAH (bug ada) dan HIJAU (bug diperbaiki)
5. Perbandingan ukuran Docker image: multi-stage vs FROM golang:1.22 langsung
6. Bukti image di registry: URL + contoh tag sha-xxxxx dan tag stable
7. Screenshot smoke test berjalan + notifikasi sukses & gagal (S4)
8. Prosedur rollback + screenshot demo rollback live (S5)
9. [Jika S6] Laporan per kategori: tool, temuan, analisis false positive, rekomendasi
10. Refleksi: keunggulan & keterbatasan tool vs tool kelompok lain
```

---

## 7. Checklist Pra-Presentasi

- [ ] Semua 3 bug diperbaiki dan test PASS (S1)
- [ ] Coverage ≥ 75% dengan bukti (S1)
- [ ] Pipeline otomatis aktif untuk push & PR (S2)
- [ ] Pipeline merah jika vet/test/coverage gagal (S2)
- [ ] Integration test dengan PostgreSQL berjalan (S2)
- [ ] Docker image multi-stage ≤ 15 MB (S3)
- [ ] Image ter-push dengan tag SHA & bisa di-pull (S3)
- [ ] Smoke test `/health` & `/api/v1/stats` otomatis (S4)
- [ ] Notifikasi sukses & gagal terkirim dengan detail lengkap (S4)
- [ ] Tag `stable` hanya update saat pipeline sukses (S5)
- [ ] `make rollback ROLLBACK_TAG=sha-xxx` berfungsi (S5)
- [ ] Demo rollback live siap (S5)
- [ ] Minimal 2 kategori security scan terintegrasi (S6)
- [ ] Pipeline gagal jika scan temukan HIGH/CRITICAL (S6)
- [ ] Artifact scan tersedia (S6)
- [ ] Laporan final lengkap 10 poin (Orang 7)
- [ ] Slide presentasi & demo script siap (Orang 7)

---

## 8. Risiko & Mitigasi

| Risiko | Mitigasi |
|--------|----------|
| Bug sulit ditemukan | Orang 1 fokus penuh 2 hari pertama, gunakan `go test -v` dan cari komentar `// BUG`. |
| Integration test gagal karena PostgreSQL | Pastikan `DATABASE_URL` benar dan service container aktif di pipeline config. |
| Docker image terlalu besar | Verifikasi multi-stage: builder → scratch, hanya copy binary + CA certs. |
| Registry tidak bisa diakses | Test push/pull sejak Hari 3, gunakan token/key yang valid. |
| Notifikasi tidak terkirim | Test webhook/bot sejak Hari 4, simpan secret dengan benar (bukan di repo!). |
| Rollback gagal saat demo | Latihan demo 2–3 kali sebelum presentasi, siapkan fallback script manual. |
| Security scan false positive banyak | Dokumentasikan dan jelaskan di laporan, jangan skip analisis. |

---

*Plan ini disusun berdasarkan `context/pbl-cicd-problem.md` dan `context/pembagian.md`.  
Update plan ini jika ada perubahan scope atau tool CI/CD kelompok.*
