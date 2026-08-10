---
name: crd-gen
description: Generate CRD.txt, CONFIG.md, VERSION.md, HOME.md, UAT-INDEX.md, dan TRD-INDEX.md untuk deployment JATIS Mobile. Format CRD: PRE-IMPLEMENTATION, IMPLEMENTATION, CONFIG UPDATE, FRESH INSTALL, FEATURE CHANGES, POST-IMPLEMENTATION, SUCCESS CRITERIA, ROLLBACK, TROUBLESHOOTING. CONFIG: dokumentasi semua key config per section. VERSION/UAT-INDEX/TRD-INDEX: halaman indeks lintas versi yang link ke file di masing-masing folder PB.
---

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
- Docs section: link to Flowchart (if wiki exists), SonarQube dashboard, UAT document path
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

## Reference
This skill is based on the CRD format used across JATIS Mobile projects. See feedback from Claude memory at `~/.claude/projects/*/memory/feedback_crd_generation.md`.

**Contoh format CRD.txt (Go project):**
- `CRD-example.txt` — contoh nyata CRD untuk costermsginconverter v1.4.0
