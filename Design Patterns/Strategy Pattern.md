---
tags:
  - design-patterns
  - oop
  - interview
type: reference
created: 2026-09-20
---

# ♟️ Strategy Pattern

> **Strategy คือ pattern ที่ห่อ algorithm/พฤติกรรมแต่ละแบบไว้เป็นคลาสแยกกัน แล้วสลับใช้ตัวไหนก็ได้โดยไม่ต้องแก้โค้ดที่เรียกใช้** — คำถามสัมภาษณ์คลาสสิกคือ "ทำไมไม่ใช้ if-else/switch ยาวๆ พอ"

---

## 1. ปัญหาที่มันแก้ — ไม่มี Strategy หน้าตาเป็นยังไง

```java
class PaymentService {
    void pay(String method, double amount) {
        if (method.equals("credit_card")) {
            // logic จ่ายผ่านบัตร
        } else if (method.equals("paypal")) {
            // logic จ่ายผ่าน PayPal
        } else if (method.equals("promptpay")) {
            // logic จ่ายผ่าน PromptPay
        }
        // เพิ่มวิธีจ่ายใหม่ทีไร ต้องมาแก้คลาสนี้ทุกครั้ง
    }
}
```

**ปัญหา:** ทุกครั้งที่มีวิธีจ่ายเงินใหม่ ต้องกลับมาแก้ `PaymentService` เพิ่ม `else if` เรื่อยๆ — คลาสนี้โตขึ้นไม่มีที่สิ้นสุดและผิดหลัก **Open/Closed Principle** (ควรเปิดให้ขยาย แต่ปิดไม่ให้ต้องแก้โค้ดเดิม)

---

## 2. มี Strategy แล้วต่างกันตรงไหน

```java
interface PaymentStrategy {
    void pay(double amount);
}

class CreditCardPayment implements PaymentStrategy {
    public void pay(double amount) { /* logic จ่ายผ่านบัตร */ }
}

class PromptPayPayment implements PaymentStrategy {
    public void pay(double amount) { /* logic จ่ายผ่าน PromptPay */ }
}

class PaymentService {
    private final PaymentStrategy strategy;   // รู้จักแค่ interface ไม่รู้ implementation จริง

    PaymentService(PaymentStrategy strategy) {
        this.strategy = strategy;
    }

    void pay(double amount) {
        strategy.pay(amount);   // ส่งต่อให้ strategy ที่ inject เข้ามาจัดการ
    }
}
```

เพิ่มวิธีจ่ายเงินใหม่ = สร้างคลาส implement `PaymentStrategy` เพิ่มอีกตัวเดียว **ไม่ต้องแก้ `PaymentService` เลยแม้แต่บรรทัดเดียว** — สังเกตว่าหน้าตาเหมือน [[Dependency Injection]] เป๊ะ (รับ interface เข้ามาจากภายนอก) เพราะ Strategy คือหนึ่งใน pattern ที่ DI ถูกออกแบบมาให้รองรับโดยตรง

---

## 3. เลือก strategy ตอน runtime ยังไง

```java
@ApplicationScoped
class PaymentService {
    @Inject
    List<PaymentStrategy> strategies;   // CDI inject ทุก implementation เข้ามาเป็น list

    void pay(String method, double amount) {
        strategies.stream()
            .filter(s -> s.supports(method))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("ไม่รองรับวิธีจ่ายนี้"))
            .pay(amount);
    }
}

interface PaymentStrategy {
    boolean supports(String method);
    void pay(double amount);
}
```

แต่ละ strategy ประกาศเองว่า `supports(...)` เงื่อนไขไหน — เพิ่ม strategy ใหม่แล้ว CDI inject เข้า list ให้อัตโนมัติโดยไม่ต้องแก้ `PaymentService`

---

## 4. เทียบกับ Java `Comparator` — Strategy ที่อยู่ใน JDK อยู่แล้ว

```java
list.sort((a, b) -> a.getPrice() - b.getPrice());       // strategy แบบ lambda
list.sort(Comparator.comparing(Item::getName));          // strategy อีกแบบ สลับได้ทันที
```

`Comparator` คือ interface ของ Strategy pattern ที่มากับ Java เอง — `Collections.sort(list, comparator)` **รับ strategy การเทียบค่าเข้ามาจากภายนอก** โดยตัว sort algorithm เองไม่ต้องรู้เลยว่าจะเทียบตามอะไร

---

## 5. Strategy vs State — คำถามสัมภาษณ์ที่สับสนกันบ่อยที่สุด

หน้าตาโค้ดแทบจะเหมือนกัน (interface + หลาย implementation + context ถือ reference) แต่**เจตนาต่างกันคนละเรื่อง**:

| | Strategy | State |
|---|---|---|
| ใครเป็นคนเลือก | **ผู้เรียกใช้** เลือก strategy เองจากภายนอก (เช่น ผู้ใช้เลือกวิธีจ่ายเงิน) | **object เปลี่ยนพฤติกรรมเอง** ตาม state ภายในที่เปลี่ยนไป (เช่น order เปลี่ยนจาก PENDING → PAID → SHIPPED) |
| รู้ตัวไหม | context อาจไม่รู้ด้วยซ้ำว่ามี "strategy อื่น" อยู่ | context รู้ทุก state และเปลี่ยนไปมาระหว่างกันเองได้ |
| ตัวอย่าง | วิธีคำนวณ, วิธีจ่ายเงิน, วิธี sort | สถานะออเดอร์, สถานะเกม, workflow ของเอกสาร |

---

## กับดัก

- **ใช้ Strategy กับเงื่อนไขที่มีแค่ 2 ทางและไม่มีทีท่าจะเพิ่ม** — over-engineer ทั้งที่ if-else ธรรมดาอ่านง่ายกว่าและพอเพียงแล้ว ใช้ Strategy คุ้มค่าตอนมีตั้งแต่ 3 ทางขึ้นไปหรือคาดว่าจะเพิ่มเรื่อยๆ
- **strategy เก็บ mutable state ไว้ในตัวเองแล้วแชร์ instance เดียวกันข้ามหลาย request** — ถ้า strategy ควรเป็น stateless (ส่วนใหญ่ควรเป็นแบบนั้น) แชร์เป็น singleton ได้ปลอดภัย แต่ถ้าเผลอใส่ state ลงไปจะเกิดปัญหาข้อมูลปนกันข้าม request เหมือน [[Singleton Pattern]] ข้อ 4
- **ลืมเคส "ไม่มี strategy ไหน supports เลย"** — ต้องมี fallback/throw exception ที่ชัดเจนเสมอ (ข้อ 3) ไม่ปล่อยให้เงียบหรือ `NullPointerException` แบบไม่มีบริบท

---

## Cheat sheet

```java
interface Strategy { void execute(); }
class ConcreteStrategyA implements Strategy { public void execute() { ... } }
class ConcreteStrategyB implements Strategy { public void execute() { ... } }

class Context {
    private final Strategy strategy;
    Context(Strategy strategy) { this.strategy = strategy; }
    void doWork() { strategy.execute(); }
}
```

## 🔗 เกี่ยวข้อง

- [[Dependency Injection]] — วิธีที่ strategy ถูก inject เข้า context จากภายนอก
- [[Singleton Pattern]] — strategy ที่ stateless มักถูกทำเป็น singleton ได้อย่างปลอดภัย

## 📖 อ่านต่อ

- [Refactoring Guru — Strategy](https://refactoring.guru/design-patterns/strategy)
