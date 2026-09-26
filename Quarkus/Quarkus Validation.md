---
tags:
  - quarkus
  - java
  - validation
type: reference
created: 2026-09-20
---

# ✅ Quarkus Validation — Bean Validation และ Custom Validator

> Quarkus ใช้ **Jakarta Bean Validation** (implementation: Hibernate Validator) ผ่าน extension `quarkus-hibernate-validator` — ใส่ annotation บน field แล้ว Quarkus ตรวจให้อัตโนมัติตอนมี request เข้ามาที่ REST endpoint ไม่ต้องเขียน if-check เองทีละฟิลด์

---

## 1. ติดตั้ง

```bash
quarkus extension add hibernate-validator
```

---

## 2. Built-in constraint annotation ที่ใช้บ่อย

```java
public class RegisterRequest {
    @NotBlank
    public String username;

    @Email
    public String email;

    @Size(min = 8, max = 100)
    public String password;

    @Min(18)
    public int age;

    @Pattern(regexp = "^[0-9]{10}$")
    public String phoneNumber;
}
```

| annotation | เช็คอะไร |
|---|---|
| `@NotNull` | ไม่ใช่ `null` |
| `@NotEmpty` | ไม่ใช่ `null` และไม่ว่างเปล่า (string ว่าง `""`/list ว่าง `[]` ไม่ผ่าน) |
| `@NotBlank` | เหมือน `@NotEmpty` แต่**เฉพาะ String** และตัด whitespace ก่อนเช็คด้วย (`"   "` ไม่ผ่าน) |
| `@Size(min, max)` | ความยาว string/ขนาด collection อยู่ในช่วง |
| `@Min`/`@Max` | ค่าตัวเลขอยู่ในช่วง |
| `@Email` | รูปแบบอีเมล |
| `@Pattern(regexp = ...)` | ตรงกับ regex ที่กำหนด |
| `@Positive`/`@Negative` | ค่าเป็นบวก/ลบ |
| `@Past`/`@Future` | วันที่อยู่ในอดีต/อนาคต |

### `@NotNull` vs `@NotEmpty` vs `@NotBlank` — สับสนกันบ่อยที่สุด

| ค่า | `@NotNull` | `@NotEmpty` | `@NotBlank` |
|---|---|---|---|
| `null` | ❌ ไม่ผ่าน | ❌ ไม่ผ่าน | ❌ ไม่ผ่าน |
| `""` (string ว่าง) | ✅ ผ่าน | ❌ ไม่ผ่าน | ❌ ไม่ผ่าน |
| `"   "` (whitespace ล้วน) | ✅ ผ่าน | ✅ ผ่าน | ❌ ไม่ผ่าน |
| `"abc"` | ✅ ผ่าน | ✅ ผ่าน | ✅ ผ่าน |

**เกือบทุกกรณีของ field ที่เป็น String ที่ต้องการค่าจริงๆ ควรใช้ `@NotBlank`** ไม่ใช่ `@NotNull` — `@NotNull` เดี่ยวๆ ปล่อยให้ค่า `""` ผ่านไปได้ทั้งที่ไม่มีความหมายอะไรเลย

---

## 3. `@Valid` — จุดที่ทำให้ validation ทำงานจริงบน REST endpoint

```java
@POST
@Path("/register")
public Response register(@Valid RegisterRequest request) {
    // ถ้ามาถึงบรรทัดนี้ = ผ่าน validation ทุกกฎแล้ว
    userService.register(request);
    return Response.ok().build();
}
```

**ไม่มี `@Valid` = annotation บน field ทั้งหมดไม่ทำงานเลย** — Quarkus ตรวจ Bean Validation เฉพาะตอนที่ endpoint parameter ถูก mark `@Valid` เท่านั้น ไม่ได้ตรวจอัตโนมัติทุก object เสมอไป

**Request ที่ไม่ผ่าน → โยน `ConstraintViolationException` → ได้ `400` พร้อม JSON บอกว่าฟิลด์ไหนผิดเงื่อนไขอะไร** (ปรับ format เองได้ผ่าน custom mapper ดู [[Quarkus Exception Mapper]] ข้อ 6)

### `@Valid` cascade เข้า nested object ด้วย

```java
public class OrderRequest {
    @Valid                    // ⚠️ ต้องใส่ ไม่งั้น constraint ข้างใน CustomerInfo จะไม่ถูกตรวจเลย
    public CustomerInfo customer;
}

public class CustomerInfo {
    @NotBlank
    public String name;
}
```

ไม่ใส่ `@Valid` ที่ field `customer` — แม้ `CustomerInfo.name` จะมี `@NotBlank` ก็จะ**ไม่ถูกตรวจ** เพราะ `@Valid` เป็นตัวบอกให้ "ไต่ลงไปตรวจ object ข้างในต่อด้วย"

---

## 4. Custom Validator — ตอน built-in ไม่พอ

### ตัวอย่าง: `@UniqueUsername` — validator ที่ต้องเช็คกับ database

```java
@Target({ ElementType.FIELD, ElementType.PARAMETER })
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueUsernameValidator.class)
public @interface UniqueUsername {
    String message() default "Username นี้ถูกใช้ไปแล้ว";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

```java
@ApplicationScoped   // ✅ ทำให้เป็น CDI bean — inject dependency ได้
public class UniqueUsernameValidator implements ConstraintValidator<UniqueUsername, String> {

    @Inject
    UserRepository userRepository;

    @Override
    public boolean isValid(String username, ConstraintValidatorContext context) {
        if (username == null) return true;   // ปล่อยให้ @NotNull จัดการกรณี null แยกต่างหาก
        return !userRepository.existsByUsername(username);
    }
}
```

```java
public class RegisterRequest {
    @NotBlank
    @UniqueUsername   // ใช้ร่วมกับ built-in annotation ได้ปกติ
    public String username;
}
```

**Quarkus ผูก Hibernate Validator เข้ากับ CDI ให้แล้ว** — custom validator ที่ mark `@ApplicationScoped` `@Inject` service/repository เข้ามาใช้ได้ตรงๆ (เช่นเช็คว่ามี record ซ้ำใน DB ไหม) ไม่ต้องหา instance เองแบบ manual — แนะนำให้ validator เองมี logic น้อยที่สุด แล้ว inject service ที่มี logic จริงเข้ามาแทน (validator ควรบางแค่ "ถามแล้วตอบ true/false")

### Class-level validation — เช็คข้ามหลาย field พร้อมกัน

```java
@Target(ElementType.TYPE)   // ผูกกับทั้งคลาส ไม่ใช่ field เดียว
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordMatchValidator.class)
public @interface PasswordMatch {
    String message() default "รหัสผ่านไม่ตรงกัน";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

```java
public class PasswordMatchValidator implements ConstraintValidator<PasswordMatch, RegisterRequest> {
    @Override
    public boolean isValid(RegisterRequest request, ConstraintValidatorContext context) {
        return request.password.equals(request.confirmPassword);
    }
}
```

```java
@PasswordMatch   // annotate ที่ตัวคลาส ไม่ใช่ field
public class RegisterRequest {
    public String password;
    public String confirmPassword;
}
```

`ConstraintValidator<PasswordMatch, RegisterRequest>` รับทั้ง**คลาส**เป็น type argument ตัวที่สอง (ไม่ใช่ `String` แบบ field-level validator) เพราะต้องเห็นทุก field พร้อมกันถึงจะเทียบกันได้

---

## กับดัก

- **ลืม `@Valid` ที่ endpoint parameter** — annotation บน field ทั้งหมดไม่ทำงานเลย แต่ไม่มี error ให้เห็นตอน compile ทำให้เข้าใจผิดว่า validate อยู่แล้ว
- **ลืม `@Valid` ที่ nested object field** — constraint ข้างใน object ลูกไม่ถูกตรวจ (ข้อ 3) ทั้งที่ตัว object แม่ผ่าน `@Valid` แล้ว
- **ใช้ `@NotNull` แทน `@NotBlank` กับ String ที่ต้องการค่าจริง** — ค่าว่าง `""`/whitespace ผ่านไปได้ทั้งที่ไม่ควร (ข้อ 2)
- **ใส่ business logic หนักๆ ลงใน custom validator โดยตรงแทนที่จะ inject service** — ทำให้ validator เทสยากและปนกับ business logic ที่ควรอยู่ service layer
- **`isValid()` ไม่เช็ค `null` ก่อน แล้วโยน `NullPointerException`** — ควร return `true` ทันทีถ้าค่าเป็น `null` แล้วปล่อยให้ `@NotNull`/`@NotBlank` (แยก annotation ต่างหาก) จัดการกรณี null เอง เป็น convention มาตรฐานของ Bean Validation
- **คาดหวังว่า validation จะรันอัตโนมัติทุกครั้งที่ save entity ผ่าน Hibernate โดยไม่ต้องทำอะไรเพิ่ม** — Hibernate ORM มี auto-validate ก่อน insert/update ให้จริง (ถ้ามี extension นี้อยู่) แต่เป็นคนละจุดกับ validation ที่ REST layer ผ่าน `@Valid` — พึ่งจุดเดียวไม่พอถ้าข้อมูลเข้ามาจากทางอื่นที่ไม่ผ่าน REST endpoint

---

## Cheat sheet

```java
@NotNull / @NotEmpty / @NotBlank
@Size(min = 1, max = 100)
@Min(0) / @Max(100)
@Email
@Pattern(regexp = "...")
```

```java
public Response endpoint(@Valid RequestDto dto) { ... }   // ต้องมี @Valid ถึงจะตรวจจริง
```

```java
@Constraint(validatedBy = MyValidator.class)
public @interface MyConstraint {
    String message() default "...";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@ApplicationScoped
public class MyValidator implements ConstraintValidator<MyConstraint, String> {
    @Inject SomeService service;
    public boolean isValid(String value, ConstraintValidatorContext ctx) { return service.check(value); }
}
```

## 🔗 เกี่ยวข้อง

- [[Quarkus Exception Mapper]] — จัดการ `ConstraintViolationException` ให้ตรง error contract ของ API เอง
- [[Dependency Injection]] — custom validator เป็น CDI bean inject service เข้ามาได้เหมือน bean ทั่วไป

## 📖 อ่านต่อ

- [Quarkus — Validation with Hibernate Validator](https://quarkus.io/guides/validation)
- [Jakarta Bean Validation — Specification](https://beanvalidation.org/)
