# Skill: bofis-gen

Generate Google Sheets timesheet + CRD documentation dari 1 baris data Bofis (PB/PMD). Input: baris Bofis (tab-separated). Output: timesheet row + CRD/CONFIG/VERSION/HOME/MONITORING-KPI.

## Cara Pakai

User tinggal paste 1 baris Bofis. Format input:
```
PB<number>	<task-title>	<dev>	<start>	<end>	<planned-mandays>	<actual-mandays>	<efficiency>	<understanding-min>	<prompting-min>	<implementation-min>	<testing-min>	<debugging-min>	<manual-dev-min>	<unit-test-min>	<documenting-min>	<ai-model>	<prompt-cycles>	<cost>	<token-amount>	<notes>
```

## Langkah 1 — Parse Bofis Row

Parse input (tab-separated atau paste manual) ke field:

| Field | Column |
|---|---|
| PB Number | col 0 |
| Task Title | col 1 |
| Developer | col 2 |
| Start Date | col 3 |
| End Date | col 4 |
| AI Estimated Mandays | col 5 |
| Planned Mandays | col 6 |
| Actual Mandays | col 7 |
| Efficiency | col 8 |
| Understanding (min) | col 9 |
| Prompting (min) | col 10 |
| Implementation (min) | col 11 |
| Testing (min) | col 12 |
| Debugging (min) | col 13 |
| Manual Dev (min) | col 14 |
| Unit Test (min) | col 15 |
| Documenting (min) | col 16 |
| AI Model | col 17 |
| Prompt Cycles | col 18 |
| Cost | col 19 |
| Token Amount | col 20 |
| Notes | col 21 |

## Langkah 2 — Generate Timesheet Row

Output format (tab-separated, langsung copas ke Google Sheets):

```
<PB>	<Task Title>	<Dev>	<Start>	<End>	<AI Est>	<Planned>	<Actual>	<Eff%>	<Understanding>	<Prompting>	<Implementation>	<Testing>	<Debugging>	<ManualDev>	<UnitTest>	<Documenting>	<Model>	<Cycles>	<Cost>	<Tokens>	<Notes>
```

## Langkah 3 — Generate CRD Documentation

Buat folder `doc/PB/<PB-folder>/` dan generate file berikut menggunakan pola dari skill `crd-gen`:

### Generate Nama Branch
Format: `feat/<PB-number>-<slug-task>` 
Slug: lowercase, replace spaces with `-`, remove special chars, max 80 chars.

### Generate PB Folder Name
Format: `<PB-number>-<slug-task>`

### Generate File List
1. **CRD.txt** — deployment guide (PRE-IMPLEMENTATION / IMPLEMENTATION / CONFIG UPDATE / FRESH INSTALL / FEATURE CHANGES / POST-IMPLEMENTATION / SUCCESS CRITERIA / ROLLBACK / TROUBLESHOOTING)
2. **CONFIG.md** — dokumentasi semua key config per section  
3. **VERSION.md** — changelog versi
4. **HOME.md** — project landing page + quickstart
5. **MONITORING-KPI.md** — Grafana dashboard, Prometheus alerts, health check
6. **UAT-INDEX.md** — indeks UAT artifacts
7. **TRD-INDEX.md** — indeks TRD documents

### Mengisi Data

Gunakan data dari Bofis row + Notes:

| Section CRD | Sumber Data |
|---|---|
| App Version | Dari pom.xml / git tag project |
| Branch | feat/<PB>-<slug> |
| SonarQube URL | Dari sonar-project.properties |
| Feature Changes | Dari Notes (kolom 21) |
| Success Criteria | Dari task description |
| Timesheet | Dari kolom 9-16 |
| AI Usage | Dari kolom 17-20 |
| Config Keys | Baca application.properties / config.yaml project |

## Langkah 4 — Ringkasan Output

Tampilkan:
1. Timesheet row (siap copas Google Sheets)
2. List file yg di-generate
3. Command git branch + commit

## Contoh

### Input:
```
PB1124271926001958YA	JATIS - Monitoring Alert WA Call - Update WebhookReceiver	Mahen	2026-07-29	2026-08-03	3	4	3	33.3	60	180	600	300	240	120	180	240	deepseek-v4-pro	30	$2.00	~1.5M	v1.3.0: MongoDB Alert + Prometheus Metrics + App Process Monitoring + 7 E2E tests + SonarQube 12/12 PASSED
```

### Output Timesheet:
```
PB1124271926001958YA	JATIS - Monitoring Alert WA Call - Update WebhookReceiver	Mahen	2026-07-29	2026-08-03	3	4	3	33.3	60	180	600	300	240	120	180	240	deepseek-v4-pro	30	$2.00	~1.5M	v1.3.0: MongoDB Alert + Prometheus Metrics + App Process Monitoring + 7 E2E tests + SonarQube 12/12 PASSED
```

### Output CRD Package:
```
doc/PB/PB1124271926001958YA-monitoring-alert-wa-call-update-webhookreceiver/
├── CRD.txt
├── CONFIG.md
├── VERSION.md
├── HOME.md
├── MONITORING-KPI.md
├── UAT-INDEX.md
└── TRD-INDEX.md
```
