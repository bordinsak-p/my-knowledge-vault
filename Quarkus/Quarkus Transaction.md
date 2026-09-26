---
tags:
  - quarkus
  - java
  - transaction
type: reference
created: 2026-09-20
---

# 💳 Quarkus Transaction — `@Transactional` และ `QuarkusTransaction`

> Quarkus จัดการ transaction ผ่าน JTA (Narayana) มี 2 ทาง: **`@Transactional`** (ประกาศที่ method — ใช้ส่วนใหญ่) และ **`QuarkusTransaction`** (เขียนโค้ดคุม transaction เองแบบละเอียด — ใช้ตอนต้องการ control ที่ annotation ทำไม่ได้)

---

## 1. `@Transactional` — ทางที่ใช้ 90% ของเวลา

```java
@ApplicationScoped
public class OrderService {

    @Transactional
    public void placeOrder(Order order) {
        orderRepository.persist(order);   // exception ตรงนี้ → rollback อัตโนมัติ
        paymentService.charge(order);
    }
}
```

### TxType — 6 แบบ ควบคุมว่า "เข้าร่วม/สร้างใหม่/ปฏิเสธ" transaction ยังไง

| TxType | ทำยังไง | ใช้เมื่อ |
|---|---|---|
| **REQUIRED** (default) | มี transaction อยู่แล้ว → เข้าร่วม, ไม่มี → สร้างใหม่ | business logic ทั่วไป (ค่า default เกือบทุกกรณี) |
| **REQUIRES_NEW** | สร้าง transaction ใหม่เสมอ (พัก transaction เดิมไว้ก่อนถ้ามี แล้วกลับมาทำต่อทีหลัง) | งานที่ต้อง commit/rollback อิสระจาก transaction หลัก เช่น audit log ที่อยากบันทึกไว้แม้ transaction หลักจะ rollback |
| **MANDATORY** | ต้องมี transaction อยู่แล้วเท่านั้น ไม่มี → throw exception | บังคับว่า method นี้ต้องถูกเรียกจากใน transaction เสมอ |
| **SUPPORTS** | มีก็เข้าร่วม ไม่มีก็รันแบบไม่มี transaction | method อเนกประสงค์ที่ใช้ได้ทั้งสองสถานการณ์ |
| **NOT_SUPPORTED** | รันโดยไม่มี transaction เสมอ (พัก transaction เดิมไว้ถ้ามี แล้วกลับมาทำต่อทีหลัง) | อ่านข้อมูลอย่างเดียว ไม่อยากมี transaction overhead |
| **NEVER** | ต้องไม่มี transaction อยู่เลย มี → throw exception | บังคับว่า method นี้ต้องไม่ถูกเรียกจากใน transaction |

```java
@Transactional(Transactional.TxType.REQUIRES_NEW)
public void auditLog(String message) { ... }
```

### Rollback — ไม่ใช่ exception ทุกแบบทำให้ rollback

**Default: `RuntimeException` (unchecked) → rollback อัตโนมัติ, checked exception → ไม่ rollback** ต้องกำหนดเองถ้าอยากได้พฤติกรรมอื่น:

```java
@Transactional(rollbackOn = CustomCheckedException.class)      // บังคับ rollback แม้เป็น checked exception
@Transactional(dontRollbackOn = SomeRuntimeException.class)    // ห้าม rollback แม้เป็น RuntimeException
```

---

## 2. `QuarkusTransaction` — คุมเองตอน annotation ไม่พอ

ใช้ตอน: ต้องตัดสินใจว่าจะเปิด transaction ไหมตาม logic runtime (annotation ตัดสินใจตอน compile ไม่ได้), ต้องการ exception handling เฉพาะจุด, หรือเขียนโค้ดในที่ที่ไม่ใช่ CDI bean method (เช่น static utility, lambda)

### Entry point หลัก

```java
QuarkusTransaction.begin();                 // เริ่มเอง คุม commit/rollback เอง
QuarkusTransaction.joiningExisting();       // มีอยู่แล้วเข้าร่วม ไม่มีสร้างใหม่ (≈ REQUIRED)
QuarkusTransaction.requiringNew();          // สร้างใหม่เสมอ พัก transaction เดิมไว้ก่อน (≈ REQUIRES_NEW)
QuarkusTransaction.suspendingExisting();    // พัก transaction เดิมไว้ รันแบบไม่มี transaction เลย (≈ NOT_SUPPORTED)
QuarkusTransaction.disallowingExisting();   // ต้องไม่มี transaction อยู่แล้วเท่านั้น (error ถ้ามี) แล้วเริ่มใหม่ให้เอง
```

**`joiningExisting()`/`requiringNew()`/`suspendingExisting()` มีพฤติกรรมคล้าย TxType ที่ชื่อใกล้เคียงกันในข้อ 1** แต่ `disallowingExisting()` ไม่มี TxType ไหนตรงเป๊ะ — ผสมระหว่าง "ต้องไม่มี transaction อยู่ก่อน" แบบ NEVER กับ "แต่ยังสร้าง transaction ใหม่ให้เอง" ซึ่ง NEVER ไม่ทำ (NEVER แค่รันแบบไม่มี transaction เฉยๆ)

### Explicit begin/commit/rollback — คุมเองทุกขั้นตอน

```java
QuarkusTransaction.begin();
try {
    orderRepository.persist(order);
    QuarkusTransaction.commit();
} catch (Exception e) {
    QuarkusTransaction.rollback();
}
```

**ข้อควรระวัง:** ถ้าลืม commit/rollback แล้วปล่อยให้ CDI request scope จบไปเฉยๆ Quarkus จะ **rollback ให้อัตโนมัติเป็น safety net** — แต่ไม่ควรพึ่งพฤติกรรมนี้ ควรปิด transaction ให้ครบเองเสมอ

### แบบ lambda — `.run()`/`.call()` พร้อม fluent option

```java
// run() — ไม่ต้องการ return ค่า
QuarkusTransaction.requiringNew().run(() -> {
    orderRepository.persist(order);
});

// call() — ต้องการ return ค่ากลับมา พร้อม timeout และ custom exception handling
int result = QuarkusTransaction.requiringNew()
    .timeout(10)                                    // วินาที ก่อน transaction timeout
    .exceptionHandler(throwable -> {
        if (throwable instanceof TemporaryException) {
            return TransactionExceptionResult.COMMIT;    // สั่งให้ commit ต่อแม้มี exception
        }
        return TransactionExceptionResult.ROLLBACK;
    })
    .call(() -> {
        return processAndReturnCount();
    });
```

`.exceptionHandler()` ให้ตัดสินใจเองเป็นรายกรณีว่า exception แบบไหนควร commit ต่อ (`COMMIT`) หรือ rollback (`ROLLBACK`) — ยืดหยุ่นกว่า `rollbackOn`/`dontRollbackOn` ของ `@Transactional` ที่กำหนดตายตัวตอน compile

---

## 3. เลือกใช้ตัวไหน

| ต้องการ | ใช้ |
|---|---|
| business logic ทั่วไป กำหนด boundary ที่ method ชัดเจน | `@Transactional` |
| ตัดสินใจเปิด/ไม่เปิด transaction ตาม logic ตอน runtime | `QuarkusTransaction` |
| exception handling เฉพาะจุดแบบละเอียด | `QuarkusTransaction` (`.exceptionHandler()`) |
| เขียนใน static method/lambda ที่ไม่ใช่ CDI bean | `QuarkusTransaction` (annotation ใช้ได้กับ CDI bean method เท่านั้น) |
| ไม่แน่ใจว่าจะเลือกอะไร | `@Transactional` — อ่านง่ายกว่า สั้นกว่า ครอบคลุมเกือบทุกกรณีจริงอยู่แล้ว |

---

## กับดัก

- **คาดหวังว่า checked exception จะ rollback ให้เหมือน RuntimeException** — default ไม่ทำ ต้องระบุ `rollbackOn` เองเสมอถ้า business logic โยน checked exception ที่ควร rollback ด้วย
- **ใช้ `REQUIRES_NEW` โดยไม่รู้ว่ามันพัก transaction เดิมไว้จริง** — ถ้า transaction หลักยังไม่ commit แล้วมาแก้ข้อมูลแถวเดียวกันใน `REQUIRES_NEW` block เสี่ยง deadlock ได้ถ้า database ล็อกแถวเดียวกันจากสอง transaction พร้อมกัน
- **ใช้ `QuarkusTransaction.begin()` แล้วลืม commit/rollback ให้ครบทุก path** (โดยเฉพาะ path ที่ throw exception) — พึ่งพา auto-rollback ตอน scope จบแทนที่จะปิดเองให้ครบ ทำให้ debug ยากเพราะจุดที่ transaction จบจริงไม่ชัดเจน
- **เรียก `@Transactional` method จากภายในคลาสเดียวกัน (self-invocation)** — ไม่ทำงาน เพราะ `@Transactional` ทำงานผ่าน CDI proxy ที่ห่อ method ไว้จากภายนอก เรียกตรงจาก `this.method()` ข้ามการห่อ proxy ไปเลย ต้อง inject ตัวเองหรือแยก logic ไปอีกคลาสหนึ่ง
- **ใช้ `NOT_SUPPORTED`/`suspendingExisting()` แล้วคาดหวังว่าจะเห็นข้อมูลที่ transaction หลักเพิ่งเขียนแต่ยังไม่ commit** — เห็นหรือไม่ขึ้นกับ isolation level ของ database เพราะรันคนละ transaction กันจริง ๆ ไม่ใช่แค่ "ไม่มี transaction wrapper" เฉยๆ

---

## Cheat sheet

```java
@Transactional                                    // REQUIRED (default)
@Transactional(Transactional.TxType.REQUIRES_NEW)
@Transactional(rollbackOn = MyCheckedException.class)
@Transactional(dontRollbackOn = SomeRuntimeException.class)
```

```java
QuarkusTransaction.begin() / .commit() / .rollback()
QuarkusTransaction.requiringNew().run(() -> { ... })
QuarkusTransaction.requiringNew().timeout(10).call(() -> value)
QuarkusTransaction.joiningExisting().run(() -> { ... })
QuarkusTransaction.suspendingExisting().run(() -> { ... })
```

| อาการ | สาเหตุ |
|---|---|
| checked exception ไม่ rollback ทั้งที่ควร | ลืมใส่ `rollbackOn` |
| `@Transactional` ไม่ทำงานเลย | self-invocation ข้าม CDI proxy |
| `REQUIRES_NEW` ค้าง/deadlock | สอง transaction ล็อกแถวเดียวกันพร้อมกัน |

## 🔗 เกี่ยวข้อง

- [[Quarkus EntityManager]] — EntityManager ผูกอยู่กับ transaction context เสมอ
- [[Quarkus Thread Pool]] — `@Transactional` ผูกกับ request scope, background thread ต้องจัดการ transaction เองแยกต่างหาก

## 📖 อ่านต่อ

- [Quarkus — Using transactions](https://quarkus.io/guides/transaction)
- [QuarkusTransaction — Javadoc](https://javadoc.io/doc/io.quarkus/quarkus-narayana-jta/latest/io/quarkus/narayana/jta/QuarkusTransaction.html)
