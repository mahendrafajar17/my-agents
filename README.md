# my-agents

Koleksi custom Claude agents dan commands untuk development workflow.

## Struktur

```
agents/      → Agent definitions (.md)
commands/    → Slash commands (.md)
```

---

## Pipelines

### `/go-wacall` — Go Microservice Pipeline

Pipeline multi-agent untuk development **Go microservice** (Gin + MySQL/sqlx + zap).
Dari TRD sampai PR-ready code dengan test otomatis.

```
orchestrator → environment → coder → tester → reviewer
```

| Agent | Tugas |
|-------|-------|
| `go-wacall-orchestrator` | Parse TRD, buat task summary |
| `go-wacall-environment` | Setup mock, fixtures, validasi interface |
| `go-wacall-coder` | Implementasi model, repository, handler |
| `go-wacall-tester` | Tulis & jalankan unit test |
| `go-wacall-reviewer` | Analisa hasil test, loop fix jika gagal |

**Cara pakai:**
```
/go-wacall <TRD kamu di sini>
```

---

### `/fullstack-mahen` — Fullstack WA Queue Pipeline

Pipeline multi-agent untuk development **Go backend + React frontend** (WA Queue project).
Dari TRD sampai PR-ready code, orkestrasi otomatis.

```
team-lead → backend-dev dan/atau frontend-dev
```

| Agent | Tugas |
|-------|-------|
| `fullstack-mahen-team-lead` | Buat TRD lengkap (arsitektur, API spec, UI spec) |
| `fullstack-mahen-backend-dev` | Implementasi Go backend (Gin, PostgreSQL, JWT, whatsmeow) |
| `fullstack-mahen-frontend-dev` | Implementasi React frontend (Vite, Zustand, React Query, Tailwind) |

**Cara pakai:**
```
/fullstack-mahen <deskripsi fitur kamu di sini>
```

---

## Setup

Copy agents dan commands ke Claude workspace:

```bash
cp agents/* ~/.claude/agents/
cp commands/* ~/.claude/commands/
```
