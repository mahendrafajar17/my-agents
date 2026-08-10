---
name: e2e-gen
description: Generate E2E test automation boilerplate dan UAT documentation untuk project Go atau Java Spring Boot. Gunakan ketika user mengetik /e2e-gen atau meminta generate E2E test, E2E automation, E2E evidence, UAT documentation, atau testcontainers boilerplate. Menghasilkan e2e/main_test.go, helpers_test.go, flow_test.go, testdata/init.sql, UAT MD, dan E2E-LOG-EVIDENCE MD.
---

Generate E2E test automation boilerplate dan UAT documentation untuk project Go atau Java Spring Boot. Menghasilkan file siap pakai: struktur folder e2e/, kode boilerplate lengkap, dan UAT .md dengan format tabel horizontal.

**Alur E2E test yang di-generate:**
```
TEAR UP   → spin up Docker (DB/queue/cache) + seed data + jalankan app
PROSES    → trigger aksi (publish queue / HTTP request)
VALIDASI  → assert DB / mock server / HTTP response
TEAR DOWN → matikan app + hapus container (otomatis via defer)
```

Baik Go maupun Java menggunakan **Testcontainers** untuk spin up Docker infra secara otomatis dari dalam test — tidak perlu setup manual.

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
3. Tidak ada keduanya → tanya user

Setelah deteksi bahasa, cek keberadaan E2E yang sudah ada:
- **Go**: cek apakah folder `e2e/` sudah ada dan berisi `main_test.go`
- **Java**: cek apakah folder `src/test/java/.../e2e/` sudah ada dan berisi `E2EContainers.java`

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

Tunggu konfirmasi user sebelum melanjutkan. Jika y → lanjut ke Langkah 4A/4B (MODE GENERATE).

---

#### Jika MODE UPDATE (e2e/ sudah ada):

Baca file yang sudah ada (Go: `e2e/flow_test.go`; Java: `*FlowE2EIT.java`), hitung jumlah test function. Baca juga `UAT-[project].md` jika ada.

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

Tunggu jawaban user, lalu lanjut ke Langkah 4C sesuai pilihan.

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

#### 4C. MODE UPDATE — sesuai pilihan user

**Pilihan a) Tambah test case baru:**

1. Baca `e2e/flow_test.go` (Go) atau `*FlowE2EIT.java` (Java) untuk memahami pola test yang sudah ada
2. Baca `UAT-[project].md` untuk melihat skenario yang sudah ada
3. Tanya user: "Skenario baru apa yang ingin ditambahkan? Jelaskan trigger dan expected result-nya."
4. Tunggu jawaban, lalu:
   - **Go**: tambahkan fungsi `TestFlow_[NamaBaru]` ke `e2e/flow_test.go` mengikuti pola yang ada. Jangan ubah fungsi yang sudah ada.
   - **Java**: tambahkan method `@Test` ke `*FlowE2EIT.java` yang sudah ada. Jika scope berbeda, buat file `[NamaBaru]FlowE2EIT.java` baru.
5. Tambahkan baris baru ke UAT-[project].md di section yang sesuai (atau buat section baru jika perlu)
6. Tampilkan ringkasan: "Ditambahkan: 1 test case di flow_test.go + 1 baris di UAT-[project].md"

**Pilihan b) Update infrastruktur:**

1. Baca file infra yang ada:
   - **Go**: baca `e2e/main_test.go` dan `e2e/helpers_test.go`
   - **Java**: baca `E2EContainers.java` dan `AbstractE2EIT.java` dan `application-e2e.properties`
2. Tampilkan infra yang terdeteksi saat ini:
   ```
   Infra saat ini:
   - Container : PostgreSQL 16, RabbitMQ 3
   - Mock      : webhookMock (port auto)
   - Config    : writeConfig() di helpers_test.go
   ```
3. Tanya user: "Apa yang ingin diubah? (contoh: tambah Redis container, tambah mock baru, ganti versi image)"
4. Tunggu jawaban, lalu update hanya bagian yang diminta:
   - Tambah container → tambah ke blok `TestMain` (Go) atau `E2EContainers static {}` (Java)
   - Tambah mock server → tambah variable dan inisialisasi di file infra yang sesuai, tambah juga ke `@DynamicPropertySource` / `writeConfig()`
   - Ganti image → update string Docker image saja
5. Jangan ubah test case di `flow_test.go` atau `*FlowE2EIT.java` kecuali memang terdampak langsung

**Pilihan c) Regenerate semua:**

Lanjutkan ke Langkah 4A atau 4B sesuai bahasa project — generate ulang semua file dengan overwrite.

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
