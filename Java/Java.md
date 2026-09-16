---
tags:
  - java
  - java/index
type: moc
created: 2026-08-18
---

# ☕ Java

หน้ารวมโน้ตภาษา Java — ตัวภาษาและ core library ล้วน ๆ
เรื่องที่ผูกกับ framework แยกไปอยู่ที่ [[Quarkus]]

> **เวอร์ชันที่ใช้อยู่: Java 17 (LTS)**
> สำคัญเพราะฟีเจอร์ใหม่หลายอย่างมาทีหลัง — ตัวอย่างที่เจอบนเว็บมักเขียนด้วย 21+ แล้วก๊อปมาใช้ไม่ได้

---

## โน้ตในนี้

| โน้ต | ว่าด้วย | สถานะ |
|---|---|---|
| [[Java Data Type]] | primitive 8 ตัว vs reference, wrapper class, autoboxing, widening/narrowing, Integer cache | ✅ |
| [[Java Exception]] | checked vs unchecked, try-with-resources, multi-catch, ออกแบบ exception เอง, exception chaining | ✅ |
| [[Java Record]] | record คืออะไร, compact constructor, ใช้ทำ DTO/projection/value object/sealed result, กับดัก | ✅ |
| [[Java Date Time]] | เลือกระหว่าง `Date`/`LocalDate`/`LocalDateTime`/`Instant`, ตัวไหนทำวันเพี้ยน, format, กับดัก | ✅ |
| [[Java Lambda]] | syntax, functional interface, method reference (`::`), effectively final, `this`, checked exception | ✅ |
| [[Java Stream]] | lazy evaluation, `collect`/`Collectors`, `map` vs `flatMap`, primitive stream, parallel stream | ✅ |

---

## แผนที่ความรู้ — ยังไม่ได้เขียน

- **Optional** — ใช้ตอนไหน ไม่ใช้ตอนไหน (อย่าใส่เป็น field หรือ parameter)
- **`equals` / `hashCode`** — สัญญาที่ต้องรักษา และผลตอนเอาไปใส่ `HashMap`/`HashSet`
- **Generics** — wildcard, type erasure, ทำไมบางอย่างเขียนไม่ได้
- **Concurrency** — `CompletableFuture`, `ExecutorService`, thread safety
- **Text block** (Java 15+) — เขียน SQL/JSON หลายบรรทัดโดยไม่ต้องต่อ string

---

## ฟีเจอร์ตามเวอร์ชัน — ตารางอ้างอิงเร็ว

| ฟีเจอร์ | ตั้งแต่ | ใช้ได้บน 17? |
|---|---|---|
| `var` (local type inference) | 10 | ✅ |
| switch expression (`->`, `yield`) | 14 | ✅ |
| **text block** (`"""`) | 15 | ✅ |
| **record** | 16 | ✅ |
| `instanceof` pattern (`if (o instanceof Foo f)`) | 16 | ✅ |
| **sealed** class/interface | 17 | ✅ |
| pattern matching for `switch` | 21 | ❌ |
| **record pattern** (แกะ component) | 21 | ❌ |
| virtual thread | 21 | ❌ |
| sequenced collection | 21 | ❌ |

---

## กฎรวมที่จำไว้ได้เลย

1. **immutable ไว้ก่อน** — สร้างเสร็จแล้วแก้ไม่ได้ ตัดปัญหา thread safety และการโดนแก้จากที่ไม่คาดคิดไปเยอะมาก
2. **แยกชั้นข้อมูลให้ชัด** — entity ของ DB, DTO ของ API, domain object ของ business logic ไม่ใช่ตัวเดียวกัน แม้หน้าตาจะเหมือนกันตอนเริ่ม
3. **ตั้งชื่อ type ให้แคบ** — `CustomerId` ดีกว่า `UUID`, `Money` ดีกว่า `BigDecimal` เพราะคอมไพเลอร์ช่วยจับที่ผิดได้
4. **ระวังตัวอย่างบนเว็บ** — Java ออกเวอร์ชันใหม่ทุก 6 เดือน โค้ดที่เจอมักใหม่กว่าที่เราใช้

---

## 🔗 ที่อื่นใน vault

- [[Quarkus]] — framework ที่ใช้ Java พวกนี้ทำงานจริง
- [[Greenhouse]] — โปรเจกต์ที่อยากทำ

## 📖 อ้างอิงหลัก

- [OpenJDK JEP Index](https://openjdk.org/jeps/0)
- [Java 17 API docs](https://docs.oracle.com/en/java/javase/17/docs/api/)
