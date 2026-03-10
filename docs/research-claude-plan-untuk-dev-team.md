# Research: Pilihan Plan Claude untuk Tim Dev (5 Orang)
**Dokumen Internal — Dipersiapkan untuk VP Engineering**

---

## Konteks

Sesuai proposal multi-agent system yang sudah diajukan, tim dev membutuhkan akses Claude yang optimal untuk mendukung workflow baru. Dokumen ini membandingkan pilihan plan Claude untuk 5 orang dev, dengan fokus pada value, limit penggunaan, dan kesesuaian dengan kebutuhan development sehari-hari.

---

## Perbandingan Plan

| | **Max 5x ÷ 2** | **Max 20x ÷ 5** | **Pro (1 akun/orang)** |
|---|---|---|---|
| **Biaya per orang/bulan** | $50 | $40 | $20 ($17 tahunan) |
| **Biaya total tim/bulan** | $100 (2 akun) | $200 (1 akun) | $100 (5 akun) |
| **Limit per orang** | ~112 pesan / 5 jam | ~180 pesan / 5 jam | ~45 pesan / 5 jam |
| **Pesan per $1** | ~2,2 | ~4,5 | ~2,25 |
| **Akses Opus** | ✅ | ✅ | ✅ |
| **Claude Code** | ✅ | ✅ | ✅ |
| **Priority access** | ✅ | ✅ | ❌ |
| **Early access fitur** | ✅ | ✅ | ❌ |
| **Limit sendiri-sendiri** | ❌ (rebutan antar 2 user) | ❌ (rebutan antar 5 user) | ✅ (dedicated per orang) |
| **Risiko ban / ToS** | ⚠️ Ada | ⚠️ Ada | ✅ Tidak ada |

---

## Analisis Mendalam

### Opsi A — Max 5x ÷ 2 ($50/orang, total $100/bulan)

**Cara kerja:** Beli 2 akun Max 5x, dibagi ke 5 orang. Setiap akun dipakai 2–3 orang bergantian.

**Kelebihan:**
- Limit per akun cukup besar (~560 pesan / 5 jam total untuk 2 akun)
- Lebih murah dari beli Max per orang

**Kekurangan:**
- Berbagi akun melanggar Terms of Service Anthropic → risiko ban permanen
- Jika 2–3 dev aktif bersamaan, kuota habis cepat dan ada yang tidak bisa kerja
- Tidak dapat early access fitur terbaru yang relevan untuk multi-agent workflow
- Tidak ada akuntabilitas penggunaan per individu

---

### Opsi B — Max 20x ÷ 5 ($40/orang, total $200/bulan)

**Cara kerja:** Beli 1 akun Max 20x, dibagi ke 5 orang. Semua pakai 1 akun bergantian.

**Kelebihan:**
- Value terbaik secara teori: ~4,5 pesan per $1 (hampir 2x lipat dari Pro)
- Limit sangat besar jika dihitung total (~900 pesan / 5 jam untuk 1 akun)
- Priority access dan early access fitur

**Kekurangan:**
- **Berbagi akun = melanggar ToS** → risiko ban lebih tinggi karena 5 orang pakai 1 akun
- Kuota saling "makan" — jika 2-3 dev aktif bersamaan, kuota bisa habis dalam hitungan jam
- Tidak ada isolasi per user: riwayat conversation, context, dan memori tercampur
- Biaya total tertinggi: $200/bulan
- Jika akun di-ban, seluruh tim langsung kehilangan akses

---

### Opsi C — Pro (1 akun per orang, $20/orang, total $100/bulan)

**Cara kerja:** Setiap dev punya akun Pro sendiri-sendiri.

**Kelebihan:**
- **Paling aman**: Tidak melanggar ToS, tidak ada risiko ban
- **Limit dedicated**: Setiap orang punya 45 pesan / 5 jam sendiri, tidak direbut siapa pun
- **Harga sama dengan Opsi A** ($100/bulan total), tapi tanpa risiko
- Riwayat conversation terisolasi per individu → lebih efektif untuk konteks panjang
- Bisa langsung upgrade ke Team jika butuh kapasitas lebih

**Kekurangan:**
- Limit per orang lebih kecil (45 pesan / 5 jam)
- Tidak dapat priority access dan early access fitur
- Untuk task intensif yang butuh banyak iterasi, limit bisa terasa kurang

---

## Alternatif yang Direkomendasikan: Claude Team

Selain ketiga opsi di atas, ada opsi resmi yang belum dipertimbangkan:

| | **Pro** | **Team Standard** |
|---|---|---|
| **Harga** | $20/bulan ($17 tahunan) | $25/bulan ($30 bulanan) |
| **Limit per orang** | ~45 pesan / 5 jam | ~56 pesan / 5 jam (1.25x Pro) |
| **Akun terpisah** | ✅ | ✅ |
| **Admin dashboard** | ❌ | ✅ |
| **Priority access** | ❌ | ✅ |
| **Kolaborasi tim** | ❌ | ✅ |
| **Risiko ban** | Tidak ada | Tidak ada |

**Team Standard** memberikan jalan tengah yang ideal: harga mendekati Pro, limit lebih besar, dan fitur kolaborasi yang relevan untuk tim dev. Harga tahunan $25/orang/bulan = $125/bulan untuk 5 orang.

---

## Matriks Keputusan

| Kriteria | Max 5x ÷ 2 | Max 20x ÷ 5 | Pro | Team |
|---|---|---|---|---|
| Keamanan & ToS | 🔴 | 🔴 | 🟢 | 🟢 |
| Limit per orang | 🟡 Tinggi tapi rebutan | 🟡 Sangat tinggi tapi rebutan | 🟡 Cukup, dedicated | 🟡 Cukup, dedicated |
| Value per $1 | 🟡 | 🟢 (teoritis) | 🟡 | 🟡 |
| Biaya total/bulan | 🟢 $100 | 🔴 $200 | 🟢 $100 | 🟡 $125 |
| Risiko operasional | 🔴 Tinggi | 🔴 Sangat tinggi | 🟢 Rendah | 🟢 Rendah |
| Siap scale | 🔴 | 🔴 | 🟡 | 🟢 |

---

## Rekomendasi

### Jangka Pendek (Sekarang)

> **Mulai dengan Pro ($20/orang) — 5 akun terpisah, total $100/bulan**

Alasan:
- Harga sama dengan Opsi A tapi tidak ada risiko ban
- Setiap dev punya kuota sendiri, tidak saling mengganggu
- 45 pesan / 5 jam sudah cukup untuk workflow normal dengan multi-agent (agent loop yang berat akan di-handle oleh pipeline, bukan oleh dev manual)
- Bisa upgrade kapan saja tanpa kehilangan data atau disruption

### Jangka Menengah (Bulan 2–3, setelah multi-agent pipeline berjalan)

> **Upgrade ke Team Standard ($25/orang/bulan tahunan) — total $125/bulan**

Alasan:
- Jika pipeline multi-agent sudah jalan, dev akan lebih aktif melakukan review dan iterasi — kebutuhan pesan naik
- Team memberikan admin dashboard untuk monitoring penggunaan
- Priority access berguna ketika butuh response cepat saat live debugging

---

## Catatan untuk VP

1. **Opsi berbagi akun (A & B) tidak direkomendasikan** — bukan hanya soal risiko ban, tapi produktivitas tim akan terganggu jika satu dev menghabiskan kuota sehingga dev lain tidak bisa pakai
2. **$100/bulan untuk Pro sudah cukup di fase awal** — investasi utama seharusnya di contract registry dan pipeline (sesuai proposal multi-agent)
3. **Jika budget ada**, langsung mulai dengan Team untuk menghindari migrasi di tengah jalan
4. **Claude Code sudah include** di semua plan Pro ke atas — tidak perlu biaya tambahan untuk CLI workflow yang relevan dengan multi-agent implementation

---

*Dokumen ini bagian dari serangkaian research untuk proposal multi-agent system. Lihat dokumen utama: "Mempercepat Delivery Dev dengan Multi-Agent System"*
