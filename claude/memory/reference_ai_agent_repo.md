---
name: reference-ai-agent-repo
description: Lokasi dan struktur repo backup agent/skill/command/memory Claude Code milik user — repo aktif saat ini adalah my-agents
metadata: 
  node_type: memory
  type: reference
  originSessionId: 98910e6d-ca6b-4ba1-9ee3-bf6bec6afd14
  modified: 2026-08-10T03:43:23.141Z
---

Repo aktif untuk backup Claude Code: `git@github.com:mahendrafajar17/my-agents.git`, di-clone ke `~/Repository/my-agents`.

Struktur (dipisah per platform, sejak reorganize 2026-08-10):
- `claude/agents/` -> mirror `~/.claude/agents` (format frontmatter `name:`)
- `claude/commands/` -> mirror `~/.claude/commands`
- `claude/skills/` -> mirror `~/.claude/skills`, flat `.md` TANPA frontmatter (jangan bungkus jadi folder+SKILL.md, itu format opencode)
- `claude/memory/` -> mirror flat dari `/Users/mahendrafajar/.claude/projects/-Users-mahendrafajar/memory/` (MEMORY.md + semua file memory)
- `opencode/agents/` -> mirror `~/.config/opencode/agents` (format frontmatter `mode: subagent`, beda dari claude meski nama sama)
- `opencode/skills/` -> mirror `~/.config/opencode/skills`, folder per-skill + `SKILL.md`
- File yang namanya sama di `claude/` vs `opencode/` BISA beda isi/frontmatter — jangan asumsikan identik, selalu diff dulu sebelum overwrite saat sync.
- `claude/agents/jatis-mahen*.md`, `claude/agents/uat-csv-generator.md`, `claude/commands/jatis-mahen.md` sudah tidak ada di `~/.claude` — sengaja dipertahankan sebagai arsip, jangan dihapus otomatis saat sync kecuali diminta eksplisit.

Repo lama `git@github.com:MyTechnoDev/ai-agent.git` (`~/Repository/Mytechnodev/ai-agent`, struktur `claude-code/` vs `opencode/` terpisah) sudah **tidak aktif** — commit terakhir 2026-06-25, sedangkan `my-agents` masih rutin di-update (terakhir 2026-08-10). Anggap `my-agents` sebagai sumber kebenaran untuk backup config Claude Code ke depannya.

**Why:** user mulai pakai repo `my-agents` yang lebih baru dan lebih sering disentuh; repo `ai-agent` lama berisiko membuat sync ke tempat yang salah/basi kalau tidak dibedakan.

**How to apply:** kalau user minta sync/backup/reinstall agent, skill, command, atau memory Claude Code, default ke `~/Repository/my-agents` (bukan `ai-agent`) kecuali user sebut repo lain secara eksplisit. Sync sifatnya one-way tambah/update (`~/.claude` → repo), jangan hapus file yang cuma ada di repo tanpa konfirmasi.
