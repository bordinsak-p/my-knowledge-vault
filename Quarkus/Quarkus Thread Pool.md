---
tags:
  - java
  - quarkus
  - concurrency
  - thread-pool
type: reference
created: 2026-09-16
---

# 🧵 Quarkus Thread Pool & Background Task

> **กฎเดียวที่ต้องเข้าใจก่อนอย่างอื่น: `@RequestScoped` bean กับของที่ผูกกับ HTTP request (เช่น `SecurityIdentity`) มีอายุแค่ "หนึ่ง request" เท่านั้น**
> พอ handler thread ตอบ response เสร็จ request context ถูกปิดทันที — ถ้างาน background พยายามอ่านของพวกนี้ทีหลังใน thread อื่น จะไม่เจออะไรเลย เพราะไม่มี request ให้ผูกอยู่แล้ว

---

## 1. ทำไมต้องมี background task แยกจาก request thread

งานบางอย่างไม่ควรทำให้ผู้ใช้รอ (ส่งอีเมล, generate report, import ไฟล์ใหญ่, sync ข้อมูลกับระบบอื่น) หรือต้องรันเองตามเวลาโดยไม่มีใครมา request (cron job, cleanup ข้อมูลเก่า) — Quarkus มีสองทางหลักให้ใช้:

| ต้องการ | ใช้ |
|---|---|
| รันตามตารางเวลา (ทุกวัน, ทุก 5 นาที) | `@Scheduled` (ข้อ 2) |
| รันทันทีแบบ async ตอบ response ก่อนแล้วค่อยทำต่อเบื้องหลัง | `ManagedExecutor`/`ExecutorService` (ข้อ 3) |

---

## 2. `@Scheduled` — งานตามตารางเวลา

```java
@ApplicationScoped
public class CleanupJob {

    @Scheduled(every = "10s")
    void everyTenSeconds() { ... }

    @Scheduled(cron = "0 15 10 * * ?")      // ทุกวัน 10:15 น.
    void dailyAt10_15() { ... }

    @Scheduled(cron = "{cleanup.cron.expr}") // อ่านจาก config property แทน hardcode
    void configurable() { ... }
}
```

**`every` กับ `cron` เลือกได้ตามความถี่ที่ต้องการ** — `every` เหมาะกับรันถี่ ๆ แบบ interval คงที่ (ทุก N วินาที), `cron` เหมาะกับเวลาที่ตายตัวตามนาฬิกา (ถ้าใส่ทั้งคู่ `cron` ชนะ) — การใส่ cron เป็น `{property.name}` แทนค่าตรง ๆ ทำให้เปลี่ยนตารางเวลาได้จาก config โดยไม่ต้อง build ใหม่ (ดู [[Quarkus Config]])

**ข้อควรรู้:** `@Scheduled` method รันบน `@ApplicationScoped` bean เท่านั้น (ไม่ใช่ `@RequestScoped`) และ**ไม่มี request context ให้ตั้งแต่แรก** — ทุกอย่างที่ต้องใช้ต้องหาทางเข้าถึงเองในตัว (query ตรง ๆ, inject bean อื่น) ไม่ใช่พึ่งพา HTTP request ที่ไม่มีอยู่จริง

---

## 3. รันงาน async เอง — `ManagedExecutor`

```java
@Inject ManagedExecutor executor;

@POST
@Path("/reports/generate")
public Response generate(ReportRequest req) {
    executor.runAsync(() -> reportService.generate(req));
    return Response.accepted().build();      // ตอบ 202 ทันที ไม่รอ report เสร็จ
}
```

**`ManagedExecutor` มาจาก MicroProfile Context Propagation** (extension `quarkus-smallrye-context-propagation`) — ต่างจาก `ExecutorService`/`Thread` ธรรมดาตรงที่มัน**พยายาม propagate บาง context ให้อัตโนมัติ** (เช่น CDI `@ApplicationScoped` bean, MDC สำหรับ log) ให้ใช้งานได้ต่อเนื่องใน thread ใหม่ — แต่ **ไม่ใช่ทุก context ที่ propagate ได้ฟรี** (ข้อ 4-5 คือเหตุผล)

```properties
# ปรับแต่ง thread ของ ManagedExecutor ได้แยกจาก thread pool หลัก
quarkus.thread-pool.core-threads=2
```

---

## 4. ⚠️ ทำไมของที่ผูกกับ request context ใช้ใน worker thread ไม่ได้

```java
@Path("/reports")
public class ReportResource {

    @Inject SecurityIdentity identity;
    @Inject ManagedExecutor executor;

    @POST
    @Path("/generate")
    public Response generate() {
        executor.runAsync(() -> {
            System.out.println(identity.getPrincipal().getName());  // ❌
        });
        return Response.accepted().build();
    }
}
```

```
jakarta.enterprise.context.ContextNotActiveException: RequestScoped context was not active
```

**เหตุผลเชิงโครงสร้าง ไม่ใช่บั๊ก:** `SecurityIdentity`, `@RequestScoped` bean, และของอื่น ๆ ที่ resolve จาก request (`HttpHeaders`, path/query param ที่ inject ตรง ๆ) ถูกสร้างขึ้น**เมื่อ request context เปิด** (ตอน HTTP request เข้ามา) และถูกทำลายทันทีที่ request จบ — `executor.runAsync(...)` สร้าง task ที่รันบน **thread คนละตัว ในจังหวะเวลาที่อาจจะหลัง request จบไปแล้วด้วยซ้ำ** ไม่มี request context ให้ผูกอยู่เลย เข้าถึงไม่ได้ไม่ว่าจะพยายามยังไง

---

## 5. วิธีแก้ที่ถูก — capture ค่าไว้ก่อน ไม่ใช่พึ่ง context ทีหลัง

### ✅ ทางที่แนะนำ: อ่านค่าที่ต้องใช้ตอนยังอยู่ใน request thread แล้วส่งเป็นพารามิเตอร์ธรรมดา

```java
@POST
@Path("/generate")
public Response generate() {
    String username = identity.getPrincipal().getName();   // ✅ อ่านตอนยัง valid อยู่

    executor.runAsync(() -> {
        reportService.generateFor(username);                // ✅ ใช้ String ธรรมดา ไม่พึ่ง context
    });
    return Response.accepted().build();
}
```

**นี่คือแพตเทิร์นมาตรฐานสำหรับงานเบื้องหลังทุกชนิด** — ดึงข้อมูลที่จำเป็น (user id, tenant, permission ที่เกี่ยวข้อง) ออกมาเป็นค่าธรรมดา (String, record, DTO) **ตั้งแต่ตอนที่ request context ยังเปิดอยู่** แล้วส่งค่าเหล่านั้นเข้าไปใน background task โดยตรง — background task ไม่ต้องพึ่ง context อะไรเลย เพราะทุกอย่างที่ต้องใช้ถูกส่งเข้ามาให้ครบแล้ว

### ทางเลือกที่สอง: `@ActivateRequestContext` — เปิด request context เทียมขึ้นมาเอง

```java
@ApplicationScoped
public class ReportWorker {

    @Inject SecurityIdentity identity;    // ใช้ได้เฉพาะตอนอยู่ใน method นี้เท่านั้น

    @ActivateRequestContext
    void process(String reportId) {
        // request context ถูกเปิดขึ้นมาใหม่ (ของว่าง ไม่ใช่ context เดิมของ request ที่สร้างงานนี้)
        // ...
    }
}
```

**ข้อจำกัดสำคัญ:** context ที่เปิดด้วย `@ActivateRequestContext` เป็น **context ใหม่ที่ว่างเปล่า ไม่ใช่ context เดิมของ HTTP request ที่สร้างงานนี้ขึ้นมา** — `SecurityIdentity`/ค่าที่เคย resolve ไว้ตอน request เดิม **ไม่ได้ตามมาด้วย** ยังต้องส่งค่าที่จำเป็นเข้ามาเองอยู่ดี วิธีนี้มีประโยชน์เมื่อโค้ดที่เรียกใช้ต้องการให้ "มี request context เปิดอยู่" (ไม่ error) มากกว่าที่จะได้ context เดิมกลับมาจริง ๆ

---

## 6. Thread pool config

| property | ค่าเริ่มต้น | ทำอะไร |
|---|---|---|
| `quarkus.thread-pool.core-threads` | `1` | จำนวน thread ขั้นต่ำที่เปิดค้างไว้เสมอ |
| `quarkus.thread-pool.max-threads` | `max(8 × จำนวน CPU, 200)` | จำนวน thread สูงสุด |
| `quarkus.thread-pool.queue-size` | ไม่จำกัด | คิวรองานก่อนจะสร้าง thread ใหม่/ปฏิเสธงาน |
| `quarkus.thread-pool.keep-alive-time` | `30s` | เวลาที่ thread ว่างเกิน core จะถูกปิดทิ้ง |

**นี่คือ thread pool ที่ blocking work (เช่น `@Blocking` endpoint, JDBC call แบบ synchronous) ใช้งานอยู่เบื้องหลัง** ไม่ใช่ event loop thread ของ Vert.x ที่รับ HTTP request เข้ามา — สองอันนี้แยกกันคนละหน้าที่ (รายละเอียดเรื่อง blocking vs non-blocking ฝั่ง REST อยู่ในหัวข้อที่ยังไม่ได้เขียนใน [[Quarkus]])

---

## 7. Virtual threads (Java 21+) — ยังใช้ไม่ได้บน Java 17

```java
@GET
@Path("/slow")
@RunOnVirtualThread
public String slow() { ... }
```

`@RunOnVirtualThread` ให้ endpoint ที่เขียนแบบ blocking ธรรมดารันบน virtual thread แทน platform thread — ได้ throughput สูงขึ้นมากสำหรับงานที่รอ I/O เยอะ โดยไม่ต้องเขียนโค้ดแบบ reactive/async ให้ซับซ้อน **ต้องการ Java 21 ขึ้นไปเสมอ ทั้งตอน build ปกติและตอน native build** (ต้อง GraalVM/Mandrel ที่รองรับ virtual thread ด้วย) — โปรเจกต์ที่ยังอยู่บน Java 17 ใช้ฟีเจอร์นี้ไม่ได้เลย ต้องใช้ `ManagedExecutor`/thread pool ปกติไปก่อน

---

## 8. กับดัก

- **inject `@RequestScoped` bean แล้วใช้ใน `runAsync()` ตรง ๆ** — `ContextNotActiveException` ทันที (ข้อ 4) วิธีแก้คือ capture ค่าก่อน ไม่ใช่หา context ทีหลัง (ข้อ 5)
- **คิดว่า `@ActivateRequestContext` ให้ context เดิมกลับมา** — จริง ๆ ได้ context ใหม่ที่ว่างเปล่า ยังต้องส่งค่าที่จำเป็นเข้าไปเองอยู่ดี (ข้อ 5)
- **`@Scheduled` method พึ่งพา request context** — ไม่มีให้ตั้งแต่แรก เพราะไม่ได้เกิดจาก HTTP request เลย
- **ลืมว่า background task ที่ throw exception ไม่มีใครเห็น** — `runAsync()` ที่ throw จะหายไปเงียบ ๆ ถ้าไม่แนบ `.exceptionally()`/`.whenComplete()` หรือ log เอง ต่างจาก endpoint ปกติที่ exception กลับไปเป็น HTTP error ให้เห็นชัดเจน
- **thread pool เล็กเกินไปสำหรับ background task ที่ใช้เวลานาน** — งาน blocking นาน ๆ ไปแย่ง thread ที่ endpoint อื่นต้องใช้ด้วย ทำให้ทั้งแอปช้าลงพร้อมกัน ควรแยก executor เฉพาะสำหรับงานหนักออกจาก thread pool หลักถ้าจำเป็น
- **ใช้ `@RunOnVirtualThread` บนโปรเจกต์ Java 17** — compile ไม่ผ่าน หรือ native build ไม่ผ่าน เพราะต้องการ Java 21 ขั้นต่ำ (ข้อ 7)

---

## 9. Cheat sheet

```java
// ตารางเวลา
@Scheduled(every = "10s") void x() {}
@Scheduled(cron = "0 15 10 * * ?") void y() {}
@Scheduled(cron = "{prop.name}") void z() {}

// รัน async
@Inject ManagedExecutor executor;
executor.runAsync(() -> doWork(capturedValue));

// ✅ capture ก่อนเข้า background
String username = identity.getPrincipal().getName();
executor.runAsync(() -> process(username));

// ❌ ห้ามทำ
executor.runAsync(() -> identity.getPrincipal().getName());  // ContextNotActiveException
```

```properties
quarkus.thread-pool.core-threads=2
quarkus.thread-pool.max-threads=200
quarkus.thread-pool.queue-size=1000
```

| อาการ | สาเหตุ |
|---|---|
| `ContextNotActiveException: RequestScoped context was not active` | ใช้ `@RequestScoped` bean/`SecurityIdentity` ใน background thread ตรง ๆ |
| `@ActivateRequestContext` แล้วยังไม่เห็นข้อมูลเดิม | context ใหม่ว่างเปล่า ไม่ใช่ context เดิมของ request ต้นทาง |
| background task พังแต่ไม่มี log/error ให้เห็นเลย | ไม่ได้แนบ `.exceptionally()`/`.whenComplete()` หรือ log เอง |
| แอปทั้งตัวช้าลงตอนมีงาน background หนัก ๆ | ใช้ thread pool เดียวกับ endpoint ปกติ ไม่ได้แยก executor |
| `@RunOnVirtualThread` compile/build ไม่ผ่าน | โปรเจกต์อยู่บน Java 17 — ต้องการ Java 21 ขึ้นไป |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus]] — หน้ารวม
- [[Quarkus Config]] — ตั้งค่า cron expression ผ่าน config property แทน hardcode
- [[Quarkus WebSocket]] — อีกที่ที่ต้อง push ข้อความจาก thread อื่นในระบบ (ไม่ใช่แค่ตอบกลับ request ตรง ๆ)

## 📖 อ่านต่อ

- [Quarkus — Scheduler reference guide](https://quarkus.io/guides/scheduler-reference/)
- [Quarkus — Context propagation](https://quarkus.io/guides/context-propagation/)
- [Quarkus — Virtual thread support](https://quarkus.io/guides/virtual-threads/)
