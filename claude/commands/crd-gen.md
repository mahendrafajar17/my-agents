# Skill: crd-gen

## When to Use
Use this skill when the user asks to create a CRD.txt for a PB (Project Brief) in JATIS Mobile projects. The CRD is the deployment guide for production operations team.

## Prerequisites
- Git repository with committed changes
- Maven-based Java project (typically)
- Existing `doc/<PB-folder>/` directory with UAT.md or E2E-EVIDENCE.md
- Access to `pom.xml` for app version

---

## Step 1: Gather Project Information

Run these commands to collect necessary data:

```bash
# App version
grep '<version>' dev/pom.xml | head -1 | sed 's/.*<version>\(.*\)<\/version>.*/\1/'

# Active branch
git branch --show-current

# Repository URL
git remote get-url origin

# Changed files overview
git diff HEAD --stat

# Full diff of source/config files
git diff HEAD -- dev/pom.xml "*.conf" dev/src/main/java/ dev/src/main/resources/

# Commit history
git log --oneline -5
```

## Step 2: Read Existing Project Documentation

Read the following files if they exist in `doc/<PB-folder>/`:
- `UAT.md` — test results, feature list, file changes
- `E2E-EVIDENCE.md` — protocol-level evidence
- `VERIFICATION-REPORT.md` — build verification

These provide the content for FEATURE CHANGES and CONFIGURATION UPDATE PROCEDURE sections.

## Step 3: Check Key Configuration Files

Read both copies of the main config file (typically `DBExecutor.conf` or `application.properties`):
- `dev/src/main/resources/<config>` — development template
- `bin/<config>` — production deployment config

Compare to identify new properties added between versions.

## Step 4: Check Build Artifacts

List `bin/` directory to identify:
- New JAR filename (e.g., `DBExecutor-2.1.0.jar`)
- Old JAR filename (e.g., `DBExecutor-2.0.jar`)
- Startup script (`start.bat` / `*.sh`)
- Any config files requiring update

## Step 5: Generate CRD.txt

Create the file at `doc/<PB-folder>/CRD.txt` using the template below.

### CRD.txt Template

```
====================================
PRE-IMPLEMENTATION PROCEDURE
====================================
Repository URL : <git-remote-url>
TRD            : <git-blob-url-to-trd-md, or omit this line if no TRD exists>
SonarQube URL  : (tidak ada / URL jika ada)
App Version    : <version>
Branch         : <branch-name>

====================================
IMPLEMENTATION PROCEDURE
====================================
1. Stop service:
   - Linux: ./DBExecutor.sh stop-all
   - Windows: ctrl+c pada command prompt yang menjalankan start.bat

2. Backup file existing:
   cp bin/<old-jar> bin/backup/<old-jar>.bak
   cp bin/<config> bin/backup/<config>.bak

3. Copy <new-jar> ke bin/

4. Update <config> di bin/ sesuai CONFIGURATION UPDATE PROCEDURE di bawah

5. Update startup script jika diperlukan:
   - Linux (DBExecutor.sh): ubah variabel "app" ke <new-jar>
   - Windows (start.bat): ubah nama JAR ke <new-jar>

6. Start service:
   - Linux: ./DBExecutor.sh start
   - Windows: jalankan start.bat

7. Cek log: tail -f out-DBExecutor.log

====================================
CONFIGURATION UPDATE PROCEDURE (v<old-version> -> v<new-version>)
====================================
Tambahkan property berikut di <config>:

<list new properties with descriptions>

====================================
FRESH INSTALLATION PROCEDURE
====================================
<Full config template for fresh install + step-by-step>

====================================
FEATURE CHANGES v<new-version>
====================================
<Detailed feature changes based on git diff and UAT.md>

====================================
POST-IMPLEMENTATION PROCEDURE
====================================
1. Cek log out-DBExecutor.log, pastikan tidak ada ERROR
2. Cek koneksi MQ: pastikan log "Opening for sending/receiving" muncul
3. Cek koneksi DB: pastikan log "connect to jdbc:..." muncul
4. Kirim test message manual (opsional) untuk verifikasi full flow
5. Cek monitoring alert (email) jika dikonfigurasi

====================================
SUCCESS CRITERIA
====================================
1. Service berjalan tanpa ERROR di log
2. Koneksi MQ established (activemq/artemis)
3. Koneksi JDBC ke MySQL established
4. Full flow MQ → DB berhasil (INSERT query tereksekusi)
5. Tidak ada regression pada fitur existing

====================================
ROLLBACK PROCEDURE
====================================
1. Stop service: ./DBExecutor.sh stop-all
2. Kembalikan <config> dari backup:
   cp bin/backup/<config>.bak bin/<config>
3. Kembalikan JAR dari backup atau copy <old-jar>:
   cp bin/backup/<old-jar> bin/<old-jar>
4. Kembalikan startup script ke versi sebelumnya
5. Start service: ./DBExecutor.sh start

====================================
TROUBLESHOOTING GUIDE
====================================
<Common issues and solutions based on component changes>
```

### Rules
- PRE-IMPLEMENTATION PROCEDURE must contain **only** these 5 fields, in this order: Repository URL,
  TRD, SonarQube URL, App Version, Branch. Omit the TRD line entirely if no TRD exists for the
  project.
- Do **not** add PB/PMD Number, PB/PMD Name, Project name, Owner, Tech Lead, Developer, target
  Server, or a notification/email list to PRE-IMPLEMENTATION PROCEDURE — even if that data is
  available (e.g. pasted from a Bofis PB/PMD form). That metadata belongs in Bofis/PB tracking, not
  in the CRD's pre-implementation block, which is a deployment-ops checklist, not a project-tracking
  summary.

## Step 5b: Generate VERSION.md

Create the file at `doc/<PB-folder>/VERSION.md` using the format below. If the file already exists, append the new version entry at the top.

### VERSION.md Template

```
## vX.Y.Z (DD Mon YYYY):

```
- Feature/change description 1
- Feature/change description 2
- Feature/change description 3
- SonarQube Quality Gate PASSED (coverage XX%, 0 bugs, 0 vuln, X code smells)
- Docs: TRD, UAT, E2E-LOG-EVIDENCE, CRD
```

## vX.Y.Z (DD Mon YYYY):

```
- ...
```
```

### Rules
- Each version entry is a `## vX.Y.Z (DD Mon YYYY):` header followed by a fenced code block with `-` bullet points
- List all new features, config changes, model changes, test additions
- Include SonarQube quality gate summary at the end
- Include list of docs generated for this version
- Keep bullet points concise — one line per change
- Order versions newest-to-oldest (latest version at top)

---

## Step 5c: Generate CONFIG.md

Create the file at `doc/<PB-folder>/CONFIG.md` using the format below. Read the actual config file (`config.json`, `config.yaml`, `application.properties`, etc.) and document every key grouped by section.

### CONFIG.md Template

```markdown
# Configuration File Explanation
This section explains the keys and values in the `<config-file>` used for the `<Project Name>` service.

## <Section 1> Configuration
- **<section>.<key>**: <description> (`<default/value>`).

## <Section 2> Configuration
- **<section>.<key>**: <description> (`<default/value>`).
```

### Rules
- Group config keys by section (e.g., MongoDB, AMQP, Telegram, Logger, Application)
- Each key: `- **<full.key.path>**: <what it does> (\`<example value>\`).`
- Read the actual config file to get exact key names and structures
- For nested keys, use dot notation (e.g., `mongodb.collection.credential`)
- Include ALL keys — not just new ones
- Place default/example values in backtick code formatting
- Section headers use `## <Section Name> Configuration` format

---

## Step 5d: Generate HOME.md

Create the file at `doc/<PB-folder>/HOME.md` as the project landing page / onboarding doc.

### HOME.md Template

```markdown
# <Project Title>

<One-liner description of what the service does>

# Docs

- [SonarQube](<sonarqube-dashboard-url>)
- [Config](https://git-rbi.jatismobile.com/<repo-path>/-/wikis/Config)
- [Version](https://git-rbi.jatismobile.com/<repo-path>/-/wikis/Version)
- [UAT](https://git-rbi.jatismobile.com/<repo-path>/-/wikis/UAT)
- [E2E Log Evidence](https://git-rbi.jatismobile.com/<repo-path>/-/wikis/E2E-Log-Evidence)
- [CRD](https://git-rbi.jatismobile.com/<repo-path>/-/wikis/CRD)
- [TRD](https://git-rbi.jatismobile.com/<repo-path>/-/blob/<branch>/doc/<PB-folder>/trd-<pb-name>.md)

## Requirements

- Golang <version> (for development)
- Docker & Docker Compose
- MongoDB <version>

## Development

\`\`\`bash
go mod tidy
cp cmd/config.json.example cmd/config.json
cd cmd
go run main.go
\`\`\`

## Data Example

### Request Payload (`POST /app`)

\`\`\`json
{
  "account": { "account": "<phone>" },
  "data": {
    "entry": [{
      "changes": [{
        "field": "calls",
        "value": {
          "contacts": [{ "wa_id": "<phone>" }],
          "calls": [{ "direction": "USER_INITIATED", "event": "connect" }]
        }
      }]
    }]
  },
  "event": "wa-call"
}
\`\`\`

### Response

\`\`\`json
{
  "message": "Request received",
  "transaction_id": "<uuid>"
}
\`\`\`

## Unit Test

### Unit test only

\`\`\`bash
go test ./... -v
\`\`\`

### Unit test with coverage (SonarQube)

\`\`\`bash
go test -coverprofile=coverage/coverage.out -covermode=count -v ./...
go tool cover -func=coverage/coverage.out | tail -1
\`\`\`

## E2E Test

\`\`\`bash
make e2e
\`\`\`

## SonarQube Scan

\`\`\`bash
export SONAR_TOKEN=<token>
make test-coverage
\`\`\`

## Deployment

### Binary (Linux)

\`\`\`bash
make build-centos
cp bin/<binary> <target-server>:~
ssh <target-server>
./launcher.sh stop-all
cp <binary> bin/
# Update launcher.sh: app="<binary>"
./launcher.sh start
\`\`\`

### Docker Build

\`\`\`bash
docker build -t <image-name>:<version> .
docker run -d --name <container-name> -v "$(pwd)"/logs:/app/logs <image-name>:<version>
\`\`\`
```

### Rules
- Title: project name exactly as in git
- Description: one line — what the service consumes, what it produces
- Docs section: **every entry except TRD must be a GitLab wiki link** —
  `https://git-rbi.jatismobile.com/<repo-path>/-/wikis/<PageName>` (no `.md` extension, no relative
  file path). This applies to Config, Version, UAT, E2E Log Evidence, CRD, and any other doc mirrored
  to the wiki. **TRD is the one exception**: it stays as a direct git blob link
  (`.../-/blob/<branch>/doc/<PB-folder>/trd-<pb-name>.md`) since the TRD is not mirrored to the wiki.
  Never link any Docs entry to a relative in-repo path (e.g. `CONFIG.md`, `UAT-x.md`) — always the
  full wiki URL.
- Data Example: show real request + response payload
- Commands: all in bash code blocks, use actual project paths
- Deployment: include both binary and Docker methods if applicable

---

## Step 5e: Generate UAT-INDEX.md

Create the file at `doc/<PB-folder>/UAT-INDEX.md` as the UAT artifact index page. This indexes all UAT artifacts (UAT.md, E2E-EVIDENCE.md, xlsx, drawio) across versions, linking to files inside each PB folder.

### UAT-INDEX.md Template

```markdown
# UAT — <Project Name>

- **vX.Y.Z** — (DD Mon YYYY) — [UAT.md](<link-to-PB-folder/UAT.md>)
- **vX.Y.Z** — (DD Mon YYYY) — [E2E-EVIDENCE.md](<link-to-PB-folder/E2E-EVIDENCE.md>)
- **vX.Y.Z** — (DD Mon YYYY) — [UAT_<Project>.xlsx](<link-to-xlsx>)
- **vX.Y.Z** — (DD Mon YYYY) — [Flowchart_<Project>_vX.Y.Z.drawio](<link-to-drawio>)
```

### Rules
- Title: `# UAT — <Project Name>`
- One bullet per artifact, format: `- **vX.Y.Z** — (DD Mon YYYY) — [<artifact-name>.<ext>](<link>)`
- Artifacts: drawio (flowchart), xlsx (UAT evidence), pdf, UAT.md, E2E-EVIDENCE.md
- Link directly to the file in the respective PB folder on the repo
- Append new entries at the top, keep older versions below

---

## Step 5f: Generate TRD-INDEX.md

Create the file at `doc/<PB-folder>/TRD-INDEX.md` as the TRD document index page. This indexes all TRD documents across versions, linking to TRD.md files inside each PB folder.

### TRD-INDEX.md Template

```markdown
# TRD — <Project Name>

- **vX.Y.Z** — (DD Mon YYYY) — [TRD.md](<link-to-PB-folder/TRD.md>)
```

### Rules
- Title: `# TRD — <Project Name>`
- One bullet per TRD document, format: `- **vX.Y.Z** — (DD Mon YYYY) — [TRD.md](<link>)`
- Link to TRD.md inside the respective PB folder on the repo
- Append new entries at the top, keep older versions below
---

## TRD Types

crd-gen supports **two types** of TRD depending on when the document is created:

| Type | When | Source Data | Section |
|---|---|---|---|
| **TRD-PRE** | Before implementation (new feature, extension, greenfield) | User requirements, research, existing architecture | Step 5g |
| **TRD-POST** | After implementation (done feature, release) | UAT.md, git diff, CRD.txt FEATURE CHANGES | Step 5h |

### Key Differences

| Aspect | TRD-PRE | TRD-POST |
|---|---|---|
| Purpose | Guide development — what to build | Document release — what was built |
| Audience | Developer + Architect | Developer + Ops + Stakeholder |
| Content | Architecture, metrics, API spec, config, acceptance criteria | File changes, test results, deployment notes |
| Flow diagrams | Mermaid.js diagrams | Mermaid.js flowcharts |
| Data model | Full schema design | Schema changes (diff) |

---

## Step 5g: Generate TRD-PRE (Pre-Implementation)

Use this when the user asks to create a TRD for a **new feature, extension, or greenfield project** that hasn't been built yet. Content is derived from user requirements, research, and existing architecture docs.

### TRD-PRE Template

Create the file at `doc/<PB-folder>/TRD-<Feature-Name>.md`.

```markdown
# Technical Requirements Document (TRD)
## <Project Name> — <Feature Name>

---

### Document Metadata

| Field | Value |
|---|---|
| Project Name | <full project name> |
| Document Version | 1.0-draft |
| Last Updated | YYYY-MM-DD |
| Status | Draft / Research / Final |
| Owner | <author name> |
| Parent TRD | <link to parent if extension> |

### Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0-draft | YYYY-MM-DD | Initial draft. |

---

## 1. Overview

### 1.1 Purpose
<One paragraph: what this service/feature does and why>

### 1.2 Goals
- <Goal 1>
- <Goal 2>

### 1.3 Non-Goals
- This service does **NOT** ...
- This service does **NOT** ...

### 1.4 Relationship to Parent System (if extension)
<How this fits with existing components. Sidecar? Replacement? New goroutine?>

---

## 2. Technology Stack

| Layer | Technology |
|---|---|
| Language | Go / Java |
| Framework | Gin / Spring Boot |
| Database | MongoDB / MySQL / PostgreSQL |
| Other | tcpdump, gopacket, Kafka, etc. |

---

## 3. System Architecture

### 3.1 High-Level Component Diagram

\`\`\`mermaid
flowchart TD
    A[Component 1<br/>goroutine] -->|writes| B[(Database)]
    C[Component 2<br/>scheduler] -->|reads| B
    D[Component 3<br/>alert] -->|reads| B
    D -->|sends| E[Telegram/Email]
    F[Component 4<br/>API] -->|queries| B
\`\`\`

### 3.2 Component Responsibilities

| # | Component | Type | Purpose |
|---|---|---|---|
| 1 | <Name> | goroutine / endpoint | <what it does> |
| 2 | <Name> | scheduler / worker | <what it does> |

---

## 4. Functional Specifications

### 4.1 Component N: <Name>

**Type:** Long-running goroutine / Scheduled / HTTP handler

#### 4.1.1 Purpose
<1-2 sentences>

#### 4.1.2 Behavior
- <Step 1>
- <Step 2>

#### 4.1.3 Key Algorithms / Formulas (if applicable)
\`\`\`
<formula or pseudo-code>
\`\`\`

---

## 5. REST API Specification (if applicable)

### 5.1 Endpoints

\`\`\`
METHOD /api/v1/resource
\`\`\`

### 5.2 Query Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|

### 5.3 Response Format

\`\`\`json
{
  "success": true,
  "data": []
}
\`\`\`

### 5.4 Error Responses

| Error | HTTP | Body |
|---|---|---|

---

## 6. Data Model

### 6.1 Collection / Table: `<name>`

\`\`\`json
{
  "_id": ObjectId("..."),
  "field": "value"
}
\`\`\`

**Indexes:**
- `{ field: 1 }`
- `{ field: 1, created_at: -1 }`

---

## 7. Configuration

\`\`\`yaml
section:
  key: "value"  # description
\`\`\`

---

## 8. Logging

| Logger | Components | File |
|---|---|---|

---

## 9. Build & Deployment

- Build command, binary name, launcher
- New system dependencies
- Runtime toggle if applicable

---

## 10. Performance Considerations

| Concern | Mitigation |
|---|---|

---

## 11. Acceptance Criteria

- [ ] Criterion 1
- [ ] Criterion 2

---

## 12. Open Questions & Risks

1. **<Topic>** — <discussion point>

---

## 13. Implementation Phasing (optional)

### Phase 1 — <Name> (week 1-2)
- <tasks>

### Phase 2 — <Name> (week 2-3)
- <tasks>

---

*End of Document*
```

### Rules for TRD-PRE
- Title format: `# Technical Requirements Document (TRD)\n## <Project Name> — <Feature Name>`
- Version starts as `1.0-draft`, graduates to `1.0` when approved
- Include **Changelog** from the start
- **Non-Goals** are critical — explicitly list what is OUT of scope
- Use **Mermaid diagrams** for component architecture — renderable in GitLab/GitHub. Use `flowchart` for component layouts, `sequenceDiagram` for call flows, `graph` for state transitions
- Include **formulas and algorithms** if the feature involves calculations (MOS, packet loss, jitter, etc.)
- **Configuration** block should show the new/extended config structure
- **Acceptance Criteria** should be testable checkboxes
- **Open Questions** section for items that need stakeholder input before implementation
- For Go microservice projects: use Go-specific patterns (goroutines, channels, context)
- For Java Spring Boot projects: use Spring patterns (beans, services, controllers)

### Example TRD-PRE files
- `TRD-WaCall-RTP-Detection.md` — full engine spec (Go, MongoDB, tcpdump)
- `TRD-WaCall-RTP-Audio-Quality.md` — extension spec (Go, gopacket, RTP parsing)

---

## Step 5h: Generate TRD-POST (Post-Implementation)

Create the file at `doc/<PB-folder>/TRD.md` as the Technical Requirement Document. Content is derived from UAT.md, git diff, and CRD.txt FEATURE CHANGES. Use this AFTER implementation is done.

### TRD-POST Template

```markdown
# <PB-number> -- <Client> - <Feature> - TRD

## 1. Introduction

### 1.1 Background
<Why this change is needed — problem statement, business context>

### 1.2 Objectives
- <Objective 1>
- <Objective 2>

### 1.3 Scope
<What is covered>

### 1.4 Out of Scope
<What is NOT covered>

## 2. Architecture

### 2.1 System Context
<Description of how the service fits in the overall system>

### 2.2 Component Changes
| Component | Before | After |
|-----------|--------|-------|
| <component> | <old> | <new> |

## 3. Data Layer

### 3.1 Database
<Database type, version, schema info>

### 3.2 Schema Changes
| Table | Change | Description |
|-------|--------|-------------|
| <table> | <change> | <desc> |

## 4. Detailed Specification

### 4.1 <Feature/Module Name>
<Detailed explanation of the change>

**Configuration:**
\`\`\`properties
<config snippet>
\`\`\`

**Flow:**
\`\`\`mermaid
flowchart TD
  A[Start] --> B[Process]
  B --> C[End]
\`\`\`

### 4.2 <Feature/Module Name>
...

## 5. Files Changed

| File | Change |
|------|--------|
| <file> | <description> |

## 6. Testing

| Test | Result |
|------|--------|
| <test> | <PASSED/FAILED> |

## 7. Deployment Notes
<Deployment notes from CRD.txt>
```

### Rules for TRD-POST
- Title format: `<PB-number> -- <Client> - <Feature> - TRD`
- Derive from existing CRD.txt FEATURE CHANGES and UAT.md
- Include mermaid.js flowcharts for key flows if applicable
- List all file changes from UAT.md "Daftar File Berubah"
- Include test results summary from UAT.md
- Use technical depth appropriate for developer audience
- Keep concise but complete — 1-2 paragraphs per section
- Renumbered from old Step 5g

---

## Step 6: Fill Sections Based on Project Data

### FEATURE CHANGES
Extract from `UAT.md` section "Daftar File Berubah" and `git diff`. Group by:
- Library/dependency upgrades
- New features
- Config additions
- Bug fixes

### CONFIGURATION UPDATE PROCEDURE
List ONLY the new/added properties (not the full config). Each property should have:
- Key name
- Description of what it does
- Default/example value
- When it's needed

### TROUBLESHOOTING GUIDE
Based on the components changed:
- MySQL 8 specific: `allowPublicKeyRetrieval`, `useSSL`, `serverTimezone`
- Artemis: authentication, protocol mismatch
- ActiveMQ Classic: version compatibility

---

## Step 7: Generate MONITORING-SHEET.txt (PB Timesheet + AI Efficiency, TSV for Google Sheets)

Create the file at `doc/<PB-folder>/MONITORING-SHEET.txt` — one TSV row per Bofis sheet, ready to paste
directly into Google Sheets. Covers TWO separate sheets that share the same PB, so generate both in
one file rather than asking the user twice.

### Step 7a: Get real token/cost data via `tokentracker` CLI — do NOT estimate

This project has `tokentracker-cli` installed globally (npm). Use it to get **real** usage numbers
instead of guessing:

```bash
tokentracker sessions --from <PB-start-date> --to <PB-end-or-today-date> --format json
```

- `--from`/`--to` should span the PB's actual start date through the last work date (today, or the
  PB's declared end date if work already finished).
- The JSON has a top-level `sessions` array. **Filter to `project_key` matching this repo's folder
  name** (e.g. `end2endmonitoring-engine`) — the array includes sessions from unrelated projects too.
- **Sum across ALL matching entries**, not just the first one. A single logical work session often
  appears as multiple array entries sharing the same `session_hash`: one long-running entry for the
  main conversation thread, plus one short entry per subagent/background agent spawned during the
  work (each subagent call is tracked as its own session with `turns: 1`). Missing the subagent
  entries under-counts tokens/cost significantly.
- Sum these fields across the filtered entries: `tokens.input_tokens`, `tokens.cached_input_tokens`
  (cache read), `tokens.cache_creation_input_tokens` (cache write), `tokens.output_tokens`,
  `total_tokens`, `cost_usd`. Use the **main thread's own `turns`** value (the single longest-duration
  entry, typically the one spanning the full PB date range) as `prompt_cycle` — do not sum `turns`
  across subagent entries, since each subagent counts as `turns: 1` regardless of how much work it
  actually did internally.
- `ai_model`: read from the main entry's `model` field (e.g. `claude-sonnet-5`). If mixed models were
  used across entries, note the primary one and mention the mix in the notes.
- Quick one-liner to compute the sums (adjust the `project_key` filter and date range):
  ```bash
  tokentracker sessions --from <start> --to <end> --format json | python3 -c "
  import json, sys
  d = json.load(sys.stdin)
  s = [x for x in d['sessions'] if x.get('project_key') == '<repo-folder-name>']
  main = max(s, key=lambda x: x.get('duration_ms', 0))
  print('input:', sum(x['tokens']['input_tokens'] for x in s))
  print('cache_read:', sum(x['tokens']['cached_input_tokens'] for x in s))
  print('cache_write:', sum(x['tokens']['cache_creation_input_tokens'] for x in s))
  print('output:', sum(x['tokens']['output_tokens'] for x in s))
  print('total_tokens:', sum(x['total_tokens'] for x in s))
  print('cost_usd: %.2f' % sum(x['cost_usd'] for x in s))
  print('prompt_cycle (main entry turns):', main['turns'])
  print('ai_model (main entry):', main['model'])
  "
  ```
- If `tokentracker` is not installed or `sessions` returns empty/unavailable for the date range,
  say so explicitly in the notes and fall back to a clearly-labeled estimate — never silently
  present a guess as if it were measured data.

### Step 7b: MONITORING-SHEET.txt Template

Column layouts below are CONFIRMED against the real sheets (verified 2026-08-13, not guessed).

```
# ============================================================
# Sheet 1 — PB Timesheet (Bofis v2.0)
# Header: PB  Task  Start  End  Status  SIT  Coverage Unit Test  SonarQube Status  Assigned Dev  Techlead  Efficiency %  Planned Mandays  Actual Mandays  Understanding the Flow  Prompting  Implementation  Testing  Debugging  Manual Development  Unit Test  Documenting  AI Token Usage  AI Model  Prompt Cycle  Cost  Notes
# ============================================================
<PB-number>	<PB-title>	<start>	<end>	Meet	- - -	<coverage 2dp>%	<sonar_status TitleCase>	<dev TitleCase>	<tl TitleCase>	<efficiency 2dp>	<planned_md>	<actual_md>	<understand>	<prompting>	<implementation>	<testing>	<debugging>	<manual_dev>	<unit_test>	<documenting>	<total_tokens>	<ai_model>	<prompt_cycle>	$<cost>	<notes>

# ============================================================
# Sheet 2 — AI Efficiency
# Header: PB No  PB Title  Dev  Start  End  AI Estimated Mandays  UH Planned Mandays  Actual Mandays  Efficiency (%)  Understanding the Flow  Prompting  Implementation  Testing  Debugging  Manual Development  Unit Test  Documenting  AI Model  Prompt Cycle  Cost  Token Amount
# ============================================================
<PB-number>	<PB-title>	<dev TitleCase>	<start>	<end>		<planned_md>	<actual_md>	<efficiency>	<understand>	<prompting>	<implementation>	<testing>	<debugging>	<manual_dev>	<unit_test>	<documenting>	<ai_model>	<prompt_cycle>	$<cost>	<input>k input, <output>k output, <cache_read>m cache read, <cache_write>m cache write ($<cost>)

# ============================================================
# Catatan (cek/isi manual sebelum submit ke Bofis)
# ============================================================
# - Status (Sheet 1) = "Meet" — this is the DEFAULT value for this column on every row seen in the
#   real sheet, not an indicator that the SIT/System phase has already happened. Always "Meet" unless
#   told otherwise (other observed values: "Late Submission", "Other" — only use those if the user
#   says the PB was actually late or is a non-dev task).
# - SIT (Sheet 1) = "- - -" — the sheet's own placeholder for "not yet in SIT", used on effectively
#   every row until SIT actually starts. Other observed values once SIT starts: "Queued",
#   "SIT OK 1 Cycle", "Research". Default to "- - -" unless the user confirms SIT status.
# - Assigned Dev / Techlead (Sheet 1) and Dev (Sheet 2): Title Case short first name (Mahen, Wisnu,
#   Faisol, Boy, Syarif, Oksa, Eka, Huda, Iqbal, Narko) — every real row uses Title Case, never
#   lowercase and never the dotted username (mahendra.fajar).
# - Coverage Unit Test & Efficiency % (Sheet 1): 2 decimal places ("92.20%", "0.00", "33.33"), not
#   1 decimal or bare integers.
# - SonarQube Status: Title Case ("Passed"/"Failed"), never ALL CAPS.
# - AI Token Usage (Sheet 1) = bare total token count as a plain number (e.g. 219525075). This is
#   DIFFERENT from Sheet 2's Token Amount, which uses the input/output/cache-read/cache-write
#   breakdown string — the two sheets intentionally use different formats for this field, don't
#   make them match.
# - efficiency = (planned_md - actual_md) / planned_md * 100. If actual is set equal to planned per
#   user instruction, efficiency = 0 (write "0.00" on Sheet 1, "0" on Sheet 2).
# - AI Estimated Mandays (Sheet 2) is left blank unless a genuinely separate AI-estimated mandays
#   figure was recorded (distinct from planned mandays).
# - AI Token Usage/Token Amount/Cost: REAL DATA from `tokentracker sessions` (see Step 7a) — state
#   the exact command run and how many session entries were combined (main thread + subagent count).
# - The 8 minute columns (Understanding the Flow ... Documenting): if there is no per-category
#   stopwatch record, use a proportional estimate based on the actual scope of work and LABEL it as
#   an estimate in the notes — never present a guess as if it were measured data.
```

### Rules
- Generate BOTH sheets in one file — user tracks the same PB across two separate Google Sheets.
- Never fabricate `ai_tokens`/`cost`/`prompt_cycle` — always derive them from `tokentracker sessions`
  per Step 7a. If unavailable, say so and mark the fallback as an estimate.
- Keep the file paste-ready: each sheet's data row must be a single physical line (TSV), with the
  header shown only as a comment above it for reference, not as part of what gets pasted.
- If the user pastes real header/example rows from their actual sheet (as opposed to relying on this
  template), always defer to what they pasted — sheet formatting conventions (status placeholder
  values, name casing, decimal precision) can differ across teams/sheets and the pasted ground truth
  always wins over this template's defaults.

---

## Reference
This skill is based on the CRD format used across JATIS Mobile projects. See feedback from Claude memory at `~/.claude/projects/*/memory/feedback_crd_generation.md`.

**Contoh format CRD.txt (Go project):**
- `~/.claude/docs/CRD-example.txt` — contoh nyata CRD untuk costermsginconverter v1.4.0
- `~/.claude/docs/MONITORING-SHEET-example.txt` — contoh nyata gabungan Sheet 1 (PB Timesheet) + Sheet 2
  (AI Efficiency) untuk end2end_monitoring_engine PB1124271926002213YA, termasuk cara memakai
  `tokentracker` CLI untuk data token/cost asli.
