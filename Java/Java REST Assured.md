---
tags:
  - java
  - testing
  - rest-assured
type: reference
created: 2026-09-20
---

# 🎯 REST Assured — `given()`/`when()`/`then()` และ Hamcrest matcher

> **REST Assured เป็น DSL สำหรับเทส REST API เขียนแบบอ่านออกเสียงได้** ("กำหนดแบบนี้ เมื่อยิงไปแล้ว ต้องได้แบบนี้") ทำงาน**บน** JUnit อีกที ไม่ใช่ของแทนกัน — เทสยังเป็น `@Test` ปกติ (ดู [[Java JUnit 5]]) แค่ตัวเนื้อในเทสเขียนด้วย DSL นี้แทนการเรียก `HttpClient`/`assertEquals` ตรง ๆ
>
> โน้ตนี้เจาะเฉพาะ**ตัวเลือกที่มีให้ใช้ใน `given()`** และ**รายการ matcher ที่ใช้ตรวจ response** — ถ้าอยากดูตัวอย่างเต็มในบริบท `@QuarkusTest` จริง ดู [[Quarkus Testing]] ข้อ 3

---

## 1. โครงหลัก

```java
given()                                    // เตรียม request
    .contentType(ContentType.JSON)
    .body(payload)
.when()
    .post("/api/products")                 // สั่งยิง
.then()
    .statusCode(201)                       // ตรวจ response
    .body("code", equalTo("P001"));
```

ต้อง static import สองที่เสมอ ไม่งั้น `given()`/`equalTo()` แดงทันที:

```java
import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;
```

---

## 2. `given()` — เตรียม request มีอะไรให้ใช้บ้าง

| method | ใช้ทำอะไร |
|---|---|
| `contentType(ContentType.JSON)` | บอก server ว่า body ที่ส่งไปเป็น type ไหน |
| `accept(ContentType.JSON)` | บอก server ว่าอยากได้ response กลับมาเป็น type ไหน |
| `header("X-Api-Key", value)` / `headers(map)` | ใส่ HTTP header เดี่ยว/หลายอันพร้อมกัน |
| `queryParam("page", 2)` | บังคับเป็น query string เสมอ (`?page=2`) ไม่ว่า method ไหน |
| `pathParam("id", 5)` | แทนค่าใน path template — คู่กับ `.when().get("/products/{id}")` |
| `formParam("username", "a")` | บังคับเป็น form field เสมอ (`application/x-www-form-urlencoded`) |
| `param("q", "widget")` | query หรือ form param **แล้วแต่ HTTP method** (GET → query, POST → form) |
| `body(payload)` | ตั้ง request body — รับ String/POJO (serialize อัตโนมัติ)/`byte[]` |
| `cookie("session", token)` | แนบ cookie |
| `auth().basic(user, pass)` / `.oauth2(token)` / `.preemptive().basic(...)` | ใส่ auth header ตามสกีมา |
| `multiPart("file", file, "image/png")` | อัปโหลดไฟล์แบบ multipart form data |
| `relaxedHTTPSValidation()` | ข้าม SSL cert verification — **ใช้เฉพาะ test/dev environment เท่านั้น** |
| `log().all()` / `log().ifValidationFails()` | print request/response เต็ม ๆ ออก console (ตัวหลังพิมพ์เฉพาะตอน assertion fail — มีประโยชน์กว่าเวลามีเทสเยอะ) |

**`RequestSpecBuilder`** ใช้รวม config ที่ซ้ำกันทุกเทส (base URI, header มาตรฐาน) ไว้ที่เดียว แล้วส่งเป็น spec เดียวแทนการเขียน `given()...` ยาว ๆ ซ้ำทุกไฟล์:

```java
RequestSpecification spec = new RequestSpecBuilder()
    .setBaseUri("http://localhost:8080")
    .setContentType(ContentType.JSON)
    .build();

given().spec(spec).when().get("/products").then().statusCode(200);
```

---

## 3. `when()` — HTTP verb

```java
.when().get("/products")
.when().post("/products")
.when().put("/products/1")
.when().patch("/products/1")
.when().delete("/products/1")
```

รายละเอียด method ไหน safe/idempotent/ควรมี body ดู [[Quarkus HTTP Methods]]

---

## 4. `then()` — ตรวจ response (นี่คือ "assert" ของ REST Assured)

```java
.then()
    .statusCode(200)
    .contentType(ContentType.JSON)
    .header("Location", containsString("/products/"))
    .body("code", equalTo("P001"))
    .time(lessThan(2000L));           // response ต้องเร็วกว่า 2 วิ
```

**REST Assured ไม่มี `assertEquals` เป็นของตัวเอง — ทุกจุดตรวจรับ Hamcrest `Matcher` เป็น argument ทั้งหมด** ต่างจาก JUnit ที่เขียน `assertEquals(expected, actual)` ตรง ๆ (ดู [[Java JUnit 5]] ข้อ 3) — สไตล์นี้ทำให้เขียนเงื่อนไขซับซ้อน (ขนาด list, ทุก element ต้องผ่านเงื่อนไข, ค่าต้องอยู่ในช่วง) ได้ในบรรทัดเดียวโดยไม่ต้องดึงค่าออกมาเช็คเอง

### ตรวจ JSON path ได้ตรง ๆ ไม่ต้อง deserialize

```java
.then()
    .body("size()", is(3))                             // array มี 3 ตัว
    .body("[0].code", equalTo("P001"))                  // element แรก
    .body("findAll { it.price > 50 }.size()", is(2))    // filter ด้วย GPath syntax
```

---

## 5. Hamcrest matcher ที่ใช้บ่อยกับ REST Assured

| ต้องการเช็ค | matcher |
|---|---|
| เท่ากับค่านี้ | `equalTo(x)` หรือ `is(x)` (`is` เป็น syntactic sugar ของ `equalTo` ล้วน ๆ) |
| กลับด้าน/ไม่เท่ากับ | `not(matcher)` |
| string มีคำนี้อยู่ | `containsString(x)` |
| ขึ้นต้น/ลงท้ายด้วย | `startsWith(x)` / `endsWith(x)` |
| ไม่เป็น null / เป็น null | `notNullValue()` / `nullValue()` |
| ขนาด array/collection | `hasSize(n)` |
| collection มี element นี้อยู่ | `hasItem(x)` / `hasItems(x, y)` |
| ทุก element ผ่านเงื่อนไข | `everyItem(matcher)` |
| map มี key/entry นี้ | `hasKey(x)` / `hasEntry(k, v)` |
| ตัวเลขเทียบมากกว่า/น้อยกว่า | `greaterThan(n)` / `lessThan(n)` / `greaterThanOrEqualTo(n)` |
| ต้องผ่าน**ทุก**เงื่อนไข (AND) | `allOf(matcher1, matcher2, ...)` |
| ผ่าน**เงื่อนไขใดก็ได้** (OR) | `anyOf(matcher1, matcher2, ...)` |
| เป็น instance ของ type นี้ | `instanceOf(MyClass.class)` |

```java
.then()
    .body("price", allOf(greaterThan(0f), lessThan(1000f)))
    .body("tags", everyItem(not(nullValue())))
    .body("status", anyOf(equalTo("ACTIVE"), equalTo("PENDING")));
```

---

## 6. `extract()` — ดึงค่าจาก response ไปใช้ต่อ

```java
String createdId =
    given()
        .contentType(ContentType.JSON)
        .body(payload)
    .when()
        .post("/products")
    .then()
        .statusCode(201)
        .extract()
        .path("id");                  // ดึงค่า field เดียวออกมาเป็น String

// เอา id ที่ได้ไปใช้เทส flow ถัดไปต่อ (ต้อง create ก่อนถึงจะ get/delete ได้จริง)
given().when().get("/products/" + createdId).then().statusCode(200);
```

`extract().response()` ดึงทั้ง `Response` object ออกมา (เอาไปใช้ `.jsonPath()`/`.asString()`/`.getHeaders()` เพิ่มเติมนอกเหนือจาก `then()`), `extract().path(...)` ดึงค่าเดียวแบบ JsonPath ตรง ๆ

---

## กับดัก

- **ลืม static import `RestAssured.*` และ `Matchers.*`** — `given()`/`equalTo()` แดง compile ไม่ผ่าน มือใหม่มักงงว่าทำไม copy ตัวอย่างมาแล้วไม่ทำงาน
- **ใช้ `param()` ทั้งที่ตั้งใจจะให้เป็น query param เสมอ** — `param()` เปลี่ยนพฤติกรรมตาม HTTP method (GET→query, POST→form) ถ้าอยากชัดเจนไม่ขึ้นกับ method ให้ใช้ `queryParam()`/`formParam()` ตรง ๆ
- **เขียน path ใน `.body(path, matcher)` แบบ Java property access** — ต้องใช้ GPath/JsonPath syntax (เช่น `"size()"` ไม่ใช่ `.length`, `"[0].code"` ไม่ใช่ `[0].getCode()`)
- **เช็ค `.body(...)` ก่อนเช็ค `.statusCode(...)`** — endpoint พังเป็น 500 แล้วไปเช็ค body ก่อน มักได้ parse error ที่อ่านไม่ออกว่าจริง ๆ แล้ว status ผิด ควรวาง `statusCode` ไว้เป็นเงื่อนไขแรกเสมอ
- **`relaxedHTTPSValidation()` หลุดไปอยู่ใน config ที่ใช้กับ environment จริง** — ปิด SSL cert verification จริงเป็นช่องโหว่ความปลอดภัย ใช้ได้เฉพาะ test/dev เท่านั้น
- **`param()`/`queryParam()` ซ้ำ key เดิมหลายครั้งโดยไม่ตั้งใจ** — REST Assured ส่งเป็น multi-value parameter ให้ (เช่น `?tag=a&tag=b`) ถ้าไม่ได้ตั้งใจจะทำให้ query ที่ server ได้รับไม่ตรงกับที่คิดไว้

---

## Cheat sheet

```java
import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

given()
    .contentType(ContentType.JSON)
    .header("Authorization", "Bearer " + token)
    .pathParam("id", 1)
    .body(payload)
.when()
    .post("/products/{id}")
.then()
    .statusCode(201)
    .body("code", equalTo("P001"))
    .body("tags", hasSize(2))
    .extract()
    .path("id");
```

| matcher | ใช้ตรวจ |
|---|---|
| `equalTo` / `is` | ค่าตรงกันเป๊ะ |
| `containsString` / `startsWith` / `endsWith` | string บางส่วน |
| `hasSize` / `hasItem` / `everyItem` | collection |
| `notNullValue` / `nullValue` | null check |
| `allOf` / `anyOf` / `not` | รวม/กลับเงื่อนไข |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus Testing]] — ตัวอย่างเต็มในบริบท `@QuarkusTest` ที่มีแอปจริงสตาร์ตรออยู่
- [[Java JUnit 5]] — `assertEquals`/`assertThrows` แบบ JUnit เทียบกับสไตล์ matcher ของโน้ตนี้
- [[Quarkus HTTP Methods]] — verb ที่ใช้ใน `when()` และ status code คู่กัน

## 📖 อ่านต่อ

- [REST Assured — Usage (official wiki)](https://github.com/rest-assured/rest-assured/wiki/usage)
- [Hamcrest — Matchers Javadoc](https://hamcrest.org/JavaHamcrest/javadoc/3.0/org/hamcrest/Matchers.html)
