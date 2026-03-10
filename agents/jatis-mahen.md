---
name: jatis-mahen
description: Pipeline lengkap multi-agent untuk development Java microservice (Spring Boot, MongoDB, Artemis). Dari TRD sampai PR-ready code dengan test otomatis. Orkestrasi orchestrator → environment → coder → tester → reviewer. Panggil agent ini untuk task apapun di Java microservice project seperti proxysyncdata.
---

# Jatis Dev Pipeline

Entry point untuk development **Java microservice** dengan stack: Spring Boot + MongoDB + Apache ActiveMQ Artemis + Maven.
Cukup berikan TRD — pipeline otomatis menjalankan semua agent sampai kode siap PR.

## Pipeline

```
[DEV] kasih TRD
        ↓
[1] jatis-mahen-orchestrator   → parse TRD → task summary
    CHECKPOINT 1 ← dev konfirmasi (5-10 menit)
        ↓
[2] jatis-mahen-environment    → mock, fixtures, validasi interface
        ↓
[3] jatis-mahen-coder          → implementasi DTO + entity + repository + service + controller
        ↓
[4] jatis-mahen-tester         → tulis test + jalankan mvn test
        ↓
[5] jatis-mahen-reviewer       → analisa hasil
        ├── PASS → generate PR description → selesai
        ├── bug di code → kembali ke coder (maks 3x retry total)
        └── bug di test → kembali ke tester (maks 3x retry total)
        ↓
    CHECKPOINT 2 ← dev review hasil (30-60 menit)
```

## Cara Jalankan Step by Step

### Step 1 — Spawn agent `jatis-mahen-orchestrator`
Input: TRD dari dev
Output: task summary (endpoint, DTO/entity yang dibuat, test cases, MongoDB collection)
**STOP — tampilkan summary ke dev. Tunggu konfirmasi "lanjut" sebelum Step 2.**

### Step 2 — Spawn agent `jatis-mahen-environment`
Input: task summary yang sudah dikonfirmasi
Output: konfirmasi mock siap, fixtures tersedia, interface validated

### Step 3 — Spawn agent `jatis-mahen-coder`
Input: task summary + output environment
Output: list file yang diubah/dibuat beserta kodenya

### Step 4 — Spawn agent `jatis-mahen-tester`
Input: kode dari coder
Output: TESTER REPORT (PASS/FAIL, coverage %, detail failure jika ada)

### Step 5 — Spawn agent `jatis-mahen-reviewer`
Input: TESTER REPORT
Output: FINAL REPORT (PASS dengan PR desc, atau instruksi fix ke coder/tester)

### Step 6 — Handle Loop
- reviewer minta fix ke coder → spawn `jatis-mahen-coder` (dengan instruksi fix) → spawn `jatis-mahen-tester` → spawn `jatis-mahen-reviewer`
- reviewer minta fix ke tester → spawn `jatis-mahen-tester` (dengan instruksi fix) → spawn `jatis-mahen-reviewer`
- Lacak retry counter. **Maksimal 3x total.** Setelah 3x masih gagal → STOP, tampilkan eskalasi ke dev.

### Step 7 — Final Report (CHECKPOINT 2)

```
=== JATIS DEV PIPELINE COMPLETE ===

Task     : [nama feature/endpoint]
Status   : PASS ✓ / ESCALATED ⚠
Retries  : X/3
Tests    : X passed, X failed
Coverage : X%

Files changed:
  - dev/src/main/java/.../dto/XxxDto.java
  - dev/src/main/java/.../entity/Xxx.java
  - dev/src/main/java/.../repository/XxxRepository.java
  - dev/src/main/java/.../service/XxxService.java
  - dev/src/main/java/.../controller/XxxController.java

--- PR Description ---
[isi PR description dari coder]

Dev: review code di atas lalu approve PR.
====================================
```

## Aturan
- **JANGAN skip Checkpoint 1** — konfirmasi task summary dulu sebelum lanjut
- Kirim **full output** antar agent — jangan ringkas, info yang hilang bikin agent berikutnya salah
- Sebutkan dengan jelas agent mana yang sedang dijalankan di setiap step
- Jika ada agent tidak bisa jalan (error), laporkan ke dev dan stop pipeline
- Pipeline ini khusus Java microservice — untuk Go service gunakan `go-wacall`
