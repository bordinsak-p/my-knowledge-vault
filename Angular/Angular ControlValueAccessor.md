---
tags:
  - angular
  - typescript
  - forms
type: reference
created: 2026-09-16
---

# 🎛️ ControlValueAccessor — ทำ component ของตัวเองให้ใช้กับ `formControlName` ได้

> **ปัญหาที่มันแก้:** `formControlName`/`[(ngModel)]` (ดู [[Angular Reactive Forms]]) ใช้ได้เองกับ `<input>`/`<select>` เพราะ Angular มี `ControlValueAccessor` ให้ในตัวสำหรับ element พวกนี้อยู่แล้ว — **component ที่เขียนเอง** (เช่น star-rating, toggle switch, custom dropdown) **ไม่มีของนี้ให้โดยอัตโนมัติ** ต้อง implement เองถึงจะเอาไปใช้กับ `formControlName` ได้เหมือน input จริง

---

## 1. Interface ที่ต้อง implement — 4 method

```typescript
export interface ControlValueAccessor {
  writeValue(value: any): void;
  registerOnChange(fn: (value: any) => void): void;
  registerOnTouched(fn: () => void): void;
  setDisabledState?(isDisabled: boolean): void;
}
```

| method | Angular เรียกตอนไหน | หน้าที่ |
|---|---|---|
| `writeValue` | form set ค่าเข้ามา (เช่น `patchValue`, ค่าเริ่มต้น) | อัปเดตหน้าตา component ให้ตรงกับค่านั้น |
| `registerOnChange` | ตอน component ถูกผูกกับ form ครั้งแรก | เก็บ callback ไว้ **เรียกเองทุกครั้งที่ผู้ใช้แก้ค่า** เพื่อบอก form ว่าเปลี่ยนแล้ว |
| `registerOnTouched` | เหมือนกัน | เก็บ callback ไว้เรียกตอนผู้ใช้ "แตะ" component (เช่น blur) — ใช้กับ `touched` state (ดู [[Angular Reactive Forms]]) |
| `setDisabledState` | form control ถูก enable/disable | อัปเดตหน้าตาให้สะท้อนสถานะ disabled |

---

## 2. ตัวอย่างเต็ม — Star Rating component

```typescript
import { Component, forwardRef, signal } from '@angular/core';
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from '@angular/forms';

@Component({
  selector: 'app-star-rating',
  standalone: true,
  template: `
    @for (star of [1,2,3,4,5]; track star) {
      <span (click)="select(star)" [class.filled]="star <= value()">★</span>
    }
  `,
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => StarRatingComponent),
      multi: true,   // ⚠️ ต้องมีเสมอ — บอกว่ามี value accessor หลายตัวได้ ไม่ใช่แทนที่ตัวอื่น
    },
  ],
})
export class StarRatingComponent implements ControlValueAccessor {
  value = signal(0);
  disabled = signal(false);

  private onChange: (value: number) => void = () => {};
  private onTouched: () => void = () => {};

  select(star: number) {
    if (this.disabled()) return;
    this.value.set(star);
    this.onChange(star);   // ✅ ต้องเรียกทุกครั้งที่ค่าเปลี่ยนจากผู้ใช้ ไม่งั้น form ไม่รู้ว่าเปลี่ยน
    this.onTouched();
  }

  writeValue(value: number): void {
    this.value.set(value ?? 0);
  }

  registerOnChange(fn: (value: number) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled.set(isDisabled);
  }
}
```

```html
<!-- ใช้เหมือน <input> ทั่วไปเลย -->
<app-star-rating formControlName="rating"></app-star-rating>
<app-star-rating [(ngModel)]="myRating"></app-star-rating>
```

---

## 3. `forwardRef` ทำไมต้องมี

```typescript
useExisting: forwardRef(() => StarRatingComponent),
```

**ตอน `providers: []` array กำลังถูกประเมิน คลาส `StarRatingComponent` ยังไม่ถูกประกาศเสร็จ** (มันอยู่ใน decorator ของคลาสตัวเองที่ยังนิยามไม่จบ) — `forwardRef(() => X)` ห่อการอ้างอิงไว้เป็นฟังก์ชัน เลื่อนการ resolve ชื่อ `X` ออกไปจนกว่าจะถูกใช้งานจริง แก้ปัญหา "อ้างถึงคลาสตัวเองก่อนมันถูกประกาศเสร็จ" ได้

---

## 4. กับดัก

- **ลืมเรียก `this.onChange(value)` ตอนผู้ใช้แก้ค่า** — form ไม่รู้เลยว่าค่าเปลี่ยน `formControlName` ที่ผูกไว้จะค้างค่าเดิมตลอด ทั้งที่หน้าจอเปลี่ยนไปแล้ว
- **ลืม `multi: true`** — ทำให้ Angular คิดว่านี่คือ value accessor ตัวเดียวของทั้งแอป ไปแทนที่ตัวอื่นที่มีอยู่แล้วโดยไม่ตั้งใจ (เช่น ของ `<input>` มาตรฐาน)
- **ลืม `forwardRef`** — error ตอน runtime เพราะอ้างถึงคลาสก่อนมันถูกประกาศเสร็จ (ข้อ 3)
- **`writeValue` ไปเรียก `onChange` ต่อ** — ทำให้เกิด loop เพราะ `writeValue` คือทาง "form → component" ส่วน `onChange` คือทาง "component → form" สองทิศทางนี้ต้องแยกจากกันชัดเจน
- **ไม่ implement `setDisabledState`** — component ยังดูเหมือนกดได้ทั้งที่ form ปิดใช้งาน (disabled) ไปแล้ว UX สับสน
- **เขียน CVA ทั้งที่ component แค่ wrap `<input>` ธรรมดา** — ถ้าแค่ต้องการ style/behavior เพิ่มเติมรอบ `<input>` ปกติ อาจไม่ต้องทำ CVA เต็มรูป ใช้ content projection ห่อ `<input>` จริงไว้ข้างในง่ายกว่า — CVA คุ้มค่าที่สุดตอน component มีค่าที่ "ไม่ใช่ text ธรรมดา" จริง ๆ (เช่น rating, toggle, date picker ที่ทำเอง)

---

## 5. Cheat sheet

```typescript
providers: [{
  provide: NG_VALUE_ACCESSOR,
  useExisting: forwardRef(() => MyComponent),
  multi: true,
}]

export class MyComponent implements ControlValueAccessor {
  private onChange = (v: T) => {};
  private onTouched = () => {};

  writeValue(v: T) { /* form → component */ }
  registerOnChange(fn: (v: T) => void) { this.onChange = fn; }
  registerOnTouched(fn: () => void) { this.onTouched = fn; }
  setDisabledState(disabled: boolean) { /* อัปเดตหน้าตา */ }

  onUserInteraction(v: T) {
    // component → form
    this.onChange(v);
    this.onTouched();
  }
}
```

| อาการ | สาเหตุ |
|---|---|
| `formControlName` ค้างค่าเดิม ไม่อัปเดตตามที่ผู้ใช้ทำ | ลืมเรียก `onChange`/`onTouched` |
| ทับ value accessor ของ input อื่นในแอป | ลืม `multi: true` |
| error ตอน runtime ตอนประกาศ providers | ลืม `forwardRef` |
| component ยังกดได้ทั้งที่ form disabled | ไม่ได้ implement `setDisabledState` |

## 🔗 เกี่ยวข้อง

- [[Angular Reactive Forms]] — `FormControl`/`Validators` ที่ CVA เข้าไปเชื่อมด้วย
- [[Angular Component Communication]] — `@Input`/`@Output` พื้นฐานที่ CVA สร้างต่อยอดขึ้นไป

## 📖 อ่านต่อ

- [Angular — Implement ControlValueAccessor for form controls](https://angular.dev/api/forms/ControlValueAccessor)
