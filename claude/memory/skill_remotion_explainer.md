---
name: skill-remotion-explainer
description: Agent/command remotion-explainer untuk generate video explainer sinematik (Vox-style) pakai Remotion
metadata: 
  node_type: memory
  type: reference
  originSessionId: 5a425e36-257c-4d6b-ac0d-0a70eacfdc6d
  modified: 2026-09-06T02:46:31.485Z
---

Agent `remotion-explainer` tersedia di `~/.claude/agents/remotion-explainer.md`, entry point command di `~/.claude/commands/remotion-explainer.md`.

**Cara panggil:** ketik `/remotion-explainer <topik/data>` atau minta "buatkan video explainer sinematik tentang [topik]".

**Gaya yang di-encode:** "Cinematic Flat-Design Explainer" — kombinasi data journalism style (Vox, Johnny Harris) dengan treatment sinematik (letterbox, vignette, film grain). Dibangun murni via kode React/Remotion, tanpa aset gambar berbayar/berlisensi tidak jelas.

**Struktur baku:** TitleScene (kinetic typography + highlight kata) → scene narasi/konflik custom (karakter flat-vector + momen "impact") → StatScene (big number count-up) → ChartScene (bar chart data-driven via props) → OutroScene (gradient cinematic + branding). Semua di-overlay dengan Vignette + FilmGrain + Letterbox di level komponen utama.

**Asset karakter/objek:** default SVG vector buatan kode (aman lisensi). Kalau user minta foto asli, sumber aman: Wikimedia Commons (CC0/PD, tanpa API key) — bukan Pinterest (lisensi repin tidak jelas).

**Why:** Dibuat setelah user puas dengan hasil video "TariffWar" (perang tarif Trump) yang di-upgrade jadi cinematic dengan karakter & asset — user minta gaya ini disimpan jadi skill reusable untuk topik lain ke depannya. Referensi implementasi konkret ada di project `~/remotion-project/src/TariffWar/`.

Lihat juga: [[project_ai_video_pricing]] (konteks video AI lain yang pernah dikerjakan user).
