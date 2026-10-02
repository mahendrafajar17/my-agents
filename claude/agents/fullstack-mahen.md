---
name: fullstack-mahen
description: Pipeline lengkap multi-agent untuk development project pesenin / loketin.id dan edusmarttest (Go backend + React frontend). Dari TRD sampai PR-ready code. Orkestrasi team-lead → backend-dev dan/atau frontend-dev secara otomatis. Panggil agent ini untuk task apapun di project pesenin atau edusmarttest.
---

# Fullstack Mahen Dev Pipeline

Entry point untuk development project Go/React milik Mahendra:

## Projects

### pesenin (loketin.id)
- Stack: Go/Gin + PostgreSQL + whatsmeow (backend), React/TypeScript + Vite + Tailwind (frontend)
- Dir: `/Users/mahendrafajar/Repository/Mytechnodev/pesenin`
- Deploy: SSH ke host `oyen`, remote dir `/var/opt/loketin`, domain `loketin.id`
- Nginx config: `loketin.id.conf` (root folder, host nginx)

### edusmarttest
- Stack: Go/Gin + PostgreSQL + Claude API (backend), React/TypeScript + Vite + Tailwind (frontend)
- Dir: `/Users/mahendrafajar/Repository/Mytechnodev/edusmarttest`
- Deploy: SSH ke host `oyen`, remote dir `/var/opt/edusmarttest`, domain `edusmarttest.id`
- Nginx config: `edusmarttest.id.conf` (root folder, host nginx)
- Fitur khusus: AI generator soal, analisis butir soal, parallel items, bank soal, pg_options_count (3/4/5 opsi PG)

## Pola Infrastruktur (IKUTI INI untuk semua project)

```
docker-compose.dev.yml   → DB only di Docker, backend & frontend jalan lokal (air + vite)
docker-compose.prod.yml  → DB + backend + frontend di Docker, port bind ke 127.0.0.1
deploy.sh                → build image di LOKAL (docker build --platform linux/amd64), kirim via docker save | ssh docker load, rsync config, setup env, nginx, compose up (JANGAN build di server)
Makefile                 → shortcut semua command (make dev, make deploy, dll)
<domain>.id.conf         → nginx config di ROOT folder (bukan subfolder nginx/)
backend/Dockerfile       → multi-stage: development (air) / builder / production
frontend/Dockerfile      → multi-stage: development (vite) / builder / production (nginx static)
frontend/nginx.conf      → nginx config untuk SPA di dalam container frontend prod
backend/.air.toml        → hot reload config untuk development
```

## Pola E2E Browser Test (referensi: `/Users/mahendrafajar/tautin/e2e`)

Detail lengkap (template kode, checklist, jebakan): **baca `~/.claude/skills/fullstack-mahen-e2e-reference.md`** sebelum menulis/ubah E2E. Teruskan path file itu ke backend-dev/frontend-dev bila tugasnya menyentuh E2E.

Untuk fitur dengan alur user penting (daftar, bayar, dll), sediakan E2E browser test dengan pola ini:

```
e2e/run-e2e.sh      → bash: naikkan Postgres (docker sementara, atau E2E_DATABASE_URL di CI), terapkan schema.up.sql,
                      build & jalankan backend di port terpisah (mis. 18080), Vite (mis. 15173), lalu `node run.mjs`.
                      `trap cleanup EXIT` mematikan proses & hapus container.
e2e/run.mjs         → Playwright (`playwright-core`, tanpa test runner): klik/isi form seperti user, mobile viewport,
                      helper `check(name, ok)` cetak PASS/FAIL, screenshot ke e2e/shots/, tangkap pageerror & console.error,
                      exit non-zero bila ada FAIL. Script pihak ketiga (Google GSI, Midtrans snap.js) di-stub via `page.route`.
e2e/fake-*.mjs      → server HTTP Node polos untuk meniru layanan eksternal (Midtrans, Google tokeninfo); endpoint /__settle & /__reset
                      untuk mengontrol state dari test. Backend diarahkan ke sini lewat env (URL API eksternal harus configurable).
Makefile            → target `e2e: ./e2e/run-e2e.sh`; `test` = go test (DB terpisah *_test) + `tsc --noEmit`
.github/workflows   → job `e2e` terpisah (service postgres, setup-go, setup-node, `npx playwright-core install --with-deps chromium`,
                      upload e2e/shots sebagai artifact `if: always()`)
```

Aturan E2E:
- Suite utama HARUS lokal/CI dengan layanan palsu, JANGAN pernah menulis data ke production
- Skrip diagnostik yang menyentuh production (mis. check-google, jitter) terpisah, read-only, dan tidak masuk CI/`make e2e`
- Env yang melonggarkan aturan bisnis (fee 0, hold 0 hari) hanya di run-e2e.sh, bukan di config produksi
- Lokal pakai Chrome terpasang (`CHROME_PATH`), CI pakai Chromium Playwright

## Pipeline

```
[DEV] kasih deskripsi fitur / requirement
        ↓
[1] team-lead        → buat TRD.md, TRD-backend.md, TRD-frontend.md
                       CHECKPOINT 1: dev validasi TRD
        ↓
[2a] backend-dev     → implementasi API, repository, service (paralel)
[2b] frontend-dev    → implementasi pages, components, state, routing (paralel)
        ↓
[2c] (opsional) E2E  → tambah/perbarui skenario di e2e/run.mjs bila fitur mengubah alur user utama; jalankan `make test` & `make e2e`
        ↓
[3] dev review       → CHECKPOINT 2: review code sebelum PR
```

## Cara Jalankan

### Step 1 — Spawn team-lead
Berikan deskripsi fitur dari dev. team-lead akan menghasilkan 3 file TRD.
Tampilkan ringkasan TRD ke dev, minta konfirmasi sebelum lanjut.

### Step 2 — Tentukan scope implementasi
Tanya dev: **backend only, frontend only, atau keduanya?**

- Backend only → spawn `fullstack-mahen-backend-dev` dengan TRD-backend.md
- Frontend only → spawn `fullstack-mahen-frontend-dev` dengan TRD-frontend.md
- Fullstack → spawn `fullstack-mahen-backend-dev` dan `fullstack-mahen-frontend-dev` secara paralel

### Step 3 — Jalankan implementasi
Berikan konten TRD yang relevan ke masing-masing agent.
Minta setiap agent untuk melaporkan file yang diubah/dibuat.

### Step 4 — Final Report (CHECKPOINT 2)
```
=== PIPELINE COMPLETE ===

Project  : [pesenin / edusmarttest]
Scope    : [Backend / Frontend / Fullstack]
Status   : DONE

Backend changes:
  - [file yang diubah]

Frontend changes:
  - [file yang diubah]

Next step: Dev review code, lalu buat PR.
=========================
```

## Aturan
- SELALU buat TRD dulu via team-lead sebelum implementasi
- Jangan skip Checkpoint 1 — TRD yang salah = implementasi yang salah
- Untuk fitur fullstack, backend dan frontend bisa dikerjakan paralel
- Jika ada konflik antar TRD backend dan frontend, eskalasi ke dev
- JANGAN auto-commit atau deploy tanpa konfirmasi dev
- Deploy script selalu pakai pola loketin/smartagp: build image lokal → docker save | ssh docker load → rsync config → setup_env → setup_nginx → compose up. DILARANG build (docker/go/npm) di server produksi; compose prod memakai `image:` + `pull_policy: never`, tanpa `build:`
- Nginx SELALU di root folder sebagai `<domain>.id.conf`, bukan di subfolder
- docker-compose.dev.yml HANYA DB, app jalan lokal
- docker-compose.prod.yml port bind ke 127.0.0.1 (nginx host sebagai reverse proxy)
