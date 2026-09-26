---
tags:
  - rxjs
  - operator
type: reference
created: 2026-09-19
---

# 🔗 combineLatest — ยิงทุกครั้งที่ตัวใดตัวหนึ่งเปลี่ยน

> `combineLatest` รวมหลาย Observable เข้าด้วยกัน แล้ว**ยิงค่าล่าสุดของทุกตัวออกมาใหม่ทุกครั้งที่มีตัวใดตัวหนึ่งปล่อยค่าใหม่** — ต้องรอให้**ทุกตัวปล่อยค่าแรกมาก่อนอย่างน้อยหนึ่งครั้ง** ถึงจะเริ่มยิงผลลัพธ์ออกมา

---

## ตัวอย่าง — UI action: validate ฟอร์มจากหลายฟิลด์พร้อมกัน

```ts
combineLatest([
  this.form.get('password')!.valueChanges,
  this.form.get('confirmPassword')!.valueChanges,
]).pipe(
  map(([password, confirm]) => password === confirm),
).subscribe(matches => this.passwordsMatch = matches);
```

`valueChanges` ของ Angular reactive form เป็น Observable อยู่แล้วในตัว — ทุกครั้งที่ผู้ใช้พิมพ์ในฟิลด์ใดฟิลด์หนึ่ง (password หรือ confirmPassword) `combineLatest` จะยิงค่าล่าสุดของ**ทั้งคู่**ออกมาใหม่ทันที ให้ validate ได้แบบ real-time

---

## จุดที่มือใหม่พลาดบ่อย — ต้องมีค่าเริ่มต้นครบทุกตัวก่อน

```ts
combineLatest([formControl1.valueChanges, formControl2.valueChanges])
```

**ถ้า field ไหนยังไม่เคยมีค่าเปลี่ยนเลยสักครั้ง (ยังไม่ fire event แรก) `combineLatest` จะไม่ยิงอะไรออกมาเลย** แม้ field อื่นจะเปลี่ยนค่าไปแล้วกี่ครั้งก็ตาม — ถ้าต้องการค่าเริ่มต้นทันทีโดยไม่ต้องรอ event แรกของทุกตัว ให้ใช้ `startWith()` ห่อแต่ละ source ไว้ก่อน:

```ts
combineLatest([
  formControl1.valueChanges.pipe(startWith(formControl1.value)),
  formControl2.valueChanges.pipe(startWith(formControl2.value)),
])
```

---

## เทียบกับ [[forkJoin]] และ [[zip]]

| operator | ยิงค่าออกมาตอนไหน | เหมาะกับ |
|---|---|---|
| **combineLatest** | ยิงทุกครั้งที่ตัวใดตัวหนึ่งปล่อยค่าใหม่ (หลังทุกตัวปล่อยค่าแรกแล้ว) | ค่าที่ derive จากหลาย stream ที่ยังเปลี่ยนต่อเนื่อง เช่น validate ฟอร์มจากหลายฟิลด์ |
| [[forkJoin]] | รอทุกตัว **complete** แล้วยิงครั้งเดียว (ค่าสุดท้ายของแต่ละตัว) | เรียก API หลายตัวพร้อมกัน รอครบทุกตัว |
| [[zip]] | จับคู่ตามลำดับ (ค่าที่ 1 คู่กับค่าที่ 1, ค่าที่ 2 คู่กับค่าที่ 2 ...) | ต้องเรียงคู่ค่าจากหลาย source ให้ตรงกันตามลำดับ |

---

## กับดัก

- **field หนึ่งยังไม่เคยยิงค่าเลยทำให้ทั้งก้อนเงียบตลอด** — ต้องมีค่าเริ่มต้นครบทุกตัวก่อน (ดูด้านบน แก้ด้วย `startWith`)
- **ใช้ `combineLatest` ตอนที่จริง ๆ อยากได้แค่ค่าตอน complete ครั้งเดียว** — เช่นเรียก API หลายตัวแล้วอยากได้ผลลัพธ์สุดท้ายก้อนเดียว ควรใช้ [[forkJoin]] แทน `combineLatest` จะยิงถี่เกินความจำเป็นถ้า source เป็น HTTP call ที่ complete เร็วอยู่แล้ว (ยิงซ้ำทุกครั้งที่มีตัวใหม่เสร็จ ไม่ใช่รอครบทีเดียว)
- **จำนวน source เยอะเกินไปจนตามยาก** — ยิ่งมี stream ใน `combineLatest` เยอะ ยิ่งยิงถี่ (ทุกตัวเปลี่ยนคือยิงใหม่) และยาก debug ว่าตัวไหนเป็นคนทำให้ยิง ควรจำกัดจำนวน source ให้พอดี

---

## Cheat sheet

```ts
combineLatest([obsA$, obsB$]).pipe(map(([a, b]) => ...))
combineLatest([obsA$.pipe(startWith(initA)), obsB$.pipe(startWith(initB))])   // กันเงียบตอนยังไม่มีค่าเริ่มต้น
```

## 🔗 เกี่ยวข้อง

- [[RxJS]] — use case แบบเต็มที่ combineLatest ปรากฏอยู่
- [[forkJoin]] · [[zip]] — operator รวม Observable ตัวอื่นที่ต้องเลือกให้ถูก

## 📖 อ่านต่อ

- [RxJS — combineLatest](https://rxjs.dev/api/index/function/combineLatest)
