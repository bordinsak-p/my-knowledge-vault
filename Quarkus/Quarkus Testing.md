---
tags:
  - java
  - quarkus
  - testing
  - junit
type: reference
created: 2026-08-18
---

# 🧪 Quarkus Testing — `@QuarkusTest` และ REST Assured

> **stage "test" ใน CI pipeline ไร้ความหมายถ้าไม่มีเทสให้รันจริง** — pipeline ที่เขียนไว้สวยงามแต่ `mvn test` ไม่เจออะไรให้รัน (หรือแย่กว่านั้นคือเทสถูก comment ทิ้งไว้) จะสีเขียวเสมอไม่ว่าโค้ดจะพังแค่ไหน
> โน้ตนี้คือพื้นฐานพอจะเขียน/เปิดใช้เทสของ Quarkus ให้กลับมามีความหมายจริง

---

## 1. `@QuarkusTest` ทำอะไรให้

```java
@QuarkusTest
class ProductResourceTest {

    @Test
    void listProducts_returns200() {
        given()
          .when().get("/api/products")
          .then()
             .statusCode(200);
    }
}
```

**แค่ใส่ `@QuarkusTest` บนคลาส เทสทั้งไฟล์จะรันโดยมีแอป Quarkus ตัวจริงสตาร์ตขึ้นมารออยู่** — ไม่ใช่ mock, ไม่ใช่ unit test แยกส่วน แต่เป็นแอปที่มี HTTP endpoint จริง ต่อ DB จริง (หรือ DB ทดสอบ) แล้วเทสยิง request เข้าไปเหมือนเป็น client จริง ๆ

```
@Test ตัวแรกในคลาสนี้ ──► Quarkus สตาร์ตแอปขึ้นมาหนึ่งครั้ง (ใช้เวลาหน่อยตอนแรก)
                                    │
        ทุก @Test ในคลาสถัดไป ──────┘ ใช้แอปตัวเดียวกัน ไม่สตาร์ตซ้ำ (เร็วขึ้นเรื่อย ๆ)
```

**Quarkus ค่อนข้างเร็วเรื่องนี้เมื่อเทียบกับ framework อื่น** — เวลาสตาร์ตแอปสำหรับเทสมักเป็นวินาทีเดียว ไม่ใช่สิบวินาทีเหมือนหลาย framework เก่า

---

## 2. Dependency ที่ต้องมี

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-junit5</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <scope>test</scope>
</dependency>
```

โปรเจกต์ที่สร้างผ่าน `code.quarkus.io` มีสองตัวนี้ให้อัตโนมัติอยู่แล้วเกือบทุกครั้ง

---

## 3. REST Assured — ภาษาที่อ่านเหมือนภาษาคน

```java
@Test
void createProduct_returns201() {
    given()
        .contentType(ContentType.JSON)
        .body("""
            { "code": "P001", "name": "Widget", "price": 99.90 }
            """)
    .when()
        .post("/api/products")
    .then()
        .statusCode(201)
        .header("Location", containsString("/api/products/"))
        .body("code", equalTo("P001"))
        .body("price", equalTo(99.9f));
}
```

```
given()   ← เตรียม request (body, header, param)
  .when() ← สั่งยิง (get/post/put/delete)
  .then() ← ตรวจ response (status, body, header)
```

**อ่านออกเสียงได้ตรงตัว: "กำหนดแบบนี้ เมื่อยิงไปแล้ว ต้องได้แบบนี้"** — นี่คือเหตุผลที่ REST Assured เป็นตัวเลือกมาตรฐานสำหรับเทส REST API แทนการเขียน `HttpClient` ดิบ ๆ เอง

### ตรวจ JSON path ได้ตรง ๆ ไม่ต้อง deserialize

```java
.then()
    .body("size()", is(3))                    // array มี 3 ตัว
    .body("[0].code", equalTo("P001"))         // ตัวแรก
    .body("findAll { it.price > 50 }.size()", is(2))  // filter ด้วย GPath
```

---

## 4. เทส layer อื่นที่ไม่ใช่ HTTP — inject bean ตรง ๆ

```java
@QuarkusTest
class ProductServiceTest {

    @Inject ProductService service;

    @Test
    void duplicateCode_throws() {
        service.create(new CreateRequest("P001", "A", BigDecimal.ONE));
        assertThrows(IllegalArgumentException.class,
            () -> service.create(new CreateRequest("P001", "B", BigDecimal.ONE)));
    }
}
```

**`@QuarkusTest` ไม่ได้จำกัดแค่เทส REST endpoint** — inject `@ApplicationScoped` bean ใด ๆ เข้ามาเทสตรง ๆ ได้เหมือนโค้ดจริง เหมาะกับการเทส business logic โดยไม่ต้องผ่าน HTTP layer

---

## 5. Transaction — ไม่ให้เทสทิ้งขยะไว้ใน DB

```java
@Test
@TestTransaction              // ← rollback อัตโนมัติหลังเทสจบ
void create_persists() {
    Product p = service.create(new CreateRequest("P001", "A", BigDecimal.ONE));
    assertNotNull(p.id);
}
```

**`@TestTransaction` ทำให้ทุกอย่างที่เทสเขียนลง DB ถูก rollback ทันทีที่เทสจบ** — เทสตัวถัดไปเริ่มจาก DB สะอาดเสมอ ไม่ต้องเขียนโค้ดล้างข้อมูลเอง และเทสหลายตัวไม่ชนกันเรื่องข้อมูลค้าง

ดูรายละเอียดเรื่อง transaction/persistence context เพิ่มที่ [[Quarkus EntityManager]]

---

## 6. Mock — เฉพาะของที่ควร mock จริง ๆ

```java
@QuarkusTest
class NotificationServiceTest {

    @InjectMock
    EmailClient emailClient;         // ← แทนที่ bean จริงด้วย mock

    @Inject NotificationService service;

    @Test
    void sendsEmail() {
        service.notifyUser("user@example.com", "hello");
        verify(emailClient).send(eq("user@example.com"), any());
    }
}
```

**`@InjectMock` มาจาก `quarkus-junit5-mockito`** ใช้แทน bean จริงด้วย Mockito mock — **ควร mock เฉพาะของที่ไม่อยากให้เกิดผลข้างเคียงจริงตอนเทส** เช่น ส่งอีเมลจริง, เรียก API ภายนอกจริง, เก็บเงินจริง

**ไม่ควร mock DB/repository พร่ำเพรื่อ** — เทสที่ mock DB ไปหมดพิสูจน์ได้แค่ว่า "โค้ดเรียก mock ถูกลำดับ" ไม่ได้พิสูจน์ว่า query จริงทำงานถูก การมี integration test ที่ยิงลง DB จริง (หรือ Dev Services ด้านล่าง) มีค่ามากกว่าในหลายกรณี

---

## 7. Dev Services — DB จริงโดยไม่ต้องตั้งอะไรเอง

**ถ้าไม่ตั้ง `quarkus.datasource.jdbc.url` ไว้ใน test profile เลย Quarkus จะสตาร์ต database ในคอนเทนเนอร์ให้อัตโนมัติตอนรันเทส** (ต้องมี Docker ในเครื่อง/CI) — เทสยิงลง PostgreSQL/MariaDB จริงได้โดยไม่ต้องมี DBติดตั้งไว้ล่วงหน้าเลย

```properties
# ไม่ต้องมีบรรทัดพวกนี้ใน %test — ปล่อยว่างแล้ว Dev Services จัดการเอง
# %test.quarkus.datasource.jdbc.url=...
```

**ข้อดีสำหรับ CI โดยเฉพาะ:** ไม่ต้องมี step ติดตั้ง/รัน database เองใน pipeline ก่อนเทส — Quarkus เรียก Docker เองผ่าน Testcontainers ข้างใต้

> ต้องมี Docker daemon ให้เรียกใช้ในสภาพแวดล้อมที่รันเทส — บน GitHub Actions cloud runner มี Docker ให้อยู่แล้วเป็นค่าเริ่มต้น แต่ถ้าใช้ [[Self-Hosted Runners]] ต้องเช็คว่าเครื่องนั้นมี Docker พร้อมใช้ด้วย

---

## 8. เอาเทสที่ถูก comment ไว้กลับมาให้รันได้ — ขั้นตอนที่แนะนำ

โค้ดเก่าจำนวนมากมีเทสที่ถูก comment ทิ้งไว้ทั้งไฟล์ (มักเพราะเคย build ไม่ผ่านแล้วไม่มีใครแก้) นี่คือลำดับที่แนะนำแทนที่จะลบทิ้งหรือ uncomment รวดเดียวทั้งหมด:

```
1. Uncomment ทีละไฟล์ ไม่ใช่ทั้งหมดพร้อมกัน
   (ถ้า uncomment รวดเดียวแล้วพัง 20 ที่ จะไม่รู้ว่าเริ่มไล่จากไหน)

2. รันแยกเดี่ยว ๆ ก่อน (mvn test -Dtest=ProductResourceTest)
   ดูว่า error จริงคืออะไร — บางทีแค่ import ผิด บางทีแอปเปลี่ยนไปมากจนเทสล้าสมัย

3. ถ้าเทสทดสอบพฤติกรรมที่ไม่มีอยู่แล้ว (โค้ดเปลี่ยนไปมาก) — ลบเทสนั้นทิ้งไปเลย
   ไม่ต้องพยายามซ่อมเทสที่ทดสอบของที่ไม่มีอยู่แล้ว

4. ถ้าเทสยัง valid แต่ผูกกับสภาพแวดล้อมที่ไม่มีแล้ว (DB ตายตัว, mock เก่า)
   ปรับให้ใช้ Dev Services (ข้อ 7) แทนการพึ่ง DB ภายนอกที่ตั้งค่าตายตัวไว้

5. เมื่อไฟล์หนึ่งผ่านแล้ว commit แยกต่างหาก อย่ารวมกับการ uncomment ไฟล์อื่น
   จะได้ bisect ย้อนกลับได้ง่ายถ้ามีอะไรพังทีหลัง

6. เปิด stage test ใน CI pipeline ให้ "fail จริง" เมื่อเทส fail
   ไม่ใช่แค่รันแล้วดู log เฉย ๆ — บาง config ตั้ง continue-on-error ไว้โดยไม่ตั้งใจ
   ทำให้ทั้ง pipeline สีเขียวแม้เทส fail จริง
```

**ข้อ 6 คือจุดที่พลาดบ่อยที่สุดหลังจากอุตส่าห์ uncomment เทสสำเร็จ** — เช็ค [[GitHub Actions Workflow Syntax]] ว่า step รันเทสไม่มี `continue-on-error: true` หรือ `if: always()` ที่ทำให้ pipeline มองข้าม failure ไป

---

## 9. Test profile — แยกค่าที่ใช้เฉพาะตอนเทส

```properties
%test.quarkus.log.level=WARN
%test.some.feature.enabled=false
```

เหมือนหลักการเดียวกับ `%dev`/`%prod` — ค่าที่ควรต่างกันเฉพาะตอนรันเทส (ปิด log ที่ไม่จำเป็น, ปิดฟีเจอร์ที่ยังไม่เกี่ยวกับเทสนี้) แยกไว้ใน `%test.` prefix

---

## 10. กับดัก

- **เทสถูก comment ทิ้งไว้เฉย ๆ นาน ๆ** — เท่ากับไม่มีเทส แต่ CI ยัง "ดูเหมือน" มี stage test อยู่ หลอกคนอ่านว่าระบบมีการทดสอบคุ้มครองอยู่
- **mock ทุกอย่างจนเทสไม่ได้พิสูจน์อะไรจริง** — เทสผ่านสวยงามแต่ของจริงพังเพราะไม่เคยแตะ DB/service จริงเลย
- **ไม่ใช้ `@TestTransaction`** — ข้อมูลจากเทสตัวหนึ่งค้างอยู่ กระทบเทสตัวถัดไป ทำให้ผลไม่แน่นอน (บางทีผ่าน บางทีไม่ผ่าน ขึ้นกับลำดับรัน)
- **CI stage test ตั้ง `continue-on-error`** — เทส fail จริงแต่ pipeline ยังสีเขียว หลอกทุกคนว่าไม่มีปัญหา
- **ไม่มี Docker ในสภาพแวดล้อมที่รันเทส แต่พึ่ง Dev Services** — เทสพังตอน CI ทั้งที่ผ่านตอน local เพราะ self-hosted runner บางเครื่องไม่มี Docker
- **uncomment เทสทั้งหมดรวดเดียว** — พังหลายสิบจุดพร้อมกัน หาต้นตอไม่ได้ว่าอะไรทำให้พังตัวไหน

---

## 11. Cheat sheet

```java
@QuarkusTest
class MyTest {

    @Inject MyService service;

    @InjectMock ExternalClient externalClient;   // mock เฉพาะของภายนอก

    @Test
    @TestTransaction                              // rollback อัตโนมัติ
    void test() {
        given().contentType(JSON).body(payload)
        .when().post("/api/x")
        .then().statusCode(201).body("id", notNullValue());
    }
}
```

```bash
mvn test                          # รันทั้งหมด
mvn test -Dtest=ProductResourceTest   # รันไฟล์เดียว
mvn test -Dtest=ProductResourceTest#createProduct_returns201   # รันเทสเดียว
```

| อาการ | สาเหตุ |
|---|---|
| pipeline เขียวทั้งที่เทส fail จริง | CI step ตั้ง `continue-on-error`/`if: always()` |
| เทสผ่านบาง ครั้งไม่ผ่านบางครั้ง | ไม่ได้ `@TestTransaction` ข้อมูลค้างข้ามเทส |
| เทสผ่าน local แต่พังบน CI | ไม่มี Docker ให้ Dev Services ใช้บน runner นั้น |
| uncomment เทสแล้วพังเป็นสิบจุด | uncomment ทีเดียวทั้งหมดแทนที่จะทีละไฟล์ |
| เทสผ่านสวยแต่ของจริงพัง | mock ทุกอย่างจนไม่เหลือ integration test จริง |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus]] — หน้ารวม
- [[Quarkus EntityManager]] — transaction/persistence context ที่ `@TestTransaction` พึ่งพา
- [[CI-CD]] — ทำไม stage test ต้องมีเทสจริงให้รัน
- [[GitHub Actions Workflow Syntax]] — เช็คว่า step test ไม่ได้ตั้ง continue-on-error

## 📖 อ่านต่อ

- [Quarkus — Testing your application](https://quarkus.io/guides/getting-started-testing)
- [Quarkus — Continuous testing](https://quarkus.io/guides/continuous-testing)
