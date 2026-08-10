# My Agents

Koleksi agents, skills, commands, dan memory untuk OpenCode dan Claude Code.

## Structure

```
my-agents/
├── claude/          # Agents, commands, skills, memory untuk Claude Code
│   ├── agents/      # mirror ~/.claude/agents
│   ├── commands/    # mirror ~/.claude/commands
│   ├── skills/      # mirror ~/.claude/skills (flat .md, tanpa frontmatter)
│   └── memory/      # mirror auto-memory Claude Code (MEMORY.md + file per topik)
├── opencode/        # Agents & skills untuk OpenCode
│   ├── agents/      # mirror ~/.config/opencode/agents
│   └── skills/      # mirror ~/.config/opencode/skills (folder per-skill + SKILL.md)
├── docs/            # Documentation
└── migrations/      # DB migrations
```

`claude/` dan `opencode/` isinya bisa beda meski nama agent sama — format frontmatter dan detail konten disesuaikan per platform, jadi jangan asumsikan file yang namanya sama otomatis identik.

Beberapa file di `claude/agents/` dan `claude/commands/` (`jatis-mahen*`, `uat-csv-generator.md`) sudah tidak ada di `~/.claude` saat ini — sengaja dipertahankan sebagai arsip, jangan dihapus otomatis saat sync.

## Agents

### Claude Code (`claude/agents/`)
| Agent | Purpose |
|----|----|
| `go-wacall` | Pipeline Go microservice (orchestrator → coder → tester → reviewer) |
| `fullstack-mahen` | Pipeline fullstack pesenin/loketin.id & edusmarttest (team-lead → backend-dev → frontend-dev) |
| `fullstack-mahen-mobile-dev` | Flutter mobile development (Lokasir) |
| `mirai-reconciliation` | Rekonsiliasi rekening koran vs database |
| `ide-konten` | Generate ide konten social media |
| `kata-sugesti` | NLP Ericksonian kata sugesti Bahasa Indonesia |
| `konten-sosmed` | Generate script video konten TikTok/Reels |
| `sugesti-video` | Video konten dengan kata sugesti NLP |
| `jatis-mahen*` | *(arsip)* Pipeline Java Spring Boot |
| `uat-csv-generator` | *(arsip)* Generate UAT CSV |

### OpenCode (`opencode/agents/`)
Sama daftar dasar seperti Claude Code (`go-wacall`, `fullstack-mahen`, `ide-konten`, `kata-sugesti`, `konten-sosmed`, `mirai-reconciliation`, `sugesti-video`) — frontmatter & format disesuaikan konvensi OpenCode.

## Skills

| Skill | Platform | Purpose |
|----|----|----|
| `developer_profile` | Claude Code | Profil developer freelance + estimasi |
| `pricing_rules` | Claude Code | Aturan pricing komersial |
| `rab_structure` | Claude Code | Struktur RAB proyek |
| `infrastructure_cost` | Claude Code | Biaya infrastruktur standar |
| `crd-gen` | OpenCode | Generate CRD.txt, CONFIG.md, VERSION.md untuk deployment |
| `e2e-gen` | OpenCode | Generate E2E test automation + UAT docs |
| `sonarqube-gate` | OpenCode | Autonomous SonarQube Quality Gate fix |
| `bofis-gen` | OpenCode | Generate BOFIS document |
| `pb-entry` | OpenCode | Generate PB entry document |

## Commands

`claude/commands/` — slash command Claude Code (`fullstack-mahen`, `go-wacall`, `kata-sugesti`, `mirai-reconciliation`, `sugesti-video`, `pm`, `e2e-gen`, `foto-docx-gen`, `invoice-gen`, `seo-optimizer`, plus arsip `jatis-mahen`).

## Memory

`claude/memory/` — backup auto-memory Claude Code (`MEMORY.md` sebagai index + file per topik: user, feedback, project, reference).

## Sync

```bash
# Claude Code -> repo
cp ~/.claude/agents/*.md claude/agents/
cp ~/.claude/commands/*.md claude/commands/
cp ~/.claude/skills/*.md claude/skills/
cp ~/.claude/projects/-Users-mahendrafajar/memory/*.md claude/memory/

# OpenCode -> repo
cp -r ~/.config/opencode/agents/* opencode/agents/
cp -r ~/.config/opencode/skills/* opencode/skills/
```
