---
tags:
  - java
  - quarkus
  - logging
  - observability
  - ops
type: reference
created: 2026-08-18
---

# 📋 Quarkus Logging — ตั้งค่า level บน server

> Quarkus ใช้ **JBoss LogManager** อยู่ข้างใต้ และรองรับ API หลายตัว (JBoss Logging, SLF4J, JUL, Log4j) โดยเปลี่ยนเส้นทางมาที่ตัวเดียวกันหมด
> ค่าเริ่มต้นคือส่ง log ออก **stdout** ซึ่งเป็นสิ่งที่ถูกต้องแล้วสำหรับการรันในคอนเทนเนอร์

---

## 1. ⭐ ระดับ log ทำงานยังไง

### 1.1 บันไดระดับ

ระดับ log เป็น**บันไดเรียงตามความรุนแรง** แต่ละขั้นมีตัวเลขกำกับอยู่ข้างใน

| ระดับ   | ค่าตัวเลข | ใช้เมื่อ                                                       |
| ------- | --------- | -------------------------------------------------------------- |
| `OFF`   | —         | ปิดสนิท ไม่เอาอะไรเลย                                          |
| `FATAL` | 1100      | ระบบไปต่อไม่ได้ กำลังจะตาย                                     |
| `ERROR` | 1000      | งานนั้นล้มเหลว แต่ระบบยังอยู่ — **ต้องมีคนมาดู**               |
| `WARN`  | 900       | ผิดปกติแต่ยังทำงานต่อได้ เช่น retry สำเร็จ, ใช้ค่า default แทน |
| `INFO`  | 800       | เหตุการณ์สำคัญทางธุรกิจ เช่น สตาร์ตเสร็จ, สร้างเอกสารแล้ว      |
| `DEBUG` | 500       | รายละเอียดสำหรับไล่ปัญหา ค่าตัวแปร เส้นทางที่โค้ดเดิน          |
| `TRACE` | 400       | ละเอียดที่สุด ทุกก้าว — ใช้ชั่วคราวเท่านั้น                    |
| `ALL`   | —         | เอาหมดทุกอย่าง                                                 |

### 1.2 ตั้งค่าหนึ่ง แล้วเห็นอะไรบ้าง

**การตั้ง level คือการตั้ง "พื้น" ไม่ใช่การเลือกเฉพาะระดับนั้น** — ข้อความจะถูกแสดงเมื่อ **ระดับของข้อความ ≥ ระดับที่ตั้งไว้** แปลว่าตั้งไว้ระดับไหน จะเห็นระดับนั้น**ขึ้นไปจนถึง FATAL เสมอ**

| ตั้งเป็น | เห็น | ไม่เห็น |
|---|---|---|
| `TRACE` | TRACE · DEBUG · INFO · WARN · ERROR · FATAL | — |
| `DEBUG` | DEBUG · INFO · WARN · ERROR · FATAL | TRACE |
| `INFO` ← ค่าเริ่มต้น | INFO · WARN · ERROR · FATAL | TRACE · DEBUG |
| `WARN` | WARN · ERROR · FATAL | TRACE · DEBUG · INFO |
| `ERROR` | ERROR · FATAL | ทุกอย่างที่ต่ำกว่า |
| `FATAL` | FATAL | ทุกอย่างที่ต่ำกว่า |
| `OFF` | — | ทุกอย่าง |

```
TRACE   DEBUG   INFO   WARN   ERROR   FATAL
  ·       ·      ·       ·       ·       ·
          └───────────── level=DEBUG เห็นตั้งแต่ตรงนี้ไปทางขวา ──────────┘
                         └──── level=WARN เห็นตั้งแต่ตรงนี้ ────┘
```

**ไม่มีวิธีเลือกเฉพาะช่วง** เช่น "เอาแค่ DEBUG กับ INFO แต่ไม่เอา ERROR" ทำไม่ได้ด้วย config ปกติ เพราะกลไกมันเป็นเพดานล่างอย่างเดียว — ถ้าจำเป็นจริง ๆ ต้องเขียน custom filter เอง ซึ่งแทบไม่มีเหตุผลให้ทำ

### 1.3 ⚠️ มีด่านกรอง 3 ชั้น ต้องผ่านให้ครบทุกชั้น

**นี่คือสาเหตุอันดับหนึ่งที่ "ตั้ง DEBUG แล้วยังไม่เห็น log"** — คนมักตั้งชั้นเดียวแล้วคิดว่าจบ

```
ข้อความ DEBUG ถูกเขียนในโค้ด
        │
        ▼
┌──────────────────────────────┐
│ ด่าน 1 · min-level            │  build time — โค้ดถูกสร้างไว้ไหม
│ quarkus.log.min-level        │  ต่ำกว่านี้ = ถูกตัดทิ้งตอนคอมไพล์
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ ด่าน 2 · level ของ logger     │  runtime — category นี้เปิดถึงระดับไหน
│ quarkus.log.level            │  ใช้ค่าของ category ที่เจาะจงที่สุด
│ quarkus.log.category."x".level│
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ ด่าน 3 · level ของ handler    │  runtime — ปลายทางยอมรับถึงระดับไหน
│ quarkus.log.console.level    │  console / file / syslog แยกกันคนละค่า
│ quarkus.log.file.level       │
└──────────────┬───────────────┘
               ▼
          เห็นใน log
```

ตัวอย่างที่ตั้งแล้วยังไม่เห็น เพราะตกด่านที่ 3

```properties
quarkus.log.level=DEBUG            # ด่าน 2 ผ่าน
quarkus.log.console.level=INFO     # ด่าน 3 ตก ← DEBUG ไม่โผล่บน console
```

ตัวอย่างที่ตกด่านที่ 1

```properties
quarkus.log.min-level=INFO         # ด่าน 1 ตก — โค้ด DEBUG ไม่มีอยู่ตั้งแต่แรก
quarkus.log.level=DEBUG            # ตั้งไปก็ไม่มีผล
```

> **ค่าเริ่มต้นของ handler คือ `ALL`** ดังนั้นถ้าไม่เคยไปตั้ง `console.level` เอง ด่าน 3 จะไม่ขวางอะไร — ปัญหานี้เกิดเฉพาะกับคนที่เคยไปตั้งไว้แล้วลืม

### 1.4 `min-level` — ด่านที่แก้ทีหลังไม่ได้

| | `quarkus.log.level` | `quarkus.log.min-level` |
|---|---|---|
| ตั้งเมื่อไหร่มีผล | **runtime** — เปลี่ยนตอน deploy ได้ | **build time** — อบติดไปกับ artifact |
| ค่าเริ่มต้น | `INFO` | `DEBUG` |
| ทำหน้าที่ | กรองว่าจะแสดงอะไร | **เพดานล่างว่าโค้ดรองรับได้ถึงระดับไหน** |

`min-level` คือระดับที่ Quarkus **สร้างโค้ดรองรับไว้ตอน build** อะไรที่ต่ำกว่านั้นถูกตัดทิ้งตั้งแต่ตอนคอมไพล์ (dead code elimination) เพื่อให้ native image เล็กและเร็ว

**ตั้ง `level` ต่ำกว่า `min-level` เมื่อไหร่ `min-level` ชนะเสมอ** — log ระดับนั้นไม่ได้ถูกกรองออก แต่**ไม่เคยถูกสร้างขึ้นมาตั้งแต่แรก**

> **ถ้าจะเปิด DEBUG บน prod ได้ในอนาคต ต้องปล่อย `min-level` ไว้ที่ `DEBUG` ตั้งแต่ตอน build** ไม่ใช่ตอนที่ระบบมีปัญหาแล้วค่อยมาคิด เพราะถึงตอนนั้นต้อง build ใหม่และ deploy ใหม่

ถ้าอยากได้ TRACE ต้องประกาศทั้งคู่ และควรจำกัดเฉพาะ category ที่ต้องการ

```properties
quarkus.log.category."org.hibernate.SQL".min-level=TRACE
quarkus.log.category."org.hibernate.SQL".level=TRACE
```

### 1.5 ระดับไหนควรใช้ตอนไหน

| สถานการณ์ | ตั้งเป็น |
|---|---|
| dev บนเครื่องตัวเอง | `DEBUG` เฉพาะ package ของเรา ส่วนที่เหลือ `INFO` |
| prod ปกติ | `INFO` — เห็นเหตุการณ์สำคัญ ไม่ท่วม |
| prod ที่ log เยอะจนเปลือง | `WARN` แต่ระวังว่าจะไม่เหลือบริบทตอนเกิดปัญหา |
| library ที่พูดมาก | `WARN` เฉพาะ category นั้น |
| ตอนไล่ปัญหาเฉพาะหน้า | `DEBUG` **เฉพาะ category ที่สงสัย** ไม่ใช่ทั้งแอป |

**อย่าตั้งทั้งแอปเป็น `DEBUG` บน prod** ไม่ใช่แค่เปลือง แต่ข้อความที่สำคัญจริงจะจมหายไปในกองข้อความจนหาไม่เจอ ซึ่งแย่กว่าไม่มี log เสียอีก

---

## 2. Category — คุมทีละส่วน

log จัดเป็นลำดับชั้นตามชื่อ package ค่าที่ตั้งกับ category แม่มีผลกับลูกทั้งหมด เว้นแต่ลูกจะตั้งทับ

```properties
# ทั้งแอปเงียบ ๆ
quarkus.log.level=INFO

# แต่โค้ดเราเองละเอียดหน่อย
quarkus.log.category."org.acme".level=DEBUG

# ไล่ SQL
quarkus.log.category."org.hibernate.SQL".level=DEBUG

# library ที่พูดมากเกินไป
quarkus.log.category."io.netty".level=WARN
```

**ต้องมีเครื่องหมายคำพูดรอบชื่อ category เสมอ** ไม่งั้นจุดจะถูกตีความเป็นตัวคั่นของ config

ระดับที่ใช้ได้ เรียงจากละเอียดไปหยาบ: `TRACE` → `DEBUG` → `INFO` → `WARN` → `ERROR` → `FATAL` → `OFF`

### option อื่นของ category

```properties
quarkus.log.category."org.acme".handlers=FILE       # ส่งไป handler เฉพาะ
quarkus.log.category."org.acme".use-parent-handlers=false   # ไม่ส่งต่อขึ้นไปแม่
```

`use-parent-handlers=false` มีประโยชน์เวลาอยากแยก log ของบางส่วนออกไปไฟล์เดียว โดยไม่ให้ซ้ำใน console

---

## 3. ตั้งค่าบน server — ไม่ต้อง build ใหม่

Quarkus อ่าน config จากหลายแหล่ง เรียงตามลำดับความสำคัญ **จากมากไปน้อย**

| ลำดับ | แหล่ง | ใช้ตอนไหน |
|---|---|---|
| 1 | System property (`-D`) | สั่งตอนรัน |
| 2 | Environment variable | **คอนเทนเนอร์ / Kubernetes** |
| 3 | ไฟล์ `.env` | เครื่อง dev |
| 4 | `config/application.properties` ข้าง ๆ ไฟล์รัน | แก้ config หลัง deploy โดยไม่แตะ image |
| 5 | `application.properties` ใน artifact | ค่าตั้งต้น |

### แบบ environment variable

**กฎการแปลงชื่อ** — ตัวพิมพ์ใหญ่, `.` และ `-` กลายเป็น `_`, ส่วนชื่อที่อยู่ในเครื่องหมายคำพูด (เช่น category) ใช้ **ขีดล่างคู่คร่อมหัวท้าย** และจุดข้างในกลายเป็นขีดล่างเดี่ยว

```bash
# quarkus.log.level
QUARKUS_LOG_LEVEL=DEBUG

# quarkus.log.category."org.hibernate".level
QUARKUS_LOG_CATEGORY__ORG_HIBERNATE__LEVEL=DEBUG

# quarkus.log.console.level
QUARKUS_LOG_CONSOLE_LEVEL=DEBUG
```

> ตรงนี้พลาดกันบ่อยมาก — ใช้ขีดล่างเดี่ยวคร่อม category แล้วมันจะเงียบไปเฉย ๆ ไม่มี error ไม่มีคำเตือน ตั้งแล้วไม่มีผลก็ไม่รู้ว่าเพราะอะไร

ใน Kubernetes ก็คือ `env` ธรรมดา

```yaml
env:
  - name: QUARKUS_LOG_LEVEL
    value: "INFO"
  - name: QUARKUS_LOG_CATEGORY__ORG_ACME__LEVEL
    value: "DEBUG"
```

### แบบ system property

```bash
java -Dquarkus.log.level=DEBUG \
     -Dquarkus.log.category.\"org.acme\".level=TRACE \
     -jar quarkus-run.jar
```

ต้อง escape เครื่องหมายคำพูด และ shell แต่ละตัวก็ต่างกัน — **บน server แนะนำใช้ environment variable จะเจ็บน้อยกว่า**

### แบบไฟล์ config ข้างนอก

วางไฟล์ `config/application.properties` ไว้ข้าง ๆ ไฟล์ที่รัน มันจะถูกอ่านทับค่าใน artifact
มีประโยชน์เวลาอยากปรับ config โดยไม่ต้อง build image ใหม่ และไม่อยากยัด env var เป็นสิบตัว

### ⚠️ เปลี่ยน level แล้วต้องรีสตาร์ต

Quarkus **ไม่มี endpoint สำหรับเปลี่ยน log level ตอนรันอยู่** — Dev UI ที่เปลี่ยนได้ทันทีมีเฉพาะใน dev mode
บน prod การเปลี่ยน level = restart pod

**นี่คือเหตุผลจริง ๆ ที่ต้องปล่อย `min-level` ไว้ที่ `DEBUG`** — อย่างน้อยเวลามีปัญหาจะแค่ restart ไม่ต้อง build ใหม่ทั้งรอบ

---

## 4. Handler — log ไปไหน

### Console (เปิดอยู่แล้ว)

```properties
quarkus.log.console.enable=true
quarkus.log.console.level=INFO
quarkus.log.console.format=%d{yyyy-MM-dd HH:mm:ss,SSS} %-5p [%c{2.}] (%t) %s%e%n
quarkus.log.console.color=false      # ปิดบน server ไม่งั้นได้ ANSI code ปนใน log
```

### File

```properties
quarkus.log.file.enable=true
quarkus.log.file.path=/var/log/app/application.log
quarkus.log.file.level=INFO
quarkus.log.file.rotation.max-file-size=10M
quarkus.log.file.rotation.max-backup-index=5
quarkus.log.file.rotation.file-suffix=.yyyy-MM-dd
quarkus.log.file.rotation.rotate-on-boot=true
```

> **ในคอนเทนเนอร์ไม่ควรเขียนไฟล์** — filesystem หายไปพร้อม pod, ไม่มีใครไปหมุนไฟล์ให้, และดิสก์เต็มได้ ปล่อยให้ log ออก stdout แล้วให้ platform เก็บไปเอง
> การเขียนไฟล์เหมาะกับการรันบน VM แบบดั้งเดิมเท่านั้น

### Syslog

```properties
quarkus.log.syslog.enable=true
quarkus.log.syslog.endpoint=logserver:514
quarkus.log.syslog.protocol=tcp
quarkus.log.syslog.syslog-type=rfc5424
```

### Async — ถ้า log เยอะจนกระทบ throughput

```properties
quarkus.log.console.async=true
quarkus.log.console.async.queue-length=512
quarkus.log.console.async.overflow=block   # block หรือ discard
```

`discard` เร็วกว่าแต่ **ทิ้ง log เงียบ ๆ ตอนคิวเต็ม** ซึ่งมักเกิดตอนระบบมีปัญหา — คือช่วงเวลาที่เราต้องการ log มากที่สุดพอดี เลือกให้ตรงกับว่าอะไรสำคัญกว่ากัน

---

## 5. รูปแบบข้อความ

```properties
quarkus.log.console.format=%d{yyyy-MM-dd HH:mm:ss,SSS} %-5p [%c{3.}] (%t) %s%e%n
```

| ตัวแทน | ได้อะไร |
|---|---|
| `%d{...}` | เวลา ตามรูปแบบ `SimpleDateFormat` |
| `%p` | ระดับ (`%-5p` = จัดชิดซ้ายกว้าง 5 ตัว ให้คอลัมน์ตรงกัน) |
| `%c` | ชื่อ category — `%c{3.}` ย่อ package ให้เหลือ 3 ส่วนท้าย |
| `%s` | ข้อความอย่างเดียว |
| `%m` | ข้อความ + exception |
| `%e` | exception stack trace |
| `%t` | ชื่อ thread |
| `%X{key}` | ค่าใน MDC ตัวเดียว |
| `%X` | ทุกค่าใน MDC |
| `%l` | ตำแหน่งในโค้ด (คลาส เมธอด บรรทัด) — **แพง อย่าใช้บน prod** |
| `%n` | ขึ้นบรรทัดใหม่ |

`%l` ต้องสร้าง stack trace ทุกครั้งที่เขียน log เพื่อหาว่าเรียกมาจากไหน ราคาแพงมากเมื่อ log ถี่

---

## 6. MDC — ผูก log ให้ตามรอย request ได้

ปัญหาคลาสสิกของ log บน server: มี request หลายอันวิ่งพร้อมกัน แล้ว log ปนกันจนแยกไม่ออกว่าบรรทัดไหนเป็นของใคร

**MDC (Mapped Diagnostic Context)** คือที่เก็บค่าประจำ thread ที่ถูกแนบไปกับทุกบรรทัด log อัตโนมัติ

```java
MDC.put("requestId", UUID.randomUUID().toString());
MDC.put("userId", user.getId());
try {
    process();
} finally {
    MDC.clear();       // ← ต้องล้าง ไม่งั้นค่าค้างไปกับ thread ตัวถัดไป
}
```

```properties
quarkus.log.console.format=%d{HH:mm:ss} %-5p [%X{requestId}] [%c{2.}] %s%n
```

ถ้าใช้ tracing อยู่แล้ว `traceId` / `spanId` มักถูกใส่ให้ใน MDC อัตโนมัติ — ใส่ `%X{traceId}` ใน format แล้วจะโยง log กับ trace ได้ทันที

> **MDC ไม่ตามไปกับ thread ใหม่โดยอัตโนมัติ** งานที่โยนเข้า executor หรือ reactive chain ต้องส่งค่าข้ามไปเอง

---

## 7. JSON log — สำหรับระบบรวม log

ถ้ามี log aggregator (Elastic, Loki, CloudWatch) การส่ง log เป็น JSON ทำให้ query ตามฟิลด์ได้ แทนที่จะต้อง regex เอา

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-logging-json</artifactId>
</dependency>
```

```properties
%prod.quarkus.log.console.json.enabled=true
%prod.quarkus.log.console.json.log-format=ecs        # default | ecs | gcp
%prod.quarkus.log.console.json.mdc.flat-fields=true
%dev.quarkus.log.console.json.enabled=false          # dev อ่านแบบข้อความสบายตากว่า
```

**สองเรื่องที่ต้องรู้**

- extension นี้ **ทับ format ของ console ทั้งหมด** — `quarkus.log.console.format` กับสีจะถูกเมินไปเลย
- `mdc.flat-fields=true` ดัน MDC ขึ้นมาเป็นฟิลด์ระดับบนสุด ทำให้ query ง่ายกว่าอยู่ซ้อนใน object

---

## 8. เขียน log ในโค้ด

Quarkus มี logger แบบ static ที่ไม่ต้องประกาศ field

```java
import io.quarkus.logging.Log;

public class OrderService {
    public void place(Order o) {
        Log.infof("order placed id=%s total=%s", o.id(), o.total());
        Log.debugf("payload=%s", o);
        Log.error("payment failed", e);
    }
}
```

Quarkus แปลง bytecode ตอน build ให้กลายเป็น logger ประจำคลาสนั้นเอง — **ใช้ได้เฉพาะในคลาสที่เป็น bean และเรียกจากเมธอดปกติเท่านั้น** ถ้าต้องใช้ในที่แปลก ๆ (static block, คลาสที่ไม่ถูกจัดการโดย Quarkus) ให้ประกาศ logger แบบเดิม

### `f` ต่อท้าย และการหลีกเลี่ยงการต่อ string

```java
Log.debug("user " + userId + " loaded " + list.size() + " items");   // ❌
Log.debugf("user %s loaded %d items", userId, list.size());          // ✅
```

แบบแรกต่อ string **ทุกครั้งที่โค้ดวิ่งผ่าน** แม้ระดับ DEBUG จะปิดอยู่ก็ตาม
แบบหลังส่งพารามิเตอร์เข้าไปเฉย ๆ แล้ว format ก็ต่อเมื่อจะเขียนจริง

---

## 9. กับดัก

- **ตั้ง `level` ต่ำกว่า `min-level`** → ไม่มีผล และไม่มีอะไรเตือน (ข้อ 1)
- **ชื่อ env var ของ category ผิด** → เงียบ ไม่มี error (ข้อ 3)
- **เปิด DEBUG ทิ้งไว้บน prod** → I/O หนัก, ดิสก์เต็ม, ค่าเก็บ log บาน และ log ที่สำคัญจมหายในกองข้อความ
- **`quarkus.log.console.color=true` บน server** → ได้ ANSI escape code ปนใน log ไฟล์ อ่านยากและ parser พัง
- **log ข้อมูลอ่อนไหว** — token, รหัสผ่าน, เลขบัตร, ข้อมูลส่วนบุคคล ระวังการ log object ทั้งก้อนด้วย `toString()` ซึ่งลากทุกฟิลด์ออกมา
- **log ใน loop** — วนหมื่นรอบแล้ว log ทุกรอบ ทำให้ระบบช้าลงและกลบ log อื่นจนหมด
- **`%l` ในรูปแบบข้อความ** → สร้าง stack trace ทุกบรรทัด
- **async + `discard`** → ทิ้ง log ตอนคิวเต็ม ซึ่งคือตอนที่ระบบกำลังมีปัญหา
- **ลืม `MDC.clear()`** → ค่าค้างไปกับ thread ที่ถูกใช้ซ้ำ แล้ว log ของ request ถัดไปติด id ของคนก่อน

---

## 10. ชุด config ที่ใช้ได้เลย

```properties
# ---------- ค่ากลาง ----------
quarkus.log.level=INFO
quarkus.log.min-level=DEBUG          # ปล่อยไว้ เพื่อให้เปิด DEBUG ทีหลังได้โดยไม่ต้อง build ใหม่
quarkus.log.category."org.acme".level=INFO

# ---------- dev ----------
%dev.quarkus.log.console.color=true
%dev.quarkus.log.category."org.acme".level=DEBUG
%dev.quarkus.log.console.format=%d{HH:mm:ss} %-5p [%c{2.}] %s%e%n

# ---------- prod ----------
%prod.quarkus.log.console.color=false
%prod.quarkus.log.console.json.enabled=true
%prod.quarkus.log.console.json.log-format=ecs
%prod.quarkus.log.console.json.mdc.flat-fields=true
%prod.quarkus.log.category."io.netty".level=WARN
%prod.quarkus.log.category."org.apache".level=WARN
```

### เปิด DEBUG ชั่วคราวเพื่อไล่ปัญหา

```bash
# ตั้ง env แล้ว restart — จำกัดเฉพาะส่วนที่สงสัย อย่าเปิดทั้งแอป
QUARKUS_LOG_CATEGORY__ORG_ACME_PAYMENT__LEVEL=DEBUG
```

**ตั้งเตือนตัวเองให้ปิดกลับด้วย** — DEBUG ที่ลืมปิดไว้คือสาเหตุยอดนิยมของดิสก์เต็มและบิลค่า log พุ่ง

---

## 11. Cheat sheet

| อาการ | สาเหตุ |
|---|---|
| ตั้ง DEBUG แล้วไม่เห็น log | ตกด่านใดด่านหนึ่งใน 3 ด่าน — ไล่จาก `min-level` → `level` ของ category → `level` ของ handler |
| ตั้ง `quarkus.log.level=DEBUG` แล้วยังไม่เห็นบน console | `quarkus.log.console.level` ถูกตั้งไว้สูงกว่า |
| ตั้ง `min-level` แล้วยังไม่เห็น | `min-level` เป็น build-time ต้อง build ใหม่ ไม่ใช่แค่ restart |
| อยากเห็นแค่ DEBUG กับ INFO ไม่เอา ERROR | ทำไม่ได้ — level เป็นเพดานล่างอย่างเดียว เห็นระดับนั้นขึ้นไปเสมอ |
| ตั้ง env var แล้วไม่มีผล | ชื่อ category ต้องใช้ขีดล่างคู่คร่อม `__ORG_ACME__` |
| log ไม่ขึ้นเลยหลัง deploy | `quarkus.log.console.enable=false` หรือ level ถูกตั้งเป็น `OFF` |
| log มีอักขระประหลาด `[0m` | เปิด color บน server |
| แยกไม่ออกว่า log ไหนของ request ไหน | ยังไม่ได้ใส่ MDC ใน format |
| id ใน log เป็นของคนอื่น | ลืม `MDC.clear()` |
| ตั้ง `console.format` แล้วไม่มีผล | เปิด JSON logging อยู่ — มันทับ format ทั้งหมด |
| แอปช้าลงหลังเปิด DEBUG | ปกติ — DEBUG แพงจริง จำกัดเฉพาะ category ที่ต้องดู |
| log หายเป็นช่วง ๆ ตอนระบบล่ม | async overflow เป็น `discard` |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus]] — หน้ารวม
- [[Quarkus Health Check]] — สัญญาณสุขภาพอีกด้านที่ระบบภายนอกอ่าน
- [[Quarkus Build]] — `min-level` เป็น build-time config เหมือนอีกหลายตัวใน Quarkus

## 📖 อ่านต่อ

- [Quarkus — Logging configuration](https://quarkus.io/guides/logging)
- [Quarkus Logging JSON — Quarkiverse](https://docs.quarkiverse.io/quarkus-logging-json/dev/index.html)
- [Red Hat — Logging configuration reference](https://docs.redhat.com/en/documentation/red_hat_build_of_quarkus/3.2/html/logging_configuration/logging-configuration)
