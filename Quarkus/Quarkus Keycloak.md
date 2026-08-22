---
tags:
  - java
  - quarkus
  - angular
  - keycloak
  - oidc
  - security
type: reference
created: 2026-08-18
---

# 🔐 Keycloak + Angular + Quarkus — เริ่มจากศูนย์

> สามชิ้นนี้ทำงานคนละหน้าที่ **Keycloak เป็นคนกลางที่ทุกฝ่ายเชื่อใจ** — Angular ไม่เก็บรหัสผ่านผู้ใช้เอง, Quarkus ไม่ต้องตรวจรหัสผ่านเอง ทั้งคู่แค่ **เชื่อ token ที่ Keycloak เซ็นมาให้**
> นี่คือมาตรฐาน **OIDC (OpenID Connect)** ซึ่งวางอยู่บน OAuth2 อีกที

---

## 1. ภาพรวมก่อนลงมือ — ใครทำหน้าที่อะไร

```
┌─────────────┐        1. ยังไม่ login            ┌─────────────┐
│   Angular   │ ─────────────────────────────►  │  Keycloak   │
│  (browser)  │                                 │(auth server)│
│             │ ◄─────────────────────────────  │             │
└─────────────┘   2. redirect ไปหน้า login       └─────────────┘
      │                                               │
      │  3. กรอก user/pass ที่หน้า Keycloak (ไม่ใช่ที่ Angular!)
      │                                               │
      │  4. Keycloak ส่ง token กลับมาที่ Angular          │
      │◄──────────────────────────────────────────────┘
      │
      │  5. แนบ token ไปกับทุก request
      ▼
┌─────────────┐
│   Quarkus   │  6. ตรวจลายเซ็นของ token กับ Keycloak
│  (API)      │     (ไม่ต้องถาม Keycloak ทุกครั้ง — ตรวจด้วยกุญแจสาธารณะเอง)
└─────────────┘
```

**สิ่งที่ควรจำจากภาพนี้:** ผู้ใช้กรอก username/password **ที่หน้าเว็บของ Keycloak เท่านั้น** ไม่ใช่ที่ฟอร์มของ Angular — Angular แค่ redirect ไปแล้วรอรับ token กลับมา นี่คือหัวใจของ OIDC: **แอปของเราไม่เคยเห็นรหัสผ่านจริงเลย**

### คำศัพท์ที่ต้องรู้ก่อน

| คำ | แปลว่า |
|---|---|
| **Realm** | พื้นที่แยกผู้ใช้/สิทธิ์ของแต่ละระบบ เหมือน "องค์กร" ใน Keycloak (มีได้หลาย realm ในเครื่อง Keycloak เดียว) |
| **Client** | แอปที่ขอเข้ามาใช้ Keycloak — Angular นับเป็น client หนึ่งตัว, Quarkus API นับเป็นอีกตัว |
| **public client** | client ที่ไม่มี secret เก็บได้ (เช่น Angular ที่รันในเบราว์เซอร์ ดู source ได้หมด) |
| **confidential client** | client ที่มี secret เก็บได้ (เช่น Quarkus ที่รันบน server) |
| **Access Token** | ตั๋วอายุสั้น (นาที) แนบไปกับทุก API call เพื่อพิสูจน์ตัวตน |
| **Refresh Token** | ตั๋วอายุยาวกว่า ใช้ขอ access token ใหม่โดยไม่ต้อง login ซ้ำ |
| **JWT** | รูปแบบของ token — เป็น JSON ที่เซ็นด้วยลายเซ็นดิจิทัล ตรวจสอบได้โดยไม่ต้องถาม Keycloak ทุกครั้ง |

---

## 2. เริ่มจากฝั่ง Keycloak — ตั้งค่าก่อนแตะโค้ดเลยสักบรรทัด

### รันด้วย Docker

```yaml
# docker-compose.yml
services:
  keycloak:
    image: quay.io/keycloak/keycloak:26.5
    command: start-dev
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports:
      - "8180:8080"
```

```bash
docker compose up -d
```

เปิด `http://localhost:8180` → login ด้วย `admin`/`admin`

> ⚠️ **Keycloak รุ่นใหม่ (ตั้งแต่ 17 ที่เปลี่ยนไปใช้ฐาน Quarkus) ตัด `/auth` ออกจาก path แล้ว**
> ถ้าเจอตัวอย่างเก่าที่เขียน `http://localhost:8180/auth/realms/...` ให้รู้ว่าล้าสมัย ของปัจจุบันคือ `http://localhost:8180/realms/...` **ไม่มี `/auth`**

### สร้าง Realm

```
Admin Console → สร้าง Realm ใหม่ → ตั้งชื่อ "my-app"
```

**ทำไมต้องแยก realm ไม่ใช้ realm `master`** — `master` มีไว้จัดการ Keycloak เอง ไม่ควรเอาผู้ใช้ระบบงานจริงไปปนไว้ที่นั่น

### สร้างสอง Client — หนึ่งสำหรับ Angular หนึ่งสำหรับ Quarkus

```
Client 1: angular-app
  - Client type: OpenID Connect
  - Client authentication: OFF        ← public client, ไม่มี secret
  - Standard flow: ON                 ← authorization code flow
  - Valid redirect URIs: http://localhost:4200/*
  - Web origins: http://localhost:4200

Client 2: quarkus-service
  - Client type: OpenID Connect
  - Client authentication: ON         ← confidential client, มี secret
  - Service accounts roles: ON        ← ถ้าต้องเรียก service-to-service ด้วย
```

**เหตุผลที่แยกสอง client:** Angular รันในเบราว์เซอร์ ใครก็เปิด devtools ดู source ได้ จึงเป็น client แบบไม่มี secret (public) — แต่ Quarkus รันบน server เก็บ secret ได้จริง จึงเป็น confidential client และยังใช้ยืนยันตัวตนแบบ service-to-service ได้ด้วย (ข้อ 6)

### สร้าง Role และผู้ใช้ทดสอบ

```
Realm roles → สร้าง: admin, user
Users → สร้างผู้ใช้ทดสอบ → ตั้งรหัสผ่าน → Role mapping → ผูก role ให้
```

---

## 3. ฝั่ง Quarkus — ตรวจ token ที่ client ส่งมา

### ติดตั้ง

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-oidc</artifactId>
</dependency>
```

### Config

```properties
quarkus.oidc.auth-server-url=http://localhost:8180/realms/my-app
quarkus.oidc.client-id=quarkus-service
quarkus.oidc.credentials.secret=${OIDC_SECRET}
quarkus.oidc.application-type=service          # ← "service" = bearer-only, ตรวจ token อย่างเดียว ไม่ redirect เอง
quarkus.oidc.roles.source=realm                # ← ใช้ realm role ไม่ใช่ client role
```

`application-type=service` สำคัญมาก — บอก Quarkus ว่า **"เราเป็น API เฉย ๆ ไม่ใช่หน้าเว็บที่ต้อง redirect ไป login เอง"** (เว็บที่ต้อง redirect เองใช้ `application-type=web-app` แทน ซึ่งเป็นอีกโมเดลที่ไม่เกี่ยวกับ Angular แยก frontend/backend แบบนี้)

### ป้องกัน endpoint

```java
@Path("/api/products")
public class ProductResource {

    @Inject SecurityIdentity identity;

    @GET
    @Authenticated                    // ต้อง login มา แต่ role อะไรก็ได้
    public List<Product> list() { ... }

    @POST
    @RolesAllowed("admin")            // ต้องมี realm role นี้
    public Response create(ProductCreateRequest req) { ... }

    @GET
    @Path("/me")
    @Authenticated
    public Map<String, Object> whoAmI() {
        return Map.of(
            "username", identity.getPrincipal().getName(),
            "roles", identity.getRoles());
    }
}
```

**สามชั้นของการป้องกัน**

```
ไม่มี annotation เลย   → ใครก็เรียกได้ ไม่ต้อง login
@Authenticated         → ต้อง login แล้ว (มี token ที่ถูกต้อง) role อะไรก็ได้
@RolesAllowed("admin") → ต้อง login แล้ว และมี role ตรงตามที่ระบุ
```

### อ่านค่าจาก token โดยตรง

```java
@Inject JsonWebToken jwt;

public String currentUserEmail() {
    return jwt.getClaim("email");
}
```

**ไม่ต้อง query DB เพื่อรู้ว่าใครเป็นใคร** — ข้อมูลพื้นฐาน (username, email, role) อยู่ใน token อยู่แล้ว เพราะ Keycloak เซ็นมาให้ Quarkus แค่ตรวจลายเซ็นว่าไม่ได้ถูกปลอม

---

## 4. ⚠️ CORS — จุดที่ทุกคนติดตอนต่อ Angular กับ Quarkus ครั้งแรก

**Angular กับ Quarkus รันคนละพอร์ตกันตอน dev** (`localhost:4200` vs `localhost:8080`) — เบราว์เซอร์ถือว่าเป็นคนละ origin เลยบล็อก request ข้ามไปมา จนกว่าจะอนุญาตชัดเจน

```properties
quarkus.http.cors=true
quarkus.http.cors.origins=http://localhost:4200
quarkus.http.cors.headers=Authorization,Content-Type
quarkus.http.cors.methods=GET,POST,PUT,PATCH,DELETE,OPTIONS
```

### กับดักเฉพาะทางที่เกิดจากการผสม OIDC เข้ากับ CORS

**เบราว์เซอร์ยิง `OPTIONS` (preflight) ไปถามก่อนเสมอ ก่อนจะยิง request จริงที่มี `Authorization` header** — ถ้า Quarkus บังคับ auth กับทุก path รวม `OPTIONS` ด้วย preflight จะโดนปฏิเสธเป็น 401 **ก่อนที่ CORS header จะถูกส่งกลับด้วยซ้ำ** ผลคือเบราว์เซอร์เห็นแค่ error CORS ทั้งที่ต้นตอจริงคือ auth

```properties
# อนุญาต OPTIONS โดยไม่ต้องผ่าน auth
quarkus.http.auth.permission.cors-preflight.paths=/api/*
quarkus.http.auth.permission.cors-preflight.methods=OPTIONS
quarkus.http.auth.permission.cors-preflight.policy=permit
```

### วิธีวินิจฉัยเมื่อ CORS พัง

```
1. เปิด DevTools → Network → หา request ที่เป็น OPTIONS (มาก่อน GET/POST เสมอ)
2. OPTIONS ตอบ 200 ไหม? ถ้าตอบ 401/403 = auth บล็อก preflight (ดูด้านบน)
3. response มี header Access-Control-Allow-Origin ตรงกับ origin ของ Angular ไหม?
4. ถ้า Angular ส่ง withCredentials: true → backend ต้องระบุ origin ตรง ๆ ใช้ * ไม่ได้
```

### ทางเลือกที่ตัดปัญหา CORS ทิ้งไปเลยตอน dev — Angular proxy

```json
// proxy.conf.json
{
  "/api": { "target": "http://localhost:8080", "changeOrigin": true }
}
```

Angular เรียก `/api/products` แล้ว dev server เป็นคนส่งต่อไปที่ Quarkus เอง — จากมุมมองเบราว์เซอร์เหมือนเรียก origin เดียวกันหมด **ไม่ต้องตั้ง CORS ฝั่ง Quarkus เลยตอน dev** (แต่ตอน deploy จริงส่วนใหญ่แก้ด้วยการเอา Angular กับ API ไปอยู่ domain เดียวกันผ่าน reverse proxy แทน ซึ่งก็ไม่ต้องมี CORS เหมือนกัน)

---

## 5. ฝั่ง Angular — login แล้วแนบ token อัตโนมัติ

### ติดตั้ง

```bash
npm install keycloak-angular keycloak-js
```

### ตั้งค่า

```typescript
// app.config.ts
import { provideKeycloak, withAutoRefreshToken,
         AutoRefreshTokenService, UserActivityService,
         includeBearerTokenInterceptor,
         INCLUDE_BEARER_TOKEN_INTERCEPTOR_CONFIG } from 'keycloak-angular';

export const appConfig: ApplicationConfig = {
  providers: [
    provideKeycloak({
      config: {
        url: 'http://localhost:8180',
        realm: 'my-app',
        clientId: 'angular-app',
      },
      initOptions: { onLoad: 'login-required' },   // บังคับ login ก่อนเข้าแอปเลย
      features: [ withAutoRefreshToken() ],          // ต่ออายุ token ให้อัตโนมัติ
    }),
    AutoRefreshTokenService,
    UserActivityService,

    provideHttpClient(withInterceptors([includeBearerTokenInterceptor])),
    {
      provide: INCLUDE_BEARER_TOKEN_INTERCEPTOR_CONFIG,
      useValue: [{ urlPattern: /^http:\/\/localhost:8080(\/.*)?$/ }],  // แนบ token เฉพาะ request ไป Quarkus เท่านั้น
    },
  ],
};
```

**`INCLUDE_BEARER_TOKEN_INTERCEPTOR_CONFIG` สำคัญกว่าที่คิด** — ถ้าไม่ระบุ pattern ให้ชัด interceptor อาจแนบ token ไปกับ**ทุก** request รวมถึง request ไปโดเมนอื่นที่ไม่ใช่ API ของเรา (เช่น CDN, third-party) ซึ่งเป็นการรั่วไหลของ token ที่ไม่ควรเกิด

### `onLoad` สองแบบ เลือกให้ตรงกับที่ต้องการ

| ค่า | พฤติกรรม |
|---|---|
| `login-required` | บังคับ login ทันทีที่เปิดแอป ก่อนเห็นหน้าไหนเลย |
| `check-sso` | เช็คเงียบ ๆ ว่ามี session อยู่แล้วไหม ไม่บังคับ login ทันที เหมาะกับแอปที่มีหน้า public ปนอยู่ |

### Route Guard ตาม role

```typescript
// app.routes.ts
import { authGuardFn } from 'keycloak-angular';

export const routes: Routes = [
  { path: 'admin', component: AdminComponent,
    canActivate: [authGuardFn], data: { roles: ['admin'] } },
  { path: 'dashboard', component: DashboardComponent,
    canActivate: [authGuardFn] },     // login แล้วเข้าได้ ไม่จำกัด role
];
```

### อ่านข้อมูลผู้ใช้ในโค้ด

```typescript
@Injectable({ providedIn: 'root' })
export class AuthService {
  private keycloak = inject(Keycloak);

  get username() { return this.keycloak.tokenParsed?.['preferred_username'] ?? ''; }
  get roles(): string[] { return this.keycloak.tokenParsed?.['realm_access']?.['roles'] ?? []; }
  hasRole(r: string) { return this.roles.includes(r); }
  logout() { this.keycloak.logout({ redirectUri: window.location.origin }); }
}
```

```html
<div *ngIf="authService.username as name">
  สวัสดี {{ name }}
  <span *ngIf="authService.hasRole('admin')">(ผู้ดูแลระบบ)</span>
</div>
```

---

## 6. ต่อทั้งสามฝั่งเข้าด้วยกัน — ไล่ทีละ request

```
① ผู้ใช้เปิด Angular (localhost:4200)
② keycloak-angular เช็คว่ามี session ไหม → ไม่มี → redirect ไป Keycloak
③ ผู้ใช้กรอก user/pass ที่หน้า Keycloak (localhost:8180)
④ Keycloak redirect กลับมา Angular พร้อม authorization code
⑤ keycloak-js แลก code เป็น access token + refresh token (เบื้องหลัง ไม่เห็นในหน้าจอ)
⑥ Angular เรียก GET localhost:8080/api/products
   interceptor แนบ header:  Authorization: Bearer eyJhbGc...
⑦ Quarkus (quarkus-oidc) ตรวจลายเซ็นของ token
   → ใช้กุญแจสาธารณะที่ดึงมาจาก Keycloak ตอน startup (JWKS endpoint)
   → ตรวจสอบเอง ไม่ต้องถาม Keycloak ทุก request (เร็ว)
⑧ ผ่าน → SecurityIdentity ถูกเติมด้วย username/roles จาก token
⑨ @RolesAllowed("admin") เช็ค role จาก SecurityIdentity
⑩ ตอบ response กลับ Angular
```

**จุดที่ควรเข้าใจให้แม่น: ข้อ ⑦** — Quarkus **ไม่ได้เรียก Keycloak ทุกครั้งที่มี request เข้ามา** มันดึงกุญแจสาธารณะ (public key) มาเก็บไว้ตอน startup แล้วตรวจลายเซ็นของ JWT ด้วยตัวเอง นี่คือเหตุผลที่ระบบแบบนี้ scale ได้ดี — auth server ไม่ได้กลายเป็นคอขวดที่ทุก request ต้องผ่าน

---

## 7. Service-to-service — เมื่อ Quarkus ต้องเรียก API อื่น

ถ้ามี Quarkus service ตัวที่สองที่ต้องเรียกกันเอง (ไม่ใช่ผู้ใช้เป็นคนเรียก) ใช้ **client credentials flow** — client authenticate ด้วยตัวเอง ไม่มีผู้ใช้เกี่ยวข้อง

```properties
quarkus.oidc-client.auth-server-url=http://localhost:8180/realms/my-app
quarkus.oidc-client.client-id=quarkus-service
quarkus.oidc-client.credentials.secret=${OIDC_SECRET}
quarkus.oidc-client.grant.type=client-credentials
```

```java
@RegisterRestClient(configKey = "inventory-api")
@RegisterProvider(OidcClientRequestFilter.class)   // แนบ token ให้อัตโนมัติ
public interface InventoryClient {
    @GET @Path("/items")
    List<Item> getItems();
}
```

ดูรายละเอียดเรื่อง REST client เพิ่มที่ [[Quarkus REST Client]]

---

## 8. Test — ปิด auth ตอน dev/test ได้

```java
@QuarkusTest
class ProductResourceTest {

    @Test
    @TestSecurity(user = "somchai", roles = {"admin"})
    void admin_can_create() { ... }

    @Test
    @TestSecurity(user = "malee", roles = {"user"})
    void non_admin_cannot_create() { ... }
}
```

`@TestSecurity` จำลอง `SecurityIdentity` ให้เลย **ไม่ต้องมี Keycloak รันจริงตอนรัน unit test** — สร้าง token จริงเองทุกครั้งจะช้าและเปราะ

---

## 9. กับดักเฉพาะของการผสมสามตัวนี้

- **ใช้ตัวอย่างเก่าที่มี `/auth/realms/...`** — Keycloak ตัด `/auth` ออกไปนานแล้ว (ตั้งแต่รุ่น 17) ตัวอย่างบนเว็บจำนวนมากยังเขียนแบบเก่า ก๊อปมาแล้ว 404 ทันที
- **preflight `OPTIONS` โดน auth บล็อก** — เห็น error เป็น CORS แต่ต้นตอจริงคือ security policy (ข้อ 4)
- **interceptor แนบ `Authorization` ไปทุก request รวมโดเมนอื่น** — ต้องระบุ URL pattern ให้ชัดใน `INCLUDE_BEARER_TOKEN_INTERCEPTOR_CONFIG`
- **`application-type` ผิด** — ตั้ง `web-app` ทั้งที่ Quarkus เป็น pure API (ควรเป็น `service`) ทำให้ Quarkus พยายาม redirect เองซึ่งไม่มีความหมายสำหรับ backend-only
- **Web Origins ใน Keycloak client ไม่ตรงกับ URL จริงของ Angular** — token endpoint ปฏิเสธ request จาก origin ที่ไม่ได้อนุญาตไว้
- **Angular เป็น confidential client โดยไม่ตั้งใจ** (เผลอเปิด "Client authentication") — secret ฝังอยู่ใน JS bundle ที่ใคร ๆ ก็เปิดดูได้ ทำให้ secret รั่วทันที ต้องเป็น public client เสมอสำหรับ SPA
- **ทดสอบด้วยการยิง Keycloak จริงในทุก unit test** — ช้าและเปราะ ใช้ `@TestSecurity` แทนสำหรับ logic ทั่วไป เก็บการเทส Keycloak จริงไว้แค่ integration test ไม่กี่เคส
- **role ไม่ตรงกันเพราะลืม `quarkus.oidc.roles.source=realm`** — ถ้า role ถูกสร้างไว้ที่ระดับ realm แต่ Quarkus ไปมองหาที่ client role (ค่า default บางกรณี) `@RolesAllowed` จะไม่เจอ role เลยทั้งที่ token มีอยู่จริง

---

## 10. เริ่มลองยังไง — ทีละขั้น

```
1. รัน Keycloak ด้วย docker-compose (ข้อ 2)
2. สร้าง realm + สอง client + role + user ทดสอบผ่าน Admin Console
3. สร้างโปรเจกต์ Quarkus ใหม่ผ่าน code.quarkus.io เลือก extension "OIDC"
4. ใส่ config ข้อ 3 แล้วลองยิง endpoint ผ่าน curl พร้อม token ปลอมดูก่อนว่าโดนบล็อกจริง
5. ลองขอ token จริงด้วย curl (client credentials หรือ direct grant) มายิงดู ว่า @Authenticated ผ่าน
6. สร้างโปรเจกต์ Angular ใหม่ ติดตั้ง keycloak-angular ตามข้อ 5
7. ตั้ง proxy.conf.json ตัดปัญหา CORS ไปก่อนตอนหัดเล่น
8. login ผ่าน Angular แล้วดูใน DevTools → Network ว่า Authorization header ถูกแนบไปจริงไหม
9. ค่อยลบ proxy ออก แล้วตั้ง CORS ฝั่ง Quarkus เอง เพื่อเข้าใจว่าทำไมมันจำเป็นตอน deploy จริง
```

**อย่าข้ามขั้นตอน curl (ข้อ 4-5)** — การยิง token ผ่าน curl ตรง ๆ ก่อน ทำให้เห็นว่า Keycloak/Quarkus ทำงานถูกหรือไม่ โดยไม่มีตัวแปรจาก Angular มาปน ถ้า curl ผ่านแล้ว Angular ไม่ผ่าน แปลว่าปัญหาอยู่ฝั่ง frontend/CORS แน่นอน ไม่ต้องเดา

```bash
# ตัวอย่างขอ token ด้วย curl (direct grant — เปิดใน client settings ก่อน)
curl -X POST http://localhost:8180/realms/my-app/protocol/openid-connect/token \
  -d "client_id=angular-app" \
  -d "grant_type=password" \
  -d "username=testuser" \
  -d "password=testpass"
```

---

## 11. Cheat sheet

```properties
# Quarkus
quarkus.oidc.auth-server-url=http://localhost:8180/realms/my-app
quarkus.oidc.client-id=quarkus-service
quarkus.oidc.credentials.secret=${OIDC_SECRET}
quarkus.oidc.application-type=service
quarkus.oidc.roles.source=realm
quarkus.http.cors=true
quarkus.http.cors.origins=http://localhost:4200
quarkus.http.auth.permission.cors-preflight.paths=/api/*
quarkus.http.auth.permission.cors-preflight.methods=OPTIONS
quarkus.http.auth.permission.cors-preflight.policy=permit
```

```java
@Authenticated                    // login แล้ว role อะไรก็ได้
@RolesAllowed("admin")            // ต้องมี role นี้
@Inject SecurityIdentity identity;
@Inject JsonWebToken jwt;
@TestSecurity(user = "x", roles = {"admin"})   // เทส
```

```typescript
provideKeycloak({ config: {...}, initOptions: { onLoad: 'login-required' } })
provideHttpClient(withInterceptors([includeBearerTokenInterceptor]))
canActivate: [authGuardFn], data: { roles: ['admin'] }
```

| อาการ | สาเหตุ |
|---|---|
| `404` ตอนต่อ Keycloak | ยังใช้ path เก่าที่มี `/auth` |
| CORS error แต่ config CORS ถูกแล้ว | preflight `OPTIONS` โดน auth บล็อกก่อนถึง CORS handler |
| token ถูกแนบไปโดเมนอื่นที่ไม่ใช่ API เรา | ไม่ได้ระบุ `INCLUDE_BEARER_TOKEN_INTERCEPTOR_CONFIG` |
| `@RolesAllowed` ไม่ผ่านทั้งที่ token มี role | `quarkus.oidc.roles.source` ไม่ตรงกับที่ role ถูกสร้างไว้ (realm vs client) |
| secret ของ Angular โผล่ใน devtools | ตั้ง Angular เป็น confidential client โดยไม่ตั้งใจ ต้องเป็น public |
| unit test ช้าเพราะรอ Keycloak จริง | ควรใช้ `@TestSecurity` แทน |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus REST Client]] — service-to-service auth ด้วย `quarkus-oidc-client`
- [[Quarkus HTTP Methods]] — CORS preflight เป็น `OPTIONS` request
- [[Quarkus Health Check]] — health endpoint ควรเปิดให้เข้าถึงได้โดยไม่ต้อง auth เสมอ
- [[Quarkus]] — หน้ารวม

## 📖 อ่านต่อ

- [Quarkus — Using OpenID Connect to Protect Service Applications](https://quarkus.io/guides/security-oidc-bearer-token-authentication)
- [Keycloak — Getting Started](https://www.keycloak.org/getting-started)
- [keycloak-angular — README](https://github.com/mauriciovigolo/keycloak-angular)
