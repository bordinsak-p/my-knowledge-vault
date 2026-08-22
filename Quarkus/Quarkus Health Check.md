---
tags:
  - java
  - quarkus
  - health-check
  - kubernetes
  - observability
type: reference
created: 2026-08-18
---

# 🩺 Quarkus Health Check

> **มีไว้ให้ระบบภายนอกถามสุขภาพแอปแบบอัตโนมัติ** — โดยเฉพาะ Kubernetes ที่ต้องตัดสินใจแทนเราตลอดเวลาว่าจะฆ่า pod นี้ไหม จะส่ง traffic เข้าไหม
>
> ไม่ใช่ dashboard สำหรับคนดู แต่เป็น **สัญญาที่เครื่องคุยกับเครื่อง**

---

## 1. ติดตั้งแล้วได้อะไร

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-health</artifactId>
</dependency>
```

| endpoint | ตอบคำถามว่า | ผลถ้า DOWN |
|---|---|---|
| `/q/health/live` | ยังมีชีวิตอยู่ไหม | **k8s ฆ่า pod แล้วสร้างใหม่** |
| `/q/health/ready` | พร้อมรับ traffic ไหม | ถอดออกจาก load balancer แต่**ไม่ฆ่า** |
| `/q/health/started` | บูตเสร็จหรือยัง | กันไม่ให้ liveness ฆ่าแอปที่แค่สตาร์ตช้า |
| `/q/health` | รวมทุกอัน | — (ใช้ดูเอง ไม่ควรผูกกับ probe) |

รูปแบบ response:

```json
{
  "status": "UP",
  "checks": [
    { "name": "Database connections health check", "status": "UP" },
    { "name": "external-api", "status": "DOWN",
      "data": { "error": "Connection refused" } }
  ]
}
```

`status` รวมเป็น **AND ของทุก check** — มีตัวเดียว DOWN ก็ DOWN ทั้งก้อน

มี UI ที่ `/q/health-ui/` (เปิดอัตโนมัติใน dev/test, prod ต้องเปิดเอง)

---

## 2. ⭐ หัวใจของเรื่อง: liveness ≠ readiness

**นี่คือจุดที่คนพลาดกันมากที่สุด และผลของการพลาดรุนแรง**

### เกิดอะไรขึ้นถ้าเอาการเช็ค DB ไปใส่ใน liveness

สมมติ DB ล่ม:

1. ทุก pod ตอบ `DOWN` พร้อมกัน
2. k8s ฆ่าทุก pod พร้อมกัน
3. pod ใหม่ขึ้นมา DB ก็ยังล่มอยู่ ตอบ `DOWN` อีก
4. โดนฆ่าอีก → **restart storm ทั้งคลัสเตอร์**

**การรีสตาร์ตแอปไม่ได้ทำให้ DB ฟื้น** การฆ่ามันจึงไม่มีประโยชน์ มีแต่ทำให้แย่ลง — จาก "DB ล่ม แอปตอบ error ได้" กลายเป็น "DB ล่ม และแอปหายไปทั้งหมด"

### กฎที่ใช้ตัดสิน

ถามตัวเองว่า **"รีสตาร์ตแล้วอาการนี้หายไหม"**

| | liveness | readiness |
|---|---|---|
| คำถาม | รีสตาร์ตแล้วหายไหม | ตอนนี้รับงานไหวไหม |
| ผลเมื่อ DOWN | ถูกฆ่า | พักรับงาน |
| **เช็ค dependency ภายนอก** | ❌ **ห้าม** | ✅ ใช่เลย |
| ตัวอย่างที่ควรเช็ค | deadlock, thread pool ตันถาวร, สถานะภายในพัง | DB, Redis, API ปลายทาง, cache อุ่นเสร็จหรือยัง |

**ค่าเริ่มต้นที่ปลอดภัยที่สุด: liveness ไม่ต้องเช็คอะไรเลย** แค่ตอบ UP ว่า process ยังรับ HTTP ได้ก็พอ — ซึ่งการที่มันตอบได้ก็พิสูจน์ตัวมันเองอยู่แล้ว

### แล้ว startup มีไว้ทำไม

แอปที่บูตนาน (JVM mode, โหลด cache ตอนเริ่ม, migrate DB) จะโดน liveness ฆ่าตั้งแต่ยังไม่ทันตั้งตัว
`@Startup` แยกช่วงบูตออกมา — k8s จะไม่เริ่มยิง liveness จนกว่า startup probe จะผ่าน
ทำให้ตั้ง liveness ให้ไวได้ โดยไม่ต้องเผื่อเวลาบูตไว้ใน `initialDelaySeconds`

---

## 3. เขียนเอง

```java
@Readiness                      // หรือ @Liveness / @Startup
@ApplicationScoped
public class ExternalApiHealthCheck implements HealthCheck {

    @Inject @RestClient ProductClient client;

    @Override
    public HealthCheckResponse call() {
        try {
            client.ping();
            return HealthCheckResponse.up("external-api");
        } catch (Exception e) {
            return HealthCheckResponse.named("external-api")
                    .down()
                    .withData("error", e.getMessage())
                    .build();
        }
    }
}
```

**ต้อง `try/catch` เสมอ** — ถ้า `call()` โยน exception ออกมา ตัว health endpoint เองอาจพังทั้งอัน ทำให้ probe ตีความว่าแอปตายทั้งที่จริง ๆ แค่ dependency มีปัญหา

### ตัวอย่าง liveness ที่เขียนถูก

```java
@Liveness
@ApplicationScoped
public class ThreadPoolHealthCheck implements HealthCheck {

    @Inject ManagedExecutor executor;

    @Override
    public HealthCheckResponse call() {
        // เช็คสถานะภายในตัวเอง — ของที่รีสตาร์ตแล้วหายจริง
        boolean stuck = queueDepth() > THRESHOLD && noProgressFor(Duration.ofMinutes(5));
        return HealthCheckResponse.named("worker-pool")
                .status(!stuck)
                .withData("queueDepth", queueDepth())
                .build();
    }
}
```

---

## 4. ของที่ได้ฟรี ไม่ต้องเขียน

extension หลายตัวแถม health check มาให้เอง เข้าไปอยู่ใน **readiness** อัตโนมัติ

```properties
quarkus.datasource.health.enabled=true      # เช็ค DB
```

Redis, Kafka, MongoDB, gRPC ก็มีของตัวเอง

**ปิดทั้งหมดได้ด้วย** `quarkus.health.extensions.enabled=false`

> ⚠️ ระวังเรื่องนี้: ถ้าแอปต่อ datasource หลายตัว (เช่นหลาย tenant) **ตัวใดตัวหนึ่งล่ม readiness จะ DOWN ทั้งแอป** ทั้งที่ tenant อื่นยังใช้งานได้ปกติ กรณีนี้อาจต้องปิด default check แล้วเขียนเองให้ตัดสินตามที่ควรจะเป็น

---

## 5. ผูกกับ Kubernetes

```yaml
livenessProbe:
  httpGet: { path: /q/health/live, port: 9000 }
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet: { path: /q/health/ready, port: 9000 }
  periodSeconds: 5
  failureThreshold: 3

startupProbe:
  httpGet: { path: /q/health/started, port: 9000 }
  periodSeconds: 5
  failureThreshold: 30          # ให้เวลาบูตได้ถึง 150 วินาที
```

**readiness ควรถามถี่กว่า liveness** เพราะผลของมันเบากว่า (แค่พักรับงาน) จะได้ตอบสนองไว
**liveness ควรใจเย็นกว่า** เพราะผลคือการฆ่า ไม่ควรตัดสินจากความผิดพลาดชั่วคราว

> ถ้าใช้ extension `quarkus-kubernetes` มันสร้าง manifest ส่วนนี้ให้อัตโนมัติได้

---

## 6. อย่าเปิด `/q/*` ออกสู่ภายนอก

endpoint ชุด `/q` บอกโครงสร้างระบบให้คนนอกรู้ — เห็นว่าต่อ DB อะไร มี dependency อะไรบ้าง

ย้ายไปพอร์ตแยกได้:

```properties
quarkus.management.enabled=true       # ย้าย /q/* ไปพอร์ต 9000
```

แล้วเปิดเฉพาะพอร์ต 9000 ให้ k8s เข้าถึงภายในคลัสเตอร์ ส่วนพอร์ต 8080 ที่ออกสู่ภายนอกจะไม่มี `/q` อีกต่อไป

---

## 7. กับดัก

- **เอา dependency ไปใส่ liveness** → restart storm (ข้อ 2)
- **health check แพง** — k8s ยิงทุกไม่กี่วินาที ถ้าข้างในไป `select count(*)` จากตารางใหญ่ ตัวเช็คเองจะกลายเป็นภาระ ใช้ `select 1` หรือเช็คแค่ว่าขอ connection ได้
- **ไม่ `try/catch` ใน `call()`** → exception ทำให้ endpoint พังทั้งอัน
- **ผูก probe กับ `/q/health` รวม** → liveness จะพลอยล้มตาม dependency ที่อยู่ใน readiness ผลเท่ากับข้อแรก **ต้องแยก `/live` กับ `/ready` เสมอ**
- **`initialDelaySeconds` ตั้งน้อยเกินไปกับแอป JVM** → โดนฆ่าตั้งแต่ยังบูตไม่เสร็จ ใช้ startup probe แทนการเดาเวลา
- **health check มี side effect** — บางคนเผลอเขียน log หรือ update สถานะใน check ซึ่งจะถูกเรียกทุกไม่กี่วินาทีตลอด 24 ชม.

---

## 8. Cheat sheet

```java
@Readiness @ApplicationScoped
class Check implements HealthCheck {
    public HealthCheckResponse call() {
        return HealthCheckResponse.named("x")
                .status(isOk())              // หรือ .up() / .down()
                .withData("detail", "...")
                .build();
    }
}
```

```properties
quarkus.smallrye-health.root-path=health     # default → /q/health
quarkus.health.extensions.enabled=true       # check ที่ extension แถมมา
quarkus.smallrye-health.ui.enabled=true      # UI (dev/test เปิดอยู่แล้ว)
quarkus.management.enabled=true              # ย้าย /q/* ไปพอร์ต 9000
quarkus.datasource.health.enabled=true
```

| อาการ | สาเหตุ |
|---|---|
| pod รีสตาร์ตวนไม่หยุดตอน DB ล่ม | เอา dependency ไปใส่ liveness |
| pod ถูกฆ่าตอนเพิ่ง deploy | ไม่มี startup probe / `initialDelaySeconds` น้อยไป |
| แอปช้าลงเรื่อย ๆ โดยไม่มีสาเหตุ | health check แพง ถูกเรียกทุกไม่กี่วินาที |
| readiness DOWN ทั้งที่ระบบใช้ได้ | datasource ตัวใดตัวหนึ่งใน multi-datasource ล่ม |
| health endpoint ตอบ 500 | มี check ที่โยน exception ออกมา |
| คนนอกเห็นโครงสร้างระบบ | เปิด `/q/*` บนพอร์ตสาธารณะ |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus]] — หน้ารวม
- [[Quarkus REST Client]] — เช็ค API ปลายทางใน readiness (อย่าลืม timeout สั้น ๆ)
- [[Quarkus Build]] — native บูตเร็วกว่า JVM มาก ทำให้ตั้ง probe ได้กระชับกว่า

## 📖 อ่านต่อ

- [Quarkus — SmallRye Health](https://quarkus.io/guides/smallrye-health)
- [Kubernetes — Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
