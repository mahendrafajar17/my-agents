---
name: e2e-gen
description: Generate E2E test automation boilerplate dan UAT documentation untuk project Go, Java Spring Boot, atau PHP web app (Playwright). Gunakan ketika user mengetik /e2e-gen atau meminta generate E2E test, E2E automation, E2E evidence, UAT documentation, atau testcontainers boilerplate. Go/Java menghasilkan e2e/main_test.go, helpers_test.go, flow_test.go, testdata/init.sql, UAT MD, dan E2E-LOG-EVIDENCE MD. PHP menghasilkan e2e/package.json, playwright.config.js, tests/security.spec.js, tests/functional.spec.js, UAT MD, dan E2E-LOG-EVIDENCE MD.
---

Generate E2E test automation boilerplate dan UAT documentation untuk project Go, Java Spring Boot, atau PHP web app. Menghasilkan file siap pakai: struktur folder e2e/, kode boilerplate lengkap, dan UAT .md dengan format tabel horizontal.

**Alur E2E test yang di-generate (Go/Java):**
```
TEAR UP   → spin up Docker (DB/queue/cache) + seed data + jalankan app
PROSES    → trigger aksi (publish queue / HTTP request)
VALIDASI  → assert DB / mock server / HTTP response
TEAR DOWN → matikan app + hapus container (otomatis via defer)
```

**Alur E2E test yang di-generate (PHP — browser via Playwright):**
```
TEAR UP   → webServer Playwright auto-start app (php -S / php artisan serve) — otomatis
PROSES    → buka halaman + aksi user beneran di browser (klik, isi form, submit)
VALIDASI  → assert UI (visible/hidden/class), response header security (CSP dkk),
            console error, nilai di DOM, log file (mis. csp_report_*.log)
TEAR DOWN → webServer berhenti otomatis setelah test selesai
```

Go dan Java menggunakan **Testcontainers** untuk spin up Docker infra otomatis dari dalam test. PHP web app menggunakan **Playwright** (Node.js) dengan `webServer` config — tidak perlu Docker, tidak perlu setup manual.

## Cara Pakai

```
/e2e-gen [path/ke/project]
```

Contoh:
- `/e2e-gen` — generate untuk project di working directory saat ini
- `/e2e-gen ../costerdrconverter` — generate untuk project lain

---

## Instruksi

Kamu adalah generator E2E test automation untuk JatisMobile. Ikuti langkah berikut secara berurutan. **Jangan skip langkah apapun.**

---

### Langkah 1 — Deteksi Bahasa Project & Mode

Baca root project yang diberikan (atau working directory jika tidak ada argumen):

1. Ada `go.mod` → project **Go**
2. Ada `pom.xml` → project **Java Spring Boot**
3. Ada file `*.php` (khususnya di root, seperti `index.php` / `*.php` page) → project **PHP web app** (Playwright)
4. Tidak ada semuanya → tanya user

Setelah deteksi bahasa, cek keberadaan E2E yang sudah ada:
- **Go**: cek apakah folder `e2e/` sudah ada dan berisi `main_test.go`
- **Java**: cek apakah folder `src/test/java/.../e2e/` sudah ada dan berisi `E2EContainers.java`
- **PHP**: cek apakah folder `e2e/` sudah ada dan berisi `playwright.config.js`

Jika sudah ada → lanjut ke **Langkah 2 dalam MODE UPDATE**
Jika belum ada → lanjut ke **Langkah 2 dalam MODE GENERATE**

---

### Langkah 2 — Baca Struktur Project Secara Mendalam

#### Jika Go:

Baca file-file berikut secara berurutan:

1. **`go.mod`** — ambil module name, versi Go, list dependency (testcontainers, gin, rabbit, mongo, postgres, mysql, redis, dsb)
2. **`cmd/main.go`** atau **`main.go`** — cari: cara baca config (viper/env/flag), port server, nama binary
3. **`config/`** atau **`config.go`** atau **`config.yaml`** / **`config.yml`** / **`config-example.yaml`** — baca struktur config: field DB host/port/user/pass, queue host/port, dsb
4. **`service/`** — list semua service file, baca nama struct dan method utama
5. **`handler/`** atau **`api/`** — list endpoint HTTP yang ada
6. **`repository/`** — list repository dan database yang dipakai
7. **`model/`** atau **`entity/`** — baca nama struct untuk tahu nama tabel/collection

Dari bacaan di atas, deteksi:
- **Entry point**: lokasi `main.go` dan cara jalankan app (`go run ./cmd/main.go` atau `go run .`)
- **Config loading**: viper dari file YAML? env var? flag? — ini penting untuk `writeConfig()` dan `startApp()`
- **Port**: port default HTTP server
- **Health check**: ada endpoint `/health`, `/status`, `/ping`? jika tidak ada, pakai `waitTCP`
- **Database**: PostgreSQL / MongoDB / MySQL — ambil nama field config (mis. `db.host`, `mongo.uri`, dsb)
- **Queue**: RabbitMQ / Artemis — ambil nama field config (mis. `amqp.uri`, `artemis.broker-url`)
- **Dependency HTTP eksternal**: ada panggilan ke URL luar? (webhook, telegram, 3rd party) — ini akan di-mock

#### Jika Java Spring Boot:

Baca file-file berikut secara berurutan:

1. **`pom.xml`** — ambil groupId, artifactId, version, list dependency (spring-boot, testcontainers, mongodb, rabbitmq, activemq, mysql, postgres, dsb)
2. **`src/main/resources/application.properties`** atau **`application.yml`** — baca semua key config: `server.port`, `spring.data.mongodb.uri`, `spring.datasource.url`, `app.artemis.broker-url`, dsb
3. **`src/main/java/`** — temukan package root (paling dalam yang masih `com.[company].[project]`)
4. **`src/main/java/.../Main.java`** atau **`*Application.java`** — verifikasi entry point
5. **`src/main/java/.../service/`** — list semua service, baca nama class dan method utama
6. **`src/main/java/.../handler/`** atau **`listener/`** — cari JMS/AMQP listener (ini trigger untuk test)
7. **`src/main/java/.../repository/`** — list repository dan jenis DB
8. **`src/main/java/.../model/`** atau **`entity/`** — baca nama class untuk tahu nama collection/tabel

Dari bacaan di atas, deteksi:
- **Package root**: mis. `com.jatismobile.messageintransmitter`
- **Port**: `server.port` dari application.properties
- **Database**: jenis DB dan nama field config yang dipakai
- **Queue**: jenis queue (Artemis/RabbitMQ) dan nama field config
- **Dependency HTTP eksternal**: cari `RestTemplate`, `WebClient`, `OkHttpClient` — URL yang di-call akan di-mock dengan `MockWebServer`
- **Collections/tabel runtime**: collection yang ditulis saat proses (bukan config) — ini yang di-clear antar test
- **Collections config**: collection yang dibaca saat startup (mis. `telegram_config`, `routing`) — ini yang di-seed sebelum context start

#### Jika PHP (web app):

Baca file-file berikut secara berurutan:

1. **File `*.php` di root** — list semua halaman (`index.php`, `success.php`, dsb), baca: form id/action, input id/name, inline script/style, event handler, header PHP (`header()` call)
2. **`config.php` / file config** — baca: env var yang dipakai, security headers (CSP, X-Frame-Options, dll), URL eksternal yang dipanggil
3. **`js/`** — list semua file JS yang di-load tiap halaman; identifikasi library (jQuery version, bundle vendor) dan file JS custom (form validation, dsb)
4. **`css/`** — hanya untuk tahu resource yang di-load (untuk CSP assertion)
5. **`docker-compose.yml` / `Dockerfile` / `nginx/`** — cara app dijalankan di produksi, port
6. **`logs/` atau lokasi log** — file log yang ditulis app (untuk assertion evidence)

Dari bacaan di atas, deteksi:
- **Cara run dev**: `php -S 127.0.0.1:[PORT] -t [docroot]` (docroot biasanya root project) — ini jadi `webServer.command` di Playwright
- **Halaman**: list halaman utama + halaman redirect/result (mis. index.php → success.php)
- **Security headers yang diharapkan**: CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, dsb (dari `config.php`)
- **Form flow**: form → endpoint POST → redirect; field yang divalidasi client-side; captcha/OAuth eksternal yang perlu di-stub
- **Dependency eksternal yang tidak bisa dijalankan di test**: reCAPTCHA (`grecaptcha`), Facebook SDK (`FB`), Google Analytics (`gtag`/`fbq`) — di-stub via `page.addInitScript` / route interception, atau ditoleransi dalam cek console error
- **Log file yang ditulis app** (mis. `csp_report_*.log`) — untuk assertion endpoint reporting

---

### Langkah 3 — Konfirmasi ke User

#### Jika MODE GENERATE (e2e/ belum ada):

Tampilkan ringkasan deteksi sebelum generate:

```
Terdeteksi:
- Language    : Go
- Module      : dr-converter
- Entry point : go run ./cmd/main.go
- Config      : YAML via viper (config.yaml)
- Port        : 8080
- Health check: GET /health
- Database    : PostgreSQL + MongoDB
- Queue       : RabbitMQ (exchange: fanout)
- HTTP mock   : webhook URL (app.webhook.url), telegram API

File yang akan di-generate:
  e2e/main_test.go
  e2e/helpers_test.go
  e2e/flow_test.go
  e2e/testdata/init.sql
  UAT-dr-converter.md
  Makefile (tambah target test + e2e)

Generate sekarang? (y/n)
```

Tunggu konfirmasi user sebelum melanjutkan. Jika y → lanjut ke Langkah 4A/4B/4C (MODE GENERATE).

Contoh ringkasan deteksi untuk **PHP**:

```
Terdeteksi:
- Language    : PHP (web app)
- Run dev     : php -S 127.0.0.1:8099 -t .
- Halaman     : index.php, index-id.php, success.php, success-id.php
- Flow        : form → POST accesstoken.php → redirect success(-id).php
- Security    : CSP + XFO + nosniff + Referrer-Policy (di config.php)
- Eksternal   : reCAPTCHA (grecaptcha), FB SDK (FB), gtag (di-stub/ditoleransi)
- Log         : logs/csp_report_YYYYMMDD.log

File yang akan di-generate:
  e2e/package.json
  e2e/playwright.config.js
  e2e/tests/helpers.js
  e2e/tests/security.spec.js
  e2e/tests/functional.spec.js
  UAT-embedded-sign-up.md
  Makefile (tambah target e2e)

Generate sekarang? (y/n)
```

---

#### Jika MODE UPDATE (e2e/ sudah ada):

Baca file yang sudah ada (Go: `e2e/flow_test.go`; Java: `*FlowE2EIT.java`; PHP: `e2e/tests/*.spec.js`), hitung jumlah test function. Baca juga `UAT-[project].md` jika ada.

Tampilkan menu pilihan:

```
E2E sudah ada! Terdeteksi:
- Language    : Go
- Module      : dr-converter
- Test files  : main_test.go, helpers_test.go, flow_test.go (3 test cases)
- UAT         : UAT-dr-converter.md (8 skenario)

Mau update apa?
  a) Tambah test case baru — tambah fungsi ke flow_test.go + baris ke UAT
  b) Update infrastruktur — ubah container/mock/config di main_test.go atau E2EContainers.java
  c) Regenerate semua — overwrite semua file (tidak bisa di-undo)

Pilih (a/b/c):
```

Contoh menu untuk **PHP**:

```
E2E sudah ada! Terdeteksi:
- Language    : PHP (web app)
- Test files  : security.spec.js (5 tests), functional.spec.js (6 tests)
- UAT         : UAT-embedded-sign-up.md (12 skenario)

Mau update apa?
  a) Tambah test case baru — tambah test() ke security.spec.js/functional.spec.js + baris ke UAT
  b) Update infrastruktur — ubah webServer/config/helper di playwright.config.js atau helpers.js
  c) Regenerate semua — overwrite semua file (tidak bisa di-undo)

Pilih (a/b/c):
```

Tunggu jawaban user, lalu lanjut ke Langkah 4D sesuai pilihan.

---

### Langkah 4 — Generate / Update File Kode

#### 4A. Untuk Go (MODE GENERATE):

**`e2e/main_test.go`**

Generate dengan menyesuaikan infra yang terdeteksi. Contoh jika PostgreSQL + RabbitMQ:

```go
package e2e

import (
    "context"
    "fmt"
    "net/http"
    "net/http/httptest"
    "os"
    "sync"
    "testing"
    "time"

    amqp "github.com/rabbitmq/amqp091-go"
    "github.com/testcontainers/testcontainers-go"
    tcpostgres "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/modules/rabbitmq"
)

var (
    pgConnStr   string
    amqpURI     string
    mockServer  *httptest.Server
    capturedReqs [][]byte
    mu          sync.Mutex
)

func TestMain(m *testing.M) {
    ctx := context.Background()

    // 1. Start PostgreSQL
    pgC, err := tcpostgres.Run(ctx, "postgres:16-alpine",
        tcpostgres.WithDatabase("[nama_db]_e2e"),
        tcpostgres.WithUsername("admin"),
        tcpostgres.WithPassword("admin"),
        tcpostgres.WithInitScripts("testdata/init.sql"),
        tcpostgres.BasicWaitStrategies(),
    )
    exitIf(err, "start postgres")
    defer testcontainers.TerminateContainer(pgC)
    pgConnStr, _ = pgC.ConnectionString(ctx, "sslmode=disable")

    // 2. Start RabbitMQ
    rabbitC, err := rabbitmq.Run(ctx, "rabbitmq:3-management",
        rabbitmq.WithAdminUsername("guest"),
        rabbitmq.WithAdminPassword("guest"),
    )
    exitIf(err, "start rabbitmq")
    defer testcontainers.TerminateContainer(rabbitC)
    amqpURI, _ = rabbitC.AmqpURL(ctx)

    // 3. Setup queue
    exitIf(setupQueue(), "setup queue")

    // 4. Mock server untuk dependency HTTP eksternal
    mockServer = httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // TODO: sesuaikan response dengan kebutuhan test
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"status":"ok"}`))
    }))
    defer mockServer.Close()

    // 5. Seed data
    exitIf(seedDB(ctx), "seed db")

    // 6. Start app
    exitIf(startApp(), "start app")
    defer stopApp()

    // 7. Tunggu app siap
    if !waitHTTP("http://localhost:[PORT]/health", 30*time.Second) {
        fmt.Fprintln(os.Stderr, "app tidak siap dalam 30 detik")
        os.Exit(1)
    }

    os.Exit(m.Run())
}
```

Sesuaikan:
- Tambah/hapus container sesuai infra yang terdeteksi (MongoDB, MySQL, Redis, dsb)
- Ganti `[nama_db]` dengan nama DB dari config
- Ganti `[PORT]` dengan port dari config
- Sesuaikan mock server response dengan perilaku dependency eksternal yang dipanggil app

**`e2e/helpers_test.go`**

```go
package e2e

import (
    "fmt"
    "io"
    "net"
    "net/http"
    "os"
    "os/exec"
    "syscall"
    "testing"
    "time"
)

var appCmd *exec.Cmd

func writeConfig() (string, error) {
    // TODO: sesuaikan struktur YAML dengan config project
    // Gunakan field name yang sama persis dengan yang dibaca app
    content := fmt.Sprintf(`
[field_db_host]: [host_dari_container]
[field_db_port]: [port_dari_container]
# ... sesuaikan semua field config
`, /* nilai dari container */)
    path := "/tmp/e2e-config.yaml"
    return path, os.WriteFile(path, []byte(content), 0644)
}

func startApp() error {
    configPath, err := writeConfig()
    if err != nil {
        return err
    }
    // TODO: sesuaikan entry point dan flag config
    appCmd = exec.Command("go", "run", "../cmd/main.go", "--config", configPath)
    appCmd.Stdout = os.Stdout
    appCmd.Stderr = os.Stderr
    appCmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}
    return appCmd.Start()
}

func stopApp() {
    if appCmd == nil || appCmd.Process == nil {
        return
    }
    pgid, err := syscall.Getpgid(appCmd.Process.Pid)
    if err == nil {
        syscall.Kill(-pgid, syscall.SIGKILL)
    } else {
        appCmd.Process.Kill()
    }
    appCmd.Process.Wait()
}

func waitHTTP(url string, timeout time.Duration) bool {
    client := &http.Client{Timeout: 2 * time.Second}
    deadline := time.Now().Add(timeout)
    for time.Now().Before(deadline) {
        resp, err := client.Get(url)
        if err == nil {
            resp.Body.Close()
            return true
        }
        time.Sleep(500 * time.Millisecond)
    }
    return false
}

func waitTCP(addr string, timeout time.Duration) bool {
    deadline := time.Now().Add(timeout)
    for time.Now().Before(deadline) {
        conn, err := net.DialTimeout("tcp", addr, 2*time.Second)
        if err == nil {
            conn.Close()
            return true
        }
        time.Sleep(500 * time.Millisecond)
    }
    return false
}

func waitFor(timeout time.Duration, check func() bool) bool {
    deadline := time.Now().Add(timeout)
    for time.Now().Before(deadline) {
        if check() {
            return true
        }
        time.Sleep(300 * time.Millisecond)
    }
    return false
}

func logMark(t *testing.T, label string) {
    fmt.Printf("\n>>> [E2E] %s %s @ %s <<<\n",
        label, t.Name(), time.Now().Format("15:04:05.000"))
}

func exitIf(err error, ctx string) {
    if err != nil {
        fmt.Fprintf(os.Stderr, "gagal %s: %v\n", ctx, err)
        os.Exit(1)
    }
}
```

**`e2e/flow_test.go`**

Generate 2–3 test case representatif berdasarkan service/handler yang ditemukan di project. Ikuti pola:

```go
package e2e

import (
    "context"
    "testing"
    "time"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

// TODO: ganti nama dan isi sesuai skenario bisnis utama project
func TestFlow_[NamaSkenario](t *testing.T) {
    logMark(t, "START")
    defer logMark(t, "END")
    ctx := context.Background()

    // GIVEN: bersihkan data dari test sebelumnya
    clearDB(ctx)

    // WHEN: trigger aksi (publish queue / HTTP request)
    // TODO: sesuaikan trigger dengan cara app menerima input

    // THEN: assert hasilnya
    ok := waitFor(10*time.Second, func() bool {
        // TODO: query DB atau cek mock server
        return false
    })
    assert.True(t, ok, "TODO: tulis apa yang seharusnya terjadi")
}
```

**`e2e/testdata/init.sql`** (jika pakai PostgreSQL/MySQL)

Generate DDL berdasarkan nama struct/model yang ditemukan:

```sql
-- TODO: sesuaikan dengan schema project
-- Contoh dari model yang terdeteksi:
CREATE TABLE IF NOT EXISTS [nama_tabel] (
    id          BIGSERIAL PRIMARY KEY,
    -- field lain dari struct...
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

#### 4B. Untuk Java Spring Boot (MODE GENERATE):

**`src/test/java/[package]/e2e/E2EContainers.java`**

```java
package [package_root].e2e;

import okhttp3.mockwebserver.MockWebServer;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.containers.wait.strategy.Wait;
import org.testcontainers.utility.DockerImageName;

import java.io.IOException;
import java.time.Duration;

/**
 * Singleton Docker infrastructure — dijalankan sekali untuk seluruh test run.
 * Ryuk sidecar Testcontainers membersihkan container saat JVM exit.
 */
public final class E2EContainers {

    // TODO: sesuaikan image dan container dengan infra yang terdeteksi

    // Contoh MongoDB:
    public static final MongoDBContainer MONGO;

    // Contoh ActiveMQ Artemis:
    public static final GenericContainer<?> ARTEMIS;

    // Mock HTTP server untuk setiap dependency HTTP eksternal yang ditemukan
    // TODO: tambah/hapus sesuai dependency yang ada di project
    public static final MockWebServer CLIENT_MOCK;
    public static final MockWebServer TELEGRAM_MOCK;

    public static final RecordingDispatcher CLIENT_DISPATCHER = new RecordingDispatcher();
    public static final RecordingDispatcher TELEGRAM_DISPATCHER = new RecordingDispatcher();

    static {
        MONGO = new MongoDBContainer(DockerImageName.parse("mongo:6.0"));
        MONGO.start();

        ARTEMIS = new GenericContainer<>(DockerImageName.parse("apache/activemq-artemis:2.31.2-alpine"))
                .withEnv("ARTEMIS_USER", "admin")
                .withEnv("ARTEMIS_PASSWORD", "admin")
                .withExposedPorts(61616)
                .waitingFor(Wait.forLogMessage(".*Server is now live.*", 1)
                        .withStartupTimeout(Duration.ofSeconds(120)));
        ARTEMIS.start();

        try {
            CLIENT_MOCK = startMock(CLIENT_DISPATCHER);
            TELEGRAM_MOCK = startMock(TELEGRAM_DISPATCHER);
        } catch (IOException e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    private static MockWebServer startMock(RecordingDispatcher d) throws IOException {
        MockWebServer s = new MockWebServer();
        s.setDispatcher(d);
        s.start();
        return s;
    }

    // TODO: sesuaikan method helper dengan infra yang ada
    public static String mongoUri() {
        return MONGO.getConnectionString() + "/[nama_db]_e2e";
    }

    public static String artemisBrokerUrl() {
        return String.format("tcp://%s:%d", ARTEMIS.getHost(), ARTEMIS.getMappedPort(61616));
    }

    private E2EContainers() {}
}
```

**`src/test/java/[package]/e2e/AbstractE2EIT.java`**

```java
package [package_root].e2e;

import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import com.mongodb.client.MongoDatabase;
import org.bson.Document;
import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.jms.core.JmsTemplate;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;

import java.util.Date;
import java.util.concurrent.TimeUnit;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("e2e")
public abstract class AbstractE2EIT {

    // TODO: sesuaikan konstanta dengan data test project
    protected static final String SENDER_ID = "E2E_SENDER";
    protected static final String SENDER_QUEUE = "e2e.test.queue";

    @Autowired
    protected MongoTemplate mongoTemplate;

    @Autowired
    protected JmsTemplate jmsTemplate;

    @DynamicPropertySource
    static void registerProperties(DynamicPropertyRegistry registry) {
        // Seed config collections SEBELUM Spring context start
        // (penting jika @PostConstruct app membaca DB saat startup)
        seedConfigCollections();

        // TODO: sesuaikan key properties dengan yang ada di application.properties
        registry.add("spring.data.mongodb.uri", E2EContainers::mongoUri);
        registry.add("app.artemis.broker-url", E2EContainers::artemisBrokerUrl);
        // Inject URL mock server ke config app:
        registry.add("[key.url.client]", () -> E2EContainers.CLIENT_MOCK.url("/client").toString());
        registry.add("[key.url.telegram]", () -> E2EContainers.TELEGRAM_MOCK.url("/bot").toString());
    }

    private static void seedConfigCollections() {
        try (MongoClient client = MongoClients.create(E2EContainers.MONGO.getConnectionString())) {
            MongoDatabase db = client.getDatabase("[nama_db]_e2e");

            // TODO: seed collection config yang dibaca app saat startup
            // Contoh:
            db.getCollection("telegram_config").drop();
            db.getCollection("telegram_config").insertOne(new Document()
                    .append("api_url", E2EContainers.TELEGRAM_MOCK.url("/bot").toString())
                    .append("chat_id", "-100999")
                    .append("date_created", new Date()));

            db.getCollection("routing").drop();
            db.getCollection("routing").insertOne(new Document()
                    .append("sender_id", SENDER_ID)
                    .append("queue", SENDER_QUEUE)
                    .append("date_created", new Date()));
        }
    }

    @BeforeEach
    void resetBeforeEach() {
        // Reset semua mock
        E2EContainers.CLIENT_DISPATCHER.reset();
        E2EContainers.TELEGRAM_DISPATCHER.reset();

        // TODO: clear hanya runtime collections, jangan hapus config collections
        for (String col : new String[]{/* nama collection runtime yang ada di project */}) {
            mongoTemplate.getDb().getCollection(col).drop();
        }
    }

    @BeforeEach
    void logTestStart(org.junit.jupiter.api.TestInfo info) {
        System.out.printf("%n>>> [E2E] START %s @ %s <<<%n",
            info.getDisplayName(),
            java.time.LocalTime.now().format(java.time.format.DateTimeFormatter.ofPattern("HH:mm:ss.SSS")));
    }

    @org.junit.jupiter.api.AfterEach
    void logTestEnd(org.junit.jupiter.api.TestInfo info) {
        System.out.printf(">>> [E2E] END %s @ %s <<<%n",
            info.getDisplayName(),
            java.time.LocalTime.now().format(java.time.format.DateTimeFormatter.ofPattern("HH:mm:ss.SSS")));
    }

    protected void sendToQueue(String queue, String json) {
        jmsTemplate.convertAndSend(queue, json);
    }

    protected long count(String collection) {
        return mongoTemplate.getDb().getCollection(collection).countDocuments();
    }

    protected Document findOne(String collection, String field, Object value) {
        return mongoTemplate.getDb().getCollection(collection)
                .find(new Document(field, value)).first();
    }

    protected String awaitRequestBody(RecordingDispatcher dispatcher, String token, long timeoutSeconds)
            throws InterruptedException {
        long deadline = System.currentTimeMillis() + timeoutSeconds * 1000L;
        long remaining;
        while ((remaining = deadline - System.currentTimeMillis()) > 0) {
            var req = dispatcher.poll(remaining, TimeUnit.MILLISECONDS);
            if (req == null) return null;
            String body = req.getBody().readUtf8();
            if (body.contains(token)) return body;
        }
        return null;
    }
}
```

**`src/test/java/[package]/e2e/RecordingDispatcher.java`**

```java
package [package_root].e2e;

import okhttp3.mockwebserver.Dispatcher;
import okhttp3.mockwebserver.MockResponse;
import okhttp3.mockwebserver.RecordedRequest;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;

public class RecordingDispatcher extends Dispatcher {

    private final BlockingQueue<RecordedRequest> recorded = new LinkedBlockingQueue<>();
    private volatile int responseCode = 200;
    private volatile String responseBody = "{\"status\":\"ok\"}";

    @Override
    public MockResponse dispatch(RecordedRequest request) {
        recorded.add(request);
        return new MockResponse()
                .setResponseCode(responseCode)
                .setBody(responseBody)
                .addHeader("Content-Type", "application/json");
    }

    public void setResponseCode(int code, String body) {
        this.responseCode = code;
        this.responseBody = body;
    }

    public RecordedRequest poll(long timeout, TimeUnit unit) throws InterruptedException {
        return recorded.poll(timeout, unit);
    }

    public void reset() {
        recorded.clear();
        responseCode = 200;
        responseBody = "{\"status\":\"ok\"}";
    }
}
```

**`src/test/java/[package]/e2e/[NamaService]FlowE2EIT.java`**

Generate 2–3 test case berdasarkan service utama yang ditemukan:

```java
package [package_root].e2e;

import org.bson.Document;
import org.junit.jupiter.api.Test;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

class [NamaService]FlowE2EIT extends AbstractE2EIT {

    // TODO: ganti nama method dan isi sesuai skenario bisnis utama
    @Test
    void [namaAlurSukses]() throws InterruptedException {
        // GIVEN: data sudah di-seed di AbstractE2EIT

        // WHEN: kirim trigger
        sendToQueue(SENDER_QUEUE, buildPayload("test-id-001"));

        // THEN: assert
        await().atMost(15, TimeUnit.SECONDS).untilAsserted(() -> {
            Document doc = findOne("[nama_collection]", "[field_id]", "test-id-001");
            assertThat(doc).isNotNull();
            assertThat(doc.getString("[field_status]")).isEqualTo("[nilai_sukses]");
        });
    }

    @Test
    void [namaAlurGagal]() {
        // GIVEN: override mock untuk return error
        E2EContainers.CLIENT_DISPATCHER.setResponseCode(500, "{\"error\":\"boom\"}");

        // WHEN
        sendToQueue(SENDER_QUEUE, buildPayload("test-id-002"));

        // THEN
        await().atMost(15, TimeUnit.SECONDS).untilAsserted(() -> {
            assertThat(count("[nama_collection_error]")).isGreaterThan(0);
        });
    }

    private String buildPayload(String id) {
        // TODO: sesuaikan dengan format payload yang diterima app
        return String.format("{\"id\":\"%s\",\"sender_id\":\"%s\"}", id, SENDER_ID);
    }
}
```

**`src/test/resources/application-e2e.properties`**

```properties
# Disable scheduler agar tidak jalan otomatis (dikontrol manual di test)
# TODO: sesuaikan key dengan yang ada di application.properties
app.scheduled.enabled=false

# Timeout singkat supaya test lebih cepat
app.rest-template.timeout=5
app.client.retry=1

# SMTP ke in-memory GreenMail (jika app kirim email)
spring.mail.host=127.0.0.1
spring.mail.port=33025
spring.mail.username=test
spring.mail.password=test

logging.level.[package_root]=DEBUG
```

---

#### 4C. Untuk PHP (MODE GENERATE — Playwright):

**`e2e/package.json`**

```json
{
  "name": "[nama-project]-e2e",
  "version": "1.0.0",
  "private": true,
  "description": "E2E test automation for [nama-project] (PHP)",
  "scripts": {
    "test": "playwright test",
    "test:security": "playwright test tests/security.spec.js",
    "test:functional": "playwright test tests/functional.spec.js",
    "report": "playwright show-report"
  },
  "devDependencies": {
    "@playwright/test": "^1.48.0"
  }
}
```

**`e2e/playwright.config.js`**

```js
const { defineConfig } = require('@playwright/test');

module.exports = defineConfig({
  testDir: './tests',
  timeout: 30000,
  expect: { timeout: 10000 },
  fullyParallel: false,
  workers: 1,
  retries: 0,
  reporter: [['list'], ['html', { outputFolder: 'playwright-report', open: 'never' }]],
  use: {
    baseURL: 'http://127.0.0.1:[PORT]',
    headless: true,
    screenshot: 'only-on-failure',
    trace: 'retain-on-failure',
  },
  outputDir: 'test-results',
  webServer: {
    command: 'php -S 127.0.0.1:[PORT] -t ..',  // TODO: sesuaikan docroot (biasanya '..' karena config berada di e2e/)
    cwd: __dirname,
    url: 'http://127.0.0.1:[PORT]/[halaman_utama].php',
    reuseExistingServer: false,
    timeout: 30000,
  },
});
```

**`e2e/tests/helpers.js`**

```js
// Helper bersama untuk security + functional spec
const { expect } = require('@playwright/test');

// Daftar halaman utama project (TODO: sesuaikan)
const PAGES = ['index.php', 'success.php']; // TODO: lengkapi semua halaman

// Header security yang wajib ada di semua halaman (TODO: sesuaikan dengan config.php)
const REQUIRED_HEADERS = {
  'x-frame-options': 'DENY',
  'x-content-type-options': 'nosniff',
  'referrer-policy': 'strict-origin-when-cross-origin',
};

// Asal error console yang ditoleransi (dependency eksternal yang tidak bisa dihindari di test env)
// TODO: sesuaikan — contoh: Google/Facebook domain boleh error network tapi bukan error JS app sendiri
const ALLOWED_CONSOLE_ORIGINS = [
  /googletagmanager\.com/,
  /google\.com/,
  /google-analytics\.com/,
  /gstatic\.com/,
  /facebook\.net/,
  /facebook\.com/,
];

async function assertSecurityHeaders(res, extra = {}) {
  for (const [header, expected] of Object.entries({ ...REQUIRED_HEADERS, ...extra })) {
    const value = res.headers()[header];
    expect(value, `header ${header} harus ada`).toBeTruthy();
    if (expected && expected !== '*') {
      expect(value, `header ${header} harus berisi ${expected}`).toContain(expected);
    }
  }
}

async function collectConsoleErrors(page) {
  const errors = [];
  page.on('console', (msg) => {
    if (msg.type() === 'error') errors.push(msg.text());
  });
  page.on('pageerror', (err) => errors.push(String(err)));
  return errors;
}

function assertNoAppErrors(errors, origin = window?.location?.origin || '') {
  const appErrors = errors.filter((e) => !ALLOWED_CONSOLE_ORIGINS.some((re) => re.test(e)));
  expect(appErrors, `console error tidak boleh ada:\n${appErrors.join('\n')}`).toEqual([]);
}

module.exports = {
  PAGES,
  REQUIRED_HEADERS,
  ALLOWED_CONSOLE_ORIGINS,
  assertSecurityHeaders,
  collectConsoleErrors,
  assertNoAppErrors,
};
```

**`e2e/tests/security.spec.js`**

```js
const { test, expect } = require('@playwright/test');
const {
  PAGES,
  assertSecurityHeaders,
  collectConsoleErrors,
  assertNoAppErrors,
} = require('./helpers');

// TODO: sesuaikan dengan CSP yang ada di config.php
const CSP_FRAGMENTS = [
  "default-src 'self'",
  'script-src',
  'style-src',
  'frame-ancestors',
  'report-uri',
];

test.describe('Security headers', () => {
  for (const page of PAGES) {
    test(`[${page}] security headers + CSP + no console error`, async ({ page: p }) => {
      const errors = collectConsoleErrors(p);
      const res = await p.goto(`/${page}`);

      expect(res.status(), 'HTTP status harus 200').toBe(200);
      await assertSecurityHeaders(res, {
        'content-security-policy': CSP_FRAGMENTS.join(' '),
      });
      // TODO: sesuaikan cek per-fragmen jika perlu:
      const csp = res.headers()['content-security-policy'] || '';
      for (const fragment of CSP_FRAGMENTS) {
        expect(csp, `CSP harus berisi "${fragment}"`).toContain(fragment);
      }
      await p.waitForLoadState('load');
      assertNoAppErrors(errors);
    });
  }
});

test.describe('CSP reporting endpoint', () => {
  test('POST /csp-report.php → 204 + log tertulis', async ({ request }) => {
    // TODO: sesuaikan nama endpoint + lokasi log dengan project
    const res = await request.post('/csp-report.php', {
      headers: { 'Content-Type': 'application/csp-report' },
      data: {
        'csp-report': {
          'blocked-uri': 'https://e2e-test.invalid/x.js',
          'document-uri': 'http://e2e/index.php',
          'violated-directive': 'script-src',
          'effective-directive': 'script-src',
          'original-policy': 'e2e-test',
          'source-file': 'http://e2e/index.php',
          'line-number': 10,
        },
      },
    });
    expect(res.status(), 'endpoint report harus 204').toBe(204);
    // TODO: verifikasi log file di disk (bisa via fs) lalu hapus log test
  });
});
```

**`e2e/tests/functional.spec.js`**

```js
const { test, expect } = require('@playwright/test');
const { collectConsoleErrors, assertNoAppErrors } = require('./helpers');

// TODO: sesuaikan dengan form & alur project. Prinsip:
// - alur yang TIDAK butuh dependency eksternal diuji penuh
// - dependency eksternal (grecaptcha, FB, gtag) di-stub via page.addInitScript
// - FB/OAuth asli tetap dicover manual di UAT

test.describe('[Nama Alur] — validasi form', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/index.php'); // TODO: halaman form
  });

  test('phone invalid → pesan validasi tampil, tidak submit', async ({ page }) => {
    // TODO: sesuaikan selector field + teks pesan
    await page.fill('#fullname', 'Test User');
    await page.fill('#corporate_email', 'test@corp.com');
    await page.fill('#phone_number', 'abc123');
    await page.click('button[type=submit]');
    const msg = await page.locator('#phone_number').evaluate(
      (el) => el.validationMessage
    );
    expect(msg).toContain('max 13-digit');
    expect(page.url()).not.toContain('accesstoken.php'); // tidak ada submit
  });

  test('captcha kosong → alert tampil (grecaptcha di-stub)', async ({ page }) => {
    // Stub dependency eksternal SEBELUM page load
    await page.addInitScript(() => {
      window.grecaptcha = { getResponse: () => '' };
      window.fbq = undefined;
    });
    await page.goto('/index.php');
    await page.fill('#fullname', 'Test User');
    await page.fill('#corporate_email', 'test@corp.com');
    await page.fill('#phone_number', '08123456789');
    let dialogText = '';
    page.on('dialog', async (d) => { dialogText = d.message(); await d.accept(); });
    await page.click('button[type=submit]');
    expect(dialogText).toContain('CAPTCHA'); // TODO: sesuaikan teks alert
  });

  test('submit valid → sampai FB.login (FB di-stub, capture config)', async ({ page }) => {
    await page.addInitScript(() => {
      window.grecaptcha = { getResponse: () => 'e2e-token' };
      window.fbq = () => {};
      window.__fbLoginCaptured = null;
      window.FB = {
        init: () => {},
        login: (cb, opts) => {
          window.__fbLoginCaptured = opts;
          // jangan panggil cb dengan authResponse → tidak ada POST/submit di test
        },
      };
    });
    await page.goto('/index.php');
    await page.fill('#fullname', 'Test User');
    await page.fill('#corporate_email', 'test@corp.com');
    await page.fill('#phone_number', '08123456789');
    await page.click('button[type=submit]');
    const captured = await page.evaluate(() => window.__fbLoginCaptured);
    expect(captured, 'FB.login harus dipanggil dengan config embedded signup').toBeTruthy();
    expect(captured.response_type).toBe('code'); // TODO: sesuaikan assert
  });
});

test.describe('Regresi UI', () => {
  test('menu dropdown terbuka', async ({ page }) => {
    await page.goto('/index.php');
    await page.hover('nav .dropdown'); // TODO: sesuaikan selector menu
    await expect(page.locator('nav .dropdown .dropdown-menu')).toBeVisible();
  });

  test('sticky header aktif setelah scroll', async ({ page }) => {
    await page.goto('/index.php');
    await page.evaluate(() => window.scrollTo(0, 400));
    await page.waitForTimeout(500);
    await expect(page.locator('header')).toHaveClass(/sticky/); // TODO: sesuaikan
  });
});
```

**`e2e/.gitignore`** (jangan commit artifact):

```
node_modules/
test-results/
playwright-report/
screenshots/
```

Prinsip penting untuk spec PHP:
- JANGAN hard-assert flow OAuth/captcha asli (butuh kredensial real + network) — itu dicover manual di UAT. Di spec, stub `window.FB` / `window.grecaptcha` / `window.fbq` via `page.addInitScript()` untuk membuktikan flow kode app sampai titik panggilan eksternal.
- Console error assertion harus punya daftar origin eksternal yang ditoleransi (GA/FB/reCAPTCHA bisa gagal load di test env) — error JS app sendiri tetap wajib 0.
- Untuk app yang menulis log file (mis. `csp_report_*.log`), assertion endpoint bisa baca file log via `fs` lalu hapus log test agar evidence bersih.
- Screenshot diambil otomatis saat failure (`only-on-failure`); untuk evidence UAT bisa tambah `await page.screenshot({ path: 'screenshots/[nama].png' })` pada happy-path test.

---

#### 4D. MODE UPDATE — sesuai pilihan user

**Pilihan a) Tambah test case baru:**

1. Baca `e2e/flow_test.go` (Go), `*FlowE2EIT.java` (Java), atau `e2e/tests/*.spec.js` (PHP) untuk memahami pola test yang sudah ada
2. Baca `UAT-[project].md` untuk melihat skenario yang sudah ada
3. Tanya user: "Skenario baru apa yang ingin ditambahkan? Jelaskan trigger dan expected result-nya."
4. Tunggu jawaban, lalu:
   - **Go**: tambahkan fungsi `TestFlow_[NamaBaru]` ke `e2e/flow_test.go` mengikuti pola yang ada. Jangan ubah fungsi yang sudah ada.
   - **Java**: tambahkan method `@Test` ke `*FlowE2EIT.java` yang sudah ada. Jika scope berbeda, buat file `[NamaBaru]FlowE2EIT.java` baru.
   - **PHP**: tambahkan `test(...)` ke `security.spec.js` (jika menyangkut header/endpoint) atau `functional.spec.js` (jika menyangkut UI/flow). Jika scope berbeda, buat file `[nama-baru].spec.js`.
5. Tambahkan baris baru ke UAT-[project].md di section yang sesuai (atau buat section baru jika perlu)
6. Tampilkan ringkasan: "Ditambahkan: 1 test case di [file] + 1 baris di UAT-[project].md"

**Pilihan b) Update infrastruktur:**

1. Baca file infra yang ada:
   - **Go**: baca `e2e/main_test.go` dan `e2e/helpers_test.go`
   - **Java**: baca `E2EContainers.java` dan `AbstractE2EIT.java` dan `application-e2e.properties`
   - **PHP**: baca `e2e/playwright.config.js` dan `e2e/tests/helpers.js`
2. Tampilkan infra yang terdeteksi saat ini:
   ```
   Infra saat ini:
   - Container : PostgreSQL 16, RabbitMQ 3
   - Mock      : webhookMock (port auto)
   - Config    : writeConfig() di helpers_test.go
   ```
   (Untuk PHP: tampilkan `webServer.command`, port, baseURL, dan list header yang di-assert di `helpers.js`.)
3. Tanya user: "Apa yang ingin diubah? (contoh: tambah Redis container, tambah mock baru, ganti versi image)" — untuk PHP contohnya: ganti port, tambah header yang di-assert, tambah origin toleransi console error
4. Tunggu jawaban, lalu update hanya bagian yang diminta:
   - Tambah container → tambah ke blok `TestMain` (Go) atau `E2EContainers static {}` (Java)
   - Tambah mock server → tambah variable dan inisialisasi di file infra yang sesuai, tambah juga ke `@DynamicPropertySource` / `writeConfig()`
   - Ganti image → update string Docker image saja
   - PHP: ganti port → `playwright.config.js` (webServer + baseURL); tambah header → `REQUIRED_HEADERS` di `helpers.js`; tambah toleransi console → `ALLOWED_CONSOLE_ORIGINS`
5. Jangan ubah test case di `flow_test.go`, `*FlowE2EIT.java`, atau `*.spec.js` kecuali memang terdampak langsung

**Pilihan c) Regenerate semua:**

Lanjutkan ke Langkah 4A, 4B, atau 4C sesuai bahasa project — generate ulang semua file dengan overwrite.

---

### Langkah 5 — UAT-[nama-project].md

**MODE GENERATE**: buat file baru `UAT-[nama-project].md` dengan format di bawah.
**MODE UPDATE (pilihan a)**: append baris ke section yang sesuai di file yang sudah ada. Jangan ubah baris lain. Jika skenario baru tidak cocok di section manapun, tambahkan section baru di akhir file.
**MODE UPDATE (pilihan b/c)**: langkah ini tidak perlu dijalankan.

---

Format wajib untuk file baru (MODE GENERATE), buat file `e2e/UAT-[nama-project].md` dengan format yang SAMA PERSIS seperti contoh berikut. Ini adalah format wajib:

```markdown
# PT. Jatis Mobile — [Nama Project]
## [Nama Fitur] UAT

| | |
|---|---|
| **Team Developer** | [isi dari info project] |
| **Tester** | |
| **Branch** | |
| **TRD** | |

---

## 0. Preparation

| No | Remarks | Setup Data | Steps | Expected Results | Actual Result | Pass/Fail |
|---|---|---|---|---|---|---|
| 0.1.1 | Setup [infra 1] | [prasyarat] | [langkah setup] | [kondisi sukses] | | ⬜ |
| 0.2.1 | Setup [infra 2] | [prasyarat] | [langkah setup] | [kondisi sukses] | | ⬜ |
| 0.3.1 | Insert Test Data — [collection 1] | [DB connected] | [query insert lengkap]<br>Verify: [query verify] | [data masuk dengan benar] | | ⬜ |
| 0.4.1 | Start Application | [semua infra running] | 1. [cara start app]<br>2. Cek startup logs<br>3. Verify tidak ada error | App started, semua koneksi OK | | ⬜ |

---

## 1. [Nama Alur Bisnis Utama]

| No | Skenario | Setup Data | Steps | Expected Results | Actual Result | Pass/Fail |
|---|---|---|---|---|---|---|
| 1.1.1 | [skenario happy path] | [kondisi awal lengkap] | 1. [trigger]<br>2. [monitor logs]<br>3. [cek DB/mock] | [kondisi yang diharapkan detail] | | ⬜ |

---

## 2. [Nama Alur Bisnis Kedua]

| No | Skenario | Setup Data | Steps | Expected Results | Actual Result | Pass/Fail |
|---|---|---|---|---|---|---|
| 2.1.1 | [skenario failure] | [kondisi error] | [langkah trigger] | [kondisi error yang diharapkan] | | ⬜ |

---

## 3. Edge Cases

| No | Skenario | Setup Data | Steps | Expected Results | Actual Result | Pass/Fail |
|---|---|---|---|---|---|---|
| 3.1.1 | [edge case 1] | [kondisi khusus] | [langkah] | [hasil yang diharapkan] | | ⬜ |
```

**Panduan mengisi konten UAT:**
- Section 0 (Preparation): satu baris per langkah setup (start infra, insert seed data per collection, start app)
- Section per fitur: kelompokkan berdasarkan alur bisnis yang ditemukan di service/handler
- Setup Data: isi query insert MongoDB / SQL yang nyata berdasarkan schema yang ditemukan
- Steps: isi payload/trigger yang nyata berdasarkan format yang diterima handler app
- Expected Results: kondisi spesifik di DB (collection, field, nilai) bukan deskripsi umum

**Panduan UAT untuk PHP (web app):**
- Section 0 (Preparation): deploy app (SIT/Docker/`php -S`), setup env var (`APP_ID`, `RECAPTCHA_*`, dsb), siapkan kredensial OAuth/captcha untuk test manual
- Section per fitur: kelompokkan per halaman/alur (form register → redirect success; halaman statis; menu)
- Setup Data: kredensial akun bisnis FB / akun uji yang dipakai
- Steps: langkah klik/isi di browser; sertakan juga langkah yang hanya bisa manual (OAuth login asli) — ini pelengkap dari yang sudah diotomasi Playwright
- Expected Results: kondisi UI spesifik (pesan validasi, halaman redirect, elemen terlihat) + header response bila relevan

---

### Langkah 6 — Update Makefile / pom.xml

**Go — tambahkan ke `Makefile` (unit test DAN e2e terpisah):**

```makefile
# Unit test saja (cepat, tanpa Docker)
test:
	go test ./... -count=1

# Unit test dengan coverage
test-coverage:
	go test ./... -count=1 -coverprofile=coverage.out
	go tool cover -html=coverage.out -o coverage.html

# E2E test automation (butuh Docker)
e2e:
	cd e2e && go test -v -timeout 10m ./...

# E2E test — jalankan satu test case saja
# Contoh: make e2e-run TEST=TestFlow_WebhookFail
e2e-run:
	cd e2e && go test -v -run $(TEST) -timeout 10m ./...

# E2E test — pertahankan DB setelah test untuk inspeksi manual
e2e-keep:
	cd e2e && KEEP_DB=true go test -v -timeout 10m ./...
```

**Java — tambahkan dua profile ke `pom.xml` (unit test DAN e2e terpisah):**

```xml
<profiles>
    <!-- Unit test saja (default): mvn test -->
    <profile>
        <id>unit</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <configuration>
                        <excludes>
                            <exclude>**/*E2EIT.java</exclude>
                        </excludes>
                    </configuration>
                </plugin>
            </plugins>
        </build>
    </profile>

    <!-- E2E test automation (butuh Docker): mvn test -Pe2e -->
    <profile>
        <id>e2e</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <configuration>
                        <includes>
                            <include>**/*E2EIT.java</include>
                        </includes>
                    </configuration>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

Cara run:
```bash
mvn test          # unit test saja (default)
mvn test -Pe2e    # e2e test automation
```

**PHP — tambahkan ke `Makefile` (atau buat baru jika belum ada):**

```makefile
# Setup E2E sekali saja (install deps + download browser chromium ~100MB)
e2e-setup:
	cd e2e && npm install && npx playwright install chromium

# E2E test automation (webServer php -S di-start otomatis oleh Playwright)
e2e:
	cd e2e && npx playwright test

# Jalankan satu spec / satu test saja
# Contoh: make e2e-run TEST="functional.spec.js"
e2e-run:
	cd e2e && npx playwright test $(TEST)

# Jalankan dengan browser terlihat (debug)
e2e-ui:
	cd e2e && npx playwright test --headed

# Buka HTML report hasil test terakhir
e2e-report:
	cd e2e && npx playwright show-report
```

Cara run:
```bash
make e2e-setup   # sekali saja
make e2e         # e2e test automation
```

---

### Langkah 7 — Ringkasan Output

Setelah semua file di-generate, tampilkan ringkasan:

**Untuk Go:**
```
✅ File yang di-generate:

e2e/
├── main_test.go          ← TEAR UP: spin up [infra via testcontainers], seed, start app
│                            TEAR DOWN: defer TerminateContainer + stopApp
├── helpers_test.go       ← startApp, stopApp, waitFor, logMark, clearDB
├── flow_test.go          ← PROSES + VALIDASI: [N] test case
└── testdata/
    └── init.sql          ← DDL untuk [list tabel]

UAT-[project].md          ← dokumentasi UAT format tabel horizontal

Makefile — target ditambahkan:
  make test           → unit test saja
  make test-coverage  → unit test + coverage report
  make e2e            → e2e test automation (butuh Docker)
  make e2e-run TEST=X → jalankan satu test case
  make e2e-keep       → pertahankan DB setelah test

⚠️  Perlu disesuaikan manual:
- writeConfig() di helpers_test.go — field YAML harus sama persis dengan yang dibaca app
- seedDB() di main_test.go — sesuaikan data seed dengan skenario bisnis
- clearDB() di helpers_test.go — pastikan semua tabel/collection runtime ikut di-clear
- init.sql — lengkapi DDL dengan constraint dan index yang dibutuhkan

Jalankan:
  make e2e
```

**Untuk Java:**
```
✅ File yang di-generate:

src/test/java/[package]/e2e/
├── E2EContainers.java           ← TEAR UP: Docker infra via testcontainers (singleton)
│                                   TEAR DOWN: Ryuk sidecar saat JVM exit
├── AbstractE2EIT.java           ← seed config, @BeforeEach reset, helper methods
├── RecordingDispatcher.java     ← mock HTTP recorder untuk dependency eksternal
└── [NamaService]FlowE2EIT.java  ← PROSES + VALIDASI: [N] test case

src/test/resources/
└── application-e2e.properties   ← config override (disable scheduler, timeout singkat)

UAT-[project].md                 ← dokumentasi UAT format tabel horizontal

pom.xml — profile ditambahkan:
  mvn test        → unit test saja (default, exclude *E2EIT)
  mvn test -Pe2e  → e2e test automation (butuh Docker, include *E2EIT)

⚠️  Perlu disesuaikan manual:
- E2EContainers.java — pastikan image Docker dan port sesuai dengan versi yang dipakai
- AbstractE2EIT.java — sesuaikan seedConfigCollections() dengan collection config project
- AbstractE2EIT.java — sesuaikan list collection yang di-clear di resetBeforeEach()
- application-e2e.properties — sesuaikan key properties dengan yang ada di project

Jalankan:
  mvn test -Pe2e
```

**Untuk PHP (Playwright):**
```
✅ File yang di-generate:

e2e/
├── package.json             ← deps @playwright/test + script npm
├── playwright.config.js     ← TEAR UP/TEAR DOWN: webServer auto-start php -S (otomatis)
├── .gitignore               ← node_modules/, test-results/, playwright-report/, screenshots/
└── tests/
    ├── helpers.js           ← PAGES, REQUIRED_HEADERS, ALLOWED_CONSOLE_ORIGINS, assert helper
    ├── security.spec.js     ← VALIDASI header security + CSP + endpoint report per halaman
    └── functional.spec.js   ← PROSES + VALIDASI: form flow (stub captcha/FB) + regresi UI

UAT-[project].md             ← dokumentasi UAT format tabel horizontal (flow manual: OAuth/captcha asli)
Makefile — target: e2e-setup, e2e, e2e-run, e2e-ui, e2e-report

⚠️  Perlu disesuaikan manual:
- playwright.config.js — port webServer + halaman utama di `url`
- helpers.js — daftar PAGES, REQUIRED_HEADERS (samakan dengan config.php), ALLOWED_CONSOLE_ORIGINS
- security.spec.js — fragment CSP yang di-assert + nama endpoint report (mis. csp-report.php)
- functional.spec.js — selector form/field/teks pesan validasi sesuai HTML asli

Jalankan:
  make e2e-setup && make e2e
```

---

## Referensi Implementasi Nyata

Jika perlu melihat contoh kode yang sudah berjalan, baca file-file berikut:

**Go (testcontainers + MongoDB + PostgreSQL + RabbitMQ):**
- `rte-cimb-niaga/costerdrconverter/e2e/main_test.go`
- `rte-cimb-niaga/costerdrconverter/e2e/helpers_test.go`
- `rte-cimb-niaga/costerdrconverter/e2e/flow_test.go`

**Java (Spring Boot + MongoDB + ActiveMQ Artemis):**
- `cimb_gateway/message-in-transmitter/dev/src/test/java/.../e2e/E2EContainers.java`
- `cimb_gateway/message-in-transmitter/dev/src/test/java/.../e2e/AbstractE2EIT.java`
- `cimb_gateway/message-in-transmitter/dev/src/test/java/.../e2e/MessageInFlowE2EIT.java`

**PHP (web app + Playwright):**
- `jatis_website/embedded-sign-up/e2e/package.json`
- `jatis_website/embedded-sign-up/e2e/playwright.config.js`
- `jatis_website/embedded-sign-up/e2e/tests/helpers.js`
- `jatis_website/embedded-sign-up/e2e/tests/security.spec.js`
- `jatis_website/embedded-sign-up/e2e/tests/functional.spec.js`
- `jatis_website/embedded-sign-up/docs/UAT-embedded-sign-up.md`

**Dokumentasi konsep & panduan lengkap** (tersedia lokal di `.claude/docs/`):
- `../docs/README.md` — konsep, kelebihan, perbedaan dengan unit/integration test
- `../docs/GOLANG.md` — panduan implementasi lengkap Go
- `../docs/JAVA.md` — panduan implementasi lengkap Java Spring Boot

**Contoh format UAT MD:**
- `../docs/uat-example.md` — contoh nyata UAT dokumentasi format tabel horizontal

---

## Format E2E Log Evidence

Setelah E2E test selesai dijalankan, generate evidence MD dengan format berikut:

### Struktur File
```
# {Project} — E2E Log Evidence
**Branch:** `{branch}` | **Date:** {date}
**Infra:** {DB} + {Queue} (Docker via testcontainers)
**Status:** {N}/{total} PASSED

## Test Matrix
| # | Event | Type | Direction | Field | Route | Status |
...

## Log Evidence — {Queue} + {DB}

### test01 — {event} / {type} / {direction} → PASS

*Full raw log verbatim dari app:*

```
TIMESTAMP  LEVEL REQID  Received message: {...full JSON input...}
TIMESTAMP  LEVEL REQID  [service.ProcessMsgIn] Processing WA call webhook for senderID XXXXXX
TIMESTAMP  LEVEL REQID  [gateway.SendWebhookWithAlert] Start sending webhook to: URL
TIMESTAMP  LEVEL REQID  [gateway.SendWebhookWithAlert] Curl: curl -X POST -d '{...full body...}' -H '...'
TIMESTAMP  LEVEL REQID  [gateway.SendWebhookWithAlert] Response: {"status":"ok"}
TIMESTAMP  LEVEL REQID  [gateway.SendWebhookWithAlert] Webhook call completed - URL, SenderID, Status, Response Time
TIMESTAMP  LEVEL REQID  Message acknowledged (sync mode)
```

| No | Check | Value | Status |
|:--:|-------|-------|:------:|
| 1 | AMQP consume | Message diterima dari queue | ✅ |
| 2 | Route | `getField() → ...` | ✅ |
| 3 | SenderID | `display_phone_number="..."` | ✅ |
| ... | ... | ... | ✅ |

## Infrastructure
| Service | Container | Image | Status |
...

## Full Test Run Output
```
$ go test ./e2e/ -v -timeout 10m -count=1
...
```

## Build Summary
```
Tests:     {N}
Passed:    {count}
...
```
```

### Aturan
- **Dua log terpisah**: TAMPILKAN debug log dan error log sebagai dua blok kode terpisah untuk setiap testcase. Format: `**e2e_debug.log:**` ```...``` `**e2e_error.log:**` ```...```. JANGAN gabungkan jadi satu.
- **Log verbatim**: copy-paste langsung dari file log app (`e2e/logs/e2e_debug.log` dan `e2e/logs/e2e_error.log`). JANGAN ringkas, JANGAN potong body Curl.
- **(no error)**: Jika error log kosong untuk testcase tersebut, tulis `(no error)` di blok e2e_error.log.
- **Checklist**: format tabel `| No | Check | Value | Status |`, semua icon ✅. Uniform di semua test case.
- **Timestamp + Request ID**: tampilkan lengkap agar bisa ditelusuri.
- **test yang SKIP/FAIL**: tetap cantumkan dengan penjelasan kenapa.

### Format Khusus PHP (Playwright)

Setelah E2E test selesai dijalankan, generate evidence MD dengan format browser-based:

```
# {Project} — E2E Log Evidence (Playwright)
**Branch:** `{branch}` | **Date:** {date}
**Runner:** Playwright + Chromium headless | **WebServer:** php -S 127.0.0.1:{PORT} (auto by Playwright)
**Status:** {N}/{total} PASSED

## Test Matrix
| # | Test | Spec | Assertion Utama | Status |
...

## Evidence per Test Case

### test01 — {judul} → PASS
| No | Check | Value | Status |
|:--:|-------|-------|:------:|
| 1 | HTTP status | 200 | ✅ |
| 2 | X-Frame-Options | DENY | ✅ |
| 3 | Content-Security-Policy | memuat frame-ancestors 'none' | ✅ |
| 4 | Console error (app) | (no error) | ✅ |
| 5 | {cek UI: elemen visible/class/teks} | {...} | ✅ |

**Screenshot:** `e2e/screenshots/{nama}.png`
**Console (filter app-only):**
```
(no error)   ← atau copy-paste error verbatim jika FAIL
```

## Full Test Run Output
```
$ cd e2e && npx playwright test
Running N tests using 1 worker
  ✓ tests/security.spec.js:... (Xms)
  ✓ tests/functional.spec.js:... (Xms)
  ...
N passed (Xs)
```

## Build Summary
```
Tests:     {N}
Passed:    {count}
Failed:    {count}
Skipped:   {count}
Duration:  {Xs}
```
```

Aturan tambahan PHP:
- **Screenshot wajib** untuk happy-path test (ambil via `page.screenshot()` ke `e2e/screenshots/`), lampirkan path-nya di evidence. Screenshot failure otomatis ada di `e2e/test-results/`.
- **Console evidence**: tampilkan error console yang di-capture helper `collectConsoleErrors` — filter yang origin eksternal boleh di-catat terpisah dengan label "(toleransi eksternal)".
- **Log file app**: jika app menulis log (mis. `csp_report_YYYYMMDD.log`), lampirkan isi log hasil test verbatim (atau `(no error)`), lalu hapus log test setelah evidence di-generate.
- **Header response**: buktikan tiap security header yang di-assert dengan nilai aktual dari `curl -I` atau `res.headers()`.
