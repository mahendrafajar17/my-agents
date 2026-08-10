---
name: project-trading-backtest
description: "Backtest strategi trading user — data 5 trade awal, analisa RR/win-rate, dan rekomendasi TP (full vs partial)"
metadata: 
  node_type: memory
  type: project
  originSessionId: de5c72c6-0070-4030-acd1-7ccac8443f6e
---

User sedang backtest strategi entry trading (instrumen belum disebutkan). Sample pertama: 5 trade.

## Data 5 trade (mentah)
- Trade 1: TP 1:1, lalu BEP
- Trade 2: TP 1:1, lalu BEP
- Trade 3: TP 1:1, lalu BEP
- Trade 4: TP 1:1 → TP 1:2 → TP 1:4, lalu BEP (satu-satunya trade yang lanjut jauh)
- Trade 5: TP 1:1, lalu BEP

Continuation rate dari sample ini: **20%** (1 dari 5 trade lanjut melewati RR 1:1), 80% mentok di 1:1 lalu balik breakeven.

## Kesimpulan matematis
- Break-even win rate untuk RR 1:2 = 33,3% (1/(1+RR)).
- Dengan continuation rate 20% dari sample ini, **full TP di 1:1** (ambil semua profit begitu sentuh RR 1:1, tanpa partial) matematisnya mengungguli partial TP (50% di 1:1 / 25% di 1:2 / 25% di 1:4) maupun full TP di 1:4 — expectancy +1R/trade vs +0,8R/trade.
- Titik impas: continuation rate perlu naik ke ~25% dulu supaya biarkan runner ke 1:4 mulai unggul dibanding ambil cepat di 1:1.

**Why:** user awalnya condong ke partial TP (mengunci profit + biarkan runner), tapi setelah dihitung dari data aktualnya, hasilnya menunjukkan strategi ambil cepat (full di 1:1) justru lebih optimal untuk sample ini — insight yang berlawanan dari intuisi awal.

**How to apply:** sample 5 trade ini terlalu kecil untuk jadi kesimpulan final. User berencana lanjut mencatat 20-30 trade lagi dengan format sama (level RR mana yang kena tiap trade). Kalau user melanjutkan diskusi ini, cek apakah continuation rate masih ~20% atau berubah, dan update rekomendasi (full 1:1 vs partial vs full 1:4) sesuai data terbaru — jangan asumsikan angka 20% ini masih berlaku tanpa data baru.
