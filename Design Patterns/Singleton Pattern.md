---
tags:
  - design-patterns
  - oop
  - interview
type: reference
created: 2026-09-20
---

# 🔂 Singleton Pattern

> **Singleton คือ design pattern ที่การันตีว่าคลาสหนึ่งมี instance ได้แค่ตัวเดียวในทั้งโปรแกรม** และมีจุดเข้าถึง instance นั้นแบบ global (เรียกจากที่ไหนก็ได้ตัวเดียวกันเสมอ) — เป็นหัวข้อที่คนสัมภาษณ์ชอบถามเพราะมี follow-up หลายชั้น (thread-safety, ข้อเสีย, เทียบกับ DI)

---

## 1. ทำไมต้องมี — ปัญหาที่มันแก้

บางอย่างควรมีตัวเดียวพอในทั้งแอป ถ้ามีหลายตัวจะเสียทรัพยากรโดยไม่จำเป็นหรือทำให้ state ไม่ตรงกัน เช่น: connection pool, cache กลาง, logger, configuration ที่โหลดมาครั้งเดียว

---

## 2. วิธี implement แบบคลาสสิก (Java)

```java
public class Singleton {
    private static Singleton instance;

    private Singleton() { }   // constructor เป็น private — ห้าม new จากข้างนอกโดยตรง

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

**3 องค์ประกอบที่ขาดไม่ได้:** constructor เป็น `private` (กัน `new` จากข้างนอก), เก็บ instance ไว้เป็น `static` field ของคลาสตัวเอง, มี static method (`getInstance()`) เป็นทางเข้าถึงจุดเดียว

---

## 3. Thread-safety — จุดที่คนสัมภาษณ์ถามต่อบ่อยที่สุด

โค้ดข้อ 2 ข้างบน **ไม่ thread-safe** — ถ้าสอง thread เรียก `getInstance()` พร้อมกันตอนที่ `instance` ยังเป็น `null` ทั้งคู่ อาจสร้าง object กันคนละตัว (race condition) ได้ instance ซ้ำสองตัวทั้งที่ตั้งใจให้มีตัวเดียว

### ทางแก้ที่พบบ่อย เรียงจากง่ายไปดี

| วิธี | ทำยังไง | ข้อดี/ข้อเสีย |
|---|---|---|
| **Synchronized method** | ใส่ `synchronized` ที่ `getInstance()` | thread-safe แต่ **ช้า** เพราะ lock ทุกครั้งที่เรียก แม้ instance จะถูกสร้างไปแล้วก็ตาม |
| **Eager initialization** | `private static final Singleton instance = new Singleton();` สร้างทันทีตอน class load | thread-safe โดย JVM รับประกันเอง (class loading เป็น atomic) แต่สร้างตั้งแต่ต้นแม้ยังไม่ได้ใช้ (เปลืองถ้า object สร้างแพง) |
| **Double-checked locking** | เช็ค `null` สองชั้น ล็อกเฉพาะตอนสร้างจริงครั้งแรก | เร็วกว่า synchronized method **แต่ต้องใส่ `volatile`** ที่ field ไม่งั้น JVM อาจ reorder คำสั่งจนได้ object ที่ยังสร้างไม่สมบูรณ์ |
| **Bill Pugh (static inner holder class)** | เก็บ instance ไว้ใน static nested class แยก จะถูก load ก็ต่อเมื่อถูกเรียกใช้ครั้งแรกเท่านั้น | **lazy + thread-safe โดยไม่ต้อง synchronized เลย** ได้ทั้งสองอย่างพร้อมกัน แนะนำที่สุดถ้ายังอยากทำ pattern นี้เองด้วยมือ |
| **Enum singleton** | `enum Singleton { INSTANCE; }` | วิธีที่ Joshua Bloch (Effective Java) แนะนำ — thread-safe, กัน reflection/serialization สร้าง instance ซ้ำได้ในตัวเองอัตโนมัติ (วิธีอื่นข้างบนป้องกันสองเรื่องนี้ไม่ได้ถ้าไม่เขียนโค้ดเพิ่ม) |

```java
// Bill Pugh — เขียนเองแล้วได้ lazy + thread-safe ฟรี
public class Singleton {
    private Singleton() { }
    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }
    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}

// Enum — สั้นสุดและปลอดภัยสุด
public enum Singleton {
    INSTANCE;
    public void doSomething() { /* ... */ }
}
```

---

## 4. ข้อเสียของ Singleton — เหตุผลที่คนไม่แนะนำให้ทำเองในโค้ดจริงยุคนี้

- **สร้าง global state ที่ซ่อนอยู่** — คลาสไหนก็เรียก `Singleton.getInstance()` ได้จากทุกที่ ทำให้ dependency ระหว่างคลาสไม่ชัดเจนในตัว constructor/signature (เรียกว่า **hidden dependency**)
- **เทสยาก** — ทุก unit test ใช้ instance เดียวกันตลอดทั้งโปรแกรม (state ค้างข้ามเทสได้ถ้าไม่ reset เอง) และแทนที่ด้วย mock ตอนเทสไม่ได้ง่าย ๆ เพราะโค้ดผูกกับคลาสจริงตรง ๆ
- **ผูกติดกับ implementation ที่เจาะจง (tight coupling)** — โค้ดที่เรียก `Singleton.getInstance()` ตรง ๆ สลับไปใช้ implementation อื่นแทนไม่ได้เลยถ้าไม่แก้โค้ดที่เรียก
- **ละเมิด Single Responsibility Principle** — คลาสต้องทำทั้งหน้าที่ของตัวเอง **และ** จัดการ lifecycle การสร้าง instance ของตัวเองไปพร้อมกัน

**นี่คือเหตุผลที่ Dependency Injection เข้ามาแทนที่ในโค้ดจริงสมัยใหม่** — ได้ประโยชน์ของ "มี instance เดียวใช้ร่วมกัน" เหมือนกัน แต่ไม่ต้องเจอข้อเสียพวกนี้ (ดูรายละเอียดที่ [[Dependency Injection]])

---

## กับดัก

- **ลืมใส่ `volatile` ตอนทำ double-checked locking** — ได้ object ที่ constructor ยังทำงานไม่เสร็จสมบูรณ์กลับมาใช้ (เพราะ JVM reorder คำสั่งได้) บั๊กที่เกิดแค่บางครั้งภายใต้ load สูงเท่านั้น debug ยากมาก
- **ใช้ Eager initialization กับ object ที่สร้างแพง** (เช่น เปิด connection จริง, โหลดไฟล์ใหญ่) ทั้งที่ไม่ได้ใช้ตั้งแต่ต้นโปรแกรม — เปลือง resource/เวลา startup โดยไม่จำเป็น
- **สร้าง Singleton เองในโปรเจกต์ที่มี DI framework อยู่แล้ว** (Spring/Quarkus/Angular) — ซ้ำซ้อนกับสิ่งที่ framework ทำให้ฟรีอยู่แล้วผ่าน scope แบบ singleton ของ container (ดู [[Dependency Injection]])
- **เข้าใจผิดว่า "DI container สร้าง bean แบบ singleton scope" กับ "Singleton pattern" คือเรื่องเดียวกัน** — หน้าตาคล้ายกัน (มี instance เดียว) แต่คนละกลไกกันโดยสิ้นเชิง (ดูตารางเทียบที่ [[Dependency Injection]])

---

## Cheat sheet

```java
// Bill Pugh — แนะนำที่สุดถ้าต้องเขียนเอง
public class Singleton {
    private Singleton() {}
    private static class Holder { static final Singleton INSTANCE = new Singleton(); }
    public static Singleton getInstance() { return Holder.INSTANCE; }
}

// Enum — สั้นและปลอดภัยสุด (Joshua Bloch แนะนำ)
public enum Singleton { INSTANCE; }
```

| ต้องการ | ใช้วิธีไหน |
|---|---|
| ง่ายสุด ปลอดภัยสุด ไม่สนใจ lazy | enum singleton |
| lazy + thread-safe โดยไม่ต้อง synchronized | Bill Pugh (static holder) |
| อยู่ในโปรเจกต์ที่มี DI framework อยู่แล้ว | **อย่าทำ Singleton pattern เอง** ใช้ scope ของ container แทน |

## 🔗 เกี่ยวข้อง

- [[Dependency Injection]] — วิธีได้ประโยชน์ของ singleton โดยไม่เจอข้อเสีย (ข้อ 4)
- [[Angular Services and DI]] — `providedIn: 'root'` คือ singleton scope ที่ Angular จัดการให้

## 📖 อ่านต่อ

- [Refactoring Guru — Singleton](https://refactoring.guru/design-patterns/singleton)
- [Baeldung — Singleton in Java](https://www.baeldung.com/java-singleton)
