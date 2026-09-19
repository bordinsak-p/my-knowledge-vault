---
tags:
  - angular
  - typescript
  - pipe
type: reference
created: 2026-09-16
---

# 🧪 Angular Custom Pipe

> **Pipe แปลง "ค่าที่แสดงในหน้าจอ" โดยไม่แตะ "ค่าจริงในโค้ด"** — `{{ price | currency }}` แสดงเป็น `฿1,000.00` แต่ตัวแปร `price` ในโค้ดยังเป็นเลข `1000` เหมือนเดิมทุกอย่าง ใช้เมื่อ logic การแปลงเป็นเรื่อง "แสดงผล" ล้วน ๆ ไม่ใช่ business logic

---

## 0. สร้างด้วย Angular CLI

```bash
ng generate pipe truncate
ng g p truncate              # short alias
```

ได้ไฟล์ `truncate.pipe.ts` พร้อมโครง `PipeTransform` แบบหัวข้อ 1 ด้านล่างทันที

| flag | default | ผลลัพธ์ |
|---|---|---|
| `--standalone` | `true` | pipe แบบ standalone (ค่า default อยู่แล้ว ไม่ต้องใส่เอง) |
| `--skip-tests` | `false` | ไม่สร้างไฟล์ `.spec.ts` |
| `--flat` | `true` | ไม่สร้างโฟลเดอร์ย่อยใหม่ วางไฟล์ไว้ระดับบนสุด |

---

## 1. สร้าง pipe เอง

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'truncate',
  standalone: true,   // แนวทางปัจจุบัน — pipe ไม่ผูกกับ NgModule
})
export class TruncatePipe implements PipeTransform {
  transform(value: string, maxLength: number = 50): string {
    if (!value || value.length <= maxLength) return value;
    return value.slice(0, maxLength) + '...';
  }
}
```

```html
<p>{{ longText | truncate }}</p>
<p>{{ longText | truncate:20 }}</p>   <!-- ส่ง parameter เพิ่มด้วย : -->
```

**`transform()` คือ method เดียวที่ต้อง implement** — parameter ตัวแรกคือค่าที่ pipe รับมา (ด้านซ้ายของ `|`) ตัวถัด ๆ ไปคือ argument ที่ส่งต่อท้ายด้วย `:`

---

## 2. Pure vs Impure — ผลต่อ performance ต่างกันมาก

```typescript
@Pipe({ name: 'myPipe', standalone: true })              // pure = true (default)
@Pipe({ name: 'myPipe', standalone: true, pure: false })  // impure
```

| | Pure (default) | Impure |
|---|---|---|
| รันเมื่อไหร่ | เฉพาะตอน **reference** ของ input เปลี่ยน | **ทุกรอบ** change detection (ถี่มาก) |
| เห็นการแก้ array/object โดย mutate ตรง ๆ ไหม | ❌ ไม่เห็น (reference เดิม) | ✅ เห็น (รันใหม่ทุกครั้งอยู่แล้ว) |
| performance | ดี | แพง ถ้าใช้ผิดที่กระทบทั้งแอป |

```typescript
// ตัวอย่าง pure pipe ไม่เห็นการเปลี่ยนถ้า mutate array ตรง ๆ
items.push(newItem);        // ❌ pure pipe ไม่รันใหม่ — reference ของ items ไม่เปลี่ยน
items = [...items, newItem]; // ✅ pure pipe รันใหม่ — ได้ array ใหม่ reference ใหม่
```

**เกือบทุกกรณีควรใช้ pure (ค่า default)** — ถ้าจำเป็นต้อง impure จริง ๆ (เช่น filter ตาม state ที่เปลี่ยนบ่อยโดยไม่เปลี่ยน reference) ให้พิจารณา `computed()` (ดู [[Angular Signal Primitives]]) แทนก่อนเสมอ เพราะ signal-based reactivity แม่นและเร็วกว่า impure pipe มาก

---

## 3. Chaining — ต่อ pipe หลายตัว

```html
{{ product.name | truncate:30 | uppercase }}
```

ทำงานจากซ้ายไปขวา: `truncate` ก่อน แล้วเอาผลลัพธ์ไปเข้า `uppercase` ต่อ — เหมือน pipeline เดียวกับแนวคิด `.pipe()` ของ [[RxJS]]

---

## 4. กับดัก

- **ทำ pipe แบบ impure โดยไม่จำเป็น** — กระทบ performance ทั้งแอปเพราะรันทุกรอบ change detection ให้ลองใช้ `computed()` ก่อนเสมอถ้าทำได้
- **mutate array/object ตรง ๆ แล้วคาดหวังว่า pure pipe จะเห็น** — ไม่เห็น เพราะ pure pipe เช็คแค่ reference เปลี่ยนไหม (ข้อ 2)
- **ใส่ side effect ใน `transform()`** — เรียก API, เขียน log เยอะ ๆ, แก้ state ข้างนอก — `transform()` ควรเป็นฟังก์ชัน pure ไม่ต่างจากหลักการเดียวกับ `computed()` ที่เขียนไว้ใน [[Angular Signal Primitives]]
- **ลืม `standalone: true`** — pipe ใช้กับ standalone component ไม่ได้ถ้าไม่ได้ mark standalone (หรือไม่ได้ import เข้า `imports: []` ของ component ที่ใช้)
- **เขียน pipe ทำสิ่งที่ควรเป็น business logic** — pipe ควรมีไว้ "แสดงผล" เท่านั้น ถ้า logic นั้นส่งผลต่อการตัดสินใจอื่นในแอปด้วย ควรอยู่ใน service ไม่ใช่ pipe

---

## 5. Cheat sheet

```typescript
@Pipe({ name: 'x', standalone: true })       // pure (default)
@Pipe({ name: 'x', standalone: true, pure: false })  // impure

export class XPipe implements PipeTransform {
  transform(value: T, ...args: A[]): R { ... }
}
```

```html
{{ value | x }}
{{ value | x:arg1:arg2 }}
{{ value | x | y }}   <!-- chain -->
```

| อาการ | สาเหตุ |
|---|---|
| แก้ array แล้ว pipe ไม่อัปเดต | pure pipe + mutate ตรง ๆ ไม่เปลี่ยน reference |
| แอปช้าลงหลังเพิ่ม pipe ตัวหนึ่ง | ตั้ง `pure: false` โดยไม่จำเป็น |
| pipe ใช้ไม่ได้ใน standalone component | ลืม `standalone: true` หรือไม่ได้ import |

## 🔗 เกี่ยวข้อง

- [[Angular CLI]] — คำสั่ง `ng generate` อื่น ๆ ทั้งหมด
- [[Angular Signal Primitives]] — `computed()` มักเป็นทางเลือกที่ดีกว่า impure pipe
- [[RxJS]] — แนวคิด pipeline/chain แบบเดียวกัน

## 📖 อ่านต่อ

- [Angular — Custom pipes](https://angular.dev/guide/templates/pipes)
