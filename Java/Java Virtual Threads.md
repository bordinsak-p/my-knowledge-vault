---
tags:
  - java
  - concurrency
  - virtual-threads
type: reference
created: 2026-09-21
---

# 🧵 Java Virtual Threads

> **Virtual thread คือ thread น้ำหนักเบาที่ JVM จัดการเอง ไม่ใช่ OS thread ตรง ๆ** — แก้ปัญหา thread pool ไม่พอตอนมีงาน blocking I/O ค้างพร้อมกันเยอะ (เช่นเรียก REST API ต่อเนื่องหลายที่) โดยยังเขียนโค้ดสไตล์ blocking ธรรมดาเหมือนเดิม ไม่ต้องเปลี่ยนไปเขียนแบบ reactive/async ให้ซับซ้อนขึ้นเพื่อได้ throughput สูง — มาจาก **JEP 444** (finalize ใน Java 21 ดู [[Java Updates]])

---

## 1. สร้างยังไง

```java
// สร้าง + สั่งรันทันที
Thread t = Thread.ofVirtual().start(() -> {
    System.out.println("running on " + Thread.currentThread());
});
t.join();

// สร้างไว้ก่อน ยังไม่รัน
Thread t2 = Thread.ofVirtual().unstarted(() -> doWork());
t2.start();

// ใช้แบบ ExecutorService — สไตล์ที่ใช้จริงบ่อยที่สุด
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10_000; i++) {
        int taskId = i;
        executor.submit(() -> callExternalApi(taskId));
    }
}   // try-with-resources รอทุก task จบก่อนปิด executor ให้อัตโนมัติ
```

**`newVirtualThreadPerTaskExecutor()` สร้าง virtual thread ใหม่ 1 ตัวต่อ 1 task เสมอ** — ไม่มีแนวคิด "pool ที่ reuse thread" แบบ `Executors.newFixedThreadPool(n)` อีกต่อไป เพราะ virtual thread ถูกออกแบบให้สร้าง/ทิ้งได้ถูกมาก

---

## 2. ปัญหาที่มันแก้ — thread pool ไม่พอ

```java
// แบบเดิม: platform thread pool ขนาดจำกัด (เช่น 200)
ExecutorService fixed = Executors.newFixedThreadPool(200);
// request ที่ 201 ต้องรอคิว ทั้งที่ thread ส่วนใหญ่แค่ "รอ" response จาก service อื่นเฉย ๆ ไม่ได้ใช้ CPU จริง
```

platform thread แต่ละตัวจองหน่วยความจำ stack หลัก MB และผูกกับ OS thread จริง — สร้างเป็นหมื่นเป็นแสนตัวพร้อมกันไม่ได้ ส่วน virtual thread เบามาก **สร้างเป็นล้านตัวพร้อมกันได้จริง** เพราะไม่ได้ผูกกับ OS thread ตลอดเวลา (ดูข้อ 3)

---

## 3. Carrier thread — กลไกเบื้องหลัง

Virtual thread ไม่ได้รันบน CPU core ตรง ๆ — เวลาทำงานจริงมันถูก **"mount"** ไปอยู่บน platform thread ตัวหนึ่งที่เรียกว่า **carrier thread** (มาจาก pool เล็ก ๆ ปกติเท่าจำนวน CPU core) พอ virtual thread เจอจุด **blocking I/O** (เช่นรอ response HTTP, รอ query DB) มันจะ **"unmount"** ตัวเองออกจาก carrier thread ทันที ปล่อยให้ carrier ไปรับ virtual thread ตัวอื่นทำงานต่อ แล้วพอ I/O เสร็จค่อย mount กลับเข้า carrier ตัวใดตัวหนึ่งเพื่อทำงานต่อ

```mermaid
flowchart LR
    A["Virtual Thread A เริ่มทำงาน"] --> B["mount บน Carrier Thread"]
    B --> C["เจอ blocking I/O (เช่น รอ HTTP response)"]
    C --> D["unmount ออก — carrier ว่างไปรับตัวอื่น"]
    D --> E["I/O เสร็จ — mount กลับเข้า carrier (ตัวไหนก็ได้ที่ว่าง)"]
```

นี่คือเหตุผลที่ virtual thread นับพันนับหมื่นตัวใช้ carrier thread แค่หยิบมือเดียวได้จริง — ต่างจาก platform thread ที่ผูกกับ OS thread ตายตัวตลอดอายุ

---

## 4. Pinning — ตอนที่ unmount ไม่ได้

**Pinning คือตอนที่ virtual thread บล็อกอยู่ แต่ unmount ออกจาก carrier ไม่ได้** — ทำให้ carrier ตัวนั้นถูกยึดไว้เฉย ๆ เหมือน platform thread ปกติ เสีย benefit ของ virtual thread ไปช่วงนั้น

| สถานการณ์ | ยังคง pin อยู่ไหม |
|---|---|
| `synchronized` block/method **ก่อน JDK 24** | ✅ pin (เป็นปัญหาที่รู้จักกันมากที่สุด) |
| `synchronized` block/method **JDK 24 ขึ้นไป** | ❌ ไม่ pin แล้ว — แก้ด้วย **JEP 491** |
| เรียก native code (JNI/Foreign Function API) ที่ block กลับเข้ามาที่ Java | ✅ ยัง pin อยู่เสมอ ไม่ว่าเวอร์ชันไหน |

**คำแนะนำเก่าที่บอกว่า "ห้ามใช้ `synchronized` กับ virtual thread"** ใช้ได้เฉพาะโปรเจกต์ที่ยังอยู่ JDK 21-23 เท่านั้น — ถ้าอัปเป็น 24 ขึ้นไปแล้ว ข้อจำกัดนี้หมดไป (แต่ยังต้องเช็คว่าโปรเจกต์ใช้เวอร์ชันไหนอยู่จริงก่อนเชื่อคำแนะนำนี้)

---

## 5. ไม่ได้ช่วยงาน CPU-bound

Virtual thread แก้ปัญหา "รอ I/O" เท่านั้น — งานที่ใช้ CPU หนัก ๆ ต่อเนื่อง (คำนวณ, loop หนัก ๆ) ยังคงแย่งจำนวน CPU core เท่าเดิมไม่ว่าจะสร้าง virtual thread กี่ตัว **สร้าง virtual thread เป็นล้านตัวให้ CPU-bound task ไม่ได้ช่วยอะไร อาจช้าลงด้วยซ้ำ** (context switch เยอะขึ้นโดยเปล่าประโยชน์)

---

## กับดัก

- **เอา virtual thread ไปทำ pool/reuse แบบ platform thread** — ผิดธรรมชาติตั้งแต่ต้น ออกแบบมาให้สร้างใหม่ทุกครั้งแล้วทิ้ง ถูกกว่า reuse มาก ไม่ต้องมี pool
- **ใช้ `ThreadLocal` เยอะ ๆ ร่วมกับ virtual thread จำนวนมาก** — แต่ละ virtual thread มี copy ของ `ThreadLocal` เป็นของตัวเอง สร้างเป็นล้านตัวพร้อม `ThreadLocal` ที่เก็บข้อมูลหนัก ๆ กินหน่วยความจำรวมมหาศาล ทางเลือกใหม่คือ `ScopedValue` (ดู [[Java Updates]] — finalize ใน Java 25)
- **ยังใช้ `synchronized` บน JDK ต่ำกว่า 24 แล้วแปลกใจว่าทำไม throughput ไม่ขึ้นตามที่คาด** — เจอ pinning (ข้อ 4) เช็คเวอร์ชัน JDK จริงก่อนสรุปว่า virtual thread "ไม่เห็นช่วยอะไรเลย"
- **คาดหวังว่า virtual thread จะเร่งงาน CPU-bound** — ไม่ช่วย (ข้อ 5) ใช้ผิดจุดประสงค์
- **thread pool อื่นที่ต่อจากนี้ยังจำกัดขนาดอยู่ (เช่น DB connection pool)** — ต่อให้ยิง request พร้อมกันเป็นแสนด้วย virtual thread แต่ connection pool ของ DB driver ยังจำกัดที่ 20-50 เหมือนเดิม คอขวดย้ายไปอยู่ตรงนั้นแทน ต้องดูทั้งระบบ ไม่ใช่แค่ชั้น thread

---

## Cheat sheet

```java
// virtual thread เดี่ยว
Thread.ofVirtual().start(() -> { ... });

// ExecutorService (ใช้บ่อยที่สุด) — 1 task = 1 virtual thread ใหม่เสมอ ไม่มี pool
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> callApi());
}

// เช็คว่ากำลังรันบน virtual thread อยู่ไหม
Thread.currentThread().isVirtual();
```

| อาการ | สาเหตุ |
|---|---|
| throughput ไม่ขึ้นตามคาดหลังเปลี่ยนมาใช้ virtual thread | pinning จาก `synchronized` (JDK < 24) หรือ native call (ข้อ 4) |
| memory พุ่งหลังเปลี่ยนมาใช้ virtual thread จำนวนมาก | `ThreadLocal` หนัก ๆ คูณด้วยจำนวน thread เป็นล้าน |
| CPU-bound task ไม่เร็วขึ้นเลย | ใช้ผิดจุดประสงค์ — virtual thread ช่วยเฉพาะงานรอ I/O |
| ระบบยังคอขวดที่เดิมทั้งที่ thread ไม่ขาดแล้ว | resource อื่นที่จำกัดขนาดอยู่ (DB pool, external API rate limit) |

---

## 🔗 เกี่ยวข้อง

- [[Java Updates]] — JEP 444 ในบริบท "อะไรใหม่ในแต่ละเวอร์ชัน" + JEP 491 (แก้ pinning)
- [[Quarkus Thread Pool]] — thread pool/context ฝั่ง Quarkus framework ที่ virtual thread เข้ามาเสริม
- [[Java]] — หน้ารวม

## 📖 อ่านต่อ

- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [JEP 491: Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)
- [Oracle — Virtual Threads Guide](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)
