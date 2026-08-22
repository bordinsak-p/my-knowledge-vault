---
tags:
  - java
  - quarkus
  - rest-client
  - http
type: reference
created: 2026-08-18
---

# 🔌 Quarkus REST Client

> เรียก HTTP API ตัวอื่นโดย**ประกาศเป็น interface** แทนการเขียนโค้ดยิง request เอง
> Quarkus สร้าง implementation ให้ตอน build — ไม่มี reflection ตอน runtime เลยทำงานได้ดีบน native image

---

## 1. ⚠️ เรื่องชื่อ extension — จุดที่คนพลาดมากที่สุด

ชื่อ artifact **เปลี่ยนความหมาย** ที่ Quarkus 3.9

| เวอร์ชัน | `quarkus-rest-client` หมายถึง | ตัวที่ควรใช้ |
|---|---|---|
| **ก่อน 3.9** | client ตัวเก่าที่อิง RESTEasy Classic | `quarkus-rest-client-reactive` |
| **3.9 ขึ้นไป** | ตัว reactive ที่ถูกเปลี่ยนชื่อมา | `quarkus-rest-client` |

**แปลว่าถ้าอ่าน tutorial ใหม่แล้วก๊อป `quarkus-rest-client` ไปใส่ในโปรเจกต์เวอร์ชันเก่า จะได้ client คนละตัวกันโดยไม่รู้ตัว** — ไม่ error แต่พฤติกรรมต่างกัน

ถ้าต้องการ JSON ให้เติม `-jackson` ต่อท้าย (`quarkus-rest-client-reactive-jackson` / `quarkus-rest-client-jackson`)

ดูเรื่องการเปลี่ยนชื่อชุดใหญ่ได้ที่ [[Quarkus Build]]

---

## 2. เริ่มใช้

```java
@Path("/v1/products")
@RegisterRestClient(configKey = "product-api")
public interface ProductClient {

    @GET
    List<Product> list(@QueryParam("category") String category);

    @GET
    @Path("/{id}")
    Product byId(@PathParam("id") String id);

    @POST
    Product create(Product body);
}
```

```properties
quarkus.rest-client.product-api.url=https://api.example.com
quarkus.rest-client.product-api.connect-timeout=5000
quarkus.rest-client.product-api.read-timeout=15000
```

```java
@Inject @RestClient ProductClient client;    // ← ต้องมี @RestClient ไม่ใช่ @Inject เปล่า ๆ
```

### ใช้ `configKey` เสมอ

ถ้าไม่ใส่ `configKey` ต้องอ้างด้วยชื่อคลาสเต็ม:

```properties
quarkus.rest-client."org.acme.client.ProductClient".url=...
```

ยาว เปราะ และ**พังเงียบ ๆ ตอน refactor ย้าย package** — `configKey` แก้ปัญหานี้ทั้งหมด

> **ไม่ตั้ง `url` = แอปไม่ start** ถือว่าดี เพราะพังตั้งแต่ตอน deploy ดีกว่าไปพังตอนมีคนเรียกใช้

---

## 3. Header

```java
// ค่าคงที่ หรือเรียก method คำนวณ
@ClientHeaderParam(name = "X-Api-Key", value = "${api.key}")
@ClientHeaderParam(name = "X-Request-Id", value = "{genRequestId}")
public interface ProductClient {
    default String genRequestId() { return UUID.randomUUID().toString(); }
}
```

ถ้า logic ซับซ้อนหรือต้องใส่ให้ทุก client ใช้ `ClientHeadersFactory` แล้วผูกด้วย `@RegisterClientHeaders`

พารามิเตอร์รายครั้งใช้ `@HeaderParam` ตามปกติ

---

## 4. จัดการ error

**default: ทุก status 4xx/5xx โยน `WebApplicationException`** ซึ่งไม่ได้บอกอะไรมาก

```java
@ClientExceptionMapper
static RuntimeException toException(Response r) {
    if (r.getStatus() == 404) return new ProductNotFoundException();
    if (r.getStatus() == 429) return new RateLimitedException();
    return null;   // คืน null = ปล่อยให้ mapper ตัวอื่นจัดการต่อ
}
```

แบบเต็มใช้ `ResponseExceptionMapper` ซึ่งมี priority ต่อกันเป็นลูกโซ่ได้ ปิด mapper เริ่มต้นด้วย `disable-default-mapper=true`

**ถ้าอยากได้ status/header โดยไม่ให้โยน exception** ให้ return เป็น `RestResponse<T>`

```java
@GET RestResponse<Product> byId(@PathParam("id") String id);
```

---

## 5. Timeout / Retry / Circuit breaker

**ตั้ง timeout เสมอ** — ค่า default ของ connect/read timeout ยาวกว่าที่คนคาดไว้มาก ปลายทางค้างทีเดียว thread pool เราตันตาม

```properties
quarkus.rest-client.product-api.connect-timeout=5000
quarkus.rest-client.product-api.read-timeout=15000
```

เพิ่ม `quarkus-smallrye-fault-tolerance` แล้วใช้ annotation ได้

```java
@Retry(maxRetries = 3, delay = 200, jitter = 100)
@Timeout(2000)
@CircuitBreaker(requestVolumeThreshold = 10, failureRatio = 0.5, delay = 5000)
@Fallback(fallbackMethod = "cached")
Product byId(String id);
```

### 🚨 กฎเหล็กของ retry

**retry ได้เฉพาะงานที่ทำซ้ำแล้วผลเหมือนเดิม** — `GET`, `PUT`, `DELETE` โดยทั่วไปปลอดภัย
**`POST` ที่สร้างของ / ตัดเงิน / ส่งเอกสาร ห้าม retry ลอย ๆ** เพราะ timeout ไม่ได้แปลว่าปลายทางไม่ได้ทำ — อาจทำไปแล้วแต่ตอบกลับไม่ทัน retry แล้วได้ของซ้ำ

ถ้าจำเป็นต้อง retry งานเขียนจริง ๆ ต้องมี **idempotency key** กำกับ

`jitter` สำคัญกว่าที่คิด — ถ้าทุก instance retry พร้อมกันเป๊ะ ๆ จะกระทืบปลายทางเป็นระลอก (thundering herd)

---

## 6. Async / Reactive

```java
Uni<Product> byId(String id);              // Mutiny
CompletionStage<Product> byId(String id);  // มาตรฐาน Java

@GET @Produces(MediaType.SERVER_SENT_EVENTS)
Multi<Event> stream();                     // SSE
```

แบบ reactive ทำงานบน **event loop** ไม่กิน thread ระหว่างรอ

> **ห้ามเรียกโค้ด blocking ใน callback ของ reactive client** — จะบล็อก event loop ทั้งเส้น กระทบทุก request ที่ใช้ thread เดียวกัน ถ้าต้อง blocking ให้ใช้แบบ synchronous ธรรมดา หรือย้ายไป worker pool

---

## 7. Multipart / ไฟล์

```java
@POST
@Consumes(MediaType.MULTIPART_FORM_DATA)
String upload(@RestForm("file") File file,
              @RestForm("desc") String desc);
```

`File`, `Path`, `byte[]` จะถูกส่งเป็น `application/octet-stream` โดยอัตโนมัติ
ปลายทางบางเจ้าจู้จี้เรื่องรูปแบบ encoder ปรับได้ที่ `quarkus.rest-client.multipart-post-encoder-mode` (RFC1738 / HTML5 / RFC3986)

---

## 8. Logging — และการกันข้อมูลรั่ว

```properties
quarkus.rest-client.logging.scope=request-response
quarkus.rest-client.logging.body-limit=1000
quarkus.rest-client.logging.masked-headers=Authorization,Cookie,X-Api-Key
```

**`masked-headers` ต้องตั้งทุกครั้งที่เปิด logging** ไม่งั้น bearer token กับ API key จะไหลลง log ตรง ๆ
และ **อย่าเปิด log body บน prod** — ข้อมูลส่วนบุคคลจะไปกองอยู่ในระบบ log

---

## 9. สร้าง client แบบ programmatic

เมื่อ URL รู้ตอน runtime เท่านั้น (เช่นเก็บอยู่ใน DB)

```java
ProductClient c = QuarkusRestClientBuilder.newBuilder()
        .baseUri(URI.create(dynamicUrl))
        .connectTimeout(5, TimeUnit.SECONDS)
        .build(ProductClient.class);
```

หรือเปลี่ยน base URL รายครั้งด้วย `@Url`

```java
Product byId(@Url String baseUri, @PathParam("id") String id);
```

> อย่าสร้าง client ใหม่ทุก request — มันสร้าง connection pool ใหม่ทุกครั้ง ควร cache ไว้ต่อ URL

---

## 10. ส่งต่อ token ไปปลายทาง

ถ้าปลายทางเป็น service ในระบบเดียวกันที่ต้องการ bearer token ของผู้ใช้เดิม ใช้ `quarkus-rest-client-oidc-token-propagation`
มันแนบ token จาก request ปัจจุบันให้อัตโนมัติ ไม่ต้องเขียน filter เอง และตัว reactive จะไม่บล็อก IO thread ตอนไปขอ/ต่ออายุ token

ถ้าเป็นการเรียกแบบ service-to-service ที่ไม่มีผู้ใช้ ใช้ `quarkus-oidc-client` ขอ token ด้วย client credentials แทน

---

## 11. กับดัก

- **ลืม `@RestClient` ตอน inject** — จะได้ CDI bean ธรรมดาที่ไม่ใช่ client แล้วงงว่าทำไมไม่ทำงาน
- **ไม่ตั้ง timeout** — ปลายทางค้าง แล้วลากเราค้างตาม อาการคือ thread pool เต็มโดยหาสาเหตุไม่เจอ
- **retry บน `POST`** — ดูข้อ 5
- **จับ `WebApplicationException` กว้างเกินไป** — กลืน 404 กับ 500 รวมกันทั้งที่ควรจัดการต่างกัน
- **สร้าง client ใหม่ทุกครั้งที่เรียก** — connection pool ใหม่ทุกครั้ง
- **`interface` ต้อง `public`** ไม่งั้น Quarkus generate implementation ไม่ได้
- **เปิด logging แล้วลืม mask header** — token รั่วลง log

---

## 12. Cheat sheet

```properties
# ต่อ client (ใช้ configKey)
quarkus.rest-client.my-api.url=https://api.example.com
quarkus.rest-client.my-api.connect-timeout=5000
quarkus.rest-client.my-api.read-timeout=15000
quarkus.rest-client.my-api.scope=jakarta.inject.Singleton

# global
quarkus.rest-client.logging.scope=request-response
quarkus.rest-client.logging.masked-headers=Authorization,Cookie
```

| อาการ | สาเหตุ |
|---|---|
| แอป start ไม่ขึ้น บ่นหา url | ลืมตั้ง `quarkus.rest-client.<key>.url` |
| inject แล้วเป็น null / ไม่ใช่ client | ลืม `@RestClient` |
| config ไม่มีผลหลัง refactor | ใช้ชื่อคลาสเต็มเป็น key แล้วย้าย package |
| thread pool เต็มโดยไม่มีสาเหตุ | ไม่ได้ตั้ง read-timeout |
| ข้อมูลซ้ำที่ปลายทาง | retry บน operation ที่ไม่ idempotent |
| แอปช้าลงทั้งระบบตอนเรียก API | เรียก blocking code ใน callback ของ reactive client |
| token โผล่ใน log | เปิด logging โดยไม่ตั้ง `masked-headers` |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus]] — หน้ารวม
- [[Quarkus Build]] — เรื่องการเปลี่ยนชื่อ extension ที่ 3.9
- [[Quarkus Redis]] — idempotency key สำหรับกันงานเขียนซ้ำตอน retry

## 📖 อ่านต่อ

- [Quarkus — Using the REST Client](https://quarkus.io/guides/rest-client)
- [Quarkus — REST Client Reference](https://quarkus.io/guides/rest-client-reference)
- [Quarkus — OIDC client and token propagation](https://quarkus.io/guides/security-openid-connect-client-reference)
- [Quarkus REST Client: Timeouts, Retries, and Redaction](https://www.the-main-thread.com/p/quarkus-rest-client-timeouts-retries-redaction)
