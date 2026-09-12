---
tags:
  - java
  - quarkus
  - flyway
  - database
  - migration
type: reference
created: 2026-08-18
---

# 🐦 Quarkus + Flyway

> **Flyway แก้ปัญหา "schema DB ของแต่ละคนไม่ตรงกัน"**
> แทนที่จะบอกทีมว่า "อย่าลืมรัน `alter table` ตัวนี้นะ" schema ทั้งหมดถูกเขียนเป็นไฟล์ SQL เรียงเลขเวอร์ชัน แล้ว Flyway เป็นคนไล่รันให้ตามลำดับ — เครื่องไหนก็ตาม สภาพแวดล้อมไหนก็ตาม จบที่ schema เดียวกันเป๊ะ

---

## 1. แนวคิดหลัก — schema history table

Flyway สร้างตารางพิเศษชื่อ `flyway_schema_history` เก็บว่า **migration ตัวไหนรันไปแล้วบ้าง**

```
flyway_schema_history
┌─────────────┬──────────┬───────────────────┬──────────┬─────────────────────┐
│ installed_rank │ version │ description     │ checksum │ installed_on        │
├─────────────┼──────────┼───────────────────┼──────────┼─────────────────────┤
│ 1           │ 1       │ create product     │ 8f3a2c1  │ 2026-01-10 09:00:00 │
│ 2           │ 2       │ add category       │ 1b9e7f4  │ 2026-02-15 14:20:00 │
│ 3           │ 3       │ add order table    │ ac02de9  │ 2026-08-18 10:05:00 │
└─────────────┴──────────┴───────────────────┴──────────┴─────────────────────┘
```

ทุกครั้งที่แอปสตาร์ต Flyway จะ:

```
1. อ่านตาราง flyway_schema_history → รู้ว่ารันไปถึงเวอร์ชันไหนแล้ว
2. สแกนโฟลเดอร์ migration → หาไฟล์ที่ "ยังไม่เคยรัน"
3. เรียงตามเลขเวอร์ชัน แล้วรันทีละไฟล์
4. รันเสร็จแต่ละไฟล์ → บันทึกลง history table พร้อม checksum
```

**checksum คือกลไกป้องกันที่สำคัญที่สุด** — ถ้าไฟล์ที่เคยรันไปแล้วถูกแก้ไขทีหลัง (แม้แค่ช่องว่างตัวเดียว) checksum จะไม่ตรงกับที่บันทึกไว้ **Flyway จะปฏิเสธและไม่ยอมสตาร์ตแอป** เพื่อป้องกันไม่ให้แต่ละเครื่องมี schema history คนละความจริงกัน

---

## 2. ติดตั้ง

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-flyway</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-jdbc-mariadb</artifactId>   <!-- ตามยี่ห้อ DB จริง -->
</dependency>
```

โครงสร้างไฟล์ default

```
src/main/resources/
└── db/
    └── migration/
        ├── V1__create_product.sql
        ├── V2__add_category.sql
        └── V3__add_order_table.sql
```

```properties
quarkus.flyway.migrate-at-start=true
```

**`migrate-at-start=false` เป็นค่าเริ่มต้น** — ต้องเปิดเองเสมอ ไม่งั้น Flyway โหลดขึ้นมาเฉย ๆ แต่ไม่ทำอะไร

---

## 3. ชื่อไฟล์ — สูตรตายตัวที่ต้องจำ

```
V <เวอร์ชัน> __ <คำอธิบาย> .sql
│    │        │      │
│    │        │      └── ใช้ _ คั่นคำ จะกลายเป็นช่องว่างตอนแสดงผล
│    │        └── ต้องเป็น __ (สอง underscore) เท่านั้น
│    └── ตัวเลข จะเรียง 1, 2, 10 ไม่ใช่ 1, 10, 2 (numeric ไม่ใช่ string)
└── ตัวพิมพ์ใหญ่ V = versioned migration
```

```
V1__create_product_table.sql
V2__add_category_column.sql
V2_1__add_index_on_category.sql      ← version ย่อยได้ (2.1)
V20260818__add_order_table.sql       ← ใช้วันที่แทนเลขไล่ก็ได้ ถ้าทีมใหญ่
```

### สาม prefix ที่มีความหมายต่างกัน

| Prefix | เรียกว่า | รันกี่ครั้ง | ตัวอย่าง |
|---|---|---|---|
| **`V`** | Versioned | **ครั้งเดียว** แล้วห้ามแก้อีก | `V1__create_table.sql` |
| **`R`** | Repeatable | **รันใหม่ทุกครั้งที่ checksum เปลี่ยน** | `R__product_view.sql` |
| `U` | Undo (มีใน Flyway Teams เท่านั้น) | rollback ของ V | `U1__create_table.sql` |

**`V` ห้ามแก้หลัง commit เด็ดขาด** — ถ้าอยากเปลี่ยนอะไรที่ทำไปแล้ว ให้เขียน `V` ตัวใหม่มาแก้ต่อ ไม่ใช่ไปแก้ไฟล์เก่า (เหตุผลเรื่อง checksum ในข้อ 1)

**`R` เหมาะกับ view, stored procedure, function** ที่อยากให้นิยามล่าสุดสะท้อนอยู่เสมอ — ไม่ต้องไล่เขียน `V` ใหม่ทุกครั้งที่แก้ view เดิม

```sql
-- R__product_summary_view.sql
CREATE OR REPLACE VIEW product_summary AS
SELECT p.code, p.name, c.name AS category_name
FROM product p JOIN category c ON c.id = p.category_id;
```

> ⚠️ **`R` มีความเสี่ยงที่มองไม่เห็น** — แก้ view นิดเดียว checksum เปลี่ยน Flyway รันใหม่ทันทีตอน deploy ครั้งถัดไป ถ้ามี stored procedure อื่นพึ่งพา view ตัวนี้อยู่ อาจพังแบบไม่มีใครคาดคิด **ปฏิบัติกับ `R` เหมือนโค้ดจริง ต้อง review ให้ละเอียดเท่า `V`**

---

## 4. เขียน migration แรก

```sql
-- V1__create_product_table.sql
CREATE TABLE PRODUCT (
    PRODUCT_REF   BINARY(16)     NOT NULL,
    CODE          VARCHAR(32)    NOT NULL,
    NAME          VARCHAR(200)   NOT NULL,
    PRICE         DECIMAL(15,2)  NOT NULL DEFAULT 0,
    CREATE_DATE   DATETIME       NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (PRODUCT_REF)
);

CREATE UNIQUE INDEX UX_PRODUCT_CODE ON PRODUCT (CODE);
```

```sql
-- V2__add_category_column.sql
ALTER TABLE PRODUCT ADD COLUMN CATEGORY_REF BINARY(16);

ALTER TABLE PRODUCT
    ADD CONSTRAINT FK_PRODUCT_CATEGORY
    FOREIGN KEY (CATEGORY_REF) REFERENCES CATEGORY(CATEGORY_REF);
```

**หลักการ: หนึ่งไฟล์ = หนึ่งการเปลี่ยนแปลงที่มีความหมายในตัวเอง** ไม่ต้องยัดทุกอย่างไว้ไฟล์เดียว และไม่ต้องแตกละเอียดเกินไปจนตามยาก — ระดับที่พอดีคือ "หนึ่ง feature หนึ่งไฟล์"

---

## 5. Config ที่ใช้บ่อย

```properties
quarkus.flyway.migrate-at-start=true
quarkus.flyway.locations=classpath:db/migration
quarkus.flyway.table=flyway_schema_history
quarkus.flyway.baseline-on-migrate=false
quarkus.flyway.baseline-version=1
quarkus.flyway.validate-at-start=true
quarkus.flyway.clean-disabled=true
```

| property | ทำอะไร |
|---|---|
| `migrate-at-start` | รัน migration อัตโนมัติตอนแอป start |
| `locations` | โฟลเดอร์ที่เก็บไฟล์ migration (ใส่ได้หลายที่ คั่นด้วยคอมมา) |
| `baseline-on-migrate` | ใช้ตอนมี DB เดิมอยู่แล้วที่ไม่เคยผ่าน Flyway มาก่อน (ข้อ 7) |
| `baseline-version` | เวอร์ชันเริ่มต้นที่ถือว่า "มีอยู่แล้ว" ไม่ต้องรันซ้ำ |
| `validate-at-start` | เช็ค checksum ก่อน migrate — **ควรเปิดเสมอ** |
| `clean-disabled` | ปิดคำสั่ง `clean` (ลบทุกตาราง) — **ต้องเปิดใน prod เสมอ** |
| `sql-migration-prefix` | เปลี่ยน prefix จาก `V` เป็นอย่างอื่น (ไม่ค่อยมีเหตุผลให้เปลี่ยน) |
| `placeholders.*` | แทนค่าตัวแปรใน SQL เช่น `${schema}` |

### ⚠️ `clean-disabled=true` ต้องเปิดบน prod เสมอ

```properties
%prod.quarkus.flyway.clean-disabled=true
```

`flyway.clean()` **ลบทุกตารางในทุก schema ที่ config ไว้** ทำเผลอบน prod หนึ่งครั้งคือข้อมูลหายทั้งฐาน — ใน Quarkus เวอร์ชันใหม่ `clean-disabled` เป็น `true` โดย default อยู่แล้ว แต่ **ควรใส่ให้เห็นชัดเจนใน config เอง** ไม่ใช่หวังพึ่ง default เงียบ ๆ

---

## 6. Multi-datasource / Multi-tenant

```properties
quarkus.flyway.migrate-at-start=true
quarkus.flyway.locations=db/migration/default

quarkus.flyway.framework.migrate-at-start=true
quarkus.flyway.framework.locations=db/migration/framework
```

Flyway ผูกกับชื่อ datasource เดียวกับที่ตั้งไว้ใน `quarkus.datasource.<name>.*` — แต่ละ datasource มีชุด migration และ history table เป็นของตัวเอง

```java
@Inject @FlywayDataSource("framework") Flyway flywayFramework;
```

> ระบบที่มี multitenant แบบ `DATABASE` (คนละ DB ต่อ tenant) ต้องคิดเพิ่มว่า migration ใหม่จะ apply ให้ทุก tenant ยังไง — Flyway extension มาตรฐานรัน migration ให้แค่ datasource ที่ config ไว้ตอน build ไม่รู้จัก tenant ที่เพิ่มเข้ามาทีหลังอัตโนมัติ ต้องมีกลไกแยกไปรันซ้ำทุก tenant เอง

---

## 7. Baseline — เมื่อมี DB เดิมอยู่แล้ว

**สถานการณ์:** ระบบมีอยู่แล้ว มีข้อมูลอยู่แล้ว แต่ไม่เคยใช้ Flyway มาก่อน จะเริ่มใช้ยังไงไม่ให้พังของเดิม

```properties
quarkus.flyway.baseline-on-migrate=true
quarkus.flyway.baseline-version=1
quarkus.flyway.baseline-description=existing schema
```

```
DB เดิม (มีตารางอยู่แล้ว 50 ตาราง ไม่มี flyway_schema_history)
         │
         │ baseline-on-migrate=true
         ▼
Flyway สร้าง flyway_schema_history แล้วใส่ baseline record
"เวอร์ชัน 1 = สภาพปัจจุบัน ถือว่าผ่านแล้ว ไม่ต้องรันอะไรเพิ่ม"
         │
         ▼
migration ที่เขียนใหม่ (V2 ขึ้นไป) ถึงจะเริ่มรันจากตรงนี้
```

**สำคัญมาก:** `baseline-on-migrate=true` ควรเปิดแค่ตอนเริ่มใช้ Flyway ครั้งแรกเท่านั้น จากนั้นควรปิดกลับ ไม่งั้นถ้ามีใครลบ history table โดยไม่ตั้งใจ Flyway จะ baseline ใหม่เงียบ ๆ แทนที่จะ error เตือน

---

## 8. Callback — เชื่อม lifecycle ของ Flyway

Flyway มีจุดเกาะให้แทรก logic ก่อน/หลังแต่ละขั้นตอน

```java
@ApplicationScoped
public class PreMigrateSafetyCheck implements Callback {

    @Override
    public boolean supports(Event event, Context context) {
        return event == Event.BEFORE_MIGRATE;
    }

    @Override
    public boolean canHandleInTransaction(Event event, Context context) {
        return true;
    }

    @Override
    public void handle(Event event, Context context) {
        if (isProd() && !approvedForDestructiveChange()) {
            throw new IllegalStateException("ต้อง approve ก่อน migrate บน prod");
        }
    }

    @Override
    public String getCallbackName() { return "pre-migrate-safety-check"; }
}
```

```properties
quarkus.flyway.callbacks=org.acme.PreMigrateSafetyCheck
```

**Event ที่ใช้บ่อย:** `BEFORE_MIGRATE`, `AFTER_MIGRATE`, `AFTER_EACH_MIGRATE`, `BEFORE_EACH_MIGRATE`

**สามข้อที่ต้องรู้ก่อนเขียน callback**

- **ไม่การันตีลำดับ** ถ้ามีหลาย callback ฟัง event เดียวกัน ห้ามเขียนให้พึ่งพากันว่าอันไหนรันก่อนหลัง
- **ต้อง idempotent และเร็ว** เพราะถ้า callback throw exception **แอปจะสตาร์ตไม่ขึ้นเลย** — และนั่นแปลว่า health endpoint ก็ไม่มีวันตอบด้วย เพราะแอปตายตั้งแต่ก่อนเปิด HTTP ([[Quarkus Health Check]] ช่วยอะไรไม่ได้ตรงนี้)
- **ปฏิบัติเหมือนโค้ด production จริง** ไม่ใช่ script ชั่วคราว — ห้ามมี dynamic SQL หรือ query ที่ไม่จำกัดขอบเขต

---

## 9. ⚠️ ความเสี่ยงของ `migrate-at-start=true` บน prod

**migration ที่ fail ตอน startup = แอปไม่ขึ้นเลย ไม่ใช่แค่ endpoint นั้นพัง**

```
deploy version ใหม่
      │
      ▼
Flyway เริ่ม migrate ตอน startup
      │
      ├── สำเร็จ → แอป start ต่อปกติ
      │
      └── ล้มเหลว (เช่น timeout เพราะตารางใหญ่, constraint ชนกับข้อมูลเดิม)
              │
              ▼
      แอปทั้งตัวไม่ start — 502/503 ทันที ไม่มี health endpoint ให้เช็คด้วยซ้ำ
```

**สิ่งที่ควรทำก่อน migration ที่เสี่ยงจะรันบน prod**

1. **ทดสอบกับข้อมูลจริง (หรือก็อปปี้ของจริง) ก่อนเสมอ** ไม่ใช่แค่ DB ว่าง ๆ ตอน dev — index ที่สร้างเร็วบนตารางว่าง อาจใช้เวลาเป็นชั่วโมงบนตารางที่มีสิบล้านแถว
2. **แยก migration ที่ทำลายของเดิมออกจากที่แค่เพิ่มของใหม่** — `ADD COLUMN` ปลอดภัยกว่า `DROP COLUMN` มาก
3. **คอลัมน์ที่จะลบ ให้ deprecate ก่อนสองรอบ deploy** แทนที่จะลบทันที (ดูรูปแบบ expand-contract ข้อ 10)
4. **migration ที่ใช้เวลานาน ให้รันแยกออกจาก deploy pipeline** ไม่ใช่ผูกกับ `migrate-at-start` ที่บล็อกการ start ของทุก instance พร้อมกัน

---

## 10. Zero-downtime — รูปแบบ Expand / Contract

ปัญหา: เปลี่ยนชื่อคอลัมน์หรือย้ายข้อมูลไปโครงสร้างใหม่ ระหว่างที่ deploy หลาย instance พร้อมกัน โค้ดเวอร์ชันเก่ากับใหม่ต้องรันคู่กันได้ชั่วคราว ไม่งั้น instance เก่าจะพังทันทีที่ schema เปลี่ยน

```
Deploy 1 (Expand)     Deploy 2 (Migrate data)    Deploy 3 (Contract)
──────────────────    ─────────────────────      ────────────────────
เพิ่มคอลัมน์ใหม่        โค้ดใหม่เขียนคอลัมน์ใหม่      ลบคอลัมน์เก่า
คอลัมน์เก่ายังอยู่        คอลัมน์เก่ายังอยู่ (เผื่อ       (ทำเมื่อมั่นใจว่า
โค้ดเก่ายังใช้ได้         ต้อง rollback)               ไม่มีใครใช้เก่าแล้ว)
```

```sql
-- V10__expand_add_email_normalized.sql  (deploy พร้อมโค้ดที่ยังอ่าน/เขียนคอลัมน์เก่า)
ALTER TABLE customer ADD COLUMN email_normalized VARCHAR(255);
UPDATE customer SET email_normalized = LOWER(TRIM(email));
```

```sql
-- V11__contract_drop_old_email.sql  (deploy รอบถัดไป หลังมั่นใจว่าโค้ดทุก instance ใช้คอลัมน์ใหม่แล้ว)
ALTER TABLE customer DROP COLUMN email;
```

**อย่ารวมสองขั้นตอนนี้เป็น migration เดียว** ถ้า deploy หลาย instance พร้อมกัน (rolling deployment) instance เก่าที่ยังรันโค้ดเดิมอยู่จะพังทันทีที่ instance แรก migrate สำเร็จ

---

## 11. Test

```java
@QuarkusTest
class ProductMigrationTest {

    @Inject Flyway flyway;

    @Test
    void migrations_apply_cleanly() {
        var result = flyway.info().all();
        assertTrue(Arrays.stream(result).noneMatch(i -> i.getState().isFailed()));
    }
}
```

```properties
%test.quarkus.flyway.migrate-at-start=true
%test.quarkus.flyway.clean-at-start=true    # ✅ ปลอดภัยเฉพาะใน test เท่านั้น
```

**`clean-at-start=true` ใช้ได้เฉพาะ test profile** — ล้าง DB ให้สะอาดก่อนแต่ละรอบเทส เพื่อพิสูจน์ว่า migration รันจากศูนย์ได้จริง ไม่ใช่ผ่านเพราะบังเอิญมีข้อมูลเก่าค้างอยู่

**ทดสอบว่า migration ใหม่ใช้กับ "DB ที่มีข้อมูลจริงอยู่แล้ว" ได้จริง** — ปัญหาที่เจอบ่อยที่สุดคือ migration ทดสอบผ่านบน DB ว่าง แต่พังตอน prod เพราะมีข้อมูลที่ไม่เข้าเงื่อนไข constraint ใหม่

---

## 12. กับดัก

- **แก้ไฟล์ `V` ที่เคย commit ไปแล้ว** — checksum ไม่ตรง แอปสตาร์ตไม่ขึ้นทุกเครื่องที่เคยรันไฟล์เก่าไปแล้ว
- **`clean-disabled=false` หลุดไปถึง prod** — เผลอเรียก `clean()` ครั้งเดียวข้อมูลหายทั้งฐาน
- **migration ทำงานหนักผูกกับ `migrate-at-start`** — instance ทุกตัว start ไม่ได้พร้อมกันจนกว่าจะ migrate เสร็จ
- **`R__` migration แก้ view/procedure ที่มีคนพึ่งพาอยู่ โดยไม่ review** — checksum เปลี่ยนแล้วรันใหม่เงียบ ๆ ตอน deploy
- **ลบคอลัมน์ทันทีในรอบ deploy เดียวกับที่หยุดใช้** — instance เก่าที่ยัง rolling อยู่พังทันที (ข้อ 10)
- **ทดสอบ migration บน DB ว่างอย่างเดียว** — พังตอน prod เพราะมีข้อมูลจริงที่ไม่เข้าเงื่อนไขใหม่
- **callback throw exception แล้วงงว่าทำไม health check ไม่ทำงาน** — แอปตายตั้งแต่ก่อนเปิด HTTP ไม่มี endpoint ให้เช็คเลย
- **ไม่เปิด `validate-at-start`** — schema ที่เพี้ยนไปจากที่ Flyway คิดว่าเป็น จะไม่ถูกจับจนกว่าจะพังจริง

---

## 13. Cheat sheet

```properties
quarkus.flyway.migrate-at-start=true
quarkus.flyway.validate-at-start=true
quarkus.flyway.locations=classpath:db/migration
%prod.quarkus.flyway.clean-disabled=true
%test.quarkus.flyway.clean-at-start=true
```

```
V1__create_table.sql        ← รันครั้งเดียว ห้ามแก้หลัง commit
R__some_view.sql            ← รันใหม่ทุกครั้งที่ checksum เปลี่ยน
```

| อาการ | สาเหตุ |
|---|---|
| แอปสตาร์ตไม่ขึ้น บ่น checksum mismatch | มีคนแก้ไฟล์ `V` ที่เคยรันไปแล้ว |
| migration ใหม่ไม่รันเลย | ลืมตั้ง `migrate-at-start=true` |
| ข้อมูลหายทั้งฐานหลัง deploy | เผลอเปิด `clean-at-start` หรือเรียก `clean()` บน prod |
| deploy แล้ว instance ค้าง pending พร้อมกันหมด | migration หนักผูกกับ `migrate-at-start` |
| view พังหลัง deploy ทั้งที่ไม่ได้แก้อะไรที่เกี่ยวข้อง | มีคนแก้ `R__` migration ที่มีคนอื่นพึ่งพาอยู่ |
| rolling deployment พังครึ่งทาง | ลบ/เปลี่ยนโครงสร้างคอลัมน์ในรอบเดียวที่ยังมีโค้ดเก่าใช้อยู่ |
| health check ไม่ตอบเลย หา log ก็ไม่เจอ error ปกติ | callback ใน Flyway throw exception ทำให้แอปตายก่อนเปิด HTTP |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus Hibernate]] — `quarkus.hibernate-orm.database.generation=none` ควรคู่กับ Flyway เสมอ อย่าให้ Hibernate จัดการ schema เอง
- [[Quarkus Health Check]] — migration ที่ fail ทำให้ไม่มี health endpoint ให้เช็คด้วยซ้ำ
- [[Quarkus Build]] — build-time vs runtime config หลักการเดียวกับที่ทำให้บาง Flyway property ต้องตั้งถูกตั้งแต่ build
- [[Quarkus]] — หน้ารวม

## 📖 อ่านต่อ

- [Quarkus — Using Flyway](https://quarkus.io/guides/flyway)
- [Flyway — Migrations concept](https://documentation.red-gate.com/fd/migrations-184127470.html)
- [From SQL Scripts to a Migration Lifecycle — Flyway Callbacks in Quarkus](https://www.the-main-thread.com/p/flyway-callbacks-quarkus-production-migrations)
