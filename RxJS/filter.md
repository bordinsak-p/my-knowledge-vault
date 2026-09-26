---
tags:
  - rxjs
  - operator
type: reference
created: 2026-09-19
---

# 🚦 filter — กรองก่อนทำงานต่อ

> `filter(predicate)` ปล่อยเฉพาะค่าที่ผ่านเงื่อนไข (predicate คืน `true`) ให้ไหลต่อไปใน pipeline — ค่าที่ไม่ผ่านจะถูกตัดทิ้งไปเลย ไม่ถึง operator ถัดไป

---

## ตัวอย่าง — UI action: กันกดปุ่มซ้ำระหว่างกำลังส่งฟอร์ม

```ts
buttonClicks$.pipe(
  filter(() => !this.isSubmitting),
).subscribe(() => this.submit());
```

ถ้า `isSubmitting` เป็น `true` อยู่ (ฟอร์มกำลังส่งอยู่) คลิกที่เข้ามาจะถูกกรองทิ้งทันที ไม่เรียก `submit()` ซ้ำ — คล้ายจุดประสงค์เดียวกับ [[exhaustMap]] แต่ `filter` เหมาะกับกรณีที่เงื่อนไขซับซ้อนกว่าแค่ "รอ request ก่อนหน้าเสร็จ"

---

## ตัวอย่าง — API call: กรอง response ก่อนประมวลผลต่อ

```ts
this.http.get<Item[]>('/api/items').pipe(
  map(items => items.filter(item => item.active)),   // filter ปกติของ array — คนละตัวกับ RxJS filter operator
).subscribe();
```

**ระวังสับสน:** `filter` ของ RxJS operator (กรอง**ค่าที่ไหลผ่าน stream**) กับ `Array.prototype.filter` (กรอง**element ใน array**) เป็นคนละอย่างกัน แม้ชื่อเหมือนกัน

---

## กับดัก

- **สับสน RxJS `filter` กับ `Array.filter`** — ใช้ผิดบริบทจะเกิด type error หรือพฤติกรรมไม่ตรงที่คิด (ดูตัวอย่างข้างบน)
- **ใช้ `filter` แทนที่ควรจะจัดการ error ด้วย `catchError`** — `filter` แค่ตัดค่าทิ้งเงียบ ๆ ไม่ใช่ error handling ถ้าค่านั้นควรถือเป็นข้อผิดพลาดที่ต้องแจ้งผู้ใช้ ควรใช้ [[catchError]] หรือ throw error แทน

---

## Cheat sheet

```ts
source$.pipe(filter(value => value > 0))
source$.pipe(filter((value): value is string => typeof value === 'string'))   // type guard — narrow type ให้ TypeScript ด้วย
```

## 🔗 เกี่ยวข้อง

- [[RxJS Operators]] — สรุปรวม operator ที่ใช้บ่อยทั้งหมด
- [[exhaustMap]] — อีกวิธีกันการกดซ้ำระหว่างรอ request

## 📖 อ่านต่อ

- [RxJS — filter](https://rxjs.dev/api/operators/filter)
