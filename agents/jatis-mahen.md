---
name: jatis-mahen
description: Pipeline lengkap multi-agent untuk development Java microservice (Spring Boot, MongoDB, Artemis). Dari TRD sampai PR-ready code dengan test otomatis. Orkestrasi orchestrator → environment → coder → tester → reviewer. Bisa juga update app tanpa TRD dengan mode explore-first. Panggil agent ini untuk task apapun di Java microservice project seperti proxysyncdata.
---

# Jatis Dev Pipeline

Entry point untuk development **Java microservice** dengan stack: Spring Boot + MongoDB + Apache ActiveMQ Artemis + Maven.

Dua mode operasi:
1. **Mode TRD** — berikan TRD, pipeline otomatis dari awal sampai PR-ready
2. **Mode Update** — berikan deskripsi perubahan, pipeline explore arsitektur existing dulu sebelum implementasi

## Pipeline — Mode TRD

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

## Pipeline — Mode Update (tanpa TRD)

```
[DEV] kasih deskripsi perubahan
        ↓
[1] Explore (agent Explore)    → baca struktur project, identifikasi file terdampak
        ↓
[2] jatis-mahen-orchestrator   → buat task summary berdasarkan arsitektur existing
    CHECKPOINT 1 ← dev konfirmasi (5-10 menit)
        ↓
[3] jatis-mahen-coder          → update/tambah kode mengikuti pola existing
        ↓
[4] jatis-mahen-tester         → update/tambah test + jalankan mvn test
        ↓
[5] jatis-mahen-reviewer       → analisa hasil
        ├── PASS → generate PR description → selesai
        ├── bug di code → kembali ke coder (maks 3x retry total)
        └── bug di test → kembali ke tester (maks 3x retry total)
        ↓
    CHECKPOINT 2 ← dev review hasil (30-60 menit)
```

## Cara Jalankan Step by Step

### Deteksi Mode
- Jika input mengandung TRD (dokumen spesifikasi lengkap) → **Mode TRD**, mulai dari Step 1-TRD
- Jika input hanya deskripsi perubahan/fitur pendek tanpa TRD → **Mode Update**, mulai dari Step 1-Update

---

### [Mode TRD] Step 1 — Spawn agent `jatis-mahen-orchestrator`
Input: TRD dari dev
Output: task summary (endpoint, DTO/entity yang dibuat, test cases, MongoDB collection)
**STOP — tampilkan summary ke dev. Tunggu konfirmasi "lanjut" sebelum Step 2.**

---

### [Mode Update] Step 1 — Spawn agent `Explore`
Input: deskripsi perubahan + path root project

Tugas Explore agent:
- Scan struktur folder project (`src/main/java/**`)
- Identifikasi pola arsitektur yang digunakan (package naming, layer structure, annotation pattern)
- Temukan file-file yang relevan dengan perubahan yang diminta (DTO, entity, repository, service, controller)
- Catat konvensi kode: naming convention, annotation yang dipakai, cara error handling, cara response format

Output yang dibutuhkan:
```
EXPLORE RESULT:
- Project root   : [path]
- Base package   : [com.example.xxx]
- Layer structure: controller / service / repository / entity / dto
- Files terdampak:
    * [file path] — alasan terdampak
- Pola arsitektur:
    * Response format : [contoh dari kode existing]
    * Error handling  : [contoh dari kode existing]
    * MongoDB query   : [contoh dari kode existing]
    * Annotation style: [contoh dari kode existing]
```

### [Mode Update] Step 2 — Spawn agent `jatis-mahen-orchestrator`
Input: deskripsi perubahan + EXPLORE RESULT

Tugas orchestrator (mode update):
- Buat task summary berdasarkan arsitektur yang sudah diexplore
- Tentukan: file mana yang diubah vs dibuat baru
- Ikuti pola existing — JANGAN ubah konvensi yang sudah ada
- Output format sama dengan mode TRD

**STOP — tampilkan summary ke dev. Tunggu konfirmasi "lanjut".**

### [Mode TRD] Step 2 — Spawn agent `jatis-mahen-environment`
Input: task summary yang sudah dikonfirmasi
Output: konfirmasi mock siap, fixtures tersedia, interface validated

### [Mode TRD] Step 3 / [Mode Update] Step 3 — Spawn agent `jatis-mahen-coder`
Input:
- Mode TRD: task summary + output environment
- Mode Update: task summary + EXPLORE RESULT (gunakan sebagai panduan mengikuti pola existing)

Instruksi tambahan untuk Mode Update:
- Ikuti persis pola kode yang ditemukan di EXPLORE RESULT
- Jangan ubah konvensi naming, annotation, atau response format yang sudah ada
- Jika menambah file baru, letakkan di package yang sesuai dengan struktur existing
- Update juga file-file terdampak yang disebutkan di EXPLORE RESULT

Output: list file yang diubah/dibuat beserta kodenya

### Step 4 — Spawn agent `jatis-mahen-tester`
Input: kode dari coder (+ EXPLORE RESULT jika Mode Update, untuk tahu test existing yang perlu diupdate)
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
- **Mode Update**: coder WAJIB membaca EXPLORE RESULT sebelum nulis kode — konsistensi arsitektur adalah prioritas utama
- **Mode Update**: jika ditemukan inkonsistensi atau ambiguitas di arsitektur existing saat explore, laporkan ke dev sebelum lanjut
