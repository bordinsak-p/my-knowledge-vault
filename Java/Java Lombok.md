---
tags:
  - java
  - lombok
  - boilerplate
type: reference
created: 2026-09-17
---

# 🪄 Lombok — เขียน getter/setter/constructor เองน้อยลง

> **Lombok คือ annotation processor ที่ generate โค้ด boilerplate ให้ตอน compile** — ใส่ `@Getter`/`@Setter`/`@ToString` ฯลฯ บนคลาส แล้ว Lombok เขียน method พวกนั้นลงใน `.class` ให้อัตโนมัติ ไม่ต้องพิมพ์เอง ซอร์สโค้ดที่เห็นจะไม่มี method พวกนี้เลยจนกว่าจะ compile

---

## 1. ทำงานยังไง — ทำไม IDE ต้องลง plugin เพิ่ม

Lombok เกาะเข้าไปใน annotation processing ตอน `javac` compile แล้ว **แก้ syntax tree ของคลาสโดยตรง** (ไม่ใช่แค่สร้างไฟล์ใหม่แบบ annotation processor ทั่วไป) — วิธีนี้ทำงานได้จริงแต่ไม่ใช่การใช้ annotation processing API ตามมาตรฐานเป๊ะ ๆ เป็นเหตุผลที่:

- **IDE ต้องลง Lombok plugin ก่อน** (IntelliJ มีให้ในตัวสมัยใหม่, Eclipse ต้องรัน installer) ไม่งั้น IDE จะขีดแดง "method ไม่มีอยู่จริง" ทั้งที่ compile ผ่านจริง ๆ
- **อยากดูโค้ดที่ generate จริง** ใช้คำสั่ง `delombok` แปลงกลับเป็น source เต็ม ๆ ให้ดู

---

## 2. Annotation ที่ใช้บ่อย

| Annotation | ทำอะไร |
|---|---|
| `@Getter` / `@Setter` | generate getter/setter ทุก field (หรือใส่บน field เดียวก็ได้) |
| `@ToString` | generate `toString()` |
| `@EqualsAndHashCode` | generate `equals()`/`hashCode()` จากทุก field (ปรับได้ว่าจะรวม field ไหนบ้าง) |
| `@NoArgsConstructor` | constructor ว่าง |
| `@AllArgsConstructor` | constructor รับทุก field |
| `@RequiredArgsConstructor` | constructor รับเฉพาะ field ที่เป็น `final` หรือมี `@NonNull` |
| `@Builder` | เปิดใช้ builder pattern (`Product.builder().name("x").build()`) |
| `@Data` | รวม `@Getter` + `@Setter` + `@ToString` + `@EqualsAndHashCode` + `@RequiredArgsConstructor` ในตัวเดียว |
| `@Value` | เหมือน `@Data` แต่ทำให้ **immutable** — field เป็น `private final` หมด ไม่มี setter |
| `@Slf4j` | ใส่ logger field ชื่อ `log` ให้อัตโนมัติ (ไม่ต้องเขียน `LoggerFactory.getLogger(...)` เอง) |
| `@With` | สร้าง method `withX(...)` คืน object ใหม่ที่ field นั้นเปลี่ยนไป (สไตล์เดียวกับ [[Java Record]] compact constructor immutable pattern) |

---

## 3. ตัวอย่าง — ก่อน/หลัง

```java
// ❌ ไม่มี Lombok — DTO 3 field ต้องเขียนเองหมด
public class ProductDto {
    private Long id;
    private String name;
    private BigDecimal price;

    public ProductDto() {}
    public ProductDto(Long id, String name, BigDecimal price) {
        this.id = id; this.name = name; this.price = price;
    }
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public BigDecimal getPrice() { return price; }
    public void setPrice(BigDecimal price) { this.price = price; }

    @Override public boolean equals(Object o) { /* ~10 บรรทัด */ }
    @Override public int hashCode() { /* ~5 บรรทัด */ }
    @Override public String toString() { /* ~3 บรรทัด */ }
}
```

```java
// ✅ มี Lombok — บรรทัดเดียวจบ ได้ทุกอย่างข้างบนเหมือนกัน
@Data
@NoArgsConstructor
@AllArgsConstructor
public class ProductDto {
    private Long id;
    private String name;
    private BigDecimal price;
}
```

---

## 4. กับดัก — โดยเฉพาะบน JPA entity

- **`@Data`/`@EqualsAndHashCode` บน JPA entity อันตรายกว่าที่คิด** — `equals()`/`hashCode()` ที่ generate ใช้ **ทุก field รวม `id`** โดย default พอ entity ใหม่ยังไม่ save `id` เป็น `null` ทำให้ entity ใหม่สองตัวถูกมองว่า "เท่ากัน" ผิด ๆ และพอ save เสร็จ `id` เปลี่ยนจาก `null` เป็นเลขจริง entity ตัวเดิมใน `Set`/`HashMap` จะหา "ตัวเอง" ไม่เจอเพราะ hash เปลี่ยนกลางทาง — แก้ด้วย `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` แล้วเลือกเฉพาะ business key มาใส่ ไม่ใช้ default
- **`@ToString` บน entity ที่มีความสัมพันธ์ (relation)** เสี่ยง `LazyInitializationException` (toString ไปแตะ lazy collection นอก transaction) และเสี่ยง **infinite recursion** ถ้า entity สองฝั่งอ้างกันเองไปมา (bidirectional) — ใช้ `@ToString.Exclude` ตัดฝั่งใดฝั่งหนึ่งออกเสมอ
- **`@Builder` ทำให้ไม่มี no-args constructor ให้อัตโนมัติ** (เหมือนใส่ `@AllArgsConstructor` แฝงอยู่) แต่ JPA/Hibernate ต้องการ no-args constructor เสมอ — ต้องเติม `@NoArgsConstructor` เองคู่กันทุกครั้งที่ใช้ `@Builder` บน entity
- **ลง Lombok แต่ลืมลง IDE plugin** — เห็น error สีแดงเต็มไปหมดทั้งที่ `mvn compile`/`./gradlew build` ผ่านจริง ทำให้เข้าใจผิดว่าโค้ดพัง
- **ใช้ `@Data` กับทุกคลาสโดยไม่คิด** — เหมาะกับ DTO/value object เท่านั้น entity ที่มี relation ควรเลือก annotation ทีละตัว (`@Getter`/`@Setter`/`@NoArgsConstructor` + `@EqualsAndHashCode(onlyExplicitlyIncluded = true)`) แทนการเหมา `@Data` รวดเดียว

---

## 5. Cheat sheet

```java
@Getter @Setter                        // อ่าน/เขียนได้ (ปลอดภัยสุด ใช้ได้แทบทุกที่)
@NoArgsConstructor @AllArgsConstructor // constructor เปล่า + constructor เต็ม
@Builder                                // ต้องคู่กับ @NoArgsConstructor ถ้าใช้กับ JPA entity

@Data                                   // DTO/value object เท่านั้น — ไม่ใช้กับ JPA entity ที่มี relation
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
@ToString.Exclude                       // ตัด field/relation ที่ไม่อยากให้เข้า toString

@Value                                  // immutable ทั้งคลาส (ทางเลือกก่อนมี record ใน [[Java Record]])
@Slf4j                                  // ได้ field `log` ใช้ log.info(...) ได้เลย
```

## 🔗 เกี่ยวข้อง

- [[Java MapStruct]] — ใช้คู่กันบ่อยมาก (Lombok ทำ DTO/entity, MapStruct map ระหว่างกัน) มี gotcha เรื่องลำดับ annotation processor ที่ต้องรู้ก่อนใช้คู่กัน
- [[Java Record]] — ทางเลือกของ JDK เองสำหรับ immutable data class โดยไม่ต้องพึ่ง Lombok เลย (คล้าย `@Value` แต่เป็นของภาษาเอง)

## 📖 อ่านต่อ

- [Project Lombok — Features](https://projectlombok.org/features/)
- [Lombok and JPA: What may go wrong?](https://jpa-buddy.com/blog/lombok-and-jpa-what-may-go-wrong/)
