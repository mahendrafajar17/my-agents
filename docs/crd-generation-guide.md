# CRD.txt Generation Guide

Panduan membuat `CRD.txt` untuk project **message-in-transmitter CIMB Niaga**.

## Langkah-langkah

1. Baca `git diff HEAD` untuk memahami perubahan kode (file baru, property baru, fitur baru).
2. Cek `pom.xml` untuk versi app terbaru.
3. Cek nama branch aktif dengan `git branch`.
4. Buat file `CRD.txt` di dalam folder `doc/<nama-folder-PB>/`.

Contoh path:
```
doc/PB0126288326001081YA PT. Bank CIMB Niaga Tbk. - .../CRD.txt
```

## Format CRD.txt

```
====================================
PRE-IMPLEMENTATION PROCEDURE
====================================
(Repository URL, SonarQube URL, App version, Branch)

====================================
IMPLEMENTATION PROCEDURE
====================================
(Panduan copy JAR, update application.properties, update startup script)

====================================
CONFIGURATION UPDATE PROCEDURE (vX.X.X -> vX.X.X)
====================================
(Langkah upgrade dari versi sebelumnya)

====================================
FRESH INSTALLATION PROCEDURE
====================================
(Full application.properties template + langkah instalasi baru)

====================================
FEATURE CHANGES vX.X.X
====================================
(Deskripsi fitur/perubahan baru)

====================================
POST-IMPLEMENTATION PROCEDURE
====================================
(Verifikasi setelah deploy)

====================================
SUCCESS CRITERIA
====================================
(Kondisi yang harus dipenuhi agar deployment dianggap sukses)

====================================
ROLLBACK PROCEDURE
====================================
(Langkah rollback ke versi sebelumnya)

====================================
TROUBLESHOOTING GUIDE
====================================
(Common issues dan solusinya)
```

## Tips

- Section **FEATURE CHANGES** diisi berdasarkan `git diff` — jelaskan fitur/perubahan teknis secara detail.
- Section **CONFIGURATION UPDATE PROCEDURE** mencantumkan property baru yang perlu ditambahkan ke `application.properties`.
- Nama branch di **PRE-IMPLEMENTATION PROCEDURE** harus sesuai dengan output `git branch`.

## Referensi

Contoh CRD lengkap:
```
doc/PB0326290026000818YA PT. Bank CIMB Niaga Tbk. - RTE Coster - Alert Error Code Custom - Update MessageInTransmitter/CRD.txt
```
