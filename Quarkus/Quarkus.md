---
tags:
  - quarkus
  - java
  - quarkus/index
type: moc
created: 2026-08-18
---

# ☕ Quarkus

หน้ารวมโน้ต Quarkus — เน้นเรื่องที่กัดจริงตอนใช้งาน ไม่ใช่ tutorial สอนตั้งแต่ศูนย์

> **เวอร์ชันสำคัญมากในเรื่องนี้** Quarkus ออกเวอร์ชันใหม่ถี่และเปลี่ยนของใหญ่หลายรอบใน 3.x
> ทั้งชื่อ extension, ค่า default และ API — เวลาอ่านตัวอย่างจากเว็บ ให้ดูก่อนเสมอว่าเขียนด้วยเวอร์ชันไหน

---

## โน้ตในนี้

| โน้ต                        | ว่าด้วย                                                                                                                         | สถานะ      |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| [[Quarkus Project Structure]] | โครงสร้างไฟล์ Maven มาตรฐาน, package-by-layer vs package-by-feature, naming convention, resource→service→repository→entity | ✅ |
| [[Quarkus Config]]          | ลำดับความสำคัญของ config source, profile (`%dev`/`%prod`/`%test`/custom), `@ConfigMapping`, build-time vs runtime config         | ✅          |
| [[Quarkus Thread Pool]]     | `@Scheduled`, `ManagedExecutor`, ทำไม request context ใช้ใน worker thread ไม่ได้, thread pool config, virtual threads             | ✅          |
| [[Quarkus REST Layer]]      | RESTEasy Classic vs Quarkus REST, smart dispatch, `@Blocking`/`@NonBlocking`, ทำไม block event loop ถึงร้ายแรง, `Uni`/`Multi`     | ✅          |
| [[Quarkus Hibernate]]       | Hibernate 5→6, type mapping, transaction/persistence context, N+1, bulk insert, multitenancy, native                            | ✅          |
| [[Quarkus Flyway]]          | schema history, ชื่อไฟล์ V/R, baseline, callback, multi-datasource, zero-downtime (expand/contract)                             | ✅          |
| [[Quarkus EntityManager]]   | สไตล์การเขียนแบบไม่ผ่าน Panache — lifecycle, persist/merge, query 4 แบบ, native query + การดัก null, pagination trap            | ✅          |
| [[Quarkus Panache]]         | Active record vs repository, query syntax, ข้อจำกัด, ทิศทาง Jakarta Data                                                        | ✅          |
| [[Quarkus Build]]           | **ขั้นตอนการ build แบบเต็ม** (source→container→deploy), JVM vs native, fast-jar, build-time config                              | ✅          |
| [[Quarkus Redis]]           | `quarkus-cache` vs `quarkus-redis-client`, data structure, cache-aside, distributed lock, rate limit, idempotency, Dev Services | ✅          |
| [[Quarkus REST Client]]     | `@RegisterRestClient`, configKey, header, exception mapper, timeout/retry/circuit breaker, multipart, token propagation         | ✅          |
| [[Quarkus HTTP Methods]]    | GET/POST/PUT/PATCH/DELETE/HEAD/OPTIONS ครบ, safe/idempotent/cacheable, annotation ใน Quarkus, status code คู่กัน                | ✅          |
| [[Quarkus MIME Types]]      | media type มีอะไรบ้าง, `@Produces`/`@Consumes`, content negotiation, multipart, streaming, 415 vs 406                           | ✅          |
| [[Quarkus WebSocket]]       | classic vs websockets-next, ตัวอย่างห้องแชทเต็ม, broadcast, path param, กับดัก, เริ่มลองยังไง                                   | ✅          |
| [[Quarkus Keycloak]]        | Keycloak + Angular + Quarkus OIDC ครบวงจร, CORS/preflight, service-to-service, `@TestSecurity`, เริ่มลองยังไง                   | ✅          |
| [[Quarkus Security]]        | `SecurityIdentityAugmentor`, `@PermissionsAllowed`/`@PermissionChecker`, multi-tenant OIDC (static/dynamic)                     | ✅          |
| [[Quarkus Testing]]         | `@QuarkusTest`, REST Assured, `@TestTransaction`, `@InjectMock`, Dev Services, เอาเทสที่ comment ไว้กลับมารัน                   | ✅          |
| [[Quarkus Health Check]]    | liveness vs readiness vs startup, เขียน check เอง, ผูกกับ k8s probe, management port                                            | ✅          |
| [[Quarkus Logging]]         | `level` vs `min-level`, category, ตั้งค่าผ่าน env var บน server, handler, MDC, JSON log                                         | ✅          |
| [[Quarkus Panache Example]] | โค้ดตัวอย่างครบชุด — entity, repository, service, DTO projection, paging, bulk, test                                            | 📝 example |

---

## แผนที่ความรู้ — ยังไม่ได้เขียน

เรียงตามที่น่าจะได้ใช้จริงก่อน

- **Dev mode** — live reload ทำงานยังไง และเมื่อไหร่ที่มันหลอกเรา
- **Observability** — metrics, tracing (logging เขียนแล้วที่ [[Quarkus Logging]])

---

## กฎรวมที่จำไว้ได้เลย

1. **Quarkus ตัดสินใจตอน build ไม่ใช่ตอน start** — งานที่ framework อื่นทำตอน start (scan classpath, อ่าน annotation, สร้าง proxy) Quarkus ทำตอน compile แล้วอบติดไปกับ artifact ผลคือ config บางตัวเปลี่ยนตอน runtime ไม่ได้ และของที่พึ่ง dynamic lookup มักพังตอน native
2. **เวอร์ชันเปลี่ยนชื่อของบ่อย** — เช่น `quarkus-resteasy-reactive` → `quarkus-rest` ตอน 3.9 ตัวอย่างบนเว็บมักใหม่กว่าที่เราใช้ ก๊อปมาตรง ๆ แล้ว build ไม่ผ่าน
3. **ของที่ทำงานใน request context จะไม่ทำงานใน background thread** — `@RequestScoped` bean, security context, สิ่งที่ resolve ตาม request
4. **property ที่ไม่มี prefix มีผลทุก profile รวม prod** — ของที่ตั้งใจให้เปิดเฉพาะ dev ต้องใส่ `%dev.` เสมอ

---

## 🔗 ที่อื่นใน vault

- [[Java]] — ตัวภาษาและ core library
- [[Greenhouse]] — โปรเจกต์ที่อยากทำ

## 📖 อ้างอิงหลัก

- [Quarkus Guides](https://quarkus.io/guides/)
- [Quarkus Migration Guides (wiki)](https://github.com/quarkusio/quarkus/wiki) — ดูทุกครั้งก่อนอัปเวอร์ชัน
