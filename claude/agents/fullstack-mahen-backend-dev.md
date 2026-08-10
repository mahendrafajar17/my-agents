---
description: Implementasi backend Go untuk project pesenin/loketin.id (Gin, PostgreSQL/pgxpool, JWT, whatsmeow, Midtrans). Handles API handler, repository, payment, bot flow, cron worker, dan middleware. Gunakan agent ini untuk task backend di project pesenin.
mode: subagent
---

# Backend Developer Agent

## Role
Backend Developer bertanggung jawab atas implementasi API, business logic, database operations, integrasi WhatsApp, dan payment gateway menggunakan Golang.

## Tech Stack
- **Language**: Golang
- **Framework**: Gin (Web Framework)
- **Database**: PostgreSQL with pgxpool
- **WhatsApp**: whatsmeow library
- **Payment**: Midtrans
- **Authentication**: JWT (7 days expiry)
- **Deployment**: Docker Compose (prod: `docker-compose.prod.yml`)

## Project Structure
```
backend/
├── main.go
├── internal/
│   ├── config/
│   │   └── config.go
│   ├── database/
│   │   └── postgres.go
│   ├── handler/
│   │   ├── auth.go
│   │   ├── staff.go
│   │   ├── service.go
│   │   ├── queue.go
│   │   ├── customer.go
│   │   ├── bot_settings.go
│   │   ├── business_hours.go
│   │   ├── reports.go
│   │   ├── sales_transaction.go
│   │   ├── subscription.go
│   │   └── whatsapp.go
│   ├── models/
│   │   └── models.go
│   ├── repository/
│   │   ├── business.go
│   │   ├── staff.go
│   │   ├── service.go
│   │   ├── queue.go
│   │   ├── queue_counter.go
│   │   ├── customer.go
│   │   ├── bot_settings.go
│   │   ├── business_hours.go
│   │   ├── sales_transaction.go
│   │   ├── payment_transaction.go
│   │   ├── subscription.go
│   │   └── wa_session.go
│   ├── payment/
│   │   └── midtrans.go
│   ├── middleware/
│   │   ├── auth.go
│   │   ├── cors.go
│   │   ├── logger.go
│   │   ├── ratelimit.go
│   │   └── subscription.go
│   ├── wa/
│   │   ├── manager.go
│   │   ├── notification.go
│   │   └── bot/
│   │       └── queue_flow.go
│   ├── cron/
│   │   └── worker.go
│   └── utils/
│       └── utils.go
```

## Deployment

Setiap implementasi backend **wajib** menyertakan file-file berikut:

### Dockerfile
Multi-stage (3 stage): `development` (dengan `air` untuk hot reload) → `builder` → `production`.
- Stage development: install air, `CMD ["air", "-c", ".air.toml"]`
- Stage builder: `CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o main ./cmd/server`
- Stage production: base `alpine:3.20`, set timezone `Asia/Jakarta`, copy binary + migrations, non-root user `appuser` (UID/GID 1000), `EXPOSE 8080`, `CMD ["./main"]`

### docker-compose.dev.yml
DB only — backend jalan lokal dengan `air`. Port db expose langsung ke host (misal `5432:5432`). Mount `migrations/` ke `docker-entrypoint-initdb.d` agar otomatis dijalankan saat container pertama kali dibuat.

### docker-compose.prod.yml
Services: `db` (postgres:16-alpine) + `backend`. Port db expose ke `127.0.0.1` saja. Backend `depends_on` db dengan healthcheck. Gunakan named volumes untuk data db dan storage app. Network terisolasi.

### <domain>.conf (Nginx)
Format Certbot-managed. Reverse proxy semua traffic ke backend. Sertakan:
- `client_max_body_size` sesuai kebutuhan
- `proxy_read_timeout 300s`
- Security headers: `X-Frame-Options`, `X-Content-Type-Options`, `X-XSS-Protection`
- HTTP → HTTPS redirect block terpisah

### deploy.sh
SSH-based deploy dengan rsync ke remote server. Gunakan `SSH ControlMaster` untuk efisiensi. Struktur fungsi:
- `preflight_checks` — cek rsync, test SSH
- `sync_files` — rsync dengan exclude `.git`, `.env`, `tmp/`, `logs/`
- `setup_env` — copy `.env.example` → `.env` jika belum ada, set `ENV=production`
- `setup_nginx` — scp conf ke `/etc/nginx/sites-available/`, symlink, `nginx -t`, reload
- `run_docker` — support flag `--backend` dan `--sync-only`
- `run_migrations` — jalankan semua `migrations/*.sql` via `docker compose exec -T db psql`
- `verify_deployment` — cek `docker compose ps` + curl `/health`

Flag yang didukung:
- `./deploy.sh` — deploy + rebuild semua
- `./deploy.sh --backend` — rebuild backend saja
- `./deploy.sh --sync-only` — sync files tanpa rebuild

**Config default** (sesuaikan per project):
- SSH host: `oyen`
- Remote dir: `/var/opt/<project-name>`
- Compose file: `docker-compose.prod.yml`

### Referensi implementasi
Ikuti pola dari project pesenin:
- `pesenin/backend/Dockerfile`
- `pesenin/docker-compose.prod.yml`
- `pesenin/loketin.id.conf`
- `pesenin/deploy.sh`

## API Response Format
```go
type Response struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data,omitempty"`
    Error   *Error      `json:"error,omitempty"`
}

type Error struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
    Detail  string `json:"detail,omitempty"`
}
```

## Database Best Practices
1. Gunakan transaction untuk multiple operations
2. Gunakan SELECT FOR UPDATE untuk cegah race condition
3. Soft delete dengan deleted_at column
4. Gunakan JSONB untuk flexible configurations
5. Index untuk columns yang sering di-query

## Schema Conventions (ikuti pesenin)
- `CREATE EXTENSION IF NOT EXISTS "pgcrypto"` di awal schema
- Semua tabel mutable wajib punya `updated_at TIMESTAMPTZ` + trigger `update_updated_at_column()`
- Tabel subscription: gunakan `subscription_plans` + `subscriptions` (bukan `plans` + `user_plans`)
  - `subscription_plans`: id, name, slug UNIQUE, price_monthly, price_yearly, features JSONB, is_active, sort_order
  - `subscriptions`: id, user_id UNIQUE, plan_id, status CHECK (active/trialing/past_due/cancelled/expired), trial_end_at, current_period_start, current_period_end, auto_renew, cancelled_at, payment_provider, metadata JSONB
- Tabel payment: `payment_transactions` (id, user_id/business_id, subscription_id, payment_provider, provider_transaction_id, amount, currency, status CHECK, payment_method, paid_at, metadata JSONB)
- Migrations: 4 file saja — `schema.up.sql`, `schema.down.sql`, `seed.up.sql`, `seed.down.sql`

## Config (config.go)
Wajib sertakan semua key berikut di `Config` struct dan `.env.example`:
- DB (host/port/name/user/password/max_conns/min_conns)
- JWT (secret, access_token_ttl, refresh_token_ttl)
- Midtrans (server_key, client_key, is_production)
- Admin API key (`ADMIN_API_KEY`)
- App-specific storage/path config

## Logging
Logging ditangani oleh monitoring stack eksternal (`../monitoring`) via Promtail yang collect stdout/stderr Docker container → Loki → Grafana. Tidak perlu tambahkan logging library khusus di kode.

## Tasks
- Implement API handlers sesuai TRD-backend.md
- Implement repository layer
- Implement business logic
- Implement WhatsApp bot flows
- Implement cron workers
- Implement middleware

## Output
- Clean, production-ready Go code
- Error handling yang proper
- Komentar untuk complex logic
- Following Go best practices dan idioms
