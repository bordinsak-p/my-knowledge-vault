---
tags:
  - angular
  - typescript
  - component
type: reference
created: 2026-09-16
---

# 🔗 Angular Component Communication

> **กฎทิศทางเดียวที่จำง่าย: ข้อมูลไหลลง (parent → child) ผ่าน `Input`, เหตุการณ์ไหลขึ้น (child → parent) ผ่าน `Output`** — [[Two-Way Binding คืออะไร]] ที่เขียนไว้แล้วคือการรวมสองทิศทางนี้เข้าด้วยกัน

---

## 1. `@Input`/`@Output` แบบ decorator (ดั้งเดิม ยังใช้ได้ปกติ)

```typescript
// child
export class ProductCardComponent {
  @Input() product!: Product;
  @Output() addToCart = new EventEmitter<Product>();

  onAdd() {
    this.addToCart.emit(this.product);
  }
}
```

```html
<!-- parent -->
<app-product-card [product]="p" (addToCart)="onAddToCart($event)"></app-product-card>
```

---

## 2. `input()`/`output()` แบบ signal (แนวทางปัจจุบัน)

```typescript
export class ProductCardComponent {
  product = input.required<Product>();          // อ่านค่า: this.product()
  addToCart = output<Product>();                 // ส่ง: this.addToCart.emit(...)

  onAdd() {
    this.addToCart.emit(this.product());
  }
}
```

| | `@Input()`/`@Output()` | `input()`/`output()` |
|---|---|---|
| อ่านค่า | ตรง ๆ `this.product` | เรียกเป็นฟังก์ชัน `this.product()` (เป็น signal) |
| reactive กับ `computed()`/`effect()` | ต้องแปลงเอง | ใช้ต่อกับ `computed()` ได้ทันที |
| ค่าเริ่มต้น/บังคับใส่ | `@Input() x?: T` | `input<T>()` (optional) / `input.required<T>()` (บังคับ) |
| ยัง deprecated ไหม | ❌ ไม่ ใช้ได้ปกติ | เป็นทางเลือกใหม่ ไม่ได้บังคับเปลี่ยน |

**`input()` ให้ signal กลับมา** — เอาไปต่อ `computed()` ได้ตรง ๆ โดยไม่ต้องแปลงผ่าน `toSignal()` เหมือนตอนใช้ `@Input()` แบบเดิม

---

## 3. `@ViewChild`/`viewChild()` — parent เข้าถึง child โดยตรง

```typescript
// แบบ decorator
@ViewChild(ChildComponent) child!: ChildComponent;

ngAfterViewInit() {
  this.child.someMethod();   // ต้องรอ ngAfterViewInit ถึงจะมีค่า
}

// แบบ signal
child = viewChild(ChildComponent);   // Signal<ChildComponent | undefined>

ngAfterViewInit() {
  this.child()?.someMethod();
}
```

**ใช้ตอนที่ event/binding ปกติไม่พอ** — ต้องการเรียก method ของ child ตรง ๆ จาก parent (เช่นสั่ง reset form, focus input) — ส่วนใหญ่ favour `@Input`/`@Output` ก่อนเสมอ เพราะทำให้ทิศทางข้อมูลชัดเจนกว่า `ViewChild` ที่ทำให้ parent ผูกกับ implementation ของ child มากเกินไป

---

## 4. Content projection — parent ส่ง template เข้าไปให้ child render

```typescript
// child: card.component.html
<div class="card">
  <ng-content></ng-content>   <!-- เนื้อหาจาก parent มาแสดงตรงนี้ -->
</div>
```

```html
<!-- parent -->
<app-card>
  <h2>หัวข้อที่กำหนดเอง</h2>
  <p>เนื้อหาอะไรก็ได้จาก parent</p>
</app-card>
```

ต่างจาก `@Input`: `@Input` ส่ง**ข้อมูล** เข้าไปให้ child ประมวลผล ส่วน content projection ส่ง**template ทั้งก้อน** ให้ child แค่วางไว้ในตำแหน่งที่กำหนด — ใช้ทำ component แบบ "กรอบ" ที่ไม่รู้ล่วงหน้าว่าข้างในจะมีอะไร (card, modal, layout wrapper)

---

## 5. กับดัก

- **ตั้งชื่อ `@Output` ขึ้นต้นด้วย `on`** — เช่น `@Output() onSave` ผิด convention เพราะ Angular attribute ที่ขึ้นต้น `on` สงวนไว้สำหรับ native DOM event ทำให้อ่านสับสน ตั้งชื่อ `save` เฉย ๆ พอ (event binding จะเขียน `(save)="..."`)
- **แก้ไข object/array ที่รับมาจาก `@Input` ตรง ๆ** — เพราะเป็น reference เดียวกับที่ parent ถืออยู่ แก้ใน child กระทบ parent โดยไม่ตั้งใจ (และ change detection บางโหมดอาจไม่รู้ตัวด้วยซ้ำว่าค่าเปลี่ยน) ควร copy ก่อนแก้เสมอถ้าไม่ได้ตั้งใจสื่อสารกลับ
- **เข้าถึง `@ViewChild`/`viewChild()` ก่อน `ngAfterViewInit`** — ได้ `undefined` เพราะ DOM ยังไม่ render (ดู [[Angular Lifecycle Hooks]])
- **ใช้ `ViewChild` แทนที่จะใช้ `@Input`/`@Output` ทั้งที่ควรสื่อสารผ่าน data binding** — ทำให้ parent ผูกแน่นกับ implementation ภายในของ child มากเกินไป แก้ child ทีหลังกระทบ parent ง่าย
- **ลืมว่า `input.required()` ที่ไม่ได้ใส่ค่าจาก parent จะ throw ตอน runtime** — ต้องมั่นใจว่า parent ส่งมาจริงเสมอ ถ้าอยากให้ optional ใช้ `input<T>()` (ไม่ใส่ `.required`) แทน

---

## 6. Cheat sheet

```typescript
// decorator style
@Input() x!: T;
@Output() y = new EventEmitter<T>();
@ViewChild(Comp) ref!: Comp;

// signal style (ปัจจุบัน)
x = input.required<T>();     // หรือ input<T>() ถ้า optional
y = output<T>();
ref = viewChild(Comp);       // Signal<Comp | undefined>
```

```html
<app-child [x]="value" (y)="onY($event)"></app-child>

<app-wrapper>
  <ng-content ที่ template ของ wrapper ถูกวางเนื้อหานี้เข้าไป></ng-content>
</app-wrapper>
```

| อาการ | สาเหตุ |
|---|---|
| แก้ค่าใน child แล้ว parent เปลี่ยนตามโดยไม่ตั้งใจ | แก้ object/array ที่เป็น reference เดียวกับ `@Input` ตรง ๆ |
| `@ViewChild` เป็น `undefined` | เข้าถึงก่อน `ngAfterViewInit` |
| `input.required()` throw ตอน runtime | parent ไม่ได้ส่งค่าเข้ามาจริง |
| event ของ custom component ดู confusing ปนกับ native DOM event | ตั้งชื่อ `@Output` ขึ้นต้นด้วย `on` |

## 🔗 เกี่ยวข้อง

- [[Two-Way Binding คืออะไร]] — รวม Input+Output เข้าด้วยกันเป็น `[(x)]`
- [[Angular Lifecycle Hooks]] — จังหวะที่ `@Input`/`@ViewChild` พร้อมใช้งานจริง
- [[Template Reference Variables]] — อีกวิธีเข้าถึง child จาก template โดยตรง (ไม่ผ่าน class)

## 📖 อ่านต่อ

- [Angular — Sharing data between child and parent](https://angular.dev/guide/components/inputs)
- [Angular Signal Components: input, output](https://blog.angular-university.io/angular-signal-components/)
