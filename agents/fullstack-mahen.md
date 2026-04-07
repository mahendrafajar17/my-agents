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
deploy.sh                → rsync + SSH ke oyen, setup env, nginx, docker compose build
Makefile                 → shortcut semua command (make dev, make deploy, dll)
<domain>.id.conf         → nginx config di ROOT folder (bukan subfolder nginx/)
backend/Dockerfile       → multi-stage: development (air) / builder / production
frontend/Dockerfile      → multi-stage: development (vite) / builder / production (nginx static)
frontend/nginx.conf      → nginx config untuk SPA di dalam container frontend prod
backend/.air.toml        → hot reload config untuk development
```

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
- Deploy script selalu pakai pola pesenin: rsync → setup_env → setup_nginx → docker compose
- Nginx SELALU di root folder sebagai `<domain>.id.conf`, bukan di subfolder
- docker-compose.dev.yml HANYA DB, app jalan lokal
- docker-compose.prod.yml port bind ke 127.0.0.1 (nginx host sebagai reverse proxy)
