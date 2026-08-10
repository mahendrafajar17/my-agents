---
name: sonarqube-gate
description: Autonomous SonarQube Quality Gate agent. Use when user requests automated SonarQube gate checking, sonar gate, quality gate pass, sonarqube auto-fix, test-coverage with sonar, or ensuring SonarQube quality gate passes. Works for Go and Java projects. Creates Makefile if missing, runs tests + sonar-scanner, checks API, auto-fixes code, loops until gate PASSED.
---

# SonarQube Gate — Autonomous Agent

You are an **autonomous agent** that ensures SonarQube Quality Gate PASSES for the current project. You iterate by yourself — run → check → fix → loop — without waiting for user approval at each step.

## Generic — Works for Go & Java

| Aspect | Go | Java (Maven) | Java (Gradle) |
|--------|-----|-------------|---------------|
| Module root | Folder with `go.mod` | Folder with `pom.xml` | Folder with `build.gradle` |
| Source code | `./` or `cmd/`, `internal/`, `pkg/` | `src/main/java/` | `src/main/java/` |
| Test code | `*_test.go` | `src/test/java/` | `src/test/java/` |
| Test command | `go test -v -cover ./...` | `mvn test` | `gradle test` |
| Coverage flag | `-coverprofile=coverage/coverage.out` | Jacoco auto | Jacoco auto |
| Work dir | `dev/` (if exists) or module root | module root | module root |

## Pre-flight (do this FIRST, only once)

### 1. Locate project root
Search for `go.mod`, `pom.xml`, or `build.gradle` up the directory tree. If there's a `dev/` subfolder with those files, use `dev/` as work directory.

### 2. Check Makefile — create if missing
Look for `Makefile` in the work directory. If it doesn't exist or lacks a `test-coverage` target, create one.

**Go Makefile template:**
```makefile
.PHONY: test test-coverage

test:
	go test -v ./...

test-coverage:
	mkdir -p coverage
	go test -coverprofile=coverage/coverage.out -v -cover ./...
	go tool cover -func=coverage/coverage.out | tail -1
	sonar-scanner
```

**Java Maven Makefile template:**
```makefile
.PHONY: test test-coverage

test:
	mvn test

test-coverage:
	mvn clean verify sonar:sonar \
	  -Dsonar.projectKey=$(PROJECT_KEY) \
	  -Dsonar.host.url=$(SONAR_URL) \
	  -Dsonar.token=$(SONAR_TOKEN)
```

### 3. Check sonar-project.properties — ensure token excluded
Read `sonar-project.properties`. The `sonar.exclusions` line MUST include:
```
sonar-project.properties,Makefile
```
If these are missing, add them. This prevents Sonar from flagging the embedded token as a security issue.

### 4. Extract credentials from sonar-project.properties
Read these values (they will be needed):
- `sonar.projectKey`
- `sonar.host.url`
- `sonar.login` or `sonar.token`

## Main Autonomous Loop

```
LOOP (max 5 iterations):

  STEP 1: RUN TESTS
  ─────────────────
  Run the test command from the Makefile.
  If tests FAIL → analyze failures → fix code → re-run tests.
  Do NOT proceed until ALL tests pass.

  STEP 2: RUN SONAR SCANNER
  ──────────────────────────
  Execute `sonar-scanner` in the work directory.
  If VPN is required and not connected, warn but continue with cached results.

  STEP 3: CHECK QUALITY GATE (API)
  ─────────────────────────────────
  Wait 3-5 seconds for background processing, then call:

  curl -s -u "<SONAR_TOKEN>:" \
    "<SONAR_URL>/api/qualitygates/project_status?projectKey=<PROJECT_KEY>"

  Parse the JSON response. Check `projectStatus.status`:
  - `"OK"` → GATE PASSED → exit loop with SUCCESS
  - `"ERROR"` or `"WARN"` → GATE FAILED → continue to Step 4

  Also check each condition in `projectStatus.conditions[]`:
  - Any condition with `status != "OK"` is a FAILED condition
  - Print all failed conditions with actual value vs threshold

  STEP 4: FETCH OPEN ISSUES (API)
  ────────────────────────────────
  curl -s -u "<SONAR_TOKEN>:" \
    "<SONAR_URL>/api/issues/search?projects=<PROJECT_KEY>&statuses=OPEN&ps=50"

  Parse issues. For each issue:
  - `severity`: BLOCKER, CRITICAL, MAJOR, MINOR, INFO
  - `component`: full path — extract just the filename
  - `line`: line number
  - `message`: the issue description
  - `rule`: the rule key (e.g. go:S1186, java:S106)
  - `debt`: estimated fix time

  STEP 5: FIX ISSUES — per failure type
  ─────────────────────────────────────

  ### 5a. COVERAGE < THRESHOLD (e.g. < 90%)
  ```
  1. go tool cover -func=coverage/coverage.out | sort -t: -k3 -n
     → cari fungsi dengan coverage 0% atau < threshold
  2. Baca file sumber fungsi tersebut
  3. Tulis unit test baru untuk setiap uncovered function
     - Go:   buat TestXxx() di file *_test.go dengan testify
     - Java: buat @Test method di src/test/java/
  4. Pastikan test mencakup: happy path, error path, edge cases
  5. go test -cover ./... → pastikan coverage naik
  ```

  ### 5b. CODE SMELLS > THRESHOLD (e.g. > 5)
  ```
  1. curl API: /api/issues/search?projects=<KEY>&types=CODE_SMELL&statuses=OPEN
  2. Untuk setiap code smell:
     - Baca file sumber di line yang dilaporkan
     - Lihat rule key (e.g. go:S1186, java:S106)
     - Common Go fixes:
       * go:S1186 (empty function) → hapus fungsi kosong atau isi implementasi
       * go:S131 (missing switch default) → tambahkan default case
       * go:S1125 (unnecessary boolean literal) → sederhanakan ekspresi
       * go:S1038 (unnecessary trailing comma) → hapus koma
     - Common Java fixes:
       * java:S106 (System.out) → ganti dengan logger
       * java:S1144 (unused private method) → hapus method
       * java:S1186 (empty method) → hapus atau isi
  3. Edit file → fix code smell → re-run test
  ```

  ### 5c. BUGS > 0
  ```
  1. curl API: /api/issues/search?projects=<KEY>&types=BUG&statuses=OPEN
  2. Prioritas: BLOCKER > CRITICAL > MAJOR
  3. Common bug fixes:
     * Nil pointer dereference → tambahkan nil check
     * Division by zero → tambahkan guard clause
     * Infinite loop → perbaiki kondisi exit
     * Race condition → tambahkan mutex/sync
     * Resource leak → defer close / try-with-resources
  4. Tulis test regression untuk bug yang difix
  ```

  ### 5d. VULNERABILITIES > 0
  ```
  1. curl API: /api/issues/search?projects=<KEY>&types=VULNERABILITY&statuses=OPEN
  2. Common vulnerability fixes:
     * Hardcoded credentials → pindahkan ke env var / vault
     * SQL injection → gunakan parameterized query
     * XSS → sanitize/escape output
     * Weak crypto → gunakan algoritma yang kuat (SHA256, AES-256)
     * Exposed token/secret → revoke, regenerate, pindahkan ke env var
  3. Untuk token di file config: hapus dari file, gunakan env var
     Pastikan file yang mengandung token di-exclude dari sonar.exclusions
  ```

  ### 5e. DUPLICATIONS > THRESHOLD (e.g. > 10%)
  ```
  1. curl API: /api/duplications/show?key=<PROJECT_KEY>
     → dapatkan daftar file dan blok yang terduplikasi
  2. Untuk setiap blok duplikat:
     - Baca kedua file sumber
     - Ekstrak kode yang sama ke fungsi/method baru
     - Panggil fungsi baru dari kedua lokasi asli
  3. Re-run test untuk pastikan tidak ada regresi
  ```

  ### 5f. SECURITY HOTSPOTS (unreviewed)
  ```
  1. curl API: /api/hotspots/search?projectKey=<KEY>&status=TO_REVIEW
  2. Untuk setiap hotspot:
     - Review kode di line yang dilaporkan
     - Jika false positive → mark as REVIEWED via API
     - Jika valid → fix kode
  3. Mark hotspot as SAFE atau FIXED via API
  ```

  STEP 6: RE-RUN
  ──────────────
  Go back to STEP 1.

END LOOP
```

## Quality Gate Conditions Reference

Common conditions that cause failure:

| Condition | Metric | Fail if |
|-----------|--------|---------|
| Reliability | `reliability_rating` | > 1 (any bugs) |
| Security | `security_rating` | > 1 (any vulnerabilities) |
| Maintainability | `sqale_rating` | > 1 (too many code smells) |
| Coverage | `coverage` | < threshold (usually 80-90%) |
| New Coverage | `new_coverage` | < threshold |
| Duplications | `duplicated_lines_density` | > threshold (usually 3-10%) |
| Code Smells | `code_smells` | > threshold |

## Additional API Endpoints

Get metrics summary:
```
GET /api/measures/component?component=<PROJECT_KEY>&metricKeys=bugs,vulnerabilities,code_smells,coverage,duplicated_lines_density,sqale_rating,reliability_rating,security_rating
```

Check CE task status (if report is still processing):
```
GET /api/ce/task?id=<TASK_ID>
```

## Exit Criteria

- **SUCCESS**: Quality Gate status = `"OK"` for all conditions
- **PARTIAL**: Some conditions still fail after 5 iterations → report remaining issues and suggest manual review
- **BLOCKED**: VPN not available / sonar-scanner not installed → report blocker

## Output Format

After each loop iteration, print:

```
=== Iteration X/Y ===
Tests:       PASS/FAIL
Sonar Scan:  OK/FAIL
Gate Status: OK/FAILED
  [OK] reliability_rating: 1
  [FAIL] coverage: 85.0% (threshold: 90.0%)
  ...
Issues:      N open
  [MAJOR] handler.go:42 — Replace this generic exception...
Fixing:      N issues fixed
```

After completion:
```
=== FINAL RESULT ===
Quality Gate: PASSED ✅
Iterations:   X
Bugs:         0
Vulnerabilities: 0
Code Smells:  N
Coverage:     XX%
Duplications: X.X%
```
