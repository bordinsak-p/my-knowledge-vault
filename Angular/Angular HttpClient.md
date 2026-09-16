---
tags:
  - angular
  - typescript
  - http
type: reference
created: 2026-09-16
---

# 🌐 Angular HttpClient — เรียก API จริง

> เรื่องเดียวที่ทำให้ HttpClient งงตอนเริ่มต้น: **ทุก method (`get`/`post`/...) คืน `Observable` ที่เป็น cold** — เรียกฟังก์ชันเฉย ๆ **ยังไม่มีการยิง request เกิดขึ้นจนกว่าจะ `subscribe()`** (หลักการเดียวกับที่เขียนไว้แล้วใน [[Observable]])

---

## 1. ติดตั้ง

```typescript
// app.config.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(),
  ],
};
```

---

## 2. เรียกใช้ผ่าน service

```typescript
interface Product {
  id: number;
  name: string;
  price: number;
}

@Injectable({ providedIn: 'root' })
export class ProductService {
  private http = inject(HttpClient);   // ✅ inject() ต้องเรียกตอน field init เท่านั้น (ดู [[Angular Services and DI]])

  getAll(): Observable<Product[]> {
    return this.http.get<Product[]>('/api/products');
  }

  create(product: Omit<Product, 'id'>): Observable<Product> {
    return this.http.post<Product>('/api/products', product);
  }
}
```

```typescript
// component
private productService = inject(ProductService);
products = signal<Product[]>([]);

ngOnInit() {
  this.productService.getAll().subscribe(products => this.products.set(products));
}
```

**`this.http.get(...)` ไม่ยิงอะไรเลยจนกว่าจะมี `.subscribe()`** — เขียนแล้วลืม subscribe คือกับดักอันดับหนึ่งของคนใหม่ (ดูข้อ 5)

---

## 3. จัดการ error

```typescript
import { catchError, of } from 'rxjs';

getAll(): Observable<Product[]> {
  return this.http.get<Product[]>('/api/products').pipe(
    catchError((err: HttpErrorResponse) => {
      console.error('โหลดสินค้าไม่สำเร็จ', err.status, err.message);
      return of([]);   // ให้ค่า fallback แทนที่จะปล่อย error หลุดไปหา subscriber
    }),
  );
}
```

`HttpErrorResponse` มี `status` (HTTP status code), `error` (response body ที่ server ส่งมา), `message` — เช็ค `err.status` เพื่อแยกกรณี (401 ต้อง login ใหม่, 404 ไม่เจอ, 500 server พัง) ได้ตรง ๆ

---

## 4. Interceptor — แทรก logic กลางทางทุก request

```typescript
// auth.interceptor.ts — functional interceptor (แนวทางปัจจุบัน)
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).token();
  const cloned = token ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }) : req;
  return next(cloned);
};
```

```typescript
// app.config.ts
provideHttpClient(withInterceptors([authInterceptor]));
```

**Functional interceptor คือแนวทางปัจจุบัน** — เป็นฟังก์ชันธรรมดา ไม่ใช่ class ที่ implement interface แบบเก่า (`HttpInterceptor`) ทดสอบง่ายกว่า, ใช้ `inject()` ได้ตรง ๆ, tree-shakable — เหมาะกับ: แนบ token อัตโนมัติทุก request, log, retry, แปลง error ให้เป็นรูปแบบเดียวกันทั้งแอป

---

## 5. กับดัก

- **เขียน `this.http.get(...)` แล้วไม่ `.subscribe()`** — ไม่มีอะไรเกิดขึ้นเลย ไม่มี error ให้เห็นด้วยซ้ำ (cold observable, ดูเหตุผลเต็ม ๆ ที่ [[Observable]])
- **`inject()` ถูกเรียกนอก injection context** — เช่นเรียกใน `setTimeout()` หรือ callback ที่ async จะ error ทันที ต้อง inject ตอน field initializer/constructor เท่านั้น (ดู [[Angular Services and DI]])
- **subscribe ซ้อน subscribe** — ยิง request ใน `.subscribe()` ของอีก request หนึ่ง ควรใช้ `switchMap`/`mergeMap` ต่อ pipeline แทน (ดู [[RxJS]])
- **ลืม unsubscribe stream ที่ไม่จบเอง** — HTTP request ปกติ complete เองไม่ต้อง unsubscribe แต่ถ้า wrap ด้วย `interval()`/polling ต้องจัดการเอง (ดู [[Observable]] ข้อ 8)
- **ไม่ตั้ง type ให้ `http.get<T>()`** — ได้ `any` กลับมา เสีย type safety ทั้งที่ Angular รองรับ generic เต็มรูป

---

## 6. Cheat sheet

```typescript
provideHttpClient(withInterceptors([myInterceptor]))

private http = inject(HttpClient);
http.get<T>(url)
http.post<T>(url, body)
http.put<T>(url, body)
http.delete<T>(url)

.pipe(catchError((err: HttpErrorResponse) => of(fallback)))
```

| อาการ | สาเหตุ |
|---|---|
| เรียก API แล้วไม่มีอะไรเกิดขึ้นเลย | ไม่ได้ `.subscribe()` |
| `NG0203: inject() must be called from an injection context` | เรียก `inject()` นอก field initializer/constructor |
| ได้ `any` แทนที่จะเป็น type ที่ต้องการ | ไม่ได้ใส่ `<T>` ให้ `http.get<T>()` |
| token ไม่ถูกแนบไปกับบาง request | interceptor logic ผิดเงื่อนไข หรือไม่ได้ลงทะเบียนใน `provideHttpClient` |

## 🔗 เกี่ยวข้อง

- [[Observable]] — cold observable, ทำไมต้อง subscribe ถึงจะยิง request จริง
- [[RxJS]] — operator ที่ใช้บ่อยกับ HTTP (`switchMap`, `catchError`, `retry`)
- [[Angular Services and DI]] — `inject()` และ injection context

## 📖 อ่านต่อ

- [Angular — Setting up HttpClient](https://angular.dev/guide/http/setup)
- [Angular — provideHttpClient](https://angular.dev/api/common/http/provideHttpClient)
