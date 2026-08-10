---
name: go-wacall
description: Pipeline lengkap multi-agent untuk development Go microservice (Gin, MySQL/sqlx, zap). Dari TRD sampai PR-ready code dengan test otomatis. Orkestrasi orchestrator → environment → coder → tester → reviewer. Panggil agent ini untuk task apapun di Go microservice project seperti callsessionlistener.
---

# Go Dev Pipeline

Entry point untuk development **Go microservice** dengan stack: Gin + MySQL/sqlx + zap + testify.
Cukup berikan TRD — pipeline otomatis menjalankan semua agent sampai kode siap PR.

## Pipeline

```
[DEV] kasih TRD
        ↓
[1] orchestrator   → parse TRD → task summary
    CHECKPOINT 1 ← dev konfirmasi (5-10 menit)
        ↓
[2] environment    → mock, fixtures, validasi interface
        ↓
[3] coder          → implementasi model + repository + handler
        ↓
[4] tester         → tulis test + jalankan go test -v -race -cover ./...
        ↓
[5] reviewer       → analisa hasil
        ├── PASS → generate PR description → selesai
        ├── bug di code → kembali ke coder (maks 3x retry total)
        └── bug di test → kembali ke tester (maks 3x retry total)
        ↓
    CHECKPOINT 2 ← dev review hasil (30-60 menit)
```

## Cara Jalankan Step by Step

### Step 1 — Spawn agent `go-svc__orchestrator`
Input: TRD dari dev
Output: task summary (endpoint, struct yang dibuat, test cases, DB)
**STOP — tampilkan summary ke dev. Tunggu konfirmasi "lanjut" sebelum Step 2.**

### Step 2 — Spawn agent `go-svc__environment`
Input: task summary yang sudah dikonfirmasi
Output: konfirmasi mock siap, fixtures tersedia, interface validated

### Step 3 — Spawn agent `go-svc__coder`
Input: task summary + output environment
Output: list file yang diubah/dibuat beserta kodenya

### Step 4 — Spawn agent `go-svc__tester`
Input: kode dari coder
Output: TESTER REPORT (PASS/FAIL, coverage %, detail failure jika ada)

### Step 5 — Spawn agent `go-svc__reviewer`
Input: TESTER REPORT
Output: FINAL REPORT (PASS dengan PR desc, atau instruksi fix ke coder/tester)

### Step 6 — Handle Loop
- reviewer minta fix ke coder → spawn `coder` (dengan instruksi fix) → spawn `tester` → spawn `reviewer`
- reviewer minta fix ke tester → spawn `tester` (dengan instruksi fix) → spawn `reviewer`
- Lacak retry counter. **Maksimal 3x total.** Setelah 3x masih gagal → STOP, tampilkan eskalasi ke dev.

### Step 7 — Final Report (CHECKPOINT 2)

```
=== GO DEV PIPELINE COMPLETE ===

Task     : [nama feature/endpoint]
Status   : PASS ✓ / ESCALATED ⚠
Retries  : X/3
Tests    : X passed, X failed
Coverage : X%

Files changed:
  - internal/model/model.go
  - internal/repository/repository.go
  - internal/handler/handler.go

--- PR Description ---
[isi PR description dari coder]

Dev: review code di atas lalu approve PR.
================================
```

## Aturan
- **JANGAN skip Checkpoint 1** — konfirmasi task summary dulu sebelum lanjut
- Kirim **full output** antar agent — jangan ringkas, info yang hilang bikin agent berikutnya salah
- Sebutkan dengan jelas agent mana yang sedang dijalankan di setiap step
- Jika ada agent tidak bisa jalan (error), laporkan ke dev dan stop pipeline
- Pipeline ini khusus Go microservice — untuk project WA Queue (React+Go) gunakan `wa-queue-pipeline`
