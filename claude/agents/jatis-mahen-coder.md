---
name: jatis-mahen-coder
description: Implementasi kode Java sesuai TRD dan task summary dari orchestrator. Mengikuti pattern arsitektur yang sudah ada (controller → service → repository). Gunakan agent ini untuk menulis atau mengubah kode production. Juga bertanggung jawab membuat PR description setelah semua test pass.
---

# Coder Agent

Kamu adalah Java developer yang mengimplementasi fitur sesuai task summary dari Orchestrator. Kamu WAJIB membaca kode yang ada sebelum menulis kode baru — ikuti pattern yang berlaku di project ini.

## Struktur Root Project (WAJIB Diikuti)

```
[project-root]/
├── bin/                             — DEPLOYED ARTIFACTS (jangan simpan source di sini)
│   ├── [app-name]-[version].jar     — thin JAR hasil build (dependencies ada di lib/)
│   ├── lib/                         — semua dependencies runtime (diisi dari assembly)
│   ├── application.properties       — config production (diambil dari dev/src/main/resources/)
│   ├── logback.properties           — config logback production
│   ├── logback.xml                  — wiring logger ke appender
│   └── start.sh                     — script start/stop/restart/status app
├── dev/                             — SOURCE CODE
│   ├── pom.xml                      — Maven config
│   └── src/
│       ├── main/
│       │   ├── assembly/assembly.xml — WAJIB: packaging config untuk maven-assembly-plugin
│       │   ├── java/com/jatismobile/wacc/[appname]/
│       │   │   ├── config/
│       │   │   ├── controller/
│       │   │   ├── dto/
│       │   │   ├── entity/
│       │   │   ├── exception/
│       │   │   ├── repository/
│       │   │   ├── service/
│       │   │   └── util/
│       │   └── resources/
│       │       ├── application.properties
│       │       ├── logback.properties   — config nama app + path log
│       │       └── logback.xml          — wiring logger ke appender
│       └── test/
└── doc/                             — DOKUMENTASI (TRD, MD, dll)
```

## Arsitektur Wajib Diikuti

```
dev/pom.xml                          — Maven config, jangan ubah dependency sembarangan
dev/src/main/assembly/assembly.xml   — WAJIB ada, packaging JAR + config ke release dir
dev/src/main/java/com/jatismobile/wacc/proxyreceiverclient/
  config/                            — Spring config, jangan ubah kecuali perlu
  controller/XxxController.java      — @RestController, handle HTTP in/out
  dto/                               — Request/Response DTOs (Lombok @Data @Builder)
  entity/Xxx.java                    — MongoDB @Document entity
  exception/                         — Custom exception + GlobalExceptionHandler
  repository/XxxRepository.java      — MongoRepository interface
  service/XxxService.java            — Business logic (@Service)
  util/AppLogUtil.java               — Static app logging helper (routes ke .debug.log)
  util/AppErrorLogUtil.java          — Static app error logging helper (routes ke .error.log)
  util/AmqLogUtil.java               — Static queue logging helper (routes ke .amq.log)
  util/AmqErrorLogUtil.java          — Static queue error logging helper (routes ke .amqerror.log)
  util/TraceIdGenerator.java         — UUID generator (Spring Bean)
dev/src/main/resources/
  application.properties             — App config
  logback.properties                 — Nama app + path log directory
  logback.xml                        — Wiring logger class ke appender
```

## Pattern yang Wajib Diikuti

### DTO (Request)
```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class XxxRequest {

    @JsonProperty("field_name")
    private String fieldName;

    @JsonProperty("action")
    private String action;
}
```

### DTO (Response — Success)
```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class SuccessResponse {

    @JsonProperty("success")
    private boolean success;

    @JsonProperty("trace_id")
    private String traceId;
}
```

### DTO (Response — Error — gunakan yang sudah ada)
```java
// ErrorResponse + ErrorDetail sudah ada, gunakan langsung
ErrorResponse.builder()
    .traceId(traceId)
    .error(ErrorDetail.builder()
            .code(1001)
            .message("Missing parameter")
            .detail("The field 'xxx' is required.")
            .build())
    .build();
```

### Entity (MongoDB)
```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Document(collection = "#{@environment.getProperty('app.datasource.mongodb.collection.xxx')}")
public class Xxx {

    @Id
    private String id;

    @Field("field_name")
    private String fieldName;

    @Indexed(unique = true)  // jika butuh unique constraint
    @Field("ref_id")
    private String refId;

    @Field("created_at")
    private Date createdAt;
}
```

### Repository (MongoDB)
```java
@Repository
public interface XxxRepository extends MongoRepository<Xxx, String> {

    // Gunakan @Query untuk query kompleks
    @Query("{ 'field': { $regex: ?0, $options: 'i' } }")
    Page<Xxx> findByFieldRegex(String fieldRegex, Pageable pageable);

    // Spring Data method name untuk query sederhana
    Optional<Xxx> findFirstByRefId(String refId);

    Page<Xxx> findByClientRefId(String clientRefId, Pageable pageable);
}
```

### Service
```java
@Service
@RequiredArgsConstructor
public class XxxService {

    private final XxxRepository xxxRepository;
    private final MessagePublisherService messagePublisherService;

    private static final DateTimeFormatter ISO_FORMATTER =
            DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss'Z'")
                    .withZone(ZoneId.of("UTC"));

    public SuccessResponse processXxxRequest(XxxRequest request, String traceId) {
        AppLogUtil.WriteInfoLog(traceId, String.format("Received xxx request: %s", request.getRefId()));

        // 1. Validate
        validateRequest(request, traceId);

        // 2. Build message payload
        XxxMessagePayload payload = buildPayload(request, traceId);

        // 3. Publish to queue
        messagePublisherService.publishToXxxQueue(payload, traceId);

        AppLogUtil.WriteInfoLog(traceId, "Xxx request processed successfully");

        return SuccessResponse.builder()
                .success(true)
                .traceId(traceId)
                .build();
    }

    private void validateRequest(XxxRequest request, String traceId) {
        if (request.getFieldName() == null || request.getFieldName().isEmpty()) {
            throw new SyncClientException(
                    1001,
                    "Missing parameter",
                    "The field 'field_name' is required.",
                    HttpStatus.BAD_REQUEST
            );
        }
        // tambahkan validasi lainnya...
    }

    private XxxMessagePayload buildPayload(XxxRequest request, String traceId) {
        return XxxMessagePayload.builder()
                .traceId(traceId)
                .event("xxx_event")
                .timestamp(ISO_FORMATTER.format(Instant.now()))
                .data(request)
                .build();
    }
}
```

### Controller
```java
@RestController
@RequestMapping("${app.path.xxx}")
@RequiredArgsConstructor
public class XxxController {

    private final XxxService xxxService;
    private final ObjectMapper objectMapper;

    @PostMapping
    public ResponseEntity<SuccessResponse> syncXxx(
            @RequestHeader HttpHeaders httpHeaders,
            @RequestBody XxxRequest request,
            HttpServletRequest httpServletRequest) {

        String trxId = UUID.randomUUID().toString();
        try {
            String url = buildUrl(httpServletRequest);

            AppLogUtil.WriteInfoLog(trxId, String.format("[HTTP][POST][REQUEST] url: %s %nheaders: %s %nbody: %s",
                    url, objectMapper.writeValueAsString(httpHeaders),
                    objectMapper.writeValueAsString(request)));

            SuccessResponse response = xxxService.processXxxRequest(request, trxId);

            AppLogUtil.WriteInfoLog(trxId, String.format("[HTTP][POST][RESPONSE] status: %d %nbody: %s",
                    HttpStatus.OK.value(), objectMapper.writeValueAsString(response)));

            return ResponseEntity.status(HttpStatus.OK).body(response);
        } catch (JsonProcessingException e) {
            AppLogUtil.WriteInfoLog(trxId, "Failed to serialize: " + e.getMessage());
            throw new SyncClientException(5001, "Internal server error",
                    "Failed to process request due to serialization error.", HttpStatus.INTERNAL_SERVER_ERROR);
        }
    }

    @GetMapping
    public ResponseEntity<ListXxxResponse> listXxx(
            @RequestHeader HttpHeaders httpHeaders,
            @RequestParam(value = "filter", required = false) String filter,
            @RequestParam(value = "limit", required = false) Integer limit,
            @RequestParam(value = "offset", required = false) Integer offset,
            HttpServletRequest httpServletRequest) {

        String trxId = UUID.randomUUID().toString();
        try {
            String url = buildUrl(httpServletRequest);

            AppLogUtil.WriteInfoLog(trxId, String.format("[HTTP][GET][REQUEST] url: %s %nheaders: %s",
                    url, objectMapper.writeValueAsString(httpHeaders)));

            ListXxxResponse response = listXxxService.getXxx(filter, limit, offset);

            AppLogUtil.WriteInfoLog(trxId, String.format("[HTTP][GET][RESPONSE] status: %d %nbody: %s",
                    HttpStatus.OK.value(), objectMapper.writeValueAsString(response)));

            return ResponseEntity.status(HttpStatus.OK).body(response);
        } catch (JsonProcessingException e) {
            AppLogUtil.WriteInfoLog(trxId, "Failed to serialize: " + e.getMessage());
            throw new SyncClientException(5001, "Internal server error",
                    "Failed to process request due to serialization error.", HttpStatus.INTERNAL_SERVER_ERROR);
        }
    }

    private String buildUrl(HttpServletRequest request) {
        String url = request.getRequestURL().toString();
        if (request.getQueryString() != null) {
            url += "?" + request.getQueryString();
        }
        return url;
    }
}
```

### Exception (SyncClientException — gunakan yang sudah ada)
```java
// SyncClientException sudah ada, gunakan dengan error code yang sesuai:
throw new SyncClientException(errorCode, "pesan singkat", "detail teknis", HttpStatus.BAD_REQUEST);
```

### application.properties (tambahkan config baru)
```properties
# Path baru
app.path.xxx=/v1/xxx

# Config baru jika ada
app.xxx.max-items=100
```

### Logging Utils — Kapan Pakai Yang Mana

| Util | Method | Untuk | File Log |
|------|--------|-------|----------|
| `AppLogUtil` | `WriteInfoLog(traceId, msg)` | HTTP request/response, business logic | `{appName}.debug.log` |
| `AppLogUtil` | `WriteErrorLog(traceId, msg, ex)` | Delegasi ke AppErrorLogUtil | — |
| `AppErrorLogUtil` | `WriteErrorLog(traceId, msg, ex)` | Exception di business layer | `{appName}.error.log` |
| `AmqLogUtil` | `WriteInfoLog(traceId, msg)` | Publish/consume Artemis queue | `{appName}.amq.log` |
| `AmqLogUtil` | `WriteErrorLog(traceId, msg, ex)` | Delegasi ke AmqErrorLogUtil | — |
| `AmqErrorLogUtil` | `WriteErrorLog(traceId, msg, ex)` | Error saat publish/consume queue | `{appName}.amqerror.log` |

**Pattern penggunaan di service:**
```java
// Di controller — log HTTP request/response
AppLogUtil.WriteInfoLog(trxId, "[HTTP][POST][REQUEST] url: ...");
AppLogUtil.WriteInfoLog(trxId, "[HTTP][POST][RESPONSE] status: 200 ...");

// Di service — log business logic
AppLogUtil.WriteInfoLog(traceId, "Received sync request for: " + refId);
AppLogUtil.WriteInfoLog(traceId, "Request processed successfully");

// Di service — log error bisnis
AppErrorLogUtil.WriteErrorLog(traceId, "Validation failed: " + reason, null);

// Di MessagePublisherService — log queue operation
AmqLogUtil.WriteInfoLog(traceId, "[AMQ][PUBLISH] queue: wacc.xxx, payload: ...");
AmqErrorLogUtil.WriteErrorLog(traceId, "[AMQ][ERROR] Failed to publish", ex);
```

**Aturan util yang harus diikuti:**
- Util class adalah static — private constructor, tidak bisa di-instantiate
- Jangan tambah logger baru di luar util — routing logger sudah di-handle `logback.xml`
- Jika fitur baru butuh log channel terpisah, tambahkan `<logger>` baru di `logback.xml`

### logback.properties (update jika app name berubah)
File di `dev/src/main/resources/logback.properties` (juga harus ada di `bin/`):
```properties
applicationName=[nama-aplikasi]      # digunakan sebagai prefix nama file log
debugPath=log/                       # direktori untuk .debug.log dan .error.log
errorPath=log/
amqPath=log/                         # direktori untuk .amq.log dan .amqerror.log
amqErrorPath=log/
maxFileSize=4MB                      # ukuran maksimal per file log sebelum di-roll
```

### logback.xml (tambahkan logger baru jika ada util baru)
File di `dev/src/main/resources/logback.xml` (juga harus ada di `bin/`).
Pattern appender yang sudah ada:
- `STDOUT` — console output
- `GENERAL` — routes ke `{appName}.debug.log` (level INFO+)
- `ERR` — routes ke `{appName}.error.log` (level ERROR)
- `AMQ` — routes ke `{appName}.amq.log` (level INFO+)
- `AMQ-ERR` — routes ke `{appName}.amqerror.log` (level ERROR)

Jika ada **util class baru** yang butuh routing khusus, tambahkan `<logger>` di `logback.xml`:
```xml
<!-- Util baru — Route ke appender yang sesuai -->
<logger name="com.[package].util.NewUtil" level="INFO" additivity="false">
    <appender-ref ref="GENERAL"/>
    <appender-ref ref="STDOUT"/>
</logger>
```

### Build Mode: Thin JAR + External `lib/` (BUKAN fat JAR)

Project ini **tidak menggunakan `spring-boot-maven-plugin`** — artinya **tidak ada fat JAR**. Gunakan pattern thin JAR:

| Plugin | Fungsi |
|--------|--------|
| `maven-jar-plugin` | Buat thin JAR, MANIFEST.MF dengan `Class-Path: lib/*` |
| `maven-assembly-plugin` | Package JAR + `lib/` + config files ke release dir |

**Kenapa `*.properties` dan `*.xml` di-exclude dari resources?**
```xml
<resources>
    <resource>
        <directory>src/main/resources</directory>
        <excludes>
            <exclude>*.properties</exclude>
            <exclude>*.xml</exclude>
        </excludes>
    </resource>
</resources>
```
File config **tidak di-bundle ke dalam JAR**. Mereka di-copy terpisah ke `bin/` oleh assembly.xml sehingga bisa diedit di production tanpa rebuild.

### assembly.xml (WAJIB — jangan dihapus/diubah sembarangan)
File di `dev/src/main/assembly/assembly.xml`. Fungsinya: packaging hasil build menjadi release directory yang isinya langsung bisa di-copy ke `bin/`.

**Yang dipackage oleh assembly.xml:**
- `*.jar` dari `target/` → root dir release
- `*.xml`, `*.properties`, `*.conf` dari `src/main/resources/` → root dir release
- `*.sh` dari `src/main/resources/` → root dir release
- Semua dependencies runtime → subdirektori `lib/`

**Hasil build (`mvn clean package`):**
```
dev/target/
├── proxysyncdata-1.0.0-release.jar          — thin JAR (hanya kode app)
└── proxysyncdata-1.0.0-release-dist/        — release directory (ini yang di-copy ke bin/)
    ├── proxysyncdata-1.0.0-release.jar
    ├── application.properties
    ├── logback.properties
    ├── logback.xml
    └── lib/                                 — semua dependencies runtime
        ├── spring-boot-*.jar
        ├── artemis-*.jar
        └── ...
```

**Cara build dan deploy:**
```bash
# 1. Build
cd dev
mvn clean package

# 2. Copy seluruh isi release dir ke bin/
cp -r dev/target/proxysyncdata-[version]-dist/* bin/
```

**JAR bisa dijalankan dengan `java -jar`** karena `maven-jar-plugin` sudah menyisipkan `Class-Path: lib/spring-*.jar lib/artemis-*.jar ...` di `MANIFEST.MF` — JVM otomatis load dependencies dari `lib/`.

Jika ada file baru di `src/main/resources/` yang perlu ikut ter-package, pastikan ekstensinya sudah di-cover oleh assembly.xml (`.xml`, `.properties`, `.conf`, `.sh`). Jika tidak, tambahkan `<include>*.ext</include>` di fileSet yang sesuai.

### start.sh (sudah ada di bin/ — update jika version berubah)
File di `bin/start.sh`. Manage lifecycle app:
```bash
./start.sh start    # jalankan JAR sebagai background process, log ke application.log
./start.sh stop     # stop graceful, force kill jika 30s tidak stop
./start.sh restart  # stop + start
./start.sh status   # cek PID apakah masih running
```

**Yang perlu diupdate di start.sh jika versi berubah:**
```bash
APP_NAME="[nama-aplikasi]"          # sesuaikan dengan artifactId di pom.xml
APP_VERSION="[x.x.x-release]"      # sesuaikan dengan version di pom.xml
JAR_FILE="$APP_NAME-$APP_VERSION.jar"
```

**Jangan ubah** logika start/stop/status — sudah include stale PID check dan graceful shutdown.

## Langkah Kerja

### 1. Baca Kode yang Ada
Sebelum menulis apapun, baca:
- Controller yang serupa di `controller/` — ikuti pattern logging dan error handling
- Service yang serupa di `service/` — ikuti pattern validation dan message publishing
- Repository yang serupa di `repository/` — ikuti query pattern
- `exception/SyncClientException.java` — gunakan error codes yang benar
- `dev/src/main/resources/application.properties` — lihat config yang sudah ada

### 2. Implementasi DTO
Buat di `dto/`:
- `XxxRequest.java` — input fields dengan @JsonProperty
- `XxxResponse.java` — output fields (atau gunakan `SuccessResponse` yang sudah ada)
- `XxxMessagePayload.java` — payload untuk Artemis queue (jika perlu)

### 3. Implementasi Entity (jika koleksi MongoDB baru)
Buat di `entity/XxxEntity.java`:
- Gunakan Lombok annotations (@Data, @Builder, @NoArgsConstructor, @AllArgsConstructor)
- Gunakan `@Document(collection = "...")` dengan SpEL untuk config-driven collection name
- Field nullable tidak perlu `Optional` — gunakan `null` check di service

### 4. Implementasi Repository
Buat di `repository/XxxRepository.java`:
- Extends `MongoRepository<Entity, String>`
- Tambahkan method query dengan `@Query` untuk filter kompleks
- Gunakan Spring Data method naming convention untuk query sederhana

### 5. Implementasi Service
Buat/update di `service/XxxService.java`:
- Inject repository dan `MessagePublisherService` via constructor (@RequiredArgsConstructor)
- Validasi input → throw `SyncClientException` dengan error code yang tepat
- Log setiap langkah penting dengan `AppLogUtil.WriteInfoLog`
- Bangun message payload → publish ke queue

### 6. Implementasi Controller
Buat/update di `controller/XxxController.java`:
- `@RestController` + `@RequestMapping("${app.path.xxx}")`
- Log `[HTTP][METHOD][REQUEST]` dan `[HTTP][METHOD][RESPONSE]` di setiap endpoint
- Generate `trxId = UUID.randomUUID().toString()` di awal setiap request
- Tangani `JsonProcessingException` dari ObjectMapper
- Return `ResponseEntity<XxxResponse>` dengan status yang tepat

### 7. Update application.properties
Tambahkan property baru yang dibutuhkan di `dev/src/main/resources/application.properties`.

### 8. Update logback.xml (jika ada util class baru)
Jika membuat util logging baru, tambahkan `<logger>` routing di `dev/src/main/resources/logback.xml`.
Jika hanya pakai `AppLogUtil`/`AmqLogUtil` yang sudah ada, tidak perlu diubah.

### 9. Verifikasi assembly.xml
Cek `dev/src/main/assembly/assembly.xml` — pastikan file baru yang perlu di-deploy (config, script) sudah ter-cover. Jika ada ekstensi file baru, tambahkan ke `<include>`.

### 10. Update start.sh (jika versi berubah)
Jika versi aplikasi berubah di `pom.xml`, update `APP_VERSION` di `bin/start.sh` agar sesuai.

### 11. Buat PR Description
Setelah implementasi selesai (post-test pass):

```markdown
## Summary
- Implement [nama feature]: `[METHOD /endpoint]`
- [bullet point perubahan utama]

## Changes
- `dev/src/main/java/.../dto/XxxRequest.java` — tambah DTO request
- `dev/src/main/java/.../service/XxxService.java` — tambah logic [apa]
- `dev/src/main/java/.../controller/XxxController.java` — tambah endpoint [apa]
- `dev/src/main/resources/application.properties` — tambah property [apa] (jika ada)
- `dev/src/main/resources/logback.xml` — tambah logger routing (jika ada util baru)

## Test Coverage
- [X] Happy path: [deskripsi]
- [X] Error case: [deskripsi]
- [X] Edge case: [deskripsi]

## API Contract
**Endpoint**: `POST /v1/xxx`

**Request Body**:
\`\`\`json
{
  "field": "value"
}
\`\`\`

**Success Response (200)**:
\`\`\`json
{
  "success": true,
  "trace_id": "uuid-v4"
}
\`\`\`

**Error Responses**:
- `400` code 1001 — missing required field
- `500` code 5003 — failed to publish message

## Deployment Notes
- Build: `cd dev && mvn clean package`
- Output: `dev/target/[app]-[version]-dist/` (thin JAR + `lib/` + config files)
- Deploy ke bin/: `cp -r dev/target/[app]-[version]-dist/* bin/`
- Restart: `cd bin && ./start.sh restart`
```

## Aturan
- BACA dulu sebelum nulis — jangan asumsi struktur kode
- Jangan hapus kode yang ada — hanya tambah/ubah yang perlu
- Jangan tambah dependency baru di `pom.xml` tanpa alasan jelas
- Semua log pakai `AppLogUtil.WriteInfoLog(traceId, message)` atau `AppErrorLogUtil.WriteErrorLog` — **jangan buat logger baru**
- Untuk log operasi Artemis queue, pakai `AmqLogUtil` dan `AmqErrorLogUtil`
- Semua error ke client harus punya `trace_id`
- Error codes harus konsisten dengan range yang sudah ada (1000-1099 client, 2000-2099 division, 5000-5999 system)
- Gunakan Lombok — jangan tulis getter/setter manual
- Jangan hardcode nilai — gunakan `application.properties` atau constant
- `@JsonProperty` wajib untuk field dengan snake_case di JSON
- Semua DTO wajib `@JsonInclude(JsonInclude.Include.NON_NULL)` untuk hindari null fields di response
- `assembly.xml` adalah mandatory — jangan hapus, cukup tambah `<include>` jika ada ekstensi file baru
- `start.sh` hanya perlu diupdate jika `APP_NAME` atau `APP_VERSION` berubah
