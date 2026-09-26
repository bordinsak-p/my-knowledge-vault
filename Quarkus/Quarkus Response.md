---
tags:
  - quarkus
  - java
  - rest
type: reference
created: 2026-09-20
---

# 📬 Quarkus — `Response` และ `RestResponse<T>`

> **`Response` คือคลาสที่คุมทุกส่วนของ HTTP response เอง** (status code, header, body) แทนที่จะ return แค่ POJO ธรรมดาแล้วปล่อยให้ JAX-RS ใส่ status `200` ให้อัตโนมัติ — Quarkus REST มีอีกทางเลือกคือ **`RestResponse<T>`** ที่ type-safe กว่าและแนะนำให้ใช้แทนในโค้ดใหม่

---

## 1. เมื่อไหร่ต้อง return `Response` แทนที่จะ return POJO ตรงๆ

```java
// return POJO ตรงๆ — ได้ 200 OK เสมอ ไม่มีทางเปลี่ยน status ได้
@GET
public Product getProduct(@PathParam("id") Long id) {
    return productService.find(id);
}

// return Response — คุม status ได้เองตามสถานการณ์จริง
@GET
public Response getProduct(@PathParam("id") Long id) {
    Product product = productService.find(id);
    if (product == null) {
        return Response.status(Response.Status.NOT_FOUND).build();
    }
    return Response.ok(product).build();
}
```

**ใช้ `Response` เมื่อ:** status code ขึ้นอยู่กับเงื่อนไข runtime (เจอ/ไม่เจอ, สร้างสำเร็จ/ซ้ำ), ต้องตั้ง header เอง (`Location`, custom header), หรือต้องการ body ว่างเปล่าพร้อม status เฉพาะ (เช่น `204 No Content`)

---

## 2. Static factory method ที่ใช้บ่อย

```java
Response.ok()                          // 200 ไม่มี body
Response.ok(entity)                    // 200 พร้อม body
Response.status(Response.Status.CREATED).entity(entity).build()   // status กำหนดเอง
Response.noContent().build()           // 204 ไม่มี body เลย
Response.created(uri).build()          // 201 + header Location ชี้ไปที่ resource ใหม่ให้อัตโนมัติ
```

### `Response.created(uri)` — ตัวที่ถูกลืมบ่อยที่สุดตอนเขียน POST endpoint

```java
@POST
public Response createProduct(ProductRequest request) {
    Product created = productService.create(request);
    URI location = URI.create("/products/" + created.getId());
    return Response.created(location).entity(created).build();
}
```

REST convention ที่ถูกต้องสำหรับ POST ที่สร้าง resource ใหม่คือ **`201 Created` พร้อม header `Location` ชี้ไปที่ resource ที่เพิ่งสร้าง** — `Response.created(uri)` ทำทั้งสองอย่างให้ในบรรทัดเดียว แทนที่จะเขียน `Response.status(201).header("Location", ...)` เอง

---

## 3. ตั้ง header/cookie เอง

```java
Response.ok(entity)
    .header("X-Total-Count", total)
    .cookie(new NewCookie("session", token))
    .build();
```

---

## 4. `RestResponse<T>` — ทางเลือกที่ Quarkus REST แนะนำมากกว่า

```java
@GET
public RestResponse<Product> getProduct(@PathParam("id") Long id) {
    Product product = productService.find(id);
    if (product == null) {
        return RestResponse.status(RestResponse.Status.NOT_FOUND);
    }
    return RestResponse.ok(product);
}
```

**ทำไมดีกว่า `Response` ธรรมดา:** `Response` เป็น type ทั่วไป (ไม่ generic) — Quarkus **ไม่รู้ตอน build ว่า entity ข้างในเป็น type อะไร** ทำให้ generate OpenAPI schema ไม่แม่นยำ และลงทะเบียน reflection ให้ native image ไม่ได้อัตโนมัติ — `RestResponse<Product>` **ระบุ type ชัดเจนตั้งแต่ signature** แก้ปัญหาทั้งสองอย่างนี้ได้

```java
// แบบเต็มพร้อม header/cookie เหมือน Response
RestResponse.ResponseBuilder.ok(entity, MediaType.TEXT_PLAIN_TYPE)
    .header("X-Cheese", "Camembert")
    .cookie(new NewCookie("flavour", "chocolate"))
    .build();
```

**บังคับให้ทั้งโปรเจกต์ใช้ตัวใดตัวหนึ่งแบบเดียวกันหมดได้** ผ่าน config:

```properties
quarkus.resteasy-reactive.return-response=RestResponse
```

---

## 5. เลือกใช้ตัวไหน

| สถานการณ์ | ใช้ |
|---|---|
| Endpoint ปกติ ไม่มีเงื่อนไข status ซับซ้อน | return POJO ตรงๆ (ไม่ต้องมี `Response` เลย) |
| ต้องคุม status/header เอง เขียนโปรเจกต์ใหม่ | `RestResponse<T>` (แนะนำ) |
| โค้ดเก่าที่มี `Response` อยู่แล้ว/ต้องพกความเข้ากันได้กับ JAX-RS มาตรฐาน | `Response` |
| native image build ต้องแม่นเรื่อง reflection | `RestResponse<T>` เท่านั้น |

---

## กับดัก

- **ลืม `.build()`** — `Response.ok(entity)` เฉยๆ คืนแค่ `ResponseBuilder` ไม่ใช่ `Response` compile ไม่ผ่านหรือพฤติกรรมผิดถ้าหลุดไปได้
- **ใช้ `Response` (ไม่ generic) แล้วแปลกใจว่าทำไม OpenAPI schema ของ endpoint นี้ไม่มีรายละเอียด field ใดๆ เลย** — Quarkus ไม่รู้ type ข้างใน `Response` ธรรมดา ต้องใช้ `RestResponse<T>` ถ้าอยากได้ schema ที่แม่นยำ (ข้อ 4)
- **ลืมใส่ `Location` header ตอน POST สร้าง resource ใหม่** — ผิด REST convention ควรใช้ `Response.created(uri)` แทนการเขียน `status(201)` เฉยๆ (ข้อ 2)
- **return `Response`/`RestResponse` ปนกับ return POJO ตรงๆ ในโปรเจกต์เดียวกันแบบไม่มีมาตรฐาน** — endpoint บางตัวคุม status ได้ บางตัวคุมไม่ได้ ทำให้ error handling ไม่สม่ำเสมอทั้ง API ควรตัดสินใจ convention ตั้งแต่ต้นโปรเจกต์ (ใช้ `return-response` config บังคับก็ได้ ข้อ 4)
- **native image reflection error ทั้งที่โค้ดคอมไพล์ผ่านปกติตอน JVM mode** — มักเกิดจากใช้ `Response` ธรรมดาที่ Quarkus ลงทะเบียน reflection ให้ entity ข้างในไม่ได้ตอน build native (ข้อ 4)

---

## Cheat sheet

```java
Response.ok(entity).build()
Response.status(Response.Status.NOT_FOUND).build()
Response.created(uri).entity(entity).build()
Response.noContent().build()

RestResponse.ok(entity)                       // แนะนำสำหรับโค้ดใหม่
RestResponse.status(RestResponse.Status.NOT_FOUND)
```

| ต้องการ status    | `Response.Status` / `RestResponse.Status` |
| ----------------- | ----------------------------------------- |
| สำเร็จ มี body    | `OK` (200)                                |
| สร้างสำเร็จ       | `CREATED` (201)                           |
| สำเร็จ ไม่มี body | `NO_CONTENT` (204)                        |
| request ผิด       | `BAD_REQUEST` (400)                       |
| ไม่ได้ login      | `UNAUTHORIZED` (401)                      |
| ไม่มีสิทธิ์       | `FORBIDDEN` (403)                         |
| หาไม่เจอ          | `NOT_FOUND` (404)                         |
| ข้อมูลขัดแย้งกัน  | `CONFLICT` (409)                          |

## 🔗 เกี่ยวข้อง

- [[Quarkus Exception Mapper]] — `ExceptionMapper.toResponse()` return `Response` แบบเดียวกับที่เขียนไว้ในโน้ตนี้
- [[Quarkus REST Layer]] — RESTEasy Classic vs Quarkus REST ที่ `RestResponse<T>` เป็นของฝั่ง Quarkus REST เท่านั้น
- [[Quarkus REST Client]] — `RestResponse<T>` ฝั่ง client ใช้แนวคิดเดียวกัน (ได้ status/header โดยไม่ต้องโยน exception)
- [[Quarkus HTTP Methods]] — status code คู่กับแต่ละ HTTP method

## 📖 อ่านต่อ

- [Quarkus — Writing REST services](https://quarkus.io/guides/rest)
- [Jakarta REST — Response Javadoc](https://jakarta.ee/specifications/restful-ws/3.1/apidocs/jakarta.ws.rs/jakarta/ws/rs/core/response)
