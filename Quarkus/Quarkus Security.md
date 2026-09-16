---
tags:
  - java
  - quarkus
  - security
  - oidc
type: reference
created: 2026-09-16
---

# 🔒 Quarkus Security — Custom Permission & Multi-tenant OIDC

> พื้นฐาน OIDC/Keycloak (`@Authenticated`, `@RolesAllowed`, realm/client, CORS) เขียนไว้แล้วที่ [[Quarkus Keycloak]] — โน้ตนี้ต่อยอดสองเรื่องที่ `@RolesAllowed` เพียวๆ ทำไม่ได้:
> **(1)** "มี role นี้ไหม" ไม่พอ ต้องรู้ "เป็นเจ้าของ resource ตัวนี้จริงไหม" ด้วย
> **(2)** แอปเดียวต้องรองรับหลาย realm/หลาย Keycloak instance พร้อมกัน (multi-tenant)

---

## 1. `SecurityIdentityAugmentor` — เติมข้อมูลเข้า identity หลัง token ผ่านแล้ว

JWT มีแค่ claim ที่ Keycloak ใส่มาให้ตอนออก token — ถ้าต้องการข้อมูลเพิ่มที่ไม่ได้อยู่ใน token (เช่น business unit ปัจจุบันของ user ที่อาจเปลี่ยนได้โดยไม่ต้อง login ใหม่) ต้อง query เพิ่มเองหลัง token validate ผ่านแล้ว

```java
@ApplicationScoped
public class BusinessUnitAugmentor implements SecurityIdentityAugmentor {

    @Inject BusinessUnitService businessUnitService;

    @Override
    public Uni<SecurityIdentity> augment(SecurityIdentity identity,
                                          AuthenticationRequestContext context) {
        if (identity.isAnonymous()) {
            return Uni.createFrom().item(identity);
        }
        return businessUnitService.findByUsername(identity.getPrincipal().getName())
            .onItem().transform(unit -> QuarkusSecurityIdentity.builder(identity)
                .addAttribute("businessUnit", unit)
                .build());
    }

    @Override
    public int priority() {
        return 1000;               // ลำดับรัน ถ้ามีหลาย augmentor พร้อมกัน
    }
}
```

```java
@Inject SecurityIdentity identity;

String unit = identity.getAttribute("businessUnit");
```

**ทุก `@ApplicationScoped` bean ที่ implement interface นี้จะถูกเรียกอัตโนมัติ** หลัง Quarkus ตรวจ token ผ่านแล้ว ก่อนที่ `SecurityIdentity` จะถูกส่งต่อไปให้ endpoint ใช้งาน — ไม่ต้องไปเรียกเองตรงไหน

---

## 2. Custom permission — เมื่อ role อย่างเดียวไม่พอ

`@RolesAllowed("user")` ตอบได้แค่ "มี role นี้ไหม" — ตอบไม่ได้ว่า **"user คนนี้เป็นเจ้าของ resource ตัวที่กำลังจะแก้จริงไหม"** ต้องใช้ `@PermissionsAllowed` คู่กับ `@PermissionChecker`

```java
@Path("/orders/{orderId}")
public class OrderResource {

    @POST
    @Path("/cancel")
    @PermissionsAllowed("order:cancel")
    public Response cancel(@RestPath Long orderId) {
        orderService.cancel(orderId);
        return Response.noContent().build();
    }

    @PermissionChecker("order:cancel")
    boolean canCancelOrder(SecurityIdentity identity, Long orderId) {
        Order order = orderRepository.findById(orderId);
        return order != null
            && order.ownerUsername.equals(identity.getPrincipal().getName());
    }
}
```

**กลไกการจับคู่:** `@PermissionsAllowed("order:cancel")` บน endpoint กับ `@PermissionChecker("order:cancel")` บน method อื่น **จับคู่กันด้วยชื่อ string ที่ตรงกัน** — และ **พารามิเตอร์ `orderId` ใน checker method ถูก map จาก path param `{orderId}` ของ endpoint โดยอัตโนมัติด้วยชื่อที่ตรงกัน** (`SecurityIdentity` inject ได้เสมอไม่ต้องตั้งชื่อ) — Quarkus เป็นคนเรียก checker method ให้เองก่อน endpoint จะทำงาน ไม่ต้องเขียน `if` เช็คเองในตัว endpoint เลย

**กฎของ checker method:** ต้องคืน `boolean` หรือ `Uni<Boolean>` เท่านั้น และ**ห้ามเป็น `private`**

---

## 3. Multi-tenant OIDC — แอปเดียวรองรับหลาย realm

เมื่อแอปเดียวต้องรับ token จากหลาย Keycloak realm/instance พร้อมกัน (เช่น แยกลูกค้าแต่ละราย, แยกสภาพแวดล้อมตามโดเมน) — Quarkus OIDC มี multi-tenancy ในตัว มีสองระดับ

### 3.1 Static — ตั้ง config หลายชุดล่วงหน้า

```properties
quarkus.oidc.tenant-a.auth-server-url=https://keycloak/realms/tenant-a
quarkus.oidc.tenant-a.client-id=app-a

quarkus.oidc.tenant-b.auth-server-url=https://keycloak/realms/tenant-b
quarkus.oidc.tenant-b.client-id=app-b
```

ค่าเริ่มต้น (ไม่ระบุ) Quarkus เดา tenant id จาก **path segment สุดท้าย** ของ URL หรือกำหนดเองผ่าน `TenantResolver` ง่าย ๆ ได้

```java
@ApplicationScoped
public class HeaderTenantResolver implements TenantResolver {
    @Override
    public String resolve(RoutingContext ctx) {
        return ctx.request().getHeader("X-Tenant-Id");   // อ่านจาก header แทน path
    }
}
```

### 3.2 Dynamic — `TenantConfigResolver` กำหนด config เองตอน runtime

```java
@ApplicationScoped
public class DynamicTenantConfigResolver implements TenantConfigResolver {

    @Override
    public Uni<OidcTenantConfig> resolve(RoutingContext ctx,
                                          OidcRequestContext<OidcTenantConfig> requestContext) {
        String tenantId = ctx.request().getHeader("X-Tenant-Id");
        if (tenantId == null) return Uni.createFrom().nullItem();  // null = ใช้ config เริ่มต้น

        return tenantRepository.findConfig(tenantId)
            .onItem().transform(cfg -> OidcTenantConfig.authServerUrl(cfg.issuerUrl())
                .tenantId(tenantId)
                .clientId(cfg.clientId())
                .build());
    }
}
```

**ใช้ dynamic เมื่อ:** จำนวน tenant ไม่ตายตัว/เปลี่ยนบ่อย (เก็บ config ไว้ใน DB แทนที่จะเขียนใน `application.properties` ทีละ tenant) — static พอสำหรับ tenant จำนวนน้อยที่รู้ล่วงหน้าตอน build

---

## 4. กับดัก

- **`SecurityIdentityAugmentor` ไปเรียกอะไรที่ blocking ตรง ๆ** — `augment()` อาจถูกเรียกบน event loop thread (ดู [[Quarkus REST Layer]]) เรียก DB call แบบ blocking ตรงนี้เสี่ยงบล็อก event loop ทั้งแอปเหมือนกับปัญหาที่โน้ตนั้นอธิบายไว้
- **`@PermissionChecker` เป็น `private`** — ไม่ทำงาน ต้อง package-private หรือ public เท่านั้น
- **ชื่อพารามิเตอร์ใน checker method ไม่ตรงกับ path param** — Quarkus จับคู่ด้วยชื่อ ถ้าสะกดไม่ตรงกันเป๊ะจะหา parameter ไม่เจอ
- **ลืมว่า `@RolesAllowed` กับ `@PermissionsAllowed` ตอบคำถามคนละแบบ** — role-based ตอบ "เป็นใคร" permission-based (แบบมี checker) ตอบ "ทำสิ่งนี้กับของชิ้นนี้ได้ไหม" ใช้ผิดที่ทำให้ authorization หลวมกว่าที่ตั้งใจ (เช็คแค่ role แต่ลืมเช็คความเป็นเจ้าของ)
- **multi-tenant resolver คืนค่าผิด tenant เงียบ ๆ** — ถ้า resolve ผิด user ของ tenant หนึ่งอาจได้ token validate ผ่านด้วย config ของอีก tenant โดยไม่มี error ชัดเจน ต้องเทสกรณีสลับ tenant ให้ครอบคลุม
- **TenantConfigResolver คืน config ที่ขาด field จำเป็น** — พังตอน runtime แบบไม่ชัดเจนว่าขาดอะไร ควร validate ค่าที่ดึงมาจาก DB ก่อนสร้าง `OidcTenantConfig` เสมอ

---

## 5. Cheat sheet

```java
// เติมข้อมูลเข้า identity
@ApplicationScoped
public class MyAugmentor implements SecurityIdentityAugmentor {
    public Uni<SecurityIdentity> augment(SecurityIdentity id, AuthenticationRequestContext ctx) { ... }
    public int priority() { return 1000; }
}

// custom permission ตามความเป็นเจ้าของ
@PermissionsAllowed("resource:action")
public Response endpoint(@RestPath Long id) { ... }

@PermissionChecker("resource:action")
boolean check(SecurityIdentity identity, Long id) { ... }   // ชื่อ param ต้องตรงกับ path param

// multi-tenant
quarkus.oidc.<tenant-id>.auth-server-url=...
// หรือ
@ApplicationScoped
public class MyTenantResolver implements TenantResolver { ... }
```

| อาการ | สาเหตุ |
|---|---|
| ข้อมูลเสริมใน `SecurityIdentity` ไม่มา | augmentor ไม่ได้ประกาศ `@ApplicationScoped` หรือ `priority()` ชนกับตัวอื่น |
| `@PermissionChecker` ไม่ถูกเรียกเลย | ชื่อ string ใน `@PermissionsAllowed`/`@PermissionChecker` ไม่ตรงกัน หรือ method เป็น `private` |
| checker method หา parameter ไม่เจอ | ชื่อ parameter ไม่ตรงกับชื่อ path param |
| token จาก tenant หนึ่งใช้ได้กับอีก tenant | resolver คืน tenant id ผิด หรือ default fallback ไม่ได้ตั้งใจ |
| แอปช้าลงหลังเพิ่ม augmentor | เรียก blocking call ใน `augment()` ที่รันบน event loop |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus Keycloak]] — พื้นฐาน OIDC/Keycloak+Angular, `@Authenticated`, `@RolesAllowed`, CORS
- [[Quarkus REST Layer]] — เหตุผลที่ห้าม block event loop ใน `SecurityIdentityAugmentor`/`TenantConfigResolver`
- [[Quarkus]] — หน้ารวม

## 📖 อ่านต่อ

- [Quarkus — Security Tips and Tricks (SecurityIdentityAugmentor)](https://quarkus.io/guides/security-customization)
- [Quarkus — Authorization of web endpoints (@PermissionsAllowed)](https://quarkus.io/guides/security-authorize-web-endpoints-reference/)
- [Quarkus — Using OpenID Connect (OIDC) multitenancy](https://quarkus.io/guides/security-openid-connect-multitenancy/)
