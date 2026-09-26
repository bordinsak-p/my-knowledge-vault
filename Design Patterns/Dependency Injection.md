---
tags:
  - design-patterns
  - oop
  - interview
type: reference
created: 2026-09-20
---

# 💉 Dependency Injection (DI)

> **Dependency Injection คือหลักการที่ "ส่ง" dependency ที่คลาสต้องการเข้ามาจากภายนอก แทนที่จะให้คลาสสร้างมันขึ้นมาเอง** — คำถามสัมภาษณ์คลาสสิกที่สุดคือ "DI ต่างจาก Singleton pattern ยังไง" (ข้อ 5) เตรียมตอบให้ได้แม่น ๆ

---

## 1. ปัญหาที่มันแก้ — ไม่มี DI หน้าตาเป็นยังไง

```java
class OrderService {
    private EmailService emailService = new EmailService();   // ❌ สร้างเองข้างใน

    void placeOrder(Order order) {
        emailService.send(order.getCustomerEmail(), "สั่งซื้อสำเร็จ");
    }
}
```

**ปัญหา:** `OrderService` **ผูกติดกับ implementation ที่เจาะจง** (`EmailService` ตัวจริง) โดยตรง — เปลี่ยนไปใช้ implementation อื่น (เช่น mock ตอนเทส หรือ `SmsService` แทน) ทำไม่ได้เลยถ้าไม่แก้โค้ดข้างในนี้ และเทส `OrderService` เดี่ยว ๆ โดยไม่ยิงอีเมลจริงก็ทำไม่ได้ด้วย

---

## 2. มี DI แล้วต่างกันตรงไหน — Constructor Injection

```java
class OrderService {
    private final EmailService emailService;

    OrderService(EmailService emailService) {   // ✅ รับเข้ามาจากข้างนอก ไม่สร้างเอง
        this.emailService = emailService;
    }

    void placeOrder(Order order) {
        emailService.send(order.getCustomerEmail(), "สั่งซื้อสำเร็จ");
    }
}
```

`OrderService` **ไม่รู้และไม่สนใจว่า `EmailService` ที่ได้รับมาเป็น implementation ไหน** — ตอนเทสส่ง mock เข้าไปแทนได้ตรง ๆ ผ่าน constructor เดียวกันนี้เลย ไม่ต้องแก้โค้ดของ `OrderService` แม้แต่บรรทัดเดียว

---

## 3. DI มี 3 แบบ

| แบบ | หน้าตา | ใช้เมื่อ |
|---|---|---|
| **Constructor injection** | รับผ่าน constructor (ข้อ 2) | **แนะนำเป็นค่า default เสมอ** — บังคับว่าต้องมี dependency ก่อนสร้าง object ได้ (ไม่มี object ที่อยู่ในสถานะไม่สมบูรณ์), ทำ field เป็น `final` ได้, เทสง่ายสุด |
| **Setter injection** | มี method `setEmailService(...)` แยก | dependency ที่เป็น**ทางเลือก** (optional) ไม่ใช่ทุก object ต้องมี |
| **Field injection** | ใส่ annotation ตรง field เลย เช่น `@Inject EmailService emailService;` | สะดวกเขียนสั้นสุด แต่**ไม่แนะนำ** — ซ่อน dependency ไว้ไม่เห็นจาก constructor, ทำ `final` ไม่ได้, เทสยากกว่า (ต้องพึ่ง reflection หรือ DI framework ตอนเทสด้วย) |

---

## 4. DI Container / Inversion of Control (IoC)

ในโปรเจกต์จริงไม่ได้ไปนั่ง `new` แล้วส่ง dependency เข้าไปเองทีละคลาสด้วยมือ — **framework (Spring, Quarkus/CDI, Angular) ทำหน้าที่นี้ให้อัตโนมัติ** อ่าน annotation แล้วสร้าง+ประกอบ (wire) object ทั้งหมดให้เอง เรียกหลักการนี้ว่า **Inversion of Control** — เดิมโค้ดเราเป็นคนควบคุมว่าจะสร้าง/เรียกอะไรเมื่อไหร่ พอใช้ DI container "การควบคุม" ย้ายไปอยู่ที่ framework แทน

```java
// Quarkus/CDI — framework สร้างและ inject ให้เอง ไม่ต้อง new เอง
@ApplicationScoped
class OrderService {
    @Inject
    EmailService emailService;
}
```

ดูตัวอย่างจริงเต็มรูปแบบฝั่ง Quarkus ที่ [[Quarkus Project Structure]] และฝั่ง Angular (`inject()`, `providedIn: 'root'`) ที่ [[Angular Services and DI]]

---

## 5. DI container's singleton scope ≠ Singleton Pattern — คำถามสัมภาษณ์คลาสสิก

ทั้งสองแบบทำให้ "มี instance เดียวใช้ร่วมกันทั้งแอป" เหมือนกัน **แต่คนละกลไกกันโดยสิ้นเชิง**:

| | [[Singleton Pattern]] (แบบดั้งเดิม) | DI Container (default scope) |
|---|---|---|
| ใครสร้าง instance | คลาสสร้างตัวเอง (`private constructor` + `getInstance()`) | container/framework เป็นคนสร้างให้ |
| โค้ดเข้าถึงยังไง | เรียก `Singleton.getInstance()` ตรง ๆ แบบ global | รับผ่าน constructor/field ที่ inject มาให้ ไม่รู้ตัวด้วยซ้ำว่าเป็น singleton |
| Testability | ยาก — global state, mock ยาก (ดู [[Singleton Pattern]] ข้อ 4) | ง่าย — สลับ implementation/mock ตอนเทสได้ตรง ๆ |
| Coupling | แน่น — ผูกกับคลาสจริงตรง ๆ | หลวม — ผูกกับ interface/abstraction, container ตัดสินใจว่าจะ inject อะไรจริง |
| ใครคุม lifecycle | คลาสตัวเองคุมเอง | container คุม (Inversion of Control) |

**สรุปสั้นสุดสำหรับตอบสัมภาษณ์:** *"DI container's singleton scope ให้ประโยชน์แบบเดียวกับ Singleton pattern (instance เดียวใช้ร่วมกัน) แต่ย้ายการควบคุม lifecycle ไปให้ framework แทนที่จะฝังไว้ในตัวคลาสเอง ทำให้เทสง่ายและ coupling หลวมกว่ามาก"*

---

## 6. ข้อดี / ข้อเสียของ DI (มุมมองสมดุล)

**ข้อดี:** เทสง่าย (mock dependency ได้ตรง ๆ), loose coupling, สลับ implementation ได้โดยไม่แก้โค้ดที่ใช้งาน, แยกความรับผิดชอบชัดเจน (คลาสไม่ต้องจัดการ lifecycle ของ dependency ตัวเอง)

**ข้อเสีย:** เพิ่มความซับซ้อน/indirection (ตามโค้ดว่า dependency มาจากไหนบางทีต้องไล่ดู config เพิ่ม), ใช้เกินความจำเป็นกับแอปเล็ก ๆ ทำให้ over-engineer, error บางอย่าง (เช่น missing binding) โผล่ตอน runtime/startup แทนที่จะเจอตอน compile

---

## กับดัก

- **ตอบสัมภาษณ์ว่า Singleton pattern กับ DI singleton scope "เหมือนกัน"** — ผิด ทั้งสองแค่ **ผลลัพธ์คล้ายกัน** (instance เดียว) แต่กลไกและข้อดีข้อเสียต่างกันคนละเรื่อง (ข้อ 5)
- **ใช้ Field injection เป็นหลัก** — ซ่อน dependency ไว้ไม่เห็นจาก constructor ทำให้อ่านโค้ดแล้วไม่รู้ว่าคลาสนี้ต้องพึ่งอะไรบ้างจนกว่าจะไล่ดูทุก field เอง แนะนำ Constructor injection เป็นค่า default เสมอ
- **มองว่า DI คือ "แค่ pattern เขียนโค้ดสวยขึ้น"** — จริง ๆ คือหลักการที่รองรับ Dependency Inversion Principle (ตัวสุดท้ายใน SOLID) — โมดูลระดับสูงไม่ควรพึ่ง concrete implementation ของโมดูลระดับล่างโดยตรง ทั้งคู่ควรพึ่ง abstraction (interface) แทน
- **ลืมว่า DI container ก็ยังมี "state" ที่แชร์กันอยู่ดี** ถ้า scope เป็น singleton — ถ้า bean เก็บ mutable state ไว้ใน field ของตัวเอง (ไม่ใช่แค่ dependency) ยังเจอปัญหา concurrency แบบเดียวกับ Singleton pattern ได้เหมือนกันถ้าไม่ระวัง thread-safety

---

## Cheat sheet

```java
// ✅ Constructor injection — แนะนำเป็นค่า default
class Service {
    private final Dependency dep;
    Service(Dependency dep) { this.dep = dep; }
}
```

```
Singleton pattern → คลาสควบคุมตัวเอง, global access, เทสยาก
DI (singleton scope) → container ควบคุมให้, inject เข้ามา, เทสง่าย
```

## 🔗 เกี่ยวข้อง

- [[Singleton Pattern]] — pattern ที่ DI container's default scope มักถูกเทียบด้วยเสมอ
- [[Angular Services and DI]] — ตัวอย่าง DI จริงฝั่ง Angular (`inject()`, `providedIn: 'root'`)
- [[Quarkus Project Structure]] — ตัวอย่าง DI จริงฝั่ง Quarkus/CDI (`@Inject`, `@ApplicationScoped`)

## 📖 อ่านต่อ

- [Martin Fowler — Inversion of Control Containers and the Dependency Injection pattern](https://martinfowler.com/articles/injection.html)
- [Baeldung — Dependency Injection in Java](https://www.baeldung.com/java-dependency-injection)
