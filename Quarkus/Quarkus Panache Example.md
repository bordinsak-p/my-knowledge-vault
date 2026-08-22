---
tags:
  - java
  - quarkus
  - panache
  - hibernate
  - example
type: example
created: 2026-08-18
---

# 🐼 Quarkus Panache — ตัวอย่างเขียนจริง

ตัวอย่างครบชุดแบบ copy ไปแก้ต่อได้ — โดเมนสมมติ: **Product / Category**
ทฤษฎีอยู่ที่ [[Quarkus Panache]] อันนี้เน้นโค้ดล้วน

---

## 0. Dependency

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-hibernate-orm-panache</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-jdbc-mariadb</artifactId>
</dependency>
```

```properties
quarkus.datasource.db-kind=mariadb
quarkus.datasource.jdbc.url=jdbc:mariadb://localhost:3306/shop
quarkus.datasource.username=app
quarkus.datasource.password=secret

quarkus.hibernate-orm.database.generation=none
%dev.quarkus.hibernate-orm.log.sql=true
quarkus.hibernate-orm.jdbc.statement-batch-size=50
```

---

## 1. Entity

### 1.1 แบบง่ายที่สุด — `PanacheEntity` (ID เป็น `Long` ให้อัตโนมัติ)

```java
package org.acme.catalog;

import jakarta.persistence.*;
import io.quarkus.hibernate.orm.panache.PanacheEntity;
import java.math.BigDecimal;
import java.time.LocalDate;

@Entity
@Table(name = "PRODUCT")
public class Product extends PanacheEntity {

    @Column(name = "CODE", nullable = false, unique = true, length = 32)
    public String code;

    @Column(name = "NAME", nullable = false)
    public String name;

    @Column(name = "PRICE", precision = 15, scale = 2)
    public BigDecimal price;

    @Enumerated(EnumType.STRING)
    @Column(name = "STATUS", length = 16)
    public Status status;

    @Column(name = "RELEASED_ON")
    public LocalDate releasedOn;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "CATEGORY_ID")
    public Category category;

    public enum Status { DRAFT, ACTIVE, DISCONTINUED }
}
```

> field เป็น `public` ได้ ไม่ต้องเขียน getter/setter — Panache แปลง bytecode ให้วิ่งผ่าน accessor จริงตอน build
> ถ้าวันหลังอยากใส่ logic ก็เขียน getter ทับได้ โดยไม่ต้องแก้ที่เรียกใช้

### 1.2 ID เป็น UUID — ต้องใช้ `PanacheEntityBase`

`PanacheEntity` บังคับ `Long id` ถ้าอยากคุมเองให้ extend `PanacheEntityBase` แล้วประกาศ `@Id` เอง

```java
@Entity
@Table(name = "CATEGORY")
public class Category extends PanacheEntityBase {

    @Id
    @GeneratedValue
    @UuidGenerator
    @Column(name = "CATEGORY_ID", columnDefinition = "BINARY(16)")
    public UUID id;

    @Column(name = "NAME", nullable = false)
    public String name;

    @OneToMany(mappedBy = "category")
    public List<Product> products = new ArrayList<>();
}
```

> `columnDefinition = "BINARY(16)"` สำคัญมากบน MariaDB — ดูเหตุผลที่ [[Quarkus Hibernate]]

---

## 2. สองแนวทาง — เลือกอย่างใดอย่างหนึ่ง

### 2.1 Active Record — query อยู่ใน entity

```java
@Entity
public class Product extends PanacheEntity {
    // ... fields ...

    public static Optional<Product> findByCode(String code) {
        return find("code", code).firstResultOptional();
    }

    public static List<Product> findActive() {
        return list("status", Status.ACTIVE);
    }

    public static long countByCategory(UUID categoryId) {
        return count("category.id", categoryId);
    }

    public static List<Product> search(String keyword, Status status) {
        return find("""
                    (:kw is null or lower(name) like :kw or lower(code) like :kw)
                    and (:st is null or status = :st)
                    """,
                Parameters.with("kw", keyword == null ? null : "%" + keyword.toLowerCase() + "%")
                          .and("st", status))
                .list();
    }
}
```

### 2.2 Repository — query อยู่นอก entity

**ใช้แบบนี้เมื่อ entity แก้ไม่ได้ หรือ extend base class อื่นอยู่แล้ว** เพราะ `PanacheRepository` เป็น **interface**

```java
package org.acme.catalog;

import io.quarkus.hibernate.orm.panache.PanacheRepository;
import io.quarkus.hibernate.orm.panache.PanacheRepositoryBase;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class ProductRepository implements PanacheRepository<Product> {   // id เป็น Long

    public Optional<Product> findByCode(String code) {
        return find("code", code).firstResultOptional();
    }

    public List<Product> findActive() {
        return list("status", Product.Status.ACTIVE);
    }
}

@ApplicationScoped
class CategoryRepository implements PanacheRepositoryBase<Category, UUID> {   // id ไม่ใช่ Long
}
```

---

## 3. Query syntax — เทียบกับ JPQL เต็ม

```java
// Panache เติม "from Product where" ให้เอง
find("name", "iPhone")
find("name = ?1 and status = ?2", "iPhone", Status.ACTIVE)
find("name like ?1", "%phone%")
find("price between ?1 and ?2", min, max)
find("category.name = ?1", "Electronics")          // เดินผ่าน relation ได้
find("order by price desc")                        // where อย่างเดียวก็ได้ order อย่างเดียวก็ได้

// named parameters — อ่านง่ายกว่าเมื่อมีหลายตัว
find("status = :st and price > :p",
     Parameters.with("st", Status.ACTIVE).and("p", 100))

// JPQL เต็มก็ใส่ได้ตามปกติ
find("select distinct p from Product p left join fetch p.category where p.status = ?1", Status.ACTIVE)
```

### วิธีดึงผลลัพธ์

```java
List<Product> all       = Product.listAll();
List<Product> some      = Product.list("status", Status.ACTIVE);
Product       one       = Product.find("code", "P001").firstResult();      // null ถ้าไม่เจอ
Optional<Product> opt   = Product.find("code", "P001").firstResultOptional();
Product       exactlyOne= Product.find("code", "P001").singleResult();     // throw ถ้าไม่เจอ/เจอเกิน 1
long          n         = Product.count("status", Status.ACTIVE);
boolean       exists    = Product.count("code", code) > 0;

try (Stream<Product> s = Product.streamAll()) {    // ต้องปิด และต้องอยู่ใน transaction
    s.forEach(...);
}
```

> `singleResult()` โยน exception ทั้งตอนไม่เจอและตอนเจอเกินหนึ่ง — ถ้า "ไม่เจอ" เป็นเรื่องปกติของ business logic ให้ใช้ `firstResultOptional()` แทน

---

## 4. Service layer + transaction

```java
package org.acme.catalog;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.NotFoundException;

@ApplicationScoped
public class ProductService {

    @Inject ProductRepository repo;

    public Product get(Long id) {
        return repo.findByIdOptional(id)
                   .orElseThrow(() -> new NotFoundException("product " + id + " not found"));
    }

    @Transactional
    public Product create(ProductCreateRequest req) {
        repo.findByCode(req.code()).ifPresent(p -> {
            throw new IllegalArgumentException("duplicate code: " + req.code());
        });

        Product p = new Product();
        p.code   = req.code();
        p.name   = req.name();
        p.price  = req.price();
        p.status = Product.Status.DRAFT;

        repo.persist(p);      // id ถูกเติมให้หลังจากนี้
        return p;
    }

    @Transactional
    public Product rename(Long id, String newName) {
        Product p = get(id);
        p.name = newName;
        return p;             // ← ไม่ต้องเรียก persist(), dirty checking จัดการตอน commit
    }

    @Transactional
    public void delete(Long id) {
        if (!repo.deleteById(id)) {
            throw new NotFoundException("product " + id + " not found");
        }
    }
}
```

**สามข้อที่ต้องจำ**

1. `@Transactional` อยู่ที่ **service** ไม่ใช่ resource — resource ควรบางที่สุด
2. method ที่แค่อ่าน **ไม่ต้องมี** `@Transactional`
3. แก้ field ของ entity ที่ managed อยู่แล้ว **ไม่ต้อง `persist()`** — Hibernate flush ให้ตอน commit

---

## 5. DTO / Projection — อย่าคืน entity ออก REST

```java
public record ProductSummary(Long id, String code, String name, BigDecimal price) {}
```

```java
// project() generate select เฉพาะคอลัมน์ที่ record ต้องการ ไม่ได้ลากทั้งแถวมา
List<ProductSummary> rows = Product.find("status", Status.ACTIVE)
                                   .project(ProductSummary.class)
                                   .list();
```

ถ้าชื่อ field ใน record ไม่ตรงกับ path ใน entity ใช้ `@ProjectedFieldName`

```java
public record ProductRow(
        String code,
        @ProjectedFieldName("category.name") String categoryName) {}
```

> **`project()` ไม่ใช่แค่เรื่องความสะอาด** — ตารางที่มี 40 คอลัมน์แต่หน้าจอใช้ 4 คอลัมน์ การ select เฉพาะที่ใช้ต่างกันมากเมื่อข้อมูลเยอะ

---

## 6. Paging + Sorting — endpoint ค้นหาแบบเต็ม

```java
@ApplicationScoped
public class ProductSearchService {

    @Inject ProductRepository repo;

    public PageResult<ProductSummary> search(SearchCriteria c) {

        Sort sort = Sort.by(c.sortBy() == null ? "code" : c.sortBy(),
                            "desc".equalsIgnoreCase(c.sortDir())
                                ? Sort.Direction.Descending
                                : Sort.Direction.Ascending);

        PanacheQuery<Product> q = repo.find("""
                    (:kw is null or lower(name) like :kw)
                    and (:st is null or status = :st)
                    """,
                sort,
                Parameters.with("kw", c.keyword() == null ? null : "%" + c.keyword().toLowerCase() + "%")
                          .and("st", c.status()));

        List<ProductSummary> rows = q.page(Page.of(c.page(), c.size()))
                                     .project(ProductSummary.class)
                                     .list();

        return new PageResult<>(rows, q.count(), q.pageCount(), c.page(), c.size());
    }
}

public record SearchCriteria(String keyword, Product.Status status,
                             int page, int size, String sortBy, String sortDir) {}

public record PageResult<T>(List<T> rows, long total, int pageCount, int page, int size) {}
```

**เรื่อง paging ที่ต้องรู้**

```java
q.page(Page.of(0, 20));       // หน้าแรก 20 รายการ (index เริ่มที่ 0)
q.page(Page.ofSize(20));      // เท่ากับ Page.of(0, 20)
q.nextPage(); q.previousPage();
q.hasNextPage();
q.pageCount();                // ยิง count query เพิ่มอีก 1 ครั้ง
q.range(0, 19);               // อีกแบบหนึ่ง — ผสมกับ page() ไม่ได้
```

> `count()` กับ `pageCount()` **ยิง query เพิ่ม** ถ้า UI ไม่ได้ใช้ตัวเลขรวม อย่าเรียก

---

## 7. Relation + กัน N+1

```java
// ❌ N+1 — วน 100 รอบ ยิง query 101 ครั้ง
for (Product p : Product.listAll()) {
    System.out.println(p.category.name);      // lazy load ทีละตัว
}

// ✅ JOIN FETCH — query เดียว
List<Product> ps = Product.list(
    "select distinct p from Product p left join fetch p.category");
```

ถ้าอยาก reuse query เดิมแต่คุม fetch ต่างกันตาม use case ใช้ `@EntityGraph`
หรือแก้แบบกว้าง ๆ ด้วย `@BatchSize` บน relation

รายละเอียดเรื่อง N+1 อยู่ใน [[Quarkus Hibernate]]

---

## 8. Bulk update / delete

```java
@Transactional
public int discontinueOld(LocalDate before) {
    // UPDATE ตรง ๆ ไม่โหลด entity เข้ามาก่อน
    return Product.update("status = ?1 where releasedOn < ?2",
                          Status.DISCONTINUED, before);
}

@Transactional
public long purgeDrafts() {
    return Product.delete("status", Status.DRAFT);
}
```

> ⚠️ **bulk operation ข้าม persistence context** — entity ที่โหลดไว้ก่อนหน้าจะยังเป็นค่าเก่า
> ถ้าจะใช้ต่อในทรานแซกชันเดียวกัน ต้อง `em.clear()` หรือโหลดใหม่

### insert เป็นชุด

```java
@Transactional
public void importAll(List<ProductCreateRequest> rows) {
    for (int i = 0; i < rows.size(); i++) {
        Product p = toEntity(rows.get(i));
        p.persist();
        if (i % 50 == 0) {
            Panache.getEntityManager().flush();
            Panache.getEntityManager().clear();   // ← ขาดตัวนี้ = memory บวมจนตาย
        }
    }
}
```

---

## 9. Resource layer

```java
@Path("/products")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class ProductResource {

    @Inject ProductService service;
    @Inject ProductSearchService searchService;

    @GET
    public PageResult<ProductSummary> search(
            @QueryParam("keyword") String keyword,
            @QueryParam("status") Product.Status status,
            @QueryParam("page") @DefaultValue("0") int page,
            @QueryParam("size") @DefaultValue("20") int size,
            @QueryParam("sortBy") String sortBy,
            @QueryParam("sortDir") @DefaultValue("asc") String sortDir) {
        return searchService.search(
            new SearchCriteria(keyword, status, page, size, sortBy, sortDir));
    }

    @GET @Path("/{id}")
    public ProductSummary get(@PathParam("id") Long id) {
        Product p = service.get(id);
        return new ProductSummary(p.id, p.code, p.name, p.price);
    }

    @POST
    public Response create(@Valid ProductCreateRequest req) {
        Product p = service.create(req);
        return Response.created(URI.create("/products/" + p.id)).build();
    }

    @DELETE @Path("/{id}")
    public void delete(@PathParam("id") Long id) {
        service.delete(id);
    }
}
```

---

## 10. Test

```java
@QuarkusTest
class ProductServiceTest {

    @Inject ProductService service;

    @Test
    @TestTransaction            // ← rollback อัตโนมัติหลังจบเทส
    void create_then_find() {
        Product p = service.create(
            new ProductCreateRequest("P001", "Test", BigDecimal.TEN));

        assertNotNull(p.id);
        assertEquals("P001", Product.<Product>findById(p.id).code);
    }

    @Test
    @TestTransaction
    void duplicate_code_rejected() {
        service.create(new ProductCreateRequest("P002", "A", BigDecimal.ONE));
        assertThrows(IllegalArgumentException.class,
            () -> service.create(new ProductCreateRequest("P002", "B", BigDecimal.ONE)));
    }
}
```

**`@TestTransaction` ทำให้ไม่ต้องเขียนโค้ดล้างข้อมูลเอง** — ทุกอย่าง rollback หลังเทสจบ

### mock

```java
// repository — Mockito ปกติ
@InjectMock ProductRepository repo;

// active record — static method ต้องใช้ quarkus-panache-mock
PanacheMock.mock(Product.class);
Mockito.when(Product.findByCode("P001")).thenReturn(Optional.of(sample));
```

---

## 11. Checklist ก่อน merge

- [ ] `@Transactional` อยู่ที่ service ไม่ใช่ resource
- [ ] method ที่แค่อ่าน ไม่มี `@Transactional`
- [ ] ไม่คืน entity ออก REST ตรง ๆ — ใช้ record/DTO
- [ ] endpoint ที่คืน list มี paging เสมอ ไม่มี `listAll()` โผล่ใน production path
- [ ] ที่ไหนวน loop แตะ relation ต้องมี `join fetch`
- [ ] bulk insert มี `flush()` + `clear()`
- [ ] `stream()` อยู่ใน try-with-resources
- [ ] ไม่ผสม active record กับ repository ในโปรเจกต์เดียว

---

## 🔗 เกี่ยวข้อง

- [[Quarkus Panache]] — ทฤษฎีและการเลือกแนวทาง
- [[Quarkus Hibernate]] — N+1, transaction, bulk insert แบบละเอียด
- [[Quarkus]] — หน้ารวม

## 📖 อ่านต่อ

- [Quarkus — Simplified Hibernate ORM with Panache](https://quarkus.io/guides/hibernate-orm-panache)
