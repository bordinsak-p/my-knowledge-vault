---
tags:
  - java
  - quarkus
  - hibernate
  - panache
  - jpa
type: reference
created: 2026-08-18
---

# 🐼 Quarkus Panache

> **Panache ไม่ใช่ ORM** — มันคือชั้นบาง ๆ ที่วางทับ Hibernate ORM อีกที
> ข้างล่างยังเป็น Hibernate ตัวเดิม กฎเรื่อง transaction / persistence context / N+1 ใน [[Quarkus Hibernate]] ยังใช้ได้ทั้งหมด **Panache ไม่ได้ทำให้ปัญหาพวกนั้นหายไป แค่ทำให้เขียนโค้ดสั้นลง**

ตัวอย่างโค้ดครบชุดอยู่ที่ [[Quarkus Panache Example]]

---

## 1. ปัญหาที่ Panache แก้

JPA มาตรฐานเขียนของซ้ำ ๆ เยอะมาก — DAO/Repository หนึ่งคลาสต่อหนึ่ง entity ที่ข้างในมีแต่ `em.find()`, `em.persist()`, `createQuery(...)` แบบเดียวกันหมด

```java
// JPA ปกติ
@ApplicationScoped
public class PersonRepository {
    @Inject EntityManager em;

    public List<Person> findByName(String name) {
        return em.createQuery("select p from Person p where p.name = :name", Person.class)
                 .setParameter("name", name)
                 .getResultList();
    }
}
```

```java
// Panache
Person.find("name", name).list();
```

---

## 2. สองรูปแบบการใช้ — เลือกให้ถูกตั้งแต่แรก

### Active Record — เอา query ไปไว้ใน entity

```java
@Entity
public class Person extends PanacheEntity {
    public String name;          // field public ได้เลย
    public LocalDate birth;

    public static List<Person> findByName(String name) {
        return find("name", name).list();
    }
}
```

`PanacheEntity` แถม `id` แบบ `Long` auto-generate ให้
ถ้าอยากคุม ID เอง (เช่นใช้ UUID) ใช้ **`PanacheEntityBase`** แล้วประกาศ `@Id` เอง

**field เป็น public แต่ไม่ได้แปลว่าไม่มี encapsulation** — Panache แปลง bytecode ตอน build ให้การอ่าน/เขียน field วิ่งผ่าน getter/setter จริง ๆ ถ้าวันหลังอยากใส่ logic ก็เขียน getter ทับได้โดยไม่ต้องแก้ที่เรียก

### Repository — แยก query ออกจาก entity

```java
@ApplicationScoped
public class PersonRepository implements PanacheRepository<Person> {
    public List<Person> findByName(String name) {
        return find("name", name).list();
    }
}
```

`PanacheRepositoryBase<Person, UUID>` สำหรับ ID ที่ไม่ใช่ `Long`

### เลือกอันไหน

| | Active Record | Repository |
|---|---|---|
| entity แก้ไม่ได้ / อยู่ใน jar อื่น | ❌ ต้อง extend class | ✅ **เป็น interface** ไม่ต้องแตะ entity |
| entity extend base class อื่นอยู่แล้ว | ❌ Java สืบทอดได้ทางเดียว | ✅ ไม่มีปัญหา |
| mock ใน test | ต้องใช้ `quarkus-panache-mock` (Mockito mock static ไม่ได้) | ✅ `quarkus-junit5-mockito` ปกติ |
| relation ซับซ้อนหลาย entity | เริ่มกลายเป็น anti-pattern — query ควรอยู่ที่ไหน? | ✅ ชัดกว่า |

**ความสามารถเท่ากันเป๊ะ** ต่างกันแค่ที่วางโค้ด — เลือกอันเดียวแล้วใช้ให้ทั้งทีมเหมือนกัน ปนกันเมื่อไหร่คนอ่านจะงงว่าต้องไปหา query ที่ไหน

> **ข้อจำกัดที่ตัดสินเรื่องนี้บ่อยที่สุด:** ถ้า entity ต้อง `extends` base class อยู่แล้ว (base class ที่มี audit column ร่วมกัน, base class จาก library) **active record ใช้ไม่ได้เลย** เพราะ Java สืบทอดได้ทางเดียว กรณีนี้เหลือทางเดียวคือ repository

---

## 3. Query syntax แบบย่อ

Panache เติมส่วนที่เดาได้ให้อัตโนมัติ

| เขียน | กลายเป็น |
|---|---|
| `find("name", x)` | `from Person where name = ?1` |
| `find("status = ?1 and age > ?2", s, a)` | `from Person where ...` |
| `find("order by name")` | `from Person order by name` |
| `find("select distinct p from Person p left join fetch p.orders")` | ใช้ JPQL เต็มได้ตามปกติ |

พารามิเตอร์ใช้ได้ทั้งแบบลำดับ `?1` และแบบชื่อ `:name` ผ่าน `Parameters.with("name", x)`

### Paging / Sorting / Projection

```java
// paging
PanacheQuery<Person> q = Person.find("status", ACTIVE);
List<Person> page = q.page(Page.ofSize(25)).list();
long total = q.count();

// sorting
Person.find("status", Sort.by("name").and("birth"), ACTIVE).list();

// projection ลง DTO/record — select เฉพาะคอลัมน์ที่ใช้
record PersonName(String name, LocalDate birth) {}
Person.find("status", ACTIVE).project(PersonName.class).list();
```

**`project()` มีค่ามากกว่าที่คิด** — มัน generate `select` เฉพาะคอลัมน์ที่ record ต้องการ ไม่ได้ลากทั้งแถวมาแล้วค่อยตัดทิ้ง ช่วยได้จริงกับตารางที่มีคอลัมน์เยอะ

---

## 4. กับดัก

- **ต้องมี `@Transactional`** ทุก method ที่เขียน DB ไม่งั้นเงียบ ๆ ไม่บันทึก
- `streamAll()` / `stream()` ต้องอยู่ใน transaction **และต้องปิดด้วย try-with-resources**
- ผสม `page()` กับ `range()` ในคิวรีเดียวไม่ได้
- entity ที่อยู่ใน jar ภายนอกต้องมี `META-INF/beans.xml` ไม่งั้น bytecode enhancement ไม่ทำงาน
- **query เป็น string ล้วน ไม่มีการเช็คตอน compile** — พิมพ์ `"where naem = ?1"` ผิด จะรู้ตอน runtime เท่านั้น
- generic hole: เขียน `var list = Person.listAll();` ไม่ได้ ต้อง `List<Person> list = ...`

---

## 5. ควรใช้ Panache ไหม

### ที่ควรพิจารณาก่อนตัดสินใจ

**Panache แก้ปัญหา boilerplate ของ CRUD** ถ้าโค้ดเบสมีชั้นที่ทำหน้าที่นั้นอยู่แล้ว — generic repository ที่เขียนเอง, code generator, หรือ query builder แบบ config — ประโยชน์ที่เพิ่มขึ้นจะน้อยกว่าที่คิดมาก

**จุดที่ Panache ให้ผลชัดที่สุดคือ JPQL ที่เขียนมือเยอะ ๆ** service ที่มี query ยาว ๆ กระจายอยู่หลายร้อยเมธอด ย่อได้จริงและอ่านง่ายขึ้นจริง

### ถ้าจะเริ่มใช้กับโค้ดเบสที่มีอยู่แล้ว

เริ่มแบบ **repository ทีละตัว** ได้โดยไม่กระทบของเดิม เพราะเป็น interface ไม่ต้องแตะ entity

**แต่ห้ามได้ครึ่ง ๆ กลาง ๆ** — โค้ดเบสที่มีทั้ง `EntityManager` ดิบ, repository ที่เขียนเอง และ Panache ปนกัน จะแย่กว่าเลือกอย่างใดอย่างหนึ่งไปเลย เพราะคนอ่านต้องเดาทุกครั้งว่าไฟล์นี้ใช้แบบไหน
ถ้าจะเปลี่ยนควรมีขอบเขตชัด เช่น "ของใหม่ทั้งหมดใช้ Panache" แล้วเขียนกำกับไว้

---

## 6. อนาคต: Panache Next / Jakarta Data

ทีม Quarkus ยอมรับข้อจำกัดของ Panache รุ่นปัจจุบันแล้ว และกำลังไปทาง **Jakarta Data**

- **query ตรวจตอน build** — `@Find` / `@HQL` ผ่าน annotation processor ของ Hibernate พิมพ์ผิดรู้ตอน compile ไม่ต้องรอ runtime
- **static method หายไป** — active record ที่ใช้ bytecode manipulation ทำให้ใช้ method reference (`Person::findByEmail`) ไม่ได้ และ generic inference พัง
- **เลิกแยก blocking / reactive** — เมื่อก่อนต้องเลือกโมดูลใดโมดูลหนึ่งต่อ entity ซึ่งเจ็บปวดหลัง Quarkus รองรับการใช้ปนกันในแอปเดียว
- ใช้ **stateless session** เป็นค่าเริ่มต้น — insert/update/delete ตรง ๆ ไม่ต้องแบก persistence context ทำให้ workload ที่อ่านเยอะกินหน่วยความจำต่อ request น้อยลงมาก

**สรุป:** ถ้าจะเริ่มใช้ Panache ตอนนี้ รู้ไว้ว่า API กำลังจะเปลี่ยน — เป็นอีกเหตุผลที่ไม่ต้องรีบยกเครื่องของเดิม

---

## 7. Cheat sheet

```java
// อ่าน
Person.findById(id);                       // null ถ้าไม่เจอ
Person.findByIdOptional(id);
Person.find("name", x).firstResult();
Person.list("age > ?1", 18);
Person.count("status", ACTIVE);
Person.find("status", ACTIVE).project(Dto.class).list();

// เขียน — ต้องมี @Transactional
person.persist();
person.persistAndFlush();                  // flush ทันที ถ้าอยากจับ error ตรงนั้น
Person.deleteById(id);
Person.delete("status", INACTIVE);
Person.update("status = ?1 where id = ?2", ACTIVE, id);
```

| อาการ | สาเหตุ |
|---|---|
| เขียนแล้วไม่บันทึก ไม่มี error | ลืม `@Transactional` |
| `stream()` แล้ว connection ค้าง | ไม่ได้ปิดด้วย try-with-resources |
| mock entity ใน test ไม่ได้ | active record ต้องใช้ `quarkus-panache-mock` |
| query พังตอน runtime ทั้งที่ compile ผ่าน | ปกติของ Panache — query เป็น string |
| extend `PanacheEntity` ไม่ได้ | entity สืบทอด base class อื่นอยู่แล้ว → ใช้ repository แทน |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus Panache Example]] — โค้ดตัวอย่างครบชุด
- [[Quarkus Hibernate]] — ชั้นล่างของ Panache กฎทุกข้อยังใช้อยู่
- [[Quarkus]] — หน้ารวม

## 📖 อ่านต่อ

- [Quarkus — Simplified Hibernate ORM with Panache](https://quarkus.io/guides/hibernate-orm-panache)
- [Panache Active Record vs. Repository Pattern — Sebastian Daschner](https://blog.sebastian-daschner.com/entries/quarkus-panache-active-record-vs-repository-pattern)
- [Fixing Panache: How Quarkus Unifies Blocking and Reactive Hibernate](https://www.the-main-thread.com/p/quarkus-panache-next-jakarta-data-hibernate)
