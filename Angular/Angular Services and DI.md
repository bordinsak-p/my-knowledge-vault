---
tags:
  - angular
  - typescript
  - dependency-injection
type: reference
created: 2026-09-16
---

# 💉 Angular Services & Dependency Injection

> **Service คือ class ธรรมดาที่มีหน้าที่เดียว: เก็บ logic/state ที่ไม่ใช่ของ component ใดโดยเฉพาะ** — Angular เป็นคนสร้าง instance ให้เอง แล้ว "ฉีด" (inject) เข้าไปให้ทุกที่ที่ขอใช้ ไม่ต้อง `new` เอง

---

## 1. สร้าง service — `@Injectable`

```typescript
@Injectable({ providedIn: 'root' })
export class CartService {
  private items = signal<Item[]>([]);
  readonly cartItems = this.items.asReadonly();

  add(item: Item) {
    this.items.update(current => [...current, item]);
  }
}
```

**`providedIn: 'root'` แปลว่า:** สร้าง instance เดียวใช้ร่วมกันทั้งแอป (singleton) และ **tree-shakable** — ถ้าไม่มีใคร inject service นี้เลย มันจะไม่ถูกรวมเข้า bundle ตอน build ด้วยซ้ำ (ประหยัดขนาดไฟล์อัตโนมัติ)

---

## 2. เอาไปใช้ — `inject()` vs constructor

```typescript
// ✅ แนวทางปัจจุบัน — inject() function
export class CartComponent {
  private cartService = inject(CartService);
  items = this.cartService.cartItems;
}

// เดิม — constructor injection (ยังใช้ได้ปกติ ไม่ deprecated)
export class CartComponent {
  constructor(private cartService: CartService) {}
}
```

**`inject()` ทำงานเหมือนกับ constructor injection ทุกอย่าง** แค่เขียนเป็น field initializer แทน — ข้อดีคือใช้ได้ในที่ที่ constructor เขียนไม่สะดวก (เช่นใน `provideRouter` guard function ที่ไม่มี class/constructor ให้)

### ⚠️ `inject()` ต้องเรียกใน "injection context" เท่านั้น

```typescript
// ✅ เรียกตอน field initializer
private service = inject(MyService);

// ✅ เรียกใน constructor
constructor() {
  const service = inject(MyService);
}

// ❌ เรียกใน callback/async — error ทันที
ngOnInit() {
  setTimeout(() => {
    const service = inject(MyService);   // NG0203
  });
}
```

**แก้:** ดึง service ออกมาเป็นตัวแปรตอนอยู่ใน injection context ก่อนเสมอ แล้วค่อยใช้ตัวแปรนั้นใน callback

```typescript
private service = inject(MyService);   // ✅ inject ตรงนี้

ngOnInit() {
  setTimeout(() => {
    this.service.doSomething();        // ✅ ใช้ตัวแปรที่ inject ไว้แล้ว
  });
}
```

---

## 3. ขอบเขตของ service — ไม่ได้มีแค่ `root`

| provide ที่ไหน | ได้ instance แบบไหน |
|---|---|
| `@Injectable({ providedIn: 'root' })` | ตัวเดียวทั้งแอป (มาตรฐานที่ใช้บ่อยสุด) |
| ใส่ใน `providers` ของ component เอง | instance ใหม่แยกต่างหาก เฉพาะ component นั้น (และ child ของมัน) |

```typescript
@Component({
  providers: [CartService],   // instance แยก เฉพาะ component นี้กับลูกของมัน
  ...
})
export class CheckoutComponent { }
```

**ใช้ component-level provider เมื่อ:** อยากให้แต่ละ instance ของ component มี state แยกกันจริง ๆ ไม่ใช่ใช้ร่วมกันทั้งแอป เช่น wizard/form ที่เปิดหลายอันพร้อมกันแล้วไม่อยากให้ state ปนกัน

---

## 4. กับดัก

- **`inject()` นอก injection context** — `NG0203` ทันที (ข้อ 2) แก้ด้วยการดึงเป็นตัวแปรไว้ก่อนเข้า callback
- **ลืม `providedIn: 'root'` แล้วไม่ได้ provide ที่ไหนเลย** — `NullInjectorError: No provider for X` ตอน runtime
- **provide service ซ้ำทั้งระดับ root และ component** โดยไม่ตั้งใจ — ได้ instance คนละตัวโดยไม่รู้ตัว state ที่ควรใช้ร่วมกันกลับแยกกัน แก้ที่หนึ่งไม่กระทบอีกที่
- **ยัด business logic ไว้ใน component แทนที่จะแยกเป็น service** — เทสยาก, ใช้ซ้ำไม่ได้, component ยาวเทอะทะ (แพตเทิร์นเดียวกับที่ [[Quarkus Project Structure]] อธิบายไว้ฝั่ง backend — service คือที่เก็บ logic ไม่ใช่ resource/component)

---

## 5. Cheat sheet

```typescript
@Injectable({ providedIn: 'root' })
export class MyService { }

// ใช้
private service = inject(MyService);

// scope เฉพาะ component
@Component({ providers: [MyService] })
```

| อาการ | สาเหตุ |
|---|---|
| `NG0203: inject() must be called from an injection context` | เรียก `inject()` ใน callback/async |
| `NullInjectorError: No provider for X` | ไม่มี `providedIn` และไม่ได้ provide ที่ไหนเลย |
| state ที่ควรใช้ร่วมกันกลับไม่ sync กัน | provide service ซ้ำหลายระดับโดยไม่ตั้งใจ |

## 🔗 เกี่ยวข้อง

- [[Angular HttpClient]] — service ส่วนใหญ่ห่อ HttpClient ไว้ข้างใน
- [[Quarkus Project Structure]] — แนวคิด service layer แบบเดียวกันฝั่ง backend

## 📖 อ่านต่อ

- [Angular — Dependency injection overview](https://angular.dev/guide/di)
- [Angular — Creating and using services](https://angular.dev/guide/di/creating-and-using-services)
