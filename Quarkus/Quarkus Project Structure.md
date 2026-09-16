---
tags:
  - java
  - quarkus
  - architecture
  - project-structure
type: reference
created: 2026-09-16
---

# 🗂️ Quarkus Project Structure — วางไฟล์ยังไงให้เป็นแพตเทิร์น

> **คำถามที่ตัดสินใจโครงสร้างทั้งหมด: "เวลาจะแก้ feature หนึ่ง ต้องเปิดกี่โฟลเดอร์"**
> package-by-layer (แยกตามหน้าที่ทางเทคนิค) กับ package-by-feature (แยกตามโดเมนธุรกิจ) ตอบคำถามนี้ต่างกันคนละขั้ว — ไม่มีแบบไหนถูกเสมอ ขึ้นกับขนาดโปรเจกต์

---

## 1. โครงสร้าง Maven มาตรฐาน — จุดเริ่มต้นของทุกโปรเจกต์

```
my-service/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/                      ← โค้ด Java ทั้งหมด
│   │   ├── resources/
│   │   │   ├── application.properties ← config หลัก (ดู [[Quarkus Config]])
│   │   │   ├── db/migration/          ← Flyway migration files (ดู [[Quarkus Flyway]])
│   │   │   └── META-INF/
│   │   │       └── beans.xml          ← เปิดใช้ CDI (บางเวอร์ชันไม่จำเป็นแล้ว)
│   │   └── docker/
│   │       ├── Dockerfile.jvm         ← build image แบบ JVM (ดู [[Quarkus Build]])
│   │       └── Dockerfile.native      ← build image แบบ native
│   └── test/
│       └── java/                      ← เทส (ดู [[Quarkus Testing]])
```

**`src/main/resources` ทุกไฟล์ถูกรวมเข้า native binary อัตโนมัติตอน build** — ไฟล์ static/template ที่แอปต้องอ่านตอน runtime ต้องอยู่ในนี้ (หรือ copy เข้า image เองใน Dockerfile) ไม่งั้นหายตอนรันเป็น native (รายละเอียดเต็ม ๆ ที่ [[Quarkus Build]] ข้อ 6.3)

---

## 2. Package-by-layer vs Package-by-feature — สองแนวคิดหลัก

### 2.1 Package-by-layer — แยกตามหน้าที่ทางเทคนิค

```
org.acme
├── resource/       ← JAX-RS endpoint ทั้งหมด
│   ├── ProductResource.java
│   └── OrderResource.java
├── service/        ← business logic ทั้งหมด
│   ├── ProductService.java
│   └── OrderService.java
├── repository/      ← data access (ถ้าใช้ Panache repository pattern)
├── entity/          ← JPA entity ทั้งหมด
└── dto/             ← request/response DTO ทั้งหมด
```

| ข้อดี | ข้อเสีย |
|---|---|
| เห็นภาพรวมทาง technical ชัด (endpoint ทั้งหมดอยู่ที่เดียว) | แก้ 1 feature ต้องเปิดหลายโฟลเดอร์พร้อมกัน (resource + service + entity + dto) |
| เข้าใจง่ายสำหรับทีมที่คุ้นกับ layered architecture แบบเดิม | พอโปรเจกต์ใหญ่ขึ้น แต่ละโฟลเดอร์มีไฟล์เป็นสิบ ๆ ร้อย ๆ ไฟล์ปนกันหมดทุก feature |

**เหมาะกับ:** โปรเจกต์เล็ก-กลาง ที่จำนวน feature ไม่เยอะมาก

### 2.2 Package-by-feature — แยกตามโดเมนธุรกิจ

```
org.acme
├── product/
│   ├── ProductResource.java
│   ├── ProductService.java
│   ├── Product.java              ← entity
│   ├── ProductCreateRequest.java ← dto
│   └── ProductResponse.java
└── order/
    ├── OrderResource.java
    ├── OrderService.java
    ├── Order.java
    └── OrderResponse.java
```

| ข้อดี | ข้อเสีย |
|---|---|
| แก้ feature เดียว เปิดโฟลเดอร์เดียวจบ ทุกอย่างที่เกี่ยวข้องอยู่ด้วยกัน | เห็นภาพรวม technical layer ยากขึ้น (endpoint กระจายอยู่หลายที่) |
| ตัดแบ่งเป็น microservice ทีหลังง่ายกว่ามาก (ขอบเขตชัดอยู่แล้ว) | ทีมที่คุ้นกับ layered แบบเดิมต้องปรับตัว |
| ไฟล์ที่เกี่ยวข้องกันไม่กระจัดกระจาย | ของที่ใช้ร่วมกันข้าม feature (shared util) ต้องคิดที่วางแยกต่างหาก |

**เหมาะกับ:** โปรเจกต์ที่โตขึ้นเรื่อย ๆ มีหลาย domain ชัดเจน หรือวางแผนจะแตกเป็น microservice ในอนาคต

**ไม่มีแบบไหนถูกเสมอ — ทีมใหญ่หลายทีมมักผสมทั้งสองแบบ:** แยกด้วย package-by-feature เป็นหลัก แต่ของที่ใช้ร่วมกันจริง ๆ ข้ามทุก feature (เช่น exception handler กลาง, utility, config) แยกไว้เป็น package ต่างหากแบบ layer

---

## 3. Naming convention มาตรฐาน

| suffix | ใช้กับ | ตัวอย่าง |
|---|---|---|
| `...Resource` | JAX-RS endpoint (`@Path`) | `ProductResource` |
| `...Service` | business logic (`@ApplicationScoped`) | `ProductService` |
| `...Repository` | data access layer (ถ้าใช้ Panache repository pattern แทน active record — ดู [[Quarkus Panache]]) | `ProductRepository` |
| (ไม่มี suffix พิเศษ หรือ entity ตรง ๆ) | JPA entity (`@Entity`) | `Product` |
| `...Request` / `...Response` / `...Detail` / `...Summary` | DTO (ไม่ใช่ entity) — มักเขียนเป็น `record` (ดู [[Java Record]]) | `ProductCreateRequest`, `ProductDetail` |

**กฎที่สำคัญที่สุดในตารางนี้: entity กับ DTO ต้องเป็นคนละคลาสกันเสมอ** — ห้ามคืน entity ออก REST endpoint ตรง ๆ (เหตุผลเต็ม ๆ อยู่ที่ [[Java Record]] เรื่อง lazy loading ระเบิดตอน serialize) แปลงเป็น DTO ก่อนทุกครั้ง

---

## 4. โครงสร้างสี่ชั้นมาตรฐาน — resource → service → repository → entity

```java
// resource/ProductResource.java — บาง แค่รับ/ส่ง HTTP, delegate ให้ service
@Path("/products")
public class ProductResource {
    @Inject ProductService service;

    @GET @Path("/{id}")
    public ProductDetail get(@PathParam("id") Long id) {
        return service.get(id);
    }
}

// service/ProductService.java — business logic, ขอบเขตทรานแซกชัน
@ApplicationScoped
public class ProductService {
    @Inject ProductRepository repository;

    public ProductDetail get(Long id) {
        return ProductDetail.from(repository.findById(id));   // entity → DTO ตรงนี้
    }
}

// repository/ProductRepository.java — คุยกับ DB อย่างเดียว
@ApplicationScoped
public class ProductRepository implements PanacheRepository<Product> { }

// entity/Product.java — JPA entity
@Entity
public class Product { ... }

// dto/ProductDetail.java — สิ่งที่ออก REST จริง
public record ProductDetail(Long id, String code, String name) {
    public static ProductDetail from(Product p) {
        return new ProductDetail(p.id, p.code, p.name);
    }
}
```

**แต่ละชั้นห้ามข้ามหน้าที่กัน** — resource ห้ามมี business logic เอง, service ห้ามรู้เรื่อง HTTP (`Response`, `@PathParam`), repository ห้ามมี business rule — เพิ่มการทดสอบและแก้ไขได้อิสระต่อชั้น

---

## 5. กับดัก

- **god-resource / god-service** — ยัด endpoint หรือ business logic ทุกอย่างของทั้งโดเมนไว้ในไฟล์เดียว จนไฟล์ยาวเป็นพัน ๆ บรรทัด แก้ยาก, merge conflict บ่อย, เทสยาก — สัญญาณว่าถึงเวลาแตกเป็นหลายไฟล์ตาม sub-feature แล้ว
- **entity หลุดออก REST โดยตรง** — ไม่แปลงเป็น DTO ก่อนคืนค่า เจอ `LazyInitializationException` ตอน serialize หรือ leak field ที่ไม่ควรโชว์ออกไป
- **resource เรียก repository/EntityManager ตรง ๆ ข้าม service** — ทำให้ business logic กระจายอยู่หลายที่ และไม่มีที่รวม `@Transactional` ที่ชัดเจน
- **ใช้ entity ตัวเดียวกันเป็นทั้ง request DTO และ response DTO** — เปลี่ยน schema DB กระทบ API contract โดยไม่ตั้งใจ และเปิดช่องให้ client ส่ง field ที่ไม่ควรแก้ได้ (เช่น `id`, `createdDate`) เข้ามา
- **ผสม package-by-layer กับ package-by-feature โดยไม่มีกฎชัดเจน** — บางส่วนอยู่แบบหนึ่ง บางส่วนอยู่อีกแบบ โดยไม่มีเหตุผล ทำให้หาไฟล์ไม่เจอว่าอยู่ไหนกันแน่ ถ้าจะผสมสองแบบต้องตกลงกันในทีมชัดเจนว่าอะไรอยู่ตรงไหน
- **ไฟล์ config/migration ปนกับโค้ดโดยไม่เป็นระเบียบ** — `db/migration` ที่ไม่เรียงเลขเวอร์ชันดี หรือ `application.properties` ที่ไม่แยก `%dev`/`%prod` ชัดเจน (ดู [[Quarkus Config]])

---

## 6. Cheat sheet

```
src/main/java/.../resource/    → @Path endpoint, บาง, delegate อย่างเดียว
src/main/java/.../service/     → business logic, @Transactional
src/main/java/.../repository/  → data access เท่านั้น
src/main/java/.../entity/      → @Entity, ไม่ออก REST ตรง ๆ
src/main/java/.../dto/         → record, ...Request/...Response/...Detail
src/main/resources/
  application.properties       → config
  db/migration/                → Flyway
src/main/docker/
  Dockerfile.jvm / Dockerfile.native
```

| อาการ | สาเหตุ |
|---|---|
| ไฟล์เดียวยาวเป็นพันบรรทัด แก้ทีไรกระทบทุกคน | god-resource/god-service ไม่ได้แตกตาม sub-feature |
| `LazyInitializationException` ตอน serialize response | คืน entity ออก REST ตรง ๆ แทนที่จะแปลงเป็น DTO |
| แก้ DB column แล้ว API พังโดยไม่ตั้งใจ | ใช้ entity ตัวเดียวกันเป็น DTO ด้วย |
| หาไฟล์ของ feature หนึ่งไม่เจอว่าอยู่ตรงไหน | ผสม package-by-layer กับ package-by-feature โดยไม่มีกฎ |
| ไฟล์ static หายตอนรันเป็น native | ไม่ได้อยู่ใน `src/main/resources` หรือไม่ได้ copy เข้า image |

---

## 🔗 เกี่ยวข้อง

- [[Java Record]] — DTO ควรเขียนเป็น record, เหตุผลที่ห้ามคืน entity ออก REST
- [[Quarkus Panache]] — active record vs repository pattern (กระทบว่าจะมี `...Repository` layer หรือไม่)
- [[Quarkus Build]] — ไฟล์ใน `src/main/resources`/`src/main/docker` ถูกใช้ยังไงตอน build จริง
- [[Quarkus Flyway]] — โครงสร้างไฟล์ใน `db/migration`
- [[Quarkus]] — หน้ารวม

## 📖 อ่านต่อ

- [Quarkus and Maven](https://quarkus.io/guides/maven-tooling/)
- [Quarkus best practices (community guideline)](https://github.com/andredesousa/quarkus-best-practices)
