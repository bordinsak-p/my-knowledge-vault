---
tags:
  - java
  - record
  - dto
  - api
type: reference
created: 2026-08-18
---

# 📦 Java Record

> **record คือคลาสที่ประกาศว่า "ฉันมีหน้าที่เก็บข้อมูล ไม่มีอย่างอื่น"**
> พอบอกคอมไพเลอร์แบบนั้น มันเขียน constructor / accessor / `equals` / `hashCode` / `toString` ให้ทั้งหมด

---

## 1. คืออะไร

```java
public record Point(int x, int y) {}
```

บรรทัดเดียวนี้ได้ของเทียบเท่าคลาสประมาณ 40 บรรทัด — คอมไพเลอร์สร้างให้อัตโนมัติ:

| ได้อะไร | หน้าตา |
|---|---|
| field | `private final int x;` `private final int y;` |
| constructor | `Point(int x, int y)` — เรียกว่า **canonical constructor** |
| accessor | `x()` และ `y()` — **ไม่ใช่ `getX()`** |
| `equals()` | เทียบทุก component |
| `hashCode()` | คำนวณจากทุก component |
| `toString()` | `Point[x=1, y=2]` |

**สิ่งที่ record เป็นโดยปริยาย**

- คลาสเป็น `final` — สืบทอดต่อไม่ได้
- field ทุกตัวเป็น `final` — สร้างแล้วแก้ไม่ได้
- extends คลาสอื่นไม่ได้ (แต่ `implements` interface ได้)
- เพิ่ม instance field เองไม่ได้ (static field ได้)

---

## 2. เขียนยังไง

### 2.1 ใส่ validation ด้วย compact constructor

```java
public record Money(BigDecimal amount, String currency) {

    // compact constructor — ไม่ต้องเขียน parameter list ซ้ำ ไม่ต้อง assign เอง
    public Money {
        if (amount == null)                 throw new IllegalArgumentException("amount required");
        if (amount.signum() < 0)            throw new IllegalArgumentException("amount must be >= 0");
        if (currency == null || currency.isBlank()) throw new IllegalArgumentException("currency required");

        currency = currency.toUpperCase();   // แก้ค่าก่อนถูก assign ลง field ได้
    }
}
```

**การ assign ลง field เกิดขึ้นอัตโนมัติหลังจบ block** — ในนี้แค่ตรวจหรือปรับค่าพารามิเตอร์

### 2.2 เพิ่ม method / static factory / interface

```java
public record Money(BigDecimal amount, String currency) implements Comparable<Money> {

    public static Money baht(BigDecimal amount) {
        return new Money(amount, "THB");
    }

    public Money plus(Money other) {
        requireSameCurrency(other);
        return new Money(amount.add(other.amount), currency);   // คืนตัวใหม่ ไม่แก้ตัวเดิม
    }

    public boolean isZero() {
        return amount.signum() == 0;
    }

    @Override
    public int compareTo(Money o) {
        requireSameCurrency(o);
        return amount.compareTo(o.amount);
    }

    private void requireSameCurrency(Money o) {
        if (!currency.equals(o.currency))
            throw new IllegalArgumentException("currency mismatch");
    }
}
```

### 2.3 ซ้อนกันได้

```java
public record Address(String line1, String city, String postcode) {}

public record Customer(String name, Address address, List<String> tags) {}
```

---

## 3. เอามาทำ DTO ได้ไหม — **ได้ และเหมาะที่สุด**

DTO คือ "ก้อนข้อมูลที่ส่งไปมา ไม่มีพฤติกรรม" ซึ่งตรงกับนิยามของ record เป๊ะ

### 3.1 Request DTO + validation

```java
public record ProductCreateRequest(
        @NotBlank @Size(max = 32)  String code,
        @NotBlank @Size(max = 200) String name,
        @NotNull @DecimalMin("0.00") BigDecimal price,
        @Valid Address shipFrom) {}
```

```java
@POST
public Response create(@Valid ProductCreateRequest req) { ... }
```

annotation ของ Jakarta Bean Validation ทำงานกับ record ได้ตามปกติ เพราะ constraint ส่วนใหญ่ประกาศ `@Target` ครอบทั้ง field / parameter / method อยู่แล้ว

### 3.2 Response DTO

```java
public record ProductResponse(
        Long id,
        String code,
        String name,
        BigDecimal price,
        String categoryName) {

    public static ProductResponse from(Product p) {
        return new ProductResponse(p.id, p.code, p.name, p.price,
                                   p.category == null ? null : p.category.name);
    }
}
```

**static factory `from(...)` เป็นแพตเทิร์นที่คุ้มมาก** — รวม logic การแปลงไว้ที่เดียว แทนที่จะกระจายอยู่ทุก endpoint

### 3.3 Jackson ทำงานได้เลย ไม่ต้องตั้งอะไร

Jackson รองรับ record มาตั้งแต่ **2.12** — serialize ผ่าน accessor, deserialize ผ่าน canonical constructor

```java
public record ProductResponse(
        Long id,
        @JsonProperty("product_code") String code,
        @JsonFormat(pattern = "yyyy-MM-dd") LocalDate releasedOn) {}
```

> ต้องการซ่อน field ที่เป็น null ใช้ `@JsonInclude(JsonInclude.Include.NON_NULL)` บน record ได้ตามปกติ

### 3.4 ⚠️ แต่ห้ามใช้เป็น JPA entity

```java
@Entity
public record Product(...) {}   // ❌ ใช้ไม่ได้
```

JPA ต้องการ **no-arg constructor** และต้องการ **แก้ค่า field ได้** (proxy, lazy loading, dirty checking) ซึ่งขัดกับธรรมชาติของ record ทั้งสองข้อ

**entity ต้องเป็นคลาสธรรมดา / record เป็น DTO** — และนี่เป็นเหตุผลที่ดีที่จะแยกสองชั้นนี้ออกจากกันอยู่แล้ว

---

## 4. ใช้ทำอะไรได้อีกในการเขียน API

### 4.1 Projection — ดึงเฉพาะคอลัมน์ที่ใช้

ใช้ได้ทั้ง JPA constructor expression และ Panache `project()`

```java
public record ProductSummary(Long id, String code, String name) {}
```

```java
// JPA
em.createQuery("""
    select new org.acme.ProductSummary(p.id, p.code, p.name)
    from Product p where p.status = :st
    """, ProductSummary.class)

// Panache
Product.find("status", Status.ACTIVE).project(ProductSummary.class).list();
```

ดูตัวอย่างเต็มที่ [[Quarkus Panache Example]]

### 4.2 คืนค่าหลายตัวจาก method เดียว

แทนที่จะใช้ `Object[]`, `Map`, หรือสร้างคลาสทิ้ง ๆ ขว้าง ๆ

```java
private record ValidationOutcome(boolean valid, List<String> errors) {}

private ValidationOutcome validate(Order o) { ... }
```

**ประกาศเป็น `private record` ข้างในคลาสได้เลย** ถ้าใช้แค่ที่นั่น — ไม่ต้องสร้างไฟล์ใหม่

### 4.3 รวมพารามิเตอร์ที่เยอะเกินไป

```java
// ❌ 6 พารามิเตอร์ สลับตำแหน่งเมื่อไหร่ก็ไม่รู้ตัว
public PageResult search(String kw, Status st, int page, int size, String sortBy, String dir)

// ✅
public record SearchCriteria(String keyword, Status status,
                             int page, int size, String sortBy, String sortDir) {
    public SearchCriteria {
        if (size <= 0 || size > 200) size = 20;      // ใส่ default ได้ตรงนี้
        if (page < 0) page = 0;
    }
}

public PageResult search(SearchCriteria c)
```

### 4.4 Value object — กันสลับพารามิเตอร์

```java
public record CustomerId(UUID value) {}
public record OrderId(UUID value) {}

void link(CustomerId c, OrderId o)     // สลับสองตัวนี้ = compile error
```

ถ้าใช้ `UUID` ทั้งคู่ สลับกันแล้วคอมไพล์ผ่านสบาย ไปพังตอน runtime

### 4.5 ผลลัพธ์ที่มีได้หลายแบบ — `sealed interface` + record

```java
public sealed interface PaymentResult {
    record Success(String transactionId, Money charged) implements PaymentResult {}
    record Declined(String reason)                       implements PaymentResult {}
    record Failed(String errorCode, String message)      implements PaymentResult {}
}
```

`sealed` แปลว่ามีได้แค่ 3 แบบนี้เท่านั้น — **คอมไพเลอร์บังคับให้จัดการครบทุกกรณี** เวลาเพิ่มแบบที่ 4 ทุกที่ที่ยังไม่รองรับจะ error ให้เห็นทันที ไม่หลุดไปเงียบ ๆ

### 4.6 Event / message payload

```java
public record OrderPlaced(UUID orderId, UUID customerId, Money total, Instant at) {}
```

immutable โดยธรรมชาติ = ปลอดภัยเวลาส่งข้ามเธรดหรือขึ้น message queue

### 4.7 Config ที่จับกลุ่มกัน

```java
public record RetryPolicy(int maxAttempts, Duration delay, double jitter) {
    public static final RetryPolicy DEFAULT = new RetryPolicy(3, Duration.ofMillis(200), 0.5);
}
```

### 4.8 Test fixture

`equals()` ที่ได้มาฟรีทำให้ assert ทั้งก้อนได้เลย ไม่ต้องไล่เทียบทีละ field

```java
assertEquals(new ProductSummary(1L, "P001", "Test"), actual);
```

---

## 5. record vs Lombok vs คลาสธรรมดา

| | record | Lombok `@Value` | Lombok `@Data` | คลาสเขียนเอง |
|---|---|---|---|---|
| immutable | ✅ บังคับ | ✅ | ❌ | แล้วแต่ |
| ต้องมี library | ❌ ภาษาให้มาเลย | ✅ | ✅ | ❌ |
| accessor | `name()` | `getName()` | `getName()` | แล้วแต่ |
| ใช้เป็น JPA entity | ❌ | ❌ | ✅ | ✅ |
| builder | ❌ ไม่มีให้ | `@Builder` | `@Builder` | เขียนเอง |
| สืบทอด | ❌ | ✅ | ✅ | ✅ |

**สรุปการเลือก**

- **DTO / projection / event / value object → record** ไม่ต้องพึ่ง library เลย
- **JPA entity → คลาสธรรมดา** (จะใช้ Lombok ช่วยก็ได้)
- **ต้องการ builder เพราะ field เยอะมาก → คลาส + Lombok `@Builder`**

---

## 6. กับดัก

### 6.1 ⚠️ record ไม่ได้ immutable ลึก

```java
public record Team(String name, List<String> members) {}

var t = new Team("A", new ArrayList<>(List.of("x")));
t.members().add("y");        // ← แก้ได้! reference เป็น final แต่ของข้างในไม่ได้ถูกล็อก
```

ถ้าต้องการ immutable จริง ต้องคัดลอกทั้งขาเข้าและขาออก

```java
public record Team(String name, List<String> members) {
    public Team {
        members = members == null ? List.of() : List.copyOf(members);   // ขาเข้า
    }

    @Override
    public List<String> members() {
        return Collections.unmodifiableList(members);                   // ขาออก
    }
}
```

### 6.2 accessor ไม่ใช่ `getXxx()`

library เก่าที่คาดหวังรูปแบบ JavaBean อาจมองไม่เห็น field
Jackson / Jakarta Validation / Panache รองรับ record หมดแล้ว แต่ของเก่ากว่านั้นต้องเช็คก่อน

### 6.3 `equals()` ใช้ทุก component

ถ้ามี field ที่ไม่ควรนับ (เช่น `Instant createdAt`) จะทำให้ object ที่ควรเท่ากันกลายเป็นไม่เท่า
กรณีนี้ override `equals`/`hashCode` เองได้ แต่ถ้าต้อง override แปลว่าอาจไม่ควรเป็น record ตั้งแต่แรก

### 6.4 ไม่มี builder

component เกิน 5-6 ตัวแล้วเรียก `new` จะอ่านยากและสลับตำแหน่งง่าย
แก้ด้วย static factory ที่ตั้งชื่อสื่อความหมาย หรือถ้าเยอะจริง ๆ ก็ใช้คลาส + builder

### 6.5 อย่าใช้ record ตัวเดียวหลายหน้าที่

ถ้าเริ่มรู้สึกว่าอยากเอา record ตัวเดิมไปใช้ทั้ง request, response และ projection แปลว่ามันกว้างเกินไปแล้ว
**หนึ่ง record หนึ่งหน้าที่** — request กับ response มักต้องต่างกันอยู่แล้ว (response มี `id`, request ไม่มี)

---

## 7. เวอร์ชัน Java — อะไรใช้ได้ตอนไหน

| ฟีเจอร์ | ตั้งแต่ | หมายเหตุ |
|---|---|---|
| **record** | **Java 16** | ✅ ใช้ได้บน Java 17 |
| `sealed` interface/class | Java 17 | ✅ ใช้ได้ |
| **record pattern** (แกะ record ใน `switch`/`instanceof`) | **Java 21** | ❌ **ใช้ไม่ได้บน Java 17** |
| derived record creation (withers) | ยัง preview | ❌ ยังไม่เสถียร |

**record pattern** คือการแกะ component ออกมาตรง ๆ ในการ match

```java
// ต้อง Java 21 ขึ้นไป
String describe(PaymentResult r) {
    return switch (r) {
        case Success(String id, Money m) -> "ok " + id + " " + m.amount();
        case Declined(String reason)     -> "declined: " + reason;
        case Failed(String code, var msg)-> "error " + code;
    };
}
```

บน **Java 17** ต้องเขียนแบบเดิม

```java
if (r instanceof PaymentResult.Success s) {
    return "ok " + s.transactionId();
}
```

---

## 8. Cheat sheet

```java
// พื้นฐาน
public record Point(int x, int y) {}
var p = new Point(1, 2);
p.x();                     // accessor — ไม่ใช่ getX()

// validate + normalize
public record Email(String value) {
    public Email {
        if (value == null || !value.contains("@"))
            throw new IllegalArgumentException("invalid email");
        value = value.trim().toLowerCase();
    }
}

// static factory
public record Money(BigDecimal amount, String ccy) {
    public static Money baht(BigDecimal a) { return new Money(a, "THB"); }
}

// DTO + validation
public record CreateReq(@NotBlank String code, @NotNull BigDecimal price) {}

// nested / private ใช้ในไฟล์เดียว
private record Pair<A, B>(A first, B second) {}

// sealed + record
public sealed interface Result {
    record Ok(String data)   implements Result {}
    record Err(String error) implements Result {}
}
```

| อาการ | สาเหตุ |
|---|---|
| library มองไม่เห็น field | คาดหวัง `getXxx()` แต่ record ให้ `xxx()` |
| ค่าใน list ถูกแก้จากข้างนอก | ไม่ได้ copy ตอนรับเข้า |
| object ที่ควรเท่ากันกลับไม่เท่า | มี field เวลา/id รวมอยู่ใน `equals()` |
| `@Entity` แล้ว compile ไม่ผ่าน | record เป็น JPA entity ไม่ได้ |
| `case Success(var id) ->` ไม่ผ่าน | record pattern ต้อง Java 21 |

---

## 🔗 เกี่ยวข้อง

- [[Java]] — หน้ารวม
- [[Quarkus Panache Example]] — ใช้ record เป็น projection และ DTO ในโค้ดจริง
- [[TS Record]] — `Record<K, V>` ของ TypeScript คนละแนวคิด อย่าสับสนชื่อ

## 📖 อ่านต่อ

- [JEP 395: Records](https://openjdk.org/jeps/395)
- [JEP 409: Sealed Classes](https://openjdk.org/jeps/409)
- [JEP 440: Record Patterns (Java 21)](https://openjdk.org/jeps/440)
- [JEP 468: Derived Record Creation (Preview)](https://openjdk.org/jeps/468)
