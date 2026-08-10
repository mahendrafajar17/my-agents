---
name: reference-ai-agent-repo
description: Lokasi dan struktur repo backup agent/skill Claude Code + opencode milik user
metadata: 
  node_type: memory
  type: reference
  originSessionId: a98e9f11-e616-466a-903f-afaa92992a79
---

Repo `git@github.com:MyTechnoDev/ai-agent.git` di-clone ke `~/Repository/Mytechnodev/ai-agent` — dipakai untuk backup & reinstall custom agents/skills Claude Code dan opencode lintas device.

Struktur (mirror 1:1 ke lokasi config asli, supaya tinggal copy saat ganti device):
- `claude-code/agents/` -> `~/.claude/agents`
- `claude-code/commands/` -> `~/.claude/commands` (skill via slash command)
- `claude-code/skills/` -> `~/.claude/skills` (knowledge file pendukung, flat .md)
- `opencode/agents/` -> `~/.config/opencode/agents`
- `opencode/skills/` -> `~/.config/opencode/skills` (folder per skill + SKILL.md)
- `install.sh` di root repo: copy semua folder di atas ke lokasi aslinya di device baru.
- `project-manager/SKILL.md`: folder lama (pre-existing, dibuat manual sebelumnya) yang konsolidasi beberapa skill (pm + dev profile + pricing + infra + RAB) jadi satu SKILL.md — beda gaya dari struktur mirror di atas, sengaja dibiarkan apa adanya.

**Why:** user minta backup semua agent & skill biar mudah dipasang ulang kalau ganti device — pilih pendekatan mirror sederhana (bukan konsolidasi manual ala project-manager) karena lebih cepat dan langsung bisa di-copy ke path config yang sama.

**How to apply:** kalau user minta update/sync ulang agent atau skill ke repo ini, copy ulang file dari `~/.claude/{agents,commands,skills}` dan `~/.config/opencode/{agents,skills}` ke folder mirror yang sesuai, lalu commit & push (konfirmasi dulu sebelum push kalau belum ada instruksi eksplisit).
