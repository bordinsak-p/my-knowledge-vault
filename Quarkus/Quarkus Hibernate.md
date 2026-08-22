---
tags:
  - java
  - quarkus
  - hibernate
  - jpa
type: reference
created: 2026-08-18
---

# 🗃️ Quarkus + Hibernate ORM

โน้ตอ่านทบทวน — เน้นเรื่องที่กัดจริงตอนใช้งาน ไม่ใช่ tutorial สอนตั้งแต่ศูนย์

> **Quarkus 3.x ใช้ Hibernate ORM 6+** (Quarkus 2 ใช้ Hibernate 5)
> เลข version สำคัญมากในเรื่องนี้ เพราะ Hibernate 5→6 คือการรื้อระบบ type mapping ใหม่ทั้งชุด

---

## 1. Hibernate 5 → 6: การเปลี่ยนที่กัดจริง

ไม่ใช่แค่เลขเวอร์ชัน — **ชนิดข้อมูลหลายตัวถูก map ลง DB คนละแบบกับเดิม**
ที่อันตรายคือมันไม่ error ตอน build แต่ไปพังตอน runtime หรือแย่กว่านั้นคือ **ข้อมูลเพี้ยนเงียบ ๆ**

| ชนิด | Hibernate 5 | Hibernate 6 |
|---|---|---|
| `UUID` | `Types.BINARY` | `SqlTypes.UUID` (ใช้ native type ถ้า DB รองรับ) |
| `Instant` | `TIMESTAMP` | `TIMESTAMP_UTC` |
| `ZonedDateTime` / `OffsetDateTime` | `TIMESTAMP` | `TIMESTAMP_WITH_TIMEZONE` + **normalize เป็น UTC** |
| `Duration` | `BIGINT` | `INTERVAL_SECOND` |
| `enum` | `TINYINT` เหมือนกันหมด | native enum หรือ numeric แล้วแต่ DB |

**อีกสองอย่างที่เปลี่ยนแบบเงียบ ๆ:**

- แต่ละ entity hierarchy ได้ sequence เป็นของตัวเอง ไม่แชร์ `hibernate_sequence` อีกแล้ว
- **default sequence increment เปลี่ยนจาก 1 เป็น 50** ← ถ้าไม่รู้แล้วไปดู ID ใน DB จะงงว่าทำไมกระโดด

**annotation ที่วางผิดที่ ตอนนี้ throw exception แล้ว** จากเดิมที่ถูกเมินเฉย ๆ
เช่น ใส่ `@Basic` กับ `@ManyToOne` บน field เดียวกัน หรือใส่ attribute converter บน `@Id` — ของพวกนี้ไม่เคยถูกรองรับมาแต่ไหนแต่ไร แต่เมื่อก่อนมันแค่เงียบ

### `@Temporal` ไม่ต้องใช้แล้ว

`@Temporal(TemporalType.DATE)` มีไว้สำหรับ `java.util.Date` / `Calendar` ยุคเก่า
ถ้า field เป็น `LocalDate` / `LocalDateTime` / `Instant` อยู่แล้ว **ตัวชนิดมันบอกความละเอียดในตัวเอง** `@Temporal` เลยไม่มีความหมาย และ Hibernate 6 ไม่ยอมให้ใส่บน type พวกนี้

---

## 2. UUID เป็น primary key — เรื่องที่ต้องระวังตอนอัปเกรด

การเก็บ UUID ลง DB ที่ไม่มี native UUID type (เช่น MySQL/MariaDB รุ่นก่อน 10.7) นิยมเก็บเป็น `BINARY(16)` เพราะประหยัดกว่า `CHAR(36)` มากกว่าครึ่ง

**ปัญหาคือ Hibernate 6 เปลี่ยน default** จาก `Types.BINARY` เป็น `SqlTypes.UUID` — ถ้าไม่ได้ระบุ type ไว้ชัดเจน มันจะพยายามใช้ native type และ **byte order ก็ไม่จำเป็นต้องตรงกับของเดิม** ผลคืออ่านข้อมูลเก่าไม่ออก

วิธีบังคับให้ชัด เลือกอย่างใดอย่างหนึ่ง:

```java
// ระบุที่คอลัมน์
@Column(columnDefinition = "BINARY(16)")
private UUID id;

// หรือระบุที่ JDBC type ตรง ๆ (ชัดเจนกว่า พกพาข้าม DB ได้ดีกว่า)
@JdbcTypeCode(SqlTypes.BINARY)
private UUID id;
```

**อย่าเผลอลบออกเวลา refactor** — โค้ดยังคอมไพล์ผ่าน เทสที่ใช้ DB เปล่าก็ผ่าน แต่ข้อมูลเดิมอ่านไม่ออก

### เรื่องการ generate UUID

```java
@Id
@GeneratedValue
@UuidGenerator          // Hibernate 6
private UUID id;
```

**`@GenericGenerator(strategy = "...")` แบบส่ง string deprecated ใน Hibernate 6 และถูกถอดออกใน Hibernate 7** — โค้ดเก่าที่เขียนแบบนั้นต้องแก้ก่อนอัปเกรด

---

## 3. Transaction, Session, และ persistence context

- `@Transactional` ครอบ method → เปิด transaction + persistence context
- entity ที่ `find()` มาในนั้นเป็น **managed** — แก้ field แล้ว **ไม่ต้องเรียก `persist()`** Hibernate flush ให้ตอน commit เอง (dirty checking)
- ออกนอก transaction แล้ว entity กลายเป็น **detached** — แตะ lazy field ตอนนั้น = `LazyInitializationException`

**กับดักที่เจอบ่อยที่สุด:** return entity ตรง ๆ ออกจาก REST endpoint แล้วให้ Jackson serialize
Jackson จะไล่แตะทุก getter รวมถึง lazy association → พังบ้าง หรือลาก DB ทั้งก้อนมาบ้าง

→ **นี่คือเหตุผลที่ควรแยก DTO ออกจาก entity** ไม่ใช่แค่เรื่องความสวยงามของ layering แต่เป็นเรื่องพฤติกรรมจริงตอน runtime

### `getSingleResult()` โยน exception ตลอด

`getSingleResult()` throw `NoResultException` เมื่อไม่เจอ และ `NonUniqueResultException` เมื่อเจอเกินหนึ่ง
การใช้ exception เป็น control flow ในเคสที่ "ไม่เจอ" เป็นเรื่องปกติ มันแพงและอ่านยาก

```java
// แทนที่จะ try/catch
return query.getResultStream().findFirst().orElse(null);
```

`getResultStream().findFirst()` อ่านง่ายกว่า และไม่จ่ายค่า stack trace ฟรี ๆ

---

## 4. N+1 — ปัญหาที่แพงที่สุดและมองไม่เห็น

โหลด parent 100 rows แล้ววน loop แตะ child ของแต่ละตัว = **101 queries**
ตอน dev ข้อมูลน้อยไม่รู้สึกอะไร ขึ้น prod แล้วช้าแบบหาสาเหตุไม่เจอ

**วิธีตรวจ**

- `quarkus.hibernate-orm.log.sql=true` แล้วดูว่ามี SQL หน้าตาเหมือนกันซ้ำ ๆ ไหม
- Hibernate Statistics — นับ query จริงต่อ request แล้วตั้ง threshold ใน test ได้ (แม่นกว่าการนั่งไล่อ่าน log)

**วิธีแก้ เรียงตามลำดับที่ควรลอง**

| วิธี | ใช้เมื่อ |
|---|---|
| `JOIN FETCH` ใน JPQL | รู้ล่วงหน้าว่า query นี้ต้องใช้ child แน่ ๆ — ยุบ 101 queries เหลือ 1 |
| `@EntityGraph` | อยาก reuse entity เดิมแต่คุม fetch ต่าง ๆ กันในแต่ละ use case |
| `@BatchSize(size = 25)` | แก้แบบกว้าง ๆ — จาก 101 queries เหลือ ~5 |
| `fetch.batch-size` (global) | Quarkus default = **16** อยู่แล้ว |

**อย่าแก้ด้วยการเปลี่ยน association เป็น `EAGER`** — มันย้ายปัญหาไปที่อื่น แล้วทำให้ทุก query ที่แตะ entity นั้นแบกน้ำหนักเพิ่มโดยไม่จำเป็น

---

## 5. Bulk insert / update

Hibernate ไม่ได้ออกแบบมาสำหรับ insert หมื่นแถวรวด ปัญหาคือ:

1. **persistence context บวม** — ทุก entity ที่ `persist()` ถูกจำไว้หมด → memory โต และ dirty checking ช้าลงเรื่อย ๆ แบบ O(n²)
2. **ไม่ batch ถ้าไม่สั่ง** — ต้องตั้ง `quarkus.hibernate-orm.jdbc.statement-batch-size` เอง ไม่งั้นยิงทีละ statement

```java
for (int i = 0; i < rows.size(); i++) {
    em.persist(toEntity(rows.get(i)));
    if (i % 50 == 0) {
        em.flush();
        em.clear();   // ← ตัวนี้แหละที่สำคัญ
    }
}
```

`flush()` ส่ง SQL ออกไป, `clear()` ทิ้ง entity ออกจาก persistence context
**ขาด `clear()` แล้ว memory จะบวมจนตาย** แม้จะ flush ถี่แค่ไหนก็ตาม

> **ถ้าข้อมูลเยอะจริง ๆ ให้พิจารณาข้าม JPA ไปใช้ JDBC ตรง ๆ** — งาน import ที่ไม่ต้องการ dirty checking, cascade หรือ lifecycle callback ไม่ได้ประโยชน์อะไรจาก ORM มีแต่จ่ายค่าโสหุ้ย

### bulk update/delete ข้าม persistence context

```java
em.createQuery("update Product p set p.status = :s where p.releasedOn < :d")
```

statement แบบนี้ยิงตรงไปที่ DB **entity ที่โหลดไว้ก่อนหน้าจะยังเป็นค่าเก่า** ถ้าจะใช้ต่อในทรานแซกชันเดียวกันต้อง `em.clear()` ก่อน

---

## 6. Multitenancy

Hibernate รองรับหลายแบบ ตั้งผ่าน `quarkus.hibernate-orm.multitenant`

| โหมด | แยกข้อมูลด้วย |
|---|---|
| `DATABASE` | คนละ datasource / คนละ DB |
| `SCHEMA` | DB เดียวกัน คนละ schema |
| `DISCRIMINATOR` | ตารางเดียวกัน มีคอลัมน์บอกว่าเป็นของ tenant ไหน |

**สิ่งที่ต้องจำ:** การ resolve tenant มักผูกกับ **request context** (อ่านจาก header หรือ token)
งานที่รันใน background thread **ไม่มี request context** → routing ไม่ทำงาน
ต้องส่ง tenant id เข้าไปให้ worker อย่างชัดเจน และมักต้องมี datasource/connection pool ของฝั่ง background แยกต่างหาก

---

## 7. Native image

- entity ต้องการ reflection → Quarkus ลงทะเบียนให้อัตโนมัติจาก `quarkus.hibernate-orm.packages`
- **entity ที่อยู่นอก package ที่ประกาศไว้จะเงียบหาย** ไม่ error ตอน build แต่ไม่ถูก map
- query ที่สร้าง SQL ด้วยการต่อ string ตอน runtime ยังทำงานได้ แต่เสียโอกาส optimize ตอน build

ดูเรื่อง native เพิ่มที่ [[Quarkus Build]]

---

## 8. Config ที่ควรรู้

```properties
# dev เท่านั้น — เปิดบน prod = overhead ฟรีและ log บวม
%dev.quarkus.hibernate-orm.log.sql=true
%dev.quarkus.hibernate-orm.log.format-sql=true

# bulk insert
quarkus.hibernate-orm.jdbc.statement-batch-size=50

# กันพลาด N+1 (default 16 อยู่แล้ว)
quarkus.hibernate-orm.fetch.batch-size=16

# ปล่อยให้ Hibernate ตรวจเวอร์ชัน DB เอง
# อย่าระบุ dialect ที่ผูกกับเวอร์ชัน (เช่น MariaDB103Dialect) — deprecated ใน Hibernate 6
quarkus.hibernate-orm.dialect=org.hibernate.dialect.MariaDBDialect

# prod ไม่ควรให้ Hibernate ไปแตะ schema
quarkus.hibernate-orm.database.generation=none
```

---

## 9. Cheat sheet

| อาการ | สาเหตุที่น่าจะเป็น |
|---|---|
| `LazyInitializationException` | แตะ lazy field นอก transaction — มัก serialize entity ตรง ๆ |
| SQL เดิมซ้ำ ๆ ใน log | N+1 → `JOIN FETCH` |
| import ช้าลงเรื่อย ๆ ตามจำนวนแถว | ไม่ได้ `em.clear()` |
| ID ใน DB กระโดดทีละ 50 | Hibernate 6 เปลี่ยน default sequence increment |
| อ่าน UUID เก่าไม่ออกหลังอัปเกรด | ไม่ได้บังคับ JDBC type ไว้ |
| เวลาเพี้ยนไปตาม timezone หลังอัปเกรด | Hibernate 6 normalize `Instant`/`ZonedDateTime` เป็น UTC |
| แก้ค่าแล้วไม่บันทึก | อยู่นอก `@Transactional` |
| entity ค่าเก่าหลัง bulk update | bulk statement ข้าม persistence context |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus]] — หน้ารวม
- [[Quarkus Panache]] — ชั้นที่วางทับ Hibernate อีกที
- [[Quarkus Build]] — native image

## 📖 อ่านต่อ

- [Quarkus — Using Hibernate ORM and Jakarta Persistence](https://quarkus.io/guides/hibernate-orm)
- [Migration Guide 3.0: Hibernate ORM 5 to 6](https://github.com/quarkusio/quarkus/wiki/Migration-Guide-3.0:-Hibernate-ORM-5-to-6-migration)
- [Hibernate Statistics in Quarkus: Catch N+1 Before Production](https://www.the-main-thread.com/p/hibernate-statistics-quarkus-n-plus-one-detection-java)
