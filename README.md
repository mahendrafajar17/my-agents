# My Agents

Koleksi agents, skills, dan commands untuk OpenCode dan Claude Code.

## Structure

```
my-agents/
├── opencode/        # Agents & skills untuk OpenCode
│   ├── agents/      # 16 agents (go-wacall, fullstack-mahen, dll)
│   └── skills/      # 9 skills (crd-gen, e2e-gen, sonarqube-gate, dll)
├── claude/          # Agents & skills untuk Claude Code
│   ├── agents/
│   └── skills/
├── commands/        # Custom commands (shared)
├── docs/            # Documentation
└── migrations/      # DB migrations
```

## Agents

### OpenCode
| Agent | Purpose |
|----|----|
| `go-wacall` | Pipeline Go microservice (orchestrator → coder → tester → reviewer) |
| `fullstack-mahen` | Pipeline fullstack (team-lead → backend-dev → frontend-dev) |
| `fullstack-mahen-mobile-dev` | Flutter mobile development |
| `mirai-reconciliation` | Rekonsiliasi rekening koran vs database |
| `ide-konten` | Generate ide konten social media |
| `kata-sugesti` | NLP Ericksonian kata sugesti Bahasa Indonesia |
| `konten-sosmed` | Generate script video konten TikTok/Reels |
| `sugesti-video` | Video konten dengan kata sugesti NLP |

### Claude Code
| Agent | Purpose |
|----|----|
| `jatis-mahen` | Pipeline Java Spring Boot |
| `pm` | Project management |

## Skills

| Skill | Platform | Purpose |
|----|----|----|
| `crd-gen` | OpenCode | Generate CRD.txt, CONFIG.md, VERSION.md untuk deployment |
| `e2e-gen` | OpenCode | Generate E2E test automation + UAT docs |
| `sonarqube-gate` | OpenCode | Autonomous SonarQube Quality Gate fix |
| `developer-profile` | OpenCode | Profil developer freelance + estimasi |
| `pricing-rules` | OpenCode | Aturan pricing komersial |
| `rab-structure` | OpenCode | Struktur RAB proyek |
| `infrastructure-cost` | OpenCode | Biaya infrastruktur standar |
| `bofis-gen` | OpenCode | Generate BOFIS document |
| `pb-entry` | OpenCode | Generate PB entry document |

## Sync

```bash
# Dari OpenCode config ke repo
cp -r ~/.config/opencode/agents/* opencode/agents/
cp -r ~/.config/opencode/skills/* opencode/skills/
```
