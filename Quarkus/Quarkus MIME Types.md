---
tags:
  - java
  - quarkus
  - http
  - mime
  - rest
type: reference
created: 2026-08-18
---

# 📎 MIME Type / Media Type — มีอะไรบ้าง และใช้ยังไงใน Quarkus

> MIME type (ชื่อทางการคือ **media type**) คือ**ป้ายบอกชนิดของเนื้อหา** ที่แนบไปกับ HTTP response/request
> HTTP method บอกว่า "ทำอะไร" (ดู [[Quarkus HTTP Methods]]) ส่วน media type บอกว่า **"เนื้อหาที่ส่งมาคืออะไร"** — client จะได้รู้ว่าต้องแกะยังไง

---

## 1. รูปร่างของมัน

`type / subtype ; parameter`

| ส่วน | ตัวอย่าง | ความหมาย |
|---|---|---|
| `type` | `application` | หมวดใหญ่ |
| `subtype` | `json` | ชนิดย่อย |
| `parameter` | `charset=utf-8` | ข้อมูลเสริม (ไม่บังคับ) |

```
application/json; charset=utf-8
    │            │
  หมวด          บอกว่าใช้ UTF-8 encoding

multipart/form-data; boundary=----abc123
    │                    │
  หมวด                 ตัวคั่นระหว่างแต่ละ part
```

**ควบคุมโดย IANA** (Internet Assigned Numbers Authority) — เป็น registry กลางที่ทุกคนตกลงร่วมกัน ไม่ใช่ใครก็ตั้งชื่อเองมั่ว ๆ ได้ (แม้ในทางปฏิบัติจะมีบางเจ้าตั้งชื่อ vendor-specific เองก็ตาม ดูข้อ 5)

---

## 2. มีอะไรบ้าง — แบ่งตามหมวดใหญ่

| หมวด | คืออะไร |
|---|---|
| `text/` | ข้อความอ่านได้ตรง ๆ |
| `image/` | รูปภาพ |
| `audio/` | เสียง |
| `video/` | วิดีโอ |
| `font/` | ไฟล์ font |
| `application/` | ไฟล์แบบไบนารีหรือโครงสร้างเฉพาะแอป (ที่เหลือ) |
| `multipart/` | ข้อความหลายส่วนรวมกันในก้อนเดียว |

### ที่เจอบ่อยที่สุดในงาน API

| Media type | คืออะไร |
|---|---|
| `application/json` | ข้อมูล JSON — **ค่าเริ่มต้นของ REST API สมัยนี้** |
| `application/xml` | ข้อมูล XML — ระบบเก่า/SOAP |
| `application/x-www-form-urlencoded` | ฟอร์ม HTML ธรรมดา (`key=value&key2=value2`) |
| `multipart/form-data` | ฟอร์มที่มีไฟล์แนบ |
| `application/octet-stream` | ไบนารีดิบ ไม่รู้ชนิดแน่ชัด — "ไฟล์อะไรก็ได้" |
| `text/plain` | ข้อความล้วน ไม่มี markup |
| `text/html` | หน้าเว็บ |
| `text/csv` | ตารางแบบ comma-separated |
| `application/pdf` | เอกสาร PDF |
| `application/zip` | ไฟล์บีบอัด ZIP |

### ไฟล์เอกสาร/สำนักงาน

| Media type | คืออะไร |
|---|---|
| `application/vnd.ms-excel` | Excel เก่า (.xls) |
| `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` | Excel ใหม่ (.xlsx) |
| `application/vnd.openxmlformats-officedocument.wordprocessingml.document` | Word (.docx) |
| `application/msword` | Word เก่า (.doc) |

### รูปภาพ/สื่อ

| Media type | คืออะไร |
|---|---|
| `image/png` | PNG |
| `image/jpeg` | JPEG |
| `image/svg+xml` | SVG (เป็น XML ข้างใน) |
| `image/webp` | WebP |
| `video/mp4` | วิดีโอ MP4 |
| `font/woff2` | font สำหรับเว็บ |

### Event / Streaming

| Media type | คืออะไร |
|---|---|
| `text/event-stream` | Server-Sent Events (SSE) — stream ข้อมูลทีละก้อนแบบ real-time |
| `application/x-ndjson` | JSON หลายบรรทัด บรรทัดละ 1 object — สตรีมข้อมูลจำนวนมาก |

---

## 3. `+json` / `+xml` — suffix ที่บอกว่า "ห่อด้วยอะไร"

หลาย media type ผสมกันได้ด้วย `+` เพื่อบอกว่าโครงสร้างภายในเป็นอะไร แม้ความหมายทางธุรกิจจะต่างกัน

```
application/vnd.api+json      → JSON:API spec
application/hal+json          → HAL (Hypertext Application Language)
application/problem+json      → RFC 7807 error response
image/svg+xml                 → SVG ซึ่งข้างในเป็น XML
```

**`application/problem+json` มีประโยชน์จริงที่ควรรู้** — เป็นมาตรฐานสำหรับ error response ที่ Quarkus รองรับผ่าน RESTEasy Reactive มีโครงสร้างมาตรฐาน (`title`, `status`, `detail`, `instance`) ทำให้ client ทุกตัวรู้วิธีอ่าน error โดยไม่ต้องเดารูปแบบเอง

---

## 4. `application/octet-stream` — ป้าย "ไม่รู้ว่าคืออะไร"

```
application/octet-stream = "ไฟล์ไบนารีทั่วไป ไม่รู้ชนิดแน่ชัด ให้ client เดาเอง/เซฟลงดิสก์เฉย ๆ"
```

**ใช้เมื่อไม่รู้ล่วงหน้าว่าไฟล์ที่ upload/download เป็นชนิดอะไร** เช่น endpoint อัปโหลดไฟล์อะไรก็ได้ตามที่ผู้ใช้เลือก

**ถ้ารู้ชนิดแน่นอน ควรระบุให้ตรง** — `application/pdf` แทน `application/octet-stream` เพราะ browser จะได้รู้ว่าควรเปิดดูตรง ๆ ในแท็บ หรือเสนอดาวน์โหลด ต่างจาก octet-stream ที่ browser ส่วนใหญ่จะดาวน์โหลดทันทีไม่ถามอะไร

---

## 5. `x-` และ `vnd.` — สังเกตยังไงว่าไม่เป็นทางการ

| prefix | ความหมาย |
|---|---|
| (ไม่มี) | ลงทะเบียนกับ IANA แล้ว เป็นมาตรฐานเต็มตัว |
| `vnd.` | **vendor-specific** — บริษัท/โปรเจกต์ตั้งเอง แต่ลงทะเบียนไว้แล้ว เช่น `application/vnd.api+json` |
| `x-` | ไม่เป็นทางการ ไม่ได้ลงทะเบียน — **เจอบ่อยในระบบเก่า** เช่น `application/x-zip-compressed`, `application/x-www-form-urlencoded` (ตัวหลังกลายเป็นมาตรฐานไปแล้วทั้งที่ยังมี `x-` ติดอยู่ เพราะเปลี่ยนชื่อทีหลังจะกระทบระบบเก่าเยอะเกินไป)

**ผลจริง:** ถ้าทำ API ที่ต้องรองรับ client เก่า ๆ บางทีต้อง accept ทั้งชื่อมาตรฐานและชื่อ `x-`/`vnd.` แบบเก่าคู่กัน เพราะ client บางตัวยังส่งชื่อเก่ามาอยู่

---

## 6. ใช้ใน Quarkus — `@Produces` / `@Consumes`

```java
@Path("/products")
public class ProductResource {

    @GET
    @Produces(MediaType.APPLICATION_JSON)      // ← ตอบกลับเป็น JSON
    public List<ProductSummary> list() { ... }

    @POST
    @Consumes(MediaType.APPLICATION_JSON)      // ← รับเข้าเป็น JSON
    @Produces(MediaType.APPLICATION_JSON)
    public Response create(ProductCreateRequest req) { ... }

    @GET
    @Path("/{id}/label")
    @Produces("image/png")                     // ← string ตรง ๆ ก็ได้ ไม่ต้องมี constant
    public byte[] barcode(@PathParam("id") Long id) { ... }
}
```

**`@Produces` = server ตอบเป็นอะไร, `@Consumes` = server รับอะไรเข้ามา** — คนละทิศทางกัน สลับกันแล้ว endpoint จะปฏิเสธ request ที่ถูกต้องเงียบ ๆ (คืน `415 Unsupported Media Type`)

### Constant ที่ `MediaType` มีให้ ไม่ต้องพิมพ์ string เอง

```java
MediaType.APPLICATION_JSON       // "application/json"
MediaType.APPLICATION_XML        // "application/xml"
MediaType.TEXT_PLAIN             // "text/plain"
MediaType.TEXT_HTML              // "text/html"
MediaType.MULTIPART_FORM_DATA    // "multipart/form-data"
MediaType.APPLICATION_OCTET_STREAM
MediaType.WILDCARD               // "*/*"
```

---

## 7. รับหลายชนิด / ตอบหลายชนิด — Content Negotiation

**หนึ่ง endpoint รองรับได้มากกว่าหนึ่ง media type** แล้วให้ client เลือกผ่าน header `Accept`

```java
@GET
@Path("/{id}")
@Produces({ MediaType.APPLICATION_JSON, MediaType.APPLICATION_XML })
public ProductDetail get(@PathParam("id") Long id) { ... }
```

```
client ส่ง:  Accept: application/xml
server ตอบ:  Content-Type: application/xml    ← เลือกให้อัตโนมัติ
```

นี่คือกลไก **content negotiation** — server มีของพร้อมหลายรูปแบบ ให้ client เป็นคนเลือกผ่าน `Accept` header แทนที่จะแยก endpoint คนละ URL

### Wildcard

```java
@Produces(MediaType.WILDCARD)     // "*/*" — ตอบได้ทุกชนิด (มักใช้กับ endpoint สั่งงานที่ไม่มี body คืน)
```

```
application/*     → รับ/ส่งได้ทุกชนิดในหมวด application (image/png ไม่เข้าเงื่อนไข)
*/*               → รับ/ส่งได้ทุกอย่างไม่จำกัด
```

---

## 8. Multipart — เมื่อมีทั้งไฟล์และข้อมูลอื่นในคำขอเดียว

```java
public class UploadForm {
    @RestForm("file")
    public FileUpload file;

    @RestForm("description")
    public String description;
}

@POST
@Path("/upload")
@Consumes(MediaType.MULTIPART_FORM_DATA)
public Response upload(@BeanParam UploadForm form) {
    Path path = form.file.uploadedFile();
    String name = form.file.fileName();
    ...
    return Response.ok().build();
}
```

**สังเกต — ไม่ต้องเขียน `@Consumes(MULTIPART_FORM_DATA)` เองก็ได้ในบางกรณี** — ถ้า `@RestForm` มี parameter ชนิด `File`, `Path`, `byte[]` หรือมี `@PartType` กำกับ Quarkus จะอนุมานให้อัตโนมัติว่าเป็น multipart แต่การเขียนไว้ชัด ๆ อ่านง่ายกว่าและกันความสับสน

**เบื้องหลังของ multipart:** เนื้อหาถูกแบ่งเป็นหลาย "part" คั่นด้วย `boundary` แต่ละ part มี header/media type ของตัวเอง

```
Content-Type: multipart/form-data; boundary=----abc123

------abc123
Content-Disposition: form-data; name="description"

รูปสินค้า
------abc123
Content-Disposition: form-data; name="file"; filename="photo.png"
Content-Type: image/png

(ไบนารีของรูป)
------abc123--
```

---

## 9. Streaming — เมื่อข้อมูลใหญ่หรือมาเรื่อย ๆ

```java
@GET
@Path("/notifications")
@Produces(MediaType.SERVER_SENT_EVENTS)     // text/event-stream
public Multi<Notification> stream() {
    return notificationBus.subscribe();
}
```

`Multi<T>` + `SERVER_SENT_EVENTS` ทำให้ endpoint ส่งข้อมูลทีละก้อนแบบ real-time โดยไม่ต้องปิด connection — ใช้กับ dashboard ที่ต้องอัปเดตสด, แจ้งเตือน, progress ของงานที่รันนาน

```java
@GET
@Path("/report/large")
@Produces(MediaType.APPLICATION_OCTET_STREAM)
public Response downloadLarge() {
    StreamingOutput out = output -> {
        try (var in = Files.newInputStream(reportPath)) {
            in.transferTo(output);
        }
    };
    return Response.ok(out)
                   .header("Content-Disposition", "attachment; filename=\"report.csv\"")
                   .build();
}
```

**เหตุผลที่ต้อง stream แทนโหลดทั้งไฟล์เข้า memory** — ไฟล์ export ขนาดหลายร้อย MB ถ้าอ่านเข้า `byte[]` ทั้งก้อนก่อนส่ง จะกิน heap มหาศาลและเสี่ยง `OutOfMemoryError` เมื่อมีหลาย request พร้อมกัน `StreamingOutput` ส่งข้อมูลออกไปทีละ chunk ทันทีที่มี ไม่ต้องรอครบก่อน

---

## 10. Charset — ส่วนที่ลืมกันบ่อย

```
Content-Type: text/plain; charset=utf-8
Content-Type: application/json; charset=utf-8
```

**JSON ไม่จำเป็นต้องระบุ charset** เพราะสเปก JSON บังคับเป็น UTF-8 อยู่แล้ว แต่ `text/*` ควรระบุเสมอ ไม่งั้น client บางตัวเดาผิดแล้วภาษาไทยกลายเป็นอักษรมั่ว

```java
@Produces("text/plain; charset=utf-8")
public String report() { ... }
```

---

## 11. Status code ที่เกี่ยวกับ media type

| Status | เกิดตอนไหน |
|---|---|
| `415 Unsupported Media Type` | client ส่ง `Content-Type` ที่ endpoint ไม่รองรับ (ไม่ตรงกับ `@Consumes`) |
| `406 Not Acceptable` | client ขอ `Accept` ที่ server ตอบให้ไม่ได้ (ไม่ตรงกับ `@Produces` ตัวไหนเลย) |

**415 กับ 406 สลับกันบ่อย** — จำง่าย ๆ ว่า **415 = ส่งมาผิดชนิด (input), 406 = ขอกลับไปผิดชนิด (output)**

---

## 12. กับดัก

- **สลับ `@Produces` กับ `@Consumes`** — endpoint ปฏิเสธ request ที่ถูกต้อง คืน 415 โดยไม่มีคำอธิบายชัดเจน
- **ตอบ `application/octet-stream` ทั้งที่รู้ชนิดไฟล์แน่นอน** — browser ไม่รู้จะเปิดยังไง ต้องดาวน์โหลดเสมอ ทั้งที่ควรเปิดดูตรง ๆ ได้ (เช่น PDF)
- **โหลดไฟล์ใหญ่เข้า `byte[]` แล้วส่งทีเดียว** — เสี่ยง OOM เมื่อมีหลาย request พร้อมกัน ควร stream แทน
- **ลืม `charset` บน `text/*` ที่มีภาษาไทย** — client เดา encoding ผิด ตัวอักษรกลายเป็นขยะ
- **บีบอัด (`Content-Encoding: gzip`) ไฟล์ที่บีบอยู่แล้ว** เช่น `.zip`, `.png`, `.jpg` — เปลือง CPU แต่ไม่ได้ขนาดลดลงจริง เวลาตั้ง compression filter ควร exclude MIME type พวกนี้
- **ใช้ `x-` prefix ในโค้ดใหม่** — ถ้าเป็นไปได้ใช้ชื่อมาตรฐานที่ลงทะเบียนกับ IANA แล้วแทน

---

## 13. Cheat sheet

```java
@Produces(MediaType.APPLICATION_JSON)              // ตอบ JSON
@Consumes(MediaType.APPLICATION_JSON)               // รับ JSON
@Produces({APPLICATION_JSON, APPLICATION_XML})       // เลือกได้ทั้งสอง ตาม Accept header
@Consumes(MediaType.MULTIPART_FORM_DATA)             // รับฟอร์มที่มีไฟล์
@Produces(MediaType.SERVER_SENT_EVENTS)              // stream แบบ real-time
@Produces(MediaType.WILDCARD)                        // ตอบได้ทุกชนิด
```

| ต้องการ | ใช้ |
|---|---|
| ส่ง/รับ JSON (ค่าเริ่มต้น) | `application/json` |
| ส่งไฟล์ดาวน์โหลด รู้ชนิดแน่นอน | ชนิดจริง เช่น `application/pdf` |
| ส่งไฟล์ไม่รู้ชนิดแน่นอน | `application/octet-stream` |
| รับฟอร์มพร้อมไฟล์แนบ | `multipart/form-data` |
| error response แบบมาตรฐาน | `application/problem+json` |
| ข้อมูลสตรีมสด (แจ้งเตือน, progress) | `text/event-stream` |
| export ตารางไปเปิดใน Excel | `text/csv` หรือ xlsx media type |

| อาการ | สาเหตุ |
|---|---|
| `415 Unsupported Media Type` | `Content-Type` ที่ client ส่งไม่ตรงกับ `@Consumes` |
| `406 Not Acceptable` | `Accept` ที่ client ขอไม่ตรงกับ `@Produces` ตัวไหนเลย |
| ภาษาไทยเป็นอักษรมั่วตอนโหลด text | ไม่ได้ระบุ `charset=utf-8` |
| ไฟล์ดาวน์โหลดถูกบังคับให้เซฟทันทีทั้งที่ควรเปิดดูได้ | ตอบเป็น `application/octet-stream` ทั้งที่รู้ชนิดจริง |
| เซิร์ฟเวอร์กิน memory พุ่งตอน export ไฟล์ใหญ่ | โหลดทั้งไฟล์เข้า `byte[]` แทนที่จะ stream |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus HTTP Methods]] — method บอก "ทำอะไร" media type บอก "เนื้อหาคืออะไร" ใช้คู่กันเสมอ
- [[Quarkus REST Client]] — ฝั่ง client ก็ต้องตั้ง `Content-Type`/`Accept` ให้ตรงเช่นกัน
- [[Quarkus]] — หน้ารวม

## 📖 อ่านต่อ

- [MDN — MIME types](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/MIME_types)
- [IANA — Media Types registry](https://www.iana.org/assignments/media-types)
- [Quarkus — Writing REST Services](https://quarkus.io/guides/rest)
- [RFC 7807 — Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc7807)
