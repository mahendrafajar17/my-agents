# Referensi: E2E Browser Test (Playwright, tanpa test runner)

Dokumen referensi untuk agent `fullstack-mahen*` (dibaca lewat path, bukan dipanggil sebagai agent).
Sumber pola: `/Users/mahendrafajar/tautin/e2e` (Go + React + Postgres + Midtrans + Google login). Baca file asli di sana bila perlu detail.

## 1. Prinsip

1. Suite penuh HANYA jalan di lokal / CI dengan stack sendiri: DB sementara, backend, frontend, layanan eksternal palsu. Jangan pernah menulis data ke production.
2. Skrip yang menyentuh production (diagnostik) dipisah, read-only, tidak masuk `make e2e` / CI.
3. Satu skrip Node (`run.mjs`) = satu alur panjang berurutan (daftar → ... → unduh). Bukan banyak file test kecil. Data dibuat di langkah awal dan dipakai langkah berikutnya.
4. Hasil = baris `PASS`/`FAIL`, screenshot per langkah, dan exit code (0 hanya bila semua lolos DAN tidak ada error browser).

## 2. Struktur file

```
e2e/
  package.json        { "type":"module", "scripts":{"test":"node run.mjs"}, "devDependencies":{"playwright-core":"^1.50.0"} }
  run-e2e.sh          orkestrasi (lihat 3)
  run.mjs             skenario Playwright (lihat 4)
  fake-<layanan>.mjs  server HTTP Node polos penirun layanan eksternal (lihat 5)
  check-*.mjs, jitter.mjs  (opsional) diagnostik live, read-only, manual
  shots/              screenshot (gitignore; di-upload CI sebagai artifact)
Makefile              e2e: ./e2e/run-e2e.sh      test: go test (DB *_test terpisah) + npx tsc --noEmit
.gitignore            e2e/node_modules, e2e/shots
```

## 3. `run-e2e.sh` (kerangka)

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/.."
API_PORT=${E2E_API_PORT:-18080}; WEB_PORT=${E2E_WEB_PORT:-15173}   # port TERPISAH dari dev
export FAKE_PORT=${FAKE_PORT:-19911}
DB_CONTAINER=<app>-e2e-db; TMP=$(mktemp -d); PIDS=()
cleanup() {
  for p in "${PIDS[@]:-}"; do [ -n "$p" ] && kill "$p" 2>/dev/null || true; done
  [ -z "${E2E_DATABASE_URL:-}" ] && docker rm -f $DB_CONTAINER >/dev/null 2>&1 || true
  rm -rf "$TMP"
}
trap cleanup EXIT
wait_for() { for _ in $(seq 1 60); do curl -fsS "$1" >/dev/null 2>&1 && return 0; sleep 1; done
             echo "GAGAL menunggu $2"; tail -20 "$TMP"/*.log; exit 1; }
```

Langkah-langkahnya:

1. **Database**
   - CI: `E2E_DATABASE_URL` diisi (service postgres di workflow). Jalankan `schema.down.sql` (abaikan error) lalu `schema.up.sql` dengan `psql`. `export E2E_PSQL="psql $DB_URL -q"`.
   - Lokal: `docker run -d --name $DB_CONTAINER -e POSTGRES_USER=t -e POSTGRES_PASSWORD=t -e POSTGRES_DB=t -p 127.0.0.1:55433:5432 postgres:16-alpine`; `export E2E_PSQL="docker exec -i $DB_CONTAINER psql -U t -d t -q"`.
   - **JEBAKAN:** image postgres menjalankan server SEMENTARA saat init lalu restart. `pg_isready` sudah OK di fase sementara. Tunggu log `ready to accept connections` muncul DUA kali (`docker logs | grep -c ... -ge 2`), lalu terapkan skema dengan retry 5x (sleep 2).
2. **Build & jalankan backend** ke `$TMP/server` (`go build`), env diarahkan ke layanan palsu + DB e2e + port e2e. Log ke `$TMP/backend.log`. Tambahkan PID ke `PIDS`.
3. **Layanan palsu**: `node e2e/fake-*.mjs > $TMP/fake.log &`.
4. **Frontend**: `cd frontend && VITE_API_PROXY=http://localhost:$API_PORT npx vite --port $WEB_PORT --strictPort &` (`npm ci` dulu bila belum ada node_modules).
5. `wait_for http://localhost:$API_PORT/api/health backend`, lalu `wait_for http://localhost:$WEB_PORT/ frontend`.
6. `cd e2e && E2E_BASE=http://localhost:$WEB_PORT node run.mjs` (`npm ci` dulu bila belum ada node_modules). Exit code skrip ini jadi exit code run-e2e.sh.

Env backend yang dilonggarkan KHUSUS E2E (jangan di config produksi): fee platform 0, masa tahan dana 0 hari, cooldown rekening 0 jam, minimum penarikan kecil, `ENV=development`, `JWT_SECRET=e2e`, `STORAGE_DIR`/`MEDIA_DIR` di `$TMP`, `CORS_ORIGINS`/`PUBLIC_BASE_URL` = URL web e2e.

**Syarat di backend:** semua URL layanan eksternal HARUS bisa di-override lewat env (mis. `MIDTRANS_SNAP_URL`, `MIDTRANS_API_URL`, `GOOGLE_TOKENINFO_URL`). Kalau belum bisa, ubah backend dulu; jangan stub di dalam kode produksi.

## 4. `run.mjs` (kerangka & pola)

```js
import { chromium } from "playwright-core";
import fs from "node:fs";
import { execSync } from "node:child_process";

const BASE = process.env.E2E_BASE || "http://localhost:5173";
const FAKE = `http://127.0.0.1:${process.env.FAKE_PORT || 9911}`;
const CHROME = process.env.CHROME_PATH || ["/Applications/Google Chrome.app/Contents/MacOS/Google Chrome", "/usr/bin/google-chrome"].find((p) => fs.existsSync(p));
const SHOTS = new URL("./shots/", import.meta.url).pathname; fs.mkdirSync(SHOTS, { recursive: true });

const results = [], errors = [];
const check = (name, ok, extra = "") => { results.push({ name, ok }); console.log(`${ok ? "PASS" : "FAIL"}  ${name}${extra ? "  " + extra : ""}`); };

// CHROME undefined (CI) -> pakai Chromium bawaan Playwright
const browser = await chromium.launch({ ...(CHROME ? { executablePath: CHROME } : {}), headless: true });
const mobile = { viewport: { width: 390, height: 844 }, deviceScaleFactor: 2, isMobile: true, hasTouch: true };

async function stubs(page) {                       // panggil untuk SETIAP page baru
  page.on("pageerror", (e) => errors.push("pageerror: " + e.message));
  page.on("console", (m) => m.type() === "error" && !/Failed to load resource/.test(m.text()) && errors.push("console: " + m.text()));
  await page.route("https://<script-pihak-ketiga>.js", (r) => r.fulfill({ contentType: "text/javascript", body: "window.x = ..." }));
}
// ... skenario ...
await browser.close();
console.log(`\n${results.filter((r) => r.ok).length}/${results.length} lolos`);
if (errors.length) console.log("ERRORS BROWSER:\n" + [...new Set(errors)].join("\n"));
process.exit(results.every((r) => r.ok) && errors.length === 0 ? 0 : 1);
```

Pola yang dipakai:

- **Context terpisah per peran**: `seller = browser.newContext(mobile)`, `buyer = newContext(mobile)` (tanpa login), `admin = newContext(desktop)`. Sesi tidak bocor antar peran.
- **Selector berbasis aksesibilitas**: `getByRole("button",{name})`, `getByLabel`, `getByPlaceholder`, `getByText`, `getByTestId` (tambahkan `data-testid` di frontend untuk elemen yang sulit dipilih, mis. `available`, `untracked-note`, `admin-withdrawal`, `audit-list`). Hindari CSS selector rapuh.
- **Tunggu kondisi, bukan waktu**: `waitFor()`, `waitForURL`, `waitForResponse`, `waitForFunction`. `waitForTimeout` hanya untuk transisi animasi sebelum screenshot.
- **Simpan form = tunggu respons API sungguhan** (toast lama bisa masih tampil):
  ```js
  await Promise.all([
    page.waitForResponse((r) => r.url().includes("/api/me") && r.request().method() === "PUT" && r.ok()),
    page.getByRole("button", { name: "Simpan perubahan" }).click(),
  ]);
  ```
- **Upload file tanpa fixture**: `setInputFiles({ name, mimeType, buffer })`. Gambar PNG dibuat in-memory tanpa dependensi (IHDR/IDAT/IEND + zlib + CRC32, lihat fungsi `png(w,h)` di run.mjs asli); file dokumen cukup `Buffer.from("ISI")`.
- **Unduhan**: `const [dl] = await Promise.all([page.waitForEvent("download"), page.getByText("ebook.pdf").click()]); fs.readFileSync(await dl.path(), "utf8")` lalu cek isi.
- **Verifikasi lewat API + UI**: setelah aksi UI, `fetch(BASE + "/api/...")` untuk memastikan data benar-benar tersimpan (mis. halaman publik, endpoint stats dengan `Authorization: Bearer` dari `localStorage`).
- **Manipulasi state lewat SQL** untuk kondisi yang tak bisa dicapai via UI: `execSync(\`${process.env.E2E_PSQL} -c "UPDATE users SET is_admin=true WHERE username='demo'"\`, { stdio:"pipe", shell:"/bin/bash" })`. Kembalikan state setelahnya bila langkah berikutnya bergantung padanya.
- **Pembayaran palsu**: stub `snap.js` agar memanggil callback `onPending`; lalu `await fetch(FAKE + "/__settle")`, dan test menunggu halaman order berubah sendiri (menguji polling) tanpa reload.
- **Login Google palsu**: stub `accounts.google.com/gsi/client` yang menyimpan callback ke `window.__gsiCb`; test memanggil `window.__gsiCb({ credential: "good:<email>" })`. Token `bad`/tanpa prefix `good:` -> ditolak oleh tokeninfo palsu.
- **Uji keamanan/otorisasi ikut di alur**: token peran A ditolak di API peran B (expect 401), logout mencabut token di server (token lama 401), token admin tidak ada di `localStorage`, non-admin ditolak dengan pesan generik, log audit memuat login gagal & berhasil.
- **Responsif**: ulangi halaman kunci di context desktop (1280x800) dan mobile (390x844), screenshot masing-masing.
- **Screenshot** bernomor urut `NN-nama.png` di tiap langkah penting (`fullPage: true` untuk halaman panjang).
- Urutan penomoran bagian skenario mengikuti alur bisnis; sisipkan sub-bagian (8b, 8c, ...) saat fitur baru bergantung pada data langkah sebelumnya.

## 5. `fake-<layanan>.mjs`

Server `node:http` tanpa library. Port dari env. Menyimpan state di variabel modul. Pola:

- Endpoint meniru layanan asli (Midtrans: `POST /snap/v1/transactions` → `{token, redirect_url}`, `GET /v2/<order>/status` → `transaction_status` `pending`/`settlement` + `gross_amount` sama dengan yang dibuat; Google: `GET /tokeninfo?id_token=good:<email>` → `{sub, email, email_verified:"true", aud, iss, exp}`).
- Endpoint kontrol khusus test: `/__settle`, `/__reset` untuk mengubah state dari `run.mjs`.
- Fallback `404 {}` untuk path lain.
- `aud`/client ID di respons palsu HARUS sama dengan `GOOGLE_CLIENT_ID` yang diberikan ke backend e2e.

## 6. CI (`.github/workflows/ci.yml`)

Job `e2e` terpisah dari `backend` dan `frontend`:

```yaml
e2e:
  runs-on: ubuntu-latest
  services:
    postgres:
      image: postgres:16-alpine
      env: { POSTGRES_USER: t, POSTGRES_PASSWORD: t, POSTGRES_DB: e2e }
      ports: ["5432:5432"]
      options: >-
        --health-cmd "pg_isready -U t -d e2e" --health-interval 5s --health-timeout 3s --health-retries 10
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-go@v5
      with: { go-version-file: backend/go.mod, cache-dependency-path: backend/go.sum }
    - uses: actions/setup-node@v4
      with: { node-version: 22 }
    - run: cd frontend && npm ci
    - run: cd e2e && npm ci && npx playwright-core install --with-deps chromium
    - run: ./e2e/run-e2e.sh
      env: { E2E_DATABASE_URL: "postgres://t:t@localhost:5432/e2e?sslmode=disable" }
    - uses: actions/upload-artifact@v4
      if: always()
      with: { name: e2e-screenshots, path: e2e/shots }
```

Job `backend`: service postgres + `go vet ./...` + `go test ./... -count=1` dengan `TEST_DATABASE_URL`. Job `frontend`: `npm ci` + `npx tsc --noEmit && npm run build`.

## 7. Test Go (integrasi) & `make test`

- Test integrasi backend butuh Postgres; pakai DB terpisah `<app>_test` lewat `TEST_DATABASE_URL` agar data dev aman. Target `make test` membuat DB itu bila belum ada, lalu `go test ./... -count=1`, lalu `cd frontend && npx tsc --noEmit`.
- Frontend tidak punya unit test; cek tipe + E2E sudah cukup untuk proyek ini.

## 8. Menambah skenario untuk fitur baru (checklist agent)

1. Fitur mengubah alur user utama (daftar, bayar, pencairan, admin, dll)? Jika tidak, cukup `make test`.
2. Tambahkan `data-testid` di komponen frontend yang perlu dipilih secara stabil.
3. Tambahkan blok di `run.mjs` setelah langkah yang menyiapkan datanya; beri `check()` bernama jelas dalam bahasa UI + screenshot bernomor.
4. Uji sisi negatif/otorisasi bila fitur menyentuh uang, akun, atau peran.
5. Jika butuh layanan eksternal baru: tambah endpoint di `fake-*.mjs` dan pastikan URL-nya configurable via env backend; tambahkan env itu ke `run-e2e.sh`.
6. Jika butuh state yang hanya bisa dicapai lewat DB: pakai `E2E_PSQL`, dan kembalikan state setelahnya.
7. Jalankan `make test` dan `make e2e` sampai hijau. Laporkan jumlah `PASS/total` dan error browser (kalau ada) di Final Report.

## 9. Skrip live (read-only, manual)

- `check-google.mjs [url] [--fake-id=<id>]`: muat `/register` dengan skrip Google ASLI, baca pesan GSI (origin salah, client ID salah). Default URL = domain produksi. Target `make google-check`.
- `jitter.mjs`: login akun asli (env `JIT_LOGIN`, `JIT_PW`), ukur layout shift, mutasi DOM, dan jumlah request API halaman tertentu. Tidak menulis data. Jalankan manual saja.
- Pengecekan otomatis di production hanya yang ada di `deploy.sh`: health check `/api/health` dan verifikasi HTML + content-type bundle JS/CSS. Jangan menambah test yang menulis data ke production.

## 10. Jebakan yang sudah ketemu

- Postgres docker "siap" terlalu dini (lihat 3.1).
- Toast/notifikasi lama menipu `waitFor` → tunggu respons API (lihat 4).
- State SQL yang diubah test harus dikembalikan sebelum langkah yang bergantung padanya (mis. `is_admin`, kenaikan `views`/`clicks`).
- Context Playwright baru TIDAK mewarisi `page.route`; panggil `stubs(page)` untuk tiap page.
- `--strictPort` pada Vite agar gagal jelas bila port bentrok, bukan pindah port diam-diam.
- Tunggu transisi tab/animasi sebelum screenshot (`waitForTimeout(500)`), jangan sebelum assertion.
