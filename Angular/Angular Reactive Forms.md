---
tags:
  - angular
  - typescript
  - forms
type: reference
created: 2026-09-16
---

# 📝 Angular Reactive Forms

> [[Two-Way Binding คืออะไร|`[(ngModel)]`]] ที่เขียนไว้แล้วคือ **template-driven form** — form ทั้งหมดถูกกำหนดใน HTML **โน้ตนี้คือ reactive form** — form ทั้งหมดถูกกำหนดใน**โค้ด TypeScript** แล้ว template แค่ผูกเข้ากับมัน — ควบคุมได้ละเอียดกว่า, เทสง่ายกว่า, เหมาะกับ form ที่ซับซ้อน

---

## 1. Template-driven vs Reactive — เลือกยังไง

| | Template-driven (`ngModel`) | Reactive |
|---|---|---|
| form structure อยู่ที่ไหน | ใน HTML | ใน TypeScript class |
| validation | directive ใน HTML (`required`) | `Validators` ใน code |
| เทส unit test ได้ไหม | ยาก (ต้อง render DOM) | ง่าย (เทส FormGroup ตรง ๆ ไม่ต้อง render) |
| เหมาะกับ | form ง่าย ๆ 2-3 ช่อง | form ที่ซับซ้อน, validation แบบมีเงื่อนไข, dynamic field |

---

## 2. FormControl / FormGroup / FormBuilder

```typescript
import { FormBuilder, Validators } from '@angular/forms';

export class RegisterFormComponent {
  private fb = inject(FormBuilder);

  form = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(8)]],
    age: [null as number | null, [Validators.min(18)]],
  });

  submit() {
    if (this.form.invalid) return;
    console.log(this.form.value);   // { email, password, age }
  }
}
```

```html
<form [formGroup]="form" (ngSubmit)="submit()">
  <input formControlName="email">
  <input formControlName="password" type="password">
  <button [disabled]="form.invalid">สมัคร</button>
</form>
```

**`FormBuilder.group()` คือทางลัด** — เขียนเองแบบเต็มได้เหมือนกัน (`new FormGroup({ email: new FormControl('', ...) })`) แต่ `fb.group()` สั้นกว่ามาก

---

## 3. แสดง error ให้ผู้ใช้เห็น

```html
<input formControlName="email">
@if (form.get('email')?.invalid && form.get('email')?.touched) {
  <p class="error">
    @if (form.get('email')?.hasError('required')) { กรุณากรอกอีเมล }
    @if (form.get('email')?.hasError('email')) { รูปแบบอีเมลไม่ถูกต้อง }
  </p>
}
```

**เช็ค `touched` ควบคู่กับ `invalid` เสมอ** — ไม่งั้น error message โผล่ตั้งแต่ยังไม่ได้แตะ field เลย (UX แย่มาก ผู้ใช้ยังไม่ทันพิมพ์ก็เห็น error เต็มหน้าจอ)

---

## 4. Custom validator

```typescript
function passwordMatchValidator(group: AbstractControl): ValidationErrors | null {
  const pass = group.get('password')?.value;
  const confirm = group.get('confirm')?.value;
  return pass === confirm ? null : { mismatch: true };
}

form = this.fb.group({
  password: [''],
  confirm: [''],
}, { validators: passwordMatchValidator });   // validator ระดับ group ไม่ใช่ field เดียว
```

**validator คืนค่า `null` = ผ่าน, คืน object = ไม่ผ่าน (key ในนั้นเอาไว้เช็คด้วย `hasError()`)** — validator ที่ต้องเทียบค่าระหว่างหลาย field (เช่นยืนยันรหัสผ่าน) ต้องใส่ที่ระดับ `FormGroup` ไม่ใช่ field เดียว เพราะต้องเข้าถึงทั้งสอง control พร้อมกัน

---

## 5. FormArray — จำนวน field ไม่คงที่

```typescript
form = this.fb.group({
  tags: this.fb.array([this.fb.control('')]),
});

get tags() { return this.form.get('tags') as FormArray; }

addTag() { this.tags.push(this.fb.control('')); }
removeTag(i: number) { this.tags.removeAt(i); }
```

```html
@for (tag of tags.controls; track $index) {
  <input [formControlName]="$index">
}
<button (click)="addTag()">+ เพิ่ม tag</button>
```

ใช้เมื่อจำนวนช่อง input ไม่รู้ล่วงหน้า (รายการแท็ก, รายการที่อยู่หลายที่)

---

## 6. กับดัก

- **ลืม `[formGroup]` บน `<form>`** — `formControlName` ข้างในหาไม่เจอว่าอยู่ group ไหน error ตอน runtime
- **เช็ค `.invalid` โดยไม่เช็ค `.touched`** — error message โผล่ก่อนผู้ใช้แตะ field เลย (ข้อ 3)
- **ผสม `ngModel` กับ reactive form ใน form เดียวกัน** — สอง mental model ชนกัน ทำงานไม่ตรงที่คาดไว้ เลือกแบบใดแบบหนึ่งให้ชัดต่อ form
- **custom validator ระดับ field ที่ต้องเทียบกับ field อื่น** — ต้องใส่ที่ `FormGroup` ไม่ใช่ field เดียว (ข้อ 4) เพราะ field validator เข้าถึง field อื่นในกลุ่มเดียวกันไม่ได้
- **แปลง `form.value` ส่งตรงไป backend โดยไม่กรอง** — ถ้า form มี field ที่ `disabled` มันจะไม่ติดมาใน `form.value` แต่ติดมาใน `form.getRawValue()` — สับสนกันบ่อยว่าทำไมข้อมูลหาย

---

## 7. Cheat sheet

```typescript
form = this.fb.group({
  field: ['', [Validators.required, Validators.email]],
});

form.valid / form.invalid
form.value            // ค่าปัจจุบัน (ไม่รวม disabled field)
form.getRawValue()     // ค่าปัจจุบัน (รวม disabled field ด้วย)
form.get('field')?.hasError('required')
form.reset()
form.patchValue({ field: 'x' })   // แก้บาง field
```

| อาการ | สาเหตุ |
|---|---|
| `formControlName` error ตอน runtime | ลืม `[formGroup]` บน `<form>` |
| error message โผล่ก่อนพิมพ์อะไรเลย | ไม่ได้เช็ค `.touched` ควบคู่ `.invalid` |
| field ที่ disabled หายไปจาก payload ที่ส่ง backend | ใช้ `form.value` แทน `form.getRawValue()` |
| validator เทียบสอง field ไม่ทำงาน | ใส่ validator ที่ field แทนที่จะใส่ที่ `FormGroup` |

## 🔗 เกี่ยวข้อง

- [[Two-Way Binding คืออะไร]] — template-driven form (`ngModel`) ทางเลือกที่ง่ายกว่าสำหรับ form สั้น ๆ
- [[Angular HttpClient]] — ส่ง `form.value`/`getRawValue()` เข้า API ตอน submit

## 📖 อ่านต่อ

- [Angular — Reactive forms](https://angular.dev/guide/forms/reactive-forms)
- [Angular — Validate form input](https://angular.dev/guide/forms/form-validation)
