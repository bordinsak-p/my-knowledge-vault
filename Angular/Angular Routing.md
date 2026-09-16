---
tags:
  - angular
  - typescript
  - routing
type: reference
created: 2026-09-16
---

# 🧭 Angular Routing — เปลี่ยนหน้าโดยไม่โหลดหน้าใหม่

> Router จับคู่ **URL path** กับ **component ที่ควรแสดง** — เปลี่ยนหน้าโดยไม่ reload ทั้งหน้าเว็บ (SPA) แต่ browser ยัง back/forward/bookmark URL ได้ปกติเหมือนเว็บทั่วไป

---

## 1. ติดตั้งและประกาศเส้นทาง

```typescript
// app.routes.ts
export const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'products', component: ProductListComponent },
  { path: 'products/:id', component: ProductDetailComponent },
  { path: '**', component: NotFoundComponent },   // ต้องอยู่บรรทัดสุดท้ายเสมอ
];
```

```typescript
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [provideRouter(routes)],
};
```

```html
<!-- app.component.html -->
<nav>
  <a routerLink="/">หน้าแรก</a>
  <a routerLink="/products" routerLinkActive="active">สินค้า</a>
</nav>
<router-outlet></router-outlet>   <!-- component ตาม route ปัจจุบันแสดงตรงนี้ -->
```

**`**` (wildcard) ต้องอยู่ล่างสุดเสมอ** — Router จับคู่จากบนลงล่าง เจอตัวแรกที่ match ก็หยุด ถ้าเอา `**` ไว้บนจะจับทุก path ก่อนตัวอื่นเลย

---

## 2. อ่านค่าจาก route — path param กับ query param

```typescript
export class ProductDetailComponent {
  private route = inject(ActivatedRoute);
  productId = signal<string | null>(null);

  constructor() {
    this.route.paramMap.subscribe(params => {
      this.productId.set(params.get('id'));
    });
  }
}
```

### ทางลัด — bind path param ตรงเข้า `@Input()` เลย (ไม่ต้องผ่าน `ActivatedRoute`)

```typescript
provideRouter(routes, withComponentInputBinding());
```

```typescript
export class ProductDetailComponent {
  id = input.required<string>();   // Angular เติมค่าจาก :id ให้อัตโนมัติ
}
```

**`withComponentInputBinding()` ทำให้ path/query param กลายเป็น component input ตรง ๆ** — โค้ดสั้นลงมาก ไม่ต้อง subscribe `ActivatedRoute` เอง

**กฎเลือกใช้:** path param (`/products/:id`) สำหรับสิ่งที่ระบุ "ตัวไหน" (resource identity), query param (`?sort=price`) สำหรับ "กรอง/เรียงยังไง" (context ที่ไม่ใช่ identity)

---

## 3. Lazy loading — โหลดโค้ดของหน้านั้นเมื่อจำเป็นเท่านั้น

```typescript
{
  path: 'admin',
  loadComponent: () => import('./admin/admin.component').then(m => m.AdminComponent),
}
```

**`loadComponent` แทนที่จะ `import` แบบปกติที่หัวไฟล์** — โค้ดของ `AdminComponent` จะไม่ถูกรวมเข้า bundle หลักเลย แยกเป็นไฟล์ต่างหาก โหลดเฉพาะตอนผู้ใช้เข้า path `/admin` จริง ๆ — bundle เริ่มต้นเล็กลง โหลดหน้าแรกเร็วขึ้น

---

## 4. Guard — กันไม่ให้เข้าบาง route

```typescript
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);

  if (auth.isLoggedIn()) return true;
  router.navigate(['/login']);
  return false;
};
```

```typescript
{ path: 'admin', canActivate: [authGuard], component: AdminComponent }
```

| guard | ทำงานตอนไหน | ใช้เมื่อ |
|---|---|---|
| `canActivate` | ก่อนเข้า route ที่ **โหลดโค้ดมาแล้ว** | เช็คสิทธิ์ทั่วไป |
| `canMatch` | **ก่อน**จะโหลดโค้ดของ lazy route ด้วยซ้ำ | กันไม่ให้ download โค้ดของ route ที่ไม่มีสิทธิ์เข้าเลยตั้งแต่แรก |

**`canMatch` ดีกว่า `canActivate` สำหรับ lazy-loaded route ที่ sensitive** — user ที่ไม่มีสิทธิ์จะไม่ได้แม้แต่ดาวน์โหลด JS chunk ของหน้านั้นเลย

---

## 5. กับดัก

- **ลืม import `RouterOutlet`/`RouterLink` ใน standalone component** — component ที่ใช้ `<router-outlet>`/`routerLink` ต้อง import ใน `imports: []` ของตัวเอง ไม่เหมือนสมัย NgModule ที่ import รวมไว้ที่เดียว
- **`**` ไม่ได้อยู่บรรทัดสุดท้าย** — จับทุก path ไปก่อน route อื่นที่ประกาศไว้ทีหลัง
- **ใช้ `canActivate` แทน `canMatch` กับ lazy route ที่อยากกันตั้งแต่ต้น** — โค้ดของหน้านั้นถูกดาวน์โหลดไปแล้วก่อนถึงจะเช็คสิทธิ์
- **path param กับ query param ใช้สลับหน้าที่กัน** — เอา identity (id) ไปใส่เป็น query param ทำให้ bookmark/share URL ดูแปลก ๆ
- **relative path กับ absolute path สับสน** — `routerLink="products"` (relative) กับ `routerLink="/products"` (absolute) ให้ผลต่างกันขึ้นกับตำแหน่งปัจจุบัน

---

## 6. Cheat sheet

```typescript
provideRouter(routes, withComponentInputBinding())

{ path: 'x/:id', component: X }
{ path: 'admin', loadComponent: () => import('./admin.component').then(m => m.Admin) }
{ path: 'admin', canMatch: [authGuard], loadComponent: ... }
{ path: '**', component: NotFound }   // ล่างสุดเสมอ
```

```html
<a routerLink="/products" routerLinkActive="active">สินค้า</a>
<router-outlet></router-outlet>
```

| อาการ | สาเหตุ |
|---|---|
| `<router-outlet>` ไม่แสดงผลอะไร | ลืม import `RouterOutlet` ใน component |
| ทุก path ไปตกที่หน้าเดียวกันหมด | `**` อยู่ไม่ใช่บรรทัดสุดท้าย |
| user ที่ไม่มีสิทธิ์ยังโหลด JS chunk ของหน้า admin ได้ | ใช้ `canActivate` แทน `canMatch` |
| component input ไม่ได้ค่าจาก route param | ไม่ได้เปิด `withComponentInputBinding()` |

## 🔗 เกี่ยวข้อง

- [[Angular Services and DI]] — guard เป็นฟังก์ชันที่ `inject()` service เข้ามาเช็คสิทธิ์
- [[Angular Lifecycle Hooks]] — component ที่เปลี่ยนไปตาม route ก็มี lifecycle ปกติ

## 📖 อ่านต่อ

- [Angular — Routing overview](https://angular.dev/guide/routing)
- [Angular — Customizing route behavior](https://angular.dev/guide/routing/customizing-route-behavior)
