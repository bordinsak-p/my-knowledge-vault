---
tags:
  - quarkus
  - java
  - rest
type: reference
created: 2026-09-20
---

# 🧯 Quarkus — Custom `ExceptionMapper`

> **`ExceptionMapper` แปลง exception ที่โยนออกมาจากโค้ด endpoint ให้กลายเป็น HTTP response ตามรูปแบบที่เรากำหนดเอง** แทนที่จะปล่อยให้หลุดออกไปเป็น error response แบบ default — นี่คือฝั่ง **server** (endpoint ของเราเองโยน exception) ต่างจาก [[Quarkus REST Client]] ที่เป็นฝั่ง **client** (เรียก API คนอื่นแล้วแปลง response error ที่ได้กลับมาเป็น exception)

---

## 1. ปัญหาที่มันแก้

ไม่มี custom mapper → exception ที่โยนออกจาก endpoint กลายเป็น HTTP 500 พร้อม JSON error แบบ default ของ Quarkus (dev mode โชว์ stack trace เต็ม, prod โชว์ข้อความทั่วไป) — ไม่ตรงกับ error contract ที่ API ควรมี (เช่น field `error`/`message`/`code` ที่ frontend คาดหวัง) และเสี่ยงหลุด information ที่ไม่ควรให้เห็นออกไปด้วย

---

## 2. Interface พื้นฐาน

```java
public interface ExceptionMapper<E extends Throwable> {
    Response toResponse(E exception);
}
```

---

## 3. ตัวอย่างเต็ม — custom exception + mapper

```java
// exception ของ domain เราเอง
public class ProductNotFoundException extends RuntimeException {
    public ProductNotFoundException(Long id) {
        super("Product not found: " + id);
    }
}
```

```java
@Provider   // ⚠️ ขาดไม่ได้ — บอก JAX-RS ให้ discover mapper นี้อัตโนมัติ
public class ProductNotFoundExceptionMapper implements ExceptionMapper<ProductNotFoundException> {

    @Override
    public Response toResponse(ProductNotFoundException exception) {
        return Response.status(Response.Status.NOT_FOUND)
                .entity(new ErrorResponse("product_not_found", exception.getMessage()))
                .build();
    }
}
```

```java
public record ErrorResponse(String code, String message) { }
```

ตอนนี้ endpoint ไหนก็ตามที่โยน `ProductNotFoundException` จะได้ `404` พร้อม JSON `{"code": "product_not_found", "message": "..."}` โดยไม่ต้องเขียน try-catch ซ้ำในทุก endpoint เอง — เขียน mapper ครั้งเดียวใช้ได้ทั้งแอป

---

## 4. Exception hierarchy — JAX-RS เลือก mapper ที่ "ใกล้ที่สุด" เสมอ

**กฎ: หาตัวที่ตรงกับ class ของ exception เป๊ะก่อน ถ้าไม่มีค่อยไต่ขึ้นไปหา superclass ที่ใกล้ที่สุดที่มี mapper** — ไม่ใช่สุ่มหรือตามลำดับที่ประกาศ

```java
class AppException extends RuntimeException { }
class ProductNotFoundException extends AppException { }
```

ถ้ามี mapper ทั้งของ `AppException` และ `ProductNotFoundException` — โยน `ProductNotFoundException` จะโดน mapper ของ **`ProductNotFoundException` เสมอ** (ตรงเป๊ะกว่า) ไม่ใช่ของ `AppException` แม้จะเป็น parent ที่กว้างกว่าก็ตาม

---

## 5. Catch-all mapper — กันไม่ให้อะไรหลุดออกไปแบบดิบๆ

```java
@Provider
public class GenericExceptionMapper implements ExceptionMapper<Exception> {

    private static final Logger LOG = Logger.getLogger(GenericExceptionMapper.class);

    @Override
    public Response toResponse(Exception exception) {
        LOG.error("Unhandled exception", exception);   // log stack trace เต็มไว้ฝั่ง server เท่านั้น
        return Response.status(500)
                .entity(new ErrorResponse("internal_error", "เกิดข้อผิดพลาดที่ไม่คาดคิด"))
                .build();
    }
}
```

จับ `Exception` (หรือ `Throwable` ถ้าอยากครอบคลุม `Error` ด้วย) เป็น**ตัวสุดท้าย** — exception เฉพาะทางที่มี mapper ของตัวเองจะยังโดน mapper เฉพาะทางก่อนเสมอ (ข้อ 4) ตัวนี้ทำหน้าที่แค่ "กันหลุด" ไม่ให้ error ที่ไม่คาดคิดส่ง stack trace ดิบๆ กลับไปหา client

---

## 6. ตัวอย่างที่ใช้บ่อยจริง — validation exception

```java
@Provider
public class ConstraintViolationExceptionMapper implements ExceptionMapper<ConstraintViolationException> {

    @Override
    public Response toResponse(ConstraintViolationException exception) {
        var errors = exception.getConstraintViolations().stream()
                .map(v -> v.getPropertyPath() + ": " + v.getMessage())
                .toList();
        return Response.status(400)
                .entity(new ErrorResponse("validation_failed", String.join(", ", errors)))
                .build();
    }
}
```

`@Valid`/Bean Validation โยน `ConstraintViolationException` เวลา request body ไม่ผ่านกฎ (`@NotNull`, `@Size` ฯลฯ) — ถ้าไม่ map เอง Quarkus ใช้ error format default ของตัวเองซึ่งมักไม่ตรงกับ contract ของ API เรา

---

## 7. จะรู้ได้ยังไงว่าควรทำ mapper ให้ exception ตัวไหน

### 7.1 เช็กก่อน — JAX-RS มี exception สำเร็จรูปให้แล้ว ไม่ต้อง custom เสมอไป

`jakarta.ws.rs.WebApplicationException` เป็นฐานของ exception สำเร็จรูปที่ **โยนตรงๆ ได้เลยจาก endpoint โดยไม่ต้องเขียน mapper เอง** — Quarkus แปลงเป็น HTTP status ที่ถูกต้องให้อัตโนมัติเพราะชื่อ exception ตรงกับความหมาย status นั้นอยู่แล้ว:

```java
if (product == null) {
    throw new NotFoundException("Product not found");   // ได้ 404 ทันที ไม่ต้องเขียน mapper เอง
}
```

ตัวที่ใช้บ่อย: `NotFoundException` (404), `BadRequestException` (400), `ForbiddenException` (403), `NotAuthorizedException` (401) — **ใช้ custom exception ต่อเมื่อ** ต้องการ error body ที่มีโครงสร้างเฉพาะ (ไม่ใช่แค่ plain text), ต้องแนบข้อมูลเพิ่มเติม (เช่น field ไหนผิด/id ไหนหาไม่เจอ), หรือ status ที่ built-in ไม่มีให้ตรงกับที่ต้องการ

### 7.2 เป็น exception ของ library/framework — หาให้เจอว่าเขาโยนอะไรจริง

**วิธีที่แม่นสุด: รันแล้วดู stack trace จริงตอน dev mode** — ไม่ต้องเดา อ่านชื่อ exception class เต็ม (fully-qualified) ตรงๆ จาก error ที่เกิดขึ้นจริง แล้วค่อยเขียน mapper ให้ตรงตัวนั้น

ที่เจอบ่อยในโปรเจกต์ Quarkus:

| มาจากไหน | exception ที่มักเจอ |
|---|---|
| Bean Validation (`@NotNull`, `@Size` ฯลฯ) | `jakarta.validation.ConstraintViolationException` |
| Hibernate/JPA เขียนข้อมูลชน constraint ใน DB (unique, foreign key) | `org.hibernate.exception.ConstraintViolationException` |
| optimistic locking ชนกัน | `jakarta.persistence.OptimisticLockException` |
| หา entity ไม่เจอผ่านบาง method ของ EntityManager | `jakarta.persistence.EntityNotFoundException` |
| JSON body แปลงไม่ได้ | `com.fasterxml.jackson.core.JsonProcessingException` |

**⚠️ กับดักตัวใหญ่: `ConstraintViolationException` มีสองคลาสคนละความหมาย ชื่อซ้ำกันเฉยๆ** — `jakarta.validation.ConstraintViolationException` (Bean Validation ตรวจก่อนถึง DB) กับ `org.hibernate.exception.ConstraintViolationException` (DB ปฏิเสธจริงๆ ตอนเขียนข้อมูล เช่น username ซ้ำ) — import ผิดตัวแล้ว mapper จะเงียบไม่ทำงานเลย เพราะ exception ที่โยนจริงเป็นคนละ class กับที่ mapper ประกาศไว้

### 7.3 เป็น business logic ของเราเอง — ออกแบบ exception ตามที่ผู้ใช้ต้อง "รู้" จริงๆ

ถามตัวเองว่า **"เคสนี้ผู้เรียก API ต้องรู้อะไรเป็นพิเศษไหม"** ถ้าใช่ ค่อยออกแบบ custom exception ที่พกข้อมูลนั้นติดตัว (เช่น `ProductNotFoundException` เก็บ id ที่หาไม่เจอไว้ในตัว)

**ไม่ต้องเขียน mapper แยกทุก exception** — ทำ base exception กลางที่พก status/code ติดตัวไปเลย แล้วเขียน mapper แค่ตัวเดียวจับ base class (อาศัยกฎ nearest-superclass ในข้อ 4 — ตราบใดที่ไม่มี subclass ไหนมี mapper เฉพาะของตัวเอง จะโดน mapper ของ base class นี้หมด):

```java
public abstract class AppException extends RuntimeException {
    private final Response.Status status;
    private final String code;
    protected AppException(Response.Status status, String code, String message) {
        super(message);
        this.status = status;
        this.code = code;
    }
    public Response.Status getStatus() { return status; }
    public String getCode() { return code; }
}

public class ProductNotFoundException extends AppException {
    public ProductNotFoundException(Long id) {
        super(Response.Status.NOT_FOUND, "product_not_found", "Product not found: " + id);
    }
}

public class InsufficientBalanceException extends AppException {
    public InsufficientBalanceException() {
        super(Response.Status.CONFLICT, "insufficient_balance", "Not enough balance");
    }
}

@Provider
public class AppExceptionMapper implements ExceptionMapper<AppException> {
    public Response toResponse(AppException e) {
        return Response.status(e.getStatus()).entity(new ErrorResponse(e.getCode(), e.getMessage())).build();
    }
}
```

exception domain ใหม่ที่เพิ่มทีหลัง = สืบทอดจาก `AppException` แล้วจบ **ไม่ต้องเขียน mapper เพิ่มอีกสักตัว**

### 7.4 สรุปเป็นลำดับการตัดสินใจ

1. JAX-RS มี built-in exception ที่ตรงพอดีไหม (`NotFoundException`, `BadRequestException` ฯลฯ) → ใช้เลย ไม่ต้อง custom (ข้อ 7.1)
2. เป็น business logic ของเราเอง ต้องการข้อมูล/status เฉพาะ → สร้าง custom exception สืบทอดจาก base exception กลาง (ข้อ 7.3)
3. เป็น exception จาก library ที่ Quarkus map default ไม่ตรงกับ contract ที่ต้องการ → ดู stack trace จริง หา fully-qualified class name แล้วเขียน mapper เฉพาะให้ตัวนั้น (ข้อ 7.2)
4. ที่เหลือทั้งหมดที่ไม่คาดคิด → catch-all (ข้อ 5)

---

## กับดัก

- **ลืม `@Provider`** — mapper ไม่ถูก JAX-RS runtime discover เลย exception ยังหลุดออกไปเป็น response แบบ default เหมือนไม่มี mapper อยู่จริง
- **implement `ExceptionMapper<RuntimeException>` (กว้างเกินไป) แล้วงงว่าทำไม exception เฉพาะทางที่ควรมี response ต่างกันได้ response เหมือนกันหมด** — ควรเขียน mapper แยกตาม exception type ที่เจาะจง แล้วใช้ catch-all (ข้อ 5) เป็นแค่ทางสุดท้ายจริงๆ
- **catch-all mapper ไม่ log อะไรเลย** — exception ที่ไม่คาดคิดหายไปเงียบๆ กลายเป็นแค่ "internal_error" ที่ client เห็น โดยไม่มีร่องรอยฝั่ง server ให้ debug เลย ต้อง log เต็มไว้เสมอก่อน return response ที่ตัดข้อมูลออก
- **เผลอใส่ stack trace หรือ exception message ดิบๆ ลง response ของ production** — ข้อความ exception บางตัวอาจมีข้อมูล sensitive (เช่น SQL query, path ภายใน) หลุดไปให้ client เห็น ควรกรอง/เขียน message ใหม่เองใน mapper แทนที่จะใช้ `exception.getMessage()` ตรงๆ เสมอไป
- **สร้าง `ErrorResponse` คนละรูปแบบในแต่ละ mapper** — ทำให้ error contract ของ API ไม่สม่ำเสมอ ควรใช้ DTO เดียวกันกับทุก mapper (ข้อ 3)

---

## Cheat sheet

```java
@Provider
public class MyExceptionMapper implements ExceptionMapper<MyException> {
    @Override
    public Response toResponse(MyException e) {
        return Response.status(400).entity(new ErrorResponse("code", e.getMessage())).build();
    }
}
```

| ต้องการ | ทำ |
|---|---|
| แปลง exception เฉพาะทางเป็น response ที่ต้องการ | implement `ExceptionMapper<เฉพาะทาง>` + `@Provider` |
| กันไม่ให้ error ที่ไม่คาดคิดหลุดเป็น stack trace | catch-all mapper บน `Exception` (ข้อ 5) |
| จัดการ validation error ให้ตรง contract | mapper บน `ConstraintViolationException` (ข้อ 6) |

## 🔗 เกี่ยวข้อง

- [[Quarkus REST Client]] — `ExceptionMapper` ฝั่ง client (`@ClientExceptionMapper`) แปลง response error ขาเข้าเป็น exception แทน
- [[Quarkus REST Layer]] — RESTEasy Classic vs Quarkus REST ที่ `ExceptionMapper` ใช้ interface เดียวกันทั้งคู่
- [[Java Exception]] — checked vs unchecked, exception hierarchy พื้นฐานที่กฎ "nearest superclass" ในข้อ 4 อ้างอิงอยู่
- [[Quarkus Validation]] — ต้นตอของ `ConstraintViolationException` ที่ mapper ในข้อ 6 จัดการ
- [[Quarkus Response]] — `Response`/`RestResponse<T>` แบบเต็ม ที่ `toResponse()` ใช้ return

## 📖 อ่านต่อ

- [Quarkus — Writing REST services](https://quarkus.io/guides/rest)
- [Jakarta REST — ExceptionMapper Javadoc](https://jakarta.ee/specifications/restful-ws/3.1/apidocs/jakarta.ws.rs/jakarta/ws/rs/ext/exceptionmapper)
