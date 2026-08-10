# PT. Jatis Mobile — CIMB RTE Mini Apps
## URL Monitoring (Synthetic Probing) UAT

| | |
|---|---|
| **Team Developer** | Mahen |
| **Tester** | Mahen |
| **Branch** | `feat/e2e-docker-automation` |
| **TRD** | `trd-cimb-rte-mini-apps.md` |

---

## 0. Preparation

| No | Remarks | Setup Data | Steps | Expected Results | Actual Result | Pass/Fail |
|---|---|---|---|---|---|---|
| 0.1.1 | Setup Artemis Broker | Artemis installed & accessible | 1. Verify Artemis running: `curl http://localhost:8161/console` (login: admin/admin)<br>2. Verify queue: Navigate to Queues → `cloud.message-in-transmitter_test`<br>3. Jika belum ada, buat via console atau CLI | Artemis running, console accessible di `http://localhost:8161`, queue `cloud.message-in-transmitter_test` exists, broker URL: `tcp://localhost:61616` | Passed | ✅ |
| 0.2.1 | Setup MongoDB | MongoDB running & accessible | 1. Connect: `mongosh mongodb://admin:admin@localhost:27017/whatsappCore?authSource=admin`<br>2. `use whatsappCore` → `show collections`<br>3. Verify collections ada: `routing`, `telegram_config`, `monitored_url`, `monitored_url_hit`, `push_messages`, `failed_hit` | Semua collections exist | Passed | ✅ |
| 0.3.1 | Insert routing | MongoDB connected, DB: whatsappCore | Insert ke `routing`:<br>`db.routing.insertOne({ sender_id: "6289651234567", msgin_transmitter_queue: "cloud.message-in-transmitter_test", push_message_collection: "push_messages", dr_url: "http://localhost:8085/cimb/message-in/dr", msgin_url: "http://localhost:8085/cimb/message-in/success", date_created: "2026-06-03 00:00:00" })`<br>Verify: `db.routing.find().pretty()` | Routing inserted: sender_id=6289651234567, msgin_url configured | Passed | ✅ |
| 0.4.1 | Insert monitored_url | MongoDB connected | Insert ke `monitored_url`:<br>`db.monitored_url.insertOne({ sender_id: "6289651234567", url: "http://localhost:8085/omni/health", enabled: true, date_created: new Date(), date_updated: new Date() })`<br>Verify: `db.monitored_url.find().pretty()` | Monitored URL inserted: sender_id=6289651234567, url=/omni/health, enabled=true | Passed | ✅ |
| 0.5.1 | Insert telegram_config | MongoDB connected | Insert ke `telegram_config`:<br>`db.telegram_config.insertOne({ api_url: "https://api.telegram.org/bot.../sendMessage", chat_id: "-4886766411", message_thread_id: "", date_created: "2026-06-03 00:00:00" })`<br>Verify: `db.telegram_config.find().pretty()` | Telegram config inserted, chat_id=-4886766411, ready for alerts | Passed | ✅ |
| 0.6.1 | Start Mock Server | Mock server available | 1. Start mock di port 8085<br>2. Verify: `/omni/health` → 200 OK<br>3. Verify: `/omni/health/503` → 503<br>4. Verify: `/omni/health/timeout` → delay >5s<br>`curl http://localhost:8085/omni/health` | Mock running: /omni/health=200, /omni/health/503=503, /omni/health/timeout=timeout | Passed | ✅ |
| 0.7.1 | Verify App Config | `application.properties` exists | Verify key configs:<br>`app.artemis.broker-url=tcp://localhost:61616`<br>`monitoring.url.enabled=true`<br>`monitoring.url.probe.msisdns=6281100000901,6281100000902`<br>`monitoring.url.failure.threshold=5`<br>`monitoring.url.failure.window.seconds=60`<br>`monitoring.url.hit.timeout=5`<br>`monitoring.url.scheduler.cron=0 */1 * * * ?` | Semua config verified, URL monitoring enabled, scheduler setiap 1 menit | Passed | ✅ |
| 0.8.1 | Start Application | Artemis, MongoDB, mock server running, config ready | 1. `java -jar message-in-transmitter.jar`<br>2. Cek startup logs: Artemis consumer active, MongoDB connected, Telegram config loaded, URL monitoring scheduler started<br>3. Verify tidak ada startup error | App started: consumer active, MongoDB connected, Telegram config loaded, scheduler running | Passed | ✅ |

---

## 1. Probe Happy Path

| No | Skenario | Setup Data | Steps | Expected Results | Actual Result | Pass/Fail |
|---|---|---|---|---|---|---|
| 1.1.1 | Monitored URL 200 — success hit | App running<br>`monitored_url`: sender=6289651234567 → `/omni/health` (200)<br>Probe MSISDN: 6281100000901 | 1. Publish ke queue `cloud.message-in-transmitter_test`:<br>`{"messages":[{"from":"6281100000901","id":"wamid.probe.ok","text":{"body":"probe"},"type":"text","timestamp":"1749000001"}],"contacts":[{"wa_id":"6281100000901"}]}`<br>2. Monitor logs: `[PROBE] Hitting monitored URL`<br>3. Cek `db.monitored_url_hit.find({"message_id":"wamid.probe.ok"})`<br>4. Tunggu scheduler max 1 menit | `monitored_url_hit`: 1 record, success=true, http_code=200<br>`msgin_push_history`: 0 record (probe skip client push)<br>`failed_hit`: 0 record<br>Tidak ada alert | Passed | ✅ |

---

## 2. Probe Failure Path

| No | Skenario | Setup Data | Steps | Expected Results | Actual Result | Pass/Fail |
|---|---|---|---|---|---|---|
| 2.1.1 | Monitored URL 503 — unsuccessful hit | App running<br>Update: `db.monitored_url.updateOne({"sender_id":"6289651234567"},{$set:{"url":"http://localhost:8085/omni/health/503"}})`<br>Probe MSISDN: 6281100000901 | 1. Publish payload:<br>`{"messages":[{"from":"6281100000901","id":"wamid.probe.503","text":{"body":"probe"},"type":"text","timestamp":"1749000002"}],"contacts":[{"wa_id":"6281100000901"}]}`<br>2. Monitor logs: `Monitored URL response: 503`<br>3. Cek `monitored_url_hit` | `monitored_url_hit`: 1 record, success=false, http_code=503, response_body captured | Passed | ✅ |
| 2.2.1 | Monitored URL Timeout | App running<br>Update: `db.monitored_url.updateOne({"sender_id":"6289651234567"},{$set:{"url":"http://localhost:8085/omni/health/timeout"}})`<br>`monitoring.url.hit.timeout=5`<br>Probe MSISDN: 6281100000901 | 1. Publish payload:<br>`{"messages":[{"from":"6281100000901","id":"wamid.probe.to","text":{"body":"probe"},"type":"text","timestamp":"1749000003"}],"contacts":[{"wa_id":"6281100000901"}]}`<br>2. Monitor logs: `Monitored URL timeout`<br>3. Cek `monitored_url_hit` | `monitored_url_hit`: 1 record, success=false, http_code=408, error_type=timeout | Passed | ✅ |

---

## 3. Monitoring Scheduler (Threshold Alert)

| No | Skenario | Setup Data | Steps | Expected Results | Actual Result | Pass/Fail |
|---|---|---|---|---|---|---|
| 3.1.1 | Below threshold (4 failures) — no alert | App running<br>Cleanup: `db.monitored_url_hit.deleteMany({})` | Seed 4 failed hits dalam window <60s:<br>`for (let i=1;i<=4;i++) { db.monitored_url_hit.insertOne({ message_id:"sched-low-"+i, sender_id:"6289651234567", msisdn:"6281100000901", url:"http://localhost:8085/omni/health", http_code:503, success:false, error_type:"exception", date_created:new Date() }) }`<br>Tunggu scheduler max 1 menit, cek logs | Tidak ada Telegram alert<br>Tidak ada Email alert | Passed | ✅ |
| 3.2.1 | Threshold breached (5 failures) — alert | App running<br>Cleanup: `db.monitored_url_hit.deleteMany({})` | Seed 5 failed hits dalam window <60s:<br>`for (let i=1;i<=5;i++) { db.monitored_url_hit.insertOne({ message_id:"sched-brk-"+i, sender_id:"6289651234567", msisdn:"6281100000901", url:"http://localhost:8085/omni/health", http_code:503, success:false, error_type:"exception", date_created:new Date() }) }`<br>Tunggu scheduler max 1 menit | Telegram alert diterima di chat -4886766411<br>Email alert diterima<br>Tepat 1 alert per channel | Passed | ✅ |
| 3.3.1 | Two offending URLs — one alert each | App running<br>Cleanup: `db.monitored_url_hit.deleteMany({})` | Seed 6 failures untuk URL-A dan 5 untuk URL-B dalam window<br>Tunggu scheduler max 1 menit | 2 alert dikirim: 1 untuk URL-A, 1 untuk URL-B | Passed | ✅ |

---

## 4. Alert Content Conformance

| No | Skenario | Setup Data | Steps | Expected Results | Actual Result | Pass/Fail |
|---|---|---|---|---|---|---|
| 4.1.1 | Telegram format (TRD 5.5.1) | App running<br>Cleanup: `db.monitored_url_hit.deleteMany({})` | Seed 5 failures dengan response_body untuk 1 URL<br>Tunggu scheduler, cek pesan Telegram yang diterima | Header: "Monitored URL Failure Detected"<br>WABA: 6289651234567<br>Monitored URL ditampilkan<br>"We observed 5 failed probe(s) (threshold: 5 in 60s)"<br>Breakdown per kode: "Non-2xx: 5 (HTTP 503 x5)"<br>Last failure: message_id, http_code, response_body<br>Next Action + disclaimer | Passed | ✅ |
| 4.2.1 | Email format (TRD 5.5.2) | App running<br>Seed sama seperti 4.1.1<br>`external.email.alert.enabled=true` | Trigger alert via scheduler, cek email yang diterima | Subject: "Monitored URL Failure on Webhook Detected"<br>HTML body: WABA, Monitored URL<br>Breakdown per kode HTTP<br>Last failure + response_body<br>Next Action + disclaimer | Passed | ✅ |

---

## 5. Edge Cases

| No | Skenario | Setup Data | Steps | Expected Results | Actual Result | Pass/Fail |
|---|---|---|---|---|---|---|
| 5.1.1 | Probe MSISDN tanpa monitored_url (fallback) | App running<br>Hapus mapping: `db.monitored_url.deleteMany({"sender_id":"6289651234567"})`<br>Probe MSISDN: 6281100000901 | 1. Publish payload:<br>`{"messages":[{"from":"6281100000901","id":"wamid.probe.nomap","text":{"body":"probe"},"type":"text","timestamp":"1749000005"}],"contacts":[{"wa_id":"6281100000901"}]}`<br>2. Monitor logs: `[PROBE] msisdn matched but no monitored URL - falling back to normal flow`<br>3. Cek `monitored_url_hit` | Diproses sebagai inbound normal (push ke client)<br>Inline Telegram alert dikirim<br>`monitored_url_hit`: 0 record | Passed | ✅ |
| 5.2.1 | Feature flag disabled | Set `monitoring.url.enabled=false`, restart app<br>Restore monitored_url mapping (0.4.1)<br>Probe MSISDN: 6281100000901 | 1. Publish payload:<br>`{"messages":[{"from":"6281100000901","id":"wamid.probe.off","text":{"body":"probe"},"type":"text","timestamp":"1749000006"}],"contacts":[{"wa_id":"6281100000901"}]}`<br>2. Cek `monitored_url_hit` dan `msgin_push_history` | Di-push ke client + `msgin_push_history` tercatat<br>`monitored_url_hit`: 0 record | Passed | ✅ |
| 5.3.1 | Reload monitored_url tanpa restart | App running (`monitoring.url.enabled=true`) | 1. Update `monitored_url` di Mongo (ubah url / tambah sender baru)<br>2. Publish reload trigger ke reload topic: `reload all`<br>3. Send probe untuk mapping baru<br>4. Cek `monitored_url_hit` | Mapping baru aktif tanpa restart<br>Probe mengikuti URL terbaru<br>`monitored_url_hit` mencatat URL baru | Passed | ✅ |
| 5.4.1 | TTL Expiry (7 hari) | `monitored_url_hit` memiliki TTL index 7 hari | 1. Verify TTL index: `db.monitored_url_hit.getIndexes()`<br>2. (Opsional) Insert dokumen dengan `date_created` lampau, tunggu MongoDB TTL reaper | TTL index ada pada `monitored_url_hit`<br>Dokumen >7 hari dihapus otomatis | Passed | ✅ |
