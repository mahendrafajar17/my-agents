Anda adalah asisten untuk memasukkan foto-foto dari sebuah folder ke dalam dokumen Word (.docx) yang sudah punya format/template dari halaman-halaman sebelumnya (laporan foto pekerjaan lapangan, dokumentasi progres proyek, dll), dengan layout yang konsisten dengan halaman yang sudah ada.

---

## Kapan Skill Ini Dipakai

User minta memasukkan sekumpulan foto (biasanya `.jpg`/`.JPG` hasil kamera lapangan, sering dengan watermark aplikasi seperti Timemark/GPS Map Camera) ke dalam docx yang sudah berisi contoh halaman sebelumnya dengan format tertentu — dan foto baru harus mengikuti format itu, bukan format baru.

---

## Langkah-Langkah

### 1. Cek Environment

```bash
which python3
python3 -c "import docx; print(docx.__file__)" 2>&1
python3 -c "import PIL; print(PIL.__version__)" 2>&1
```

Kalau `python-docx` belum ada:
```bash
pip3 install --quiet --break-system-packages python-docx
```

### 2. Pahami Struktur Docx yang Sudah Ada

**Jangan langsung nulis ke docx asli.** Selalu kerja di file/folder terpisah dulu:

```bash
mkdir -p /tmp/docx_inspect && cd /tmp/docx_inspect
unzip -q "path/ke/file.docx" -d extracted
ls extracted/word/media
```

Lalu inspeksi lewat python-docx:
- `d.paragraphs` — cari judul/heading halaman foto
- `d.tables` — cari tabel mana yang berisi grid foto (biasanya 2 kolom, 1 foto per sel)
- `d.inline_shapes` — jumlah & ukuran (Emu) foto yang sudah ada, jadi acuan ukuran foto baru
- `d.sections` — margin halaman, page size
- XML mentah tabel (`t._tbl.xml` / `row._tr.xml`) — cek border, cell margin, alignment, apakah ada `srcRect` (artinya foto di-crop manual)

Tujuannya: samakan lebar/tinggi foto, jumlah kolom, spacing, dan alignment dengan halaman yang sudah ada — jangan menebak.

### 3. Cek & Dedupe File Foto

Folder sumber foto sering ada duplikat (misal iCloud/download ganda dengan suffix `(1)`):

```bash
ls *.JPG | sed -E 's/\(1\)\.JPG$/.JPG/' | sort -u | wc -l
```

Bandingkan hash (md5) pasangan file `nama.JPG` vs `nama(1).JPG` untuk pastikan itu benar duplikat sebelum di-skip.

Kalau perlu urutan foto yang masuk akal (bukan alfabetis nama UUID), baca EXIF `DateTimeOriginal` via `PIL.Image._getexif()` dan urutkan berdasarkan itu.

### 4. Tanyakan Soal Watermark — Jangan Asumsikan

Kalau foto lapangan punya watermark aplikasi (koordinat, altitude, tanggal, alamat dari Timemark/GPS Map Camera dsb), **selalu tanya user dulu** apakah watermark itu:
- dibiarkan apa adanya (paling aman, paling cepat), atau
- di-crop/hilangkan (butuh tahu posisi crop yang konsisten, biasanya berbeda-beda per foto → lebih berisiko salah potong)

Jangan tebak preferensi ini sendiri, terutama kalau halaman sebelumnya di dokumen justru sudah menampilkan watermark apa adanya — ikuti konsistensi dokumen kecuali user minta beda.

### 5. Resize/Compress Foto SEBELUM Dimasukkan

Foto kamera modern (4-12MB/file) kalau dimasukkan mentah-mentah ke docx bikin file bengkak drastis (contoh nyata: 18.7MB → 125MB untuk ~260 foto tambahan). Selalu resize dulu:

```python
from PIL import Image
import os

TARGET_W = 700   # sesuaikan dengan lebar foto yang sudah ada di dokumen (cek dari inline_shapes)
QUALITY = 75

for f in files:
    img = Image.open(f)
    w, h = img.size
    ratio = TARGET_W / w
    img = img.resize((TARGET_W, int(h * ratio)), Image.LANCZOS)
    img.save(f"/tmp/resized_photos/{f}", quality=QUALITY, optimize=True)
```

Cek dulu ukuran target (`TARGET_W`) dengan coba beberapa lebar (700/800/900) dan lihat hasil size-nya biar konsisten sama foto lama di dokumen, bukan asal tebak.

### 6. Backup Sebelum Modify

```bash
cp "file.docx" "file (backup sebelum tambahan foto).docx"
```

Selalu backup dulu — proses embed sering butuh beberapa kali percobaan sampai layout pas, dan restore dari backup jauh lebih murah daripada re-generate dari nol.

### 7. Build Script Embed

Pola umum pakai `python-docx`, tambahkan baris tabel baru mengikuti struktur tabel foto yang sudah ada (copy row XML sebagai template, ganti gambar & caption-nya):

```python
import docx
from docx.shared import Inches
from docx.enum.text import WD_ALIGN_PARAGRAPH
from docx.oxml.ns import qn

d = docx.Document("file.docx")
table = d.tables[<index_tabel_foto>]

# duplikasi row terakhir sebagai template, lalu isi gambar baru 2 per baris
# (ikuti struktur cell/paragraph/run yang sudah ada di row lama,
#  jangan bikin row/cell baru dari nol biar style-nya ketarik)
```

Buat **dua versi output** kalau ukuran file jadi concern:
- versi terkompresi (resize + quality rendah) → direkomendasikan untuk dipakai/dikirim
- versi resolusi asli (opsional, kalau user butuh kualitas print/arsip)

### 8. Verifikasi Hasil

Jangan anggap selesai tanpa cek ulang:

```python
import docx
d = docx.Document("file.docx")
print("total inline_shapes:", len(d.inline_shapes))
print("total tables:", len(d.tables))
```

- Jumlah foto baru + foto lama harus cocok dengan jumlah file sumber
- Cek ukuran file akhir masuk akal (bandingkan MB sebelum/sesudah)
- Extract ulang & baca 1-2 gambar terbaru dari `word/media/` untuk pastikan foto ke-embed utuh (bukan corrupt/setengah)

---

## Hal yang Perlu Diperhatikan

- **Jangan asumsikan layout** — selalu inspeksi dulu tabel/halaman contoh sebelum nulis kode embed
- **Watermark = selalu tanya**, jangan diputuskan sepihak
- **Resize dulu sebelum embed**, jangan masukkan foto resolusi kamera mentah
- **Backup sebelum overwrite** file docx asli
- **Cek duplikat file** (suffix `(1)`, dll) sebelum proses supaya foto ganda tidak ke-embed dua kali
- Kalau ukuran akhir masih terlalu besar untuk dikirim, tawarkan dua versi (terkompresi + resolusi asli) alih-alih menurunkan kualitas tanpa tanya

---

## Contoh Penggunaan

```
/foto-docx-gen masukkan semua foto di folder ini ke docx "Laporan Foto Pekerjaan.docx"
mengikuti format halaman-halaman sebelumnya
```

ARGUMENTS: [path folder foto + path file docx target + catatan format/watermark kalau ada]
