---
name: pilih-topik-tren
description: Pilih topik konten Bunda Kirana dari data tren TikTok (JSON yang ditempel user). Syarat topik = kontroversial (memancing debat parenting), tren dipakai sebagai HOOK atau sebagai TOPIK. Saring dengan batas keras CLAUDE.md, simpan ke docs, usulkan satu set, lalu berhenti minta persetujuan.
---

# Pilih topik tren untuk Bunda Kirana

Jawab dalam bahasa Indonesia, singkat, jujur.

## Syarat topik
1. **Kontroversial**: ada dua kubu yang masuk akal dan orang tua ingin ikut berdebat/berkomentar.
   Contoh bentuk: "anak boleh dibiarkan menang atau tidak?", "kartun dewasa boleh ditonton anak?".
   Bukan kontroversi: fakta yang sudah disepakati, atau yang cuma mengundang amarah.
2. **Tren = hook ATAU topik** (tulis jelas mana yang dipilih):
   - **Hook**: tren hanya pengait 1-2 detik pertama (emosi/situasinya, bukan orangnya), isi video tentang parenting.
   - **Topik**: tren itu sendiri adalah bahan bahasan parenting (mis. tontonan viral -> tontonan anak).
3. Ada sudut parenting/MPASI yang nyata. Tren tanpa sudut itu dibuang, jangan dipaksakan.
4. **Lolos 3 pertanyaan penyaring** (jawab ya/tidak per topik, buang yang ada tidak): orang tua peduli? Bisa beda pendapat? Punya pengalaman pribadi tentang ini?
   Rumus gesekan: relevan + beda pendapat + emosi + mudah dikomentari.

## Komposisi set: 70% evergreen, 20% tren, 10% eksperimen
Tren sebagai kendaraan, masalah orang tua sebagai isi. Hook tren basi dalam 2-3 hari, set evergreen (mis. "anak dibiarkan menang atau diajari kalah?")
tetap bisa diunggah kapan saja. Cek `docs/ide-konten-*.md` dan `docs/captions/` untuk set terakhir: kalau 3 set terakhir semuanya berbasis tren,
usulkan SATU set evergreen (hook tidak bergantung berita) sebagai rekomendasi atau alternatif. Eksperimen (sekitar 1 dari 10): hook **pernyataan lawan arus**
yang menyenggol kebiasaan, bukan orang (mis. "Anak yang selalu dibela justru lebih rapuh"), bukan hanya hook pertanyaan "boleh gak?".

**Rage bait dan engagement bait BOLEH** (keputusan user 2026-10-03): hook boleh sengaja menyenggol, memancing perdebatan atau reaksi kuat,
dan mengajak komentar/bagikan. Yang tetap berlaku hanya "Batas keras" di bawah (topik sensitif, tidak menyerang orang/nama, klaim kesehatan dicek ke sumber).
Sebut di usulan kalau hook-nya masuk kategori ini, beserta risikonya.

## Batas keras (kontroversi boleh, ini tidak)
Dari CLAUDE.md "Aturan konten": buang topik yang berisi
- musibah/korban (kecelakaan, bencana, jenazah, KM Virgo dll), kasus kriminal/eksekusi, kekerasan seksual;
- taruhan, berita luar negeri yang tak nyambung, SARA, rivalitas negara/suporter/politik;
- cerai/selingkuh, nama anak, orang yang diserang atau disebut di video.
Nama tokoh/tren boleh SATU kali di caption, tidak di video.
Kontroversi ada pada **sikap parenting yang bisa diperdebatkan**, bukan pada orang atau peristiwa.
Tetap empatik, tidak menyalahkan orang tua; klaim kesehatan/psikologi dicek ke sumber tepercaya (IDAI, Kemenkes, WHO, KPAI).

## Langkah
1. Baca JSON: `Title`, `Rank`, `Play`, dan `ItemName` (caption) 5 kiriman teratas tiap tren. Abaikan URL cover.
2. Saring dengan batas keras. Sebut singkat apa yang gugur dan kenapa.
3. Untuk sisanya, tulis: sudut kontroversi (kubu A vs B), peran tren (hook/topik), hook 1 kalimat,
   risiko (rendah/sedang/tinggi) dan cara menjinakkannya.
4. Rekomendasikan SATU set (6 klip) dengan **lokasi berganti antar klip** (jangan selalu dapur) dan nama `set` baru.
   Beri satu alternatif saja.
5. Simpan ringkas: data tren ke `docs/tren-tiktok-<tanggal>.md`, ide ke `docs/ide-konten-<tanggal>.md`
   (ikuti format file tanggal sebelumnya di `docs/`).
6. **Berhenti** dan minta persetujuan. Jangan menulis skenario ke `karakter/bundakirana.json` dan jangan produksi
   sebelum user setuju. Setelah setuju, ikuti resep di CLAUDE.md "Produksi satu konten".

## Pelajaran dari data posting nyata (2026-09-30)
GTM 4 tonton, anak_sulung 1, hp_makan 107: nada empatik + hook tren besar lebih terangkat. Kontroversi yang berhasil
bersifat "dilema orang tua" yang dirasakan banyak orang, bukan tuduhan.

## Caption (setelah produksi)
Pengait + label "dibuat dengan AI" + catatan bukan pengganti dokter/psikolog + hashtag tema. Simpan di `docs/captions/<set>.txt`.
