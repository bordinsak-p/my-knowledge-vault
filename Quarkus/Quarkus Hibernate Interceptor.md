---
tags:
  - quarkus
  - java
  - hibernate
  - jpa
type: reference
created: 2026-09-30
---

# 🪝 Hibernate Interceptor — ดักวงจรชีวิต entity

> ⚠️ **"Interceptor" ตัวนี้เป็นคนละตัวกับ [[Quarkus Interceptor]] (CDI `@Interceptor`) โดยสิ้นเชิง** — ตัวนั้นดักการเรียก **method** ระดับ Java ทั่วไป (`@Transactional`, custom `@AroundInvoke`) implement `jakarta.interceptor.*` ส่วนตัวนี้ (`org.hibernate.Interceptor`) ดักเฉพาะ **วงจรชีวิตของ entity** ในชั้น Hibernate ORM เท่านั้น (ก่อน/หลัง load, persist, update, delete, flush) — คนละ interface คนละ package ไม่เกี่ยวข้องกันเลยแม้ชื่อจะเหมือนกัน

---

## 1. 3 ทางเลือกในการดัก entity lifecycle — ไม่ได้มีแค่ Interceptor

| ทาง                                       | คืออะไร                                                                                               | จุดเด่น                                                              | ใช้เมื่อ                                                                                                                      |
| ----------------------------------------- | ----------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **JPA `@PrePersist` ฯลฯ**                 | annotation มาตรฐาน JPA บน entity เอง หรือแยกไปคลาส `@EntityListeners`                                 | portable ข้าม JPA provider ใดก็ได้ ไม่ผูก Hibernate เขียนง่ายสุด     | ส่วนใหญ่ — logic เฉพาะ entity ใด entity หนึ่ง                                                                                 |
| **`org.hibernate.Interceptor`** (โน้ตนี้) | interface เดียว ครอบ **ทุก entity ในเซสชัน** ในจุดเดียว เห็น state ละเอียดกว่า (old/new value คู่กัน) | cross-cutting ข้าม entity ทุกตัวโดยไม่ต้องแก้ entity class เลยสักตัว | logic ที่ต้องใช้ร่วมกันทุก entity เช่น audit field, soft-delete filter                                                        |
| **Hibernate `EventListener` SPI**         | ลงทะเบียนแยกทีละ event type (`PreInsertEventListener` ฯลฯ) ผ่าน `Integrator`                          | ละเอียดสุด เลือกฟังเฉพาะ event ที่ต้องการ                            | เคส advanced — Hibernate เองแนะนำแทน Interceptor สำหรับเคสซับซ้อน แต่ **Quarkus ไม่มี first-class support ให้ทางนี้** (ข้อ 5) |

**ในบริบท Quarkus: `Interceptor` คือทางที่เหมาะสุดสำหรับ cross-cutting logic** เพราะมี `@PersistenceUnitExtension` รองรับให้ตรงๆ (ข้อ 4) ส่วน JPA callback เหมาะกับ logic เฉพาะ entity เดียว — ใช้ผสมกันได้ ไม่ต้องเลือกทางเดียว

---

## 2. `org.hibernate.Interceptor` — method ที่ใช้จริงบ่อย

Hibernate 6 ทำให้ทุก method ใน interface นี้เป็น **default method** แล้ว (implement ตรงๆ ได้เลย ไม่ต้องมี base class `EmptyInterceptor` เหมือน Hibernate 5 อีกต่อไป — `EmptyInterceptor` ถูก deprecate ตั้งแต่ 6.0) override เฉพาะตัวที่ต้องใช้พอ

| Method                                                                        | ทำงานตอนไหน                                                               |
| ----------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| `onLoad(entity, id, state, propertyNames, types)`                             | ก่อน entity ถูก initialize จาก DB                                         |
| `onPersist(entity, id, state, propertyNames, types)`                          | ก่อน entity ใหม่กลายเป็น persistent (แทนที่ `onSave` เดิม)                |
| `onFlushDirty(entity, id, currentState, previousState, propertyNames, types)` | ตอน dirty checking เจอว่า entity เปลี่ยนระหว่าง flush                     |
| `onRemove(entity, id, state, propertyNames, types)`                           | ก่อน entity ถูกลบ (แทนที่ `onDelete` เดิม)                                |
| `preFlush(Iterator<Object> entities)` / `postFlush(...)`                      | ก่อน/หลัง flush ทั้งก้อน — เห็น**ทุก**entity ที่กำลังจะถูก flush พร้อมกัน |
| `beforeTransactionCompletion` / `afterTransactionCompletion`                  | ก่อน commit / หลัง commit-หรือ-rollback                                   |

**`onLoad`/`onPersist`/`onFlushDirty` คืน `boolean`** — `true` แปลว่า "ฉันแก้ `state`/`currentState` แล้ว เอาค่าใหม่ไปใช้" ถ้าแก้ array แล้วลืม return `true` **การแก้ไม่มีผลอะไรเลย**

### ตัวอย่างเต็ม — auto-set `updatedAt` ให้ทุก entity โดยไม่ต้องแก้ entity class สักตัว

```java
public interface Auditable {
    void setUpdatedAt(Instant instant);
}
```

```java
import org.hibernate.Interceptor;
import org.hibernate.type.Type;
import java.time.Instant;
import java.util.Arrays;

public class AuditInterceptor implements Interceptor {

    @Override
    public boolean onFlushDirty(Object entity, Object id, Object[] currentState,
                                 Object[] previousState, String[] propertyNames, Type[] types) {
        if (!(entity instanceof Auditable)) {
            return false;            // entity นี้ไม่เกี่ยวข้อง ไม่แตะอะไร
        }
        int idx = Arrays.asList(propertyNames).indexOf("updatedAt");
        if (idx < 0) return false;

        currentState[idx] = Instant.now();  // แก้ state ก่อนที่ SQL UPDATE จะถูกสร้าง
        return true;        // ต้อง return true ไม่งั้นค่าที่แก้ไม่มีผล
    }
}
```

**จุดสำคัญ:** `onFlushDirty` ทำงานเฉพาะตอน **update** ที่มีการเปลี่ยนแปลงจริง ถ้าต้องการ `createdAt` ตอน insert ด้วย ต้อง override `onPersist` แยกอีก method หนึ่ง (คนละจังหวะกัน)

---

## 3. ลงทะเบียนใน Quarkus — `@PersistenceUnitExtension`

```java
import io.quarkus.hibernate.orm.PersistenceUnitExtension;

@PersistenceUnitExtension                          // ← แปะเพิ่มบน AuditInterceptor จากข้อ 2 บรรทัดเดียวจบ
public class AuditInterceptor implements Interceptor {
    // ... onFlushDirty ตามข้อ 2
}
```

แค่แปะ `@PersistenceUnitExtension` บน bean ที่ implement `Interceptor` — Quarkus จัดการลงทะเบียนให้ persistence unit เอง ไม่ต้องเขียน config อะไรเพิ่ม

- **ไม่ระบุ CDI scope** → ทำงานเหมือน `@Dependent`, **instance เดียวถูกสร้างต่อ persistence unit หนึ่งตัว** (ไม่ใช่ต่อ request/session) — instance นี้เลยถูกแชร์ข้ามหลาย session/thread พร้อมกัน (ข้อ 6)
- **มีหลาย persistence unit** → ระบุชื่อได้: `@PersistenceUnitExtension("nameOfYourPU")`
- `@PersistenceUnitExtension` ใช้ได้กับ extension point อื่นของ Hibernate ด้วย ไม่ใช่แค่ `Interceptor`: `StatementInspector` (ดักดู/แก้ SQL ดิบก่อนส่งไป DB — ดี logging), `FormatMapper`, `TenantResolver`/`TenantConnectionResolver` (multi-tenancy), `FunctionContributor`, `TypeContributor`

---

## 4. เทียบกับ JPA `@PrePersist`/`@EntityListeners` — ทำงานเดียวกัน คนละวิธี

```java
@Entity
@EntityListeners(AuditListener.class)      // ← ต้องแปะทุก entity ที่ต้องการ
public class Product implements Auditable {
    private Instant updatedAt;
    // getter/setter...
}

public class AuditListener {
    @PreUpdate
    public void onPreUpdate(Object entity) {
        if (entity instanceof Auditable a) {
            a.setUpdatedAt(Instant.now());
        }
    }
}
```

**ต่างกันตรงจุดเดียวแต่สำคัญมาก:** `@EntityListeners` ต้องแปะ**ทุก entity** ที่ต้องการ behavior นี้ (หรือใช้ `@MappedSuperclass` ช่วยลดงาน) ส่วน `Interceptor` ลงทะเบียน**ครั้งเดียว** แล้วเห็นทุก entity อัตโนมัติผ่าน `instanceof` เช็คเอง ไม่ต้องแก้ entity class เลยสักตัว — ถ้า entity ในระบบมีเยอะและอยากได้ behavior เดียวกันทุกตัว `Interceptor` เขียนน้อยกว่ามาก

---

## 5. ทำไม Quarkus ถึงแนะนำ Interceptor มากกว่า EventListener SPI

Hibernate เองแนะนำ `EventListener` SPI (`PreInsertEventListener`, `PreUpdateEventListener` ฯลฯ) เป็นทางเลือกที่ทันสมัยกว่า `Interceptor` สำหรับเคสซับซ้อน เพราะแยก listener ทีละ event type ชัดเจนกว่า — **แต่ใน Quarkus โดยเฉพาะ `@PersistenceUnitExtension` (ข้อ 3) รองรับแค่ `Interceptor`, `StatementInspector`, `FormatMapper`, `TenantResolver`/`TenantConnectionResolver`, `FunctionContributor`, `TypeContributor` เท่านั้น — ไม่มี raw `EventListener` interface อยู่ในรายการ** การลงทะเบียน `EventListener` เองต้องพึ่ง Hibernate `Integrator` SPI แบบ low-level ที่ Quarkus ไม่ได้ wire ให้อัตโนมัติเหมือน `@PersistenceUnitExtension` ในทางปฏิบัติ **`Interceptor` จึงเป็นทางเลือกที่เข้ากับ Quarkus ได้ลื่นไหลที่สุด** แม้ในโลก Hibernate ทั่วไปจะถือว่าเป็นของเก่ากว่าก็ตาม

---

## 🎯 แนวทางการเริ่มศึกษา

1. **เริ่มต้น** — เข้าใจก่อนว่ามี 3 ทางเลือก (ข้อ 1) ไม่ใช่มีแค่ Interceptor และ Hibernate `Interceptor` ต่างจาก CDI `@Interceptor` ที่ชื่อคล้ายกันโดยสิ้นเชิง
2. **ลงมือทำจริง** — เขียน `AuditInterceptor` ตามตัวอย่างข้อ 2 ลงทะเบียนด้วย `@PersistenceUnitExtension` แล้วลองแก้ entity ที่ implement `Auditable` ดูว่า `updatedAt` ถูกเซ็ตอัตโนมัติจริงไหมโดยไม่ต้องแตะ entity class เลย
3. **ใช้งานได้คล่อง** — รู้ว่า `onFlushDirty` ไม่ทำงานกับ insert (ต้องคู่กับ `onPersist`) และไม่ทำงานกับ bulk update/native SQL เลย (ข้อ 6), เข้าใจว่า instance ถูกแชร์ข้าม session ต้องไม่เก็บ mutable state, และรู้ว่าเมื่อไหร่ควรใช้ JPA callback แทน (ข้อ 4) แทนที่จะใช้ Interceptor กับทุกกรณี

---

## 6. กับดัก

- **สับสนกับ [[Quarkus Interceptor]] (CDI)** — คนละ interface คนละงานกันเลยแม้ชื่อเหมือนกัน (ดูตารางเปิดโน้ต)
- **แก้ `currentState`/`state` แล้วลืม `return true`** — การแก้ไม่มีผลอะไรเลย ค่าเดิมถูกใช้ต่อเหมือนไม่มีอะไรเกิดขึ้น
- **คาดหวังให้ `onFlushDirty` ทำงานตอน insert** — มันทำงานเฉพาะ **update ที่มีการเปลี่ยนแปลงจริง** เท่านั้น ต้อง override `onPersist` แยกสำหรับตอนสร้างใหม่
- **Interceptor ไม่ทำงานกับ bulk update/native SQL** — `UPDATE ... SET` แบบ JPQL/HQL หรือ native SQL ไม่ผ่าน dirty-checking lifecycle ของ session เลย ไม่ trigger interceptor (เช่นเดียวกับที่ [[Quarkus EntityManager]] ข้อ 13 พูดถึงเรื่อง bulk operation ไว้)
- **เก็บ mutable state เป็น field ของ Interceptor** — instance ถูกแชร์ข้ามหลาย session/thread โดย default (ข้อ 3) ถ้าเก็บ state ที่เปลี่ยนได้ไว้ใน field จะพังตอนมีหลาย request พร้อมกัน ต้องเขียนให้ stateless
- **ลืมว่า `Interceptor` เห็นแค่ entity ที่ผ่าน Hibernate session** — ข้อมูลที่เข้ามาทาง JDBC ดิบหรือ connection แยกนอก Hibernate จะไม่ผ่าน interceptor เลย

---

## Cheat sheet

```java
@PersistenceUnitExtension
public class MyInterceptor implements Interceptor {

    @Override
    public boolean onFlushDirty(Object entity, Object id, Object[] currentState,
            Object[] previousState, String[] propertyNames, Type[] types) {
        // แก้ currentState[i] แล้ว return true
        return false;
    }

    @Override
    public boolean onPersist(Object entity, Object id, Object[] state,
            String[] propertyNames, Type[] types) {
        // ใช้ตอน insert ใหม่ (onFlushDirty ไม่ทำงานตอนนี้)
        return false;
    }
}
```

| อาการ | สาเหตุ |
|---|---|
| แก้ `currentState` แล้วไม่มีผล | ลืม `return true` |
| ไม่ทำงานตอน insert ใหม่ | ต้องดักที่ `onPersist` แยก ไม่ใช่ `onFlushDirty` |
| ไม่ทำงานกับ bulk update/native SQL | ไม่ผ่าน dirty-checking lifecycle เลย |
| พังตอนมีหลาย request พร้อมกัน | instance เดียวแชร์ข้าม session/thread โดย default ต้อง stateless |
| สับสนกับ [[Quarkus Interceptor]] | คนละ interface กันเลย — ดูตารางเปรียบเทียบข้อ 1 |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus Interceptor]] — CDI method interceptor คนละตัวกับโน้ตนี้โดยสิ้นเชิง (disambiguation)
- [[Quarkus Hibernate]] — ภาพรวม Hibernate ORM ใน Quarkus, type mapping 5→6
- [[Quarkus EntityManager]] — bulk operation ที่ไม่ผ่าน interceptor (ข้อ 13), entity lifecycle state
- [[Java Date Time]] — `Instant`/timestamp ที่มักใช้ใน audit field

## 📖 อ่านต่อ

- [Hibernate — Interceptor (Javadoc 6.6)](https://docs.hibernate.org/orm/6.6/javadocs/org/hibernate/Interceptor.html)
- [Quarkus — Using Hibernate ORM and Jakarta Persistence](https://quarkus.io/guides/hibernate-orm/)
- [Hibernate — Interceptors and events (User Guide)](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#events)
