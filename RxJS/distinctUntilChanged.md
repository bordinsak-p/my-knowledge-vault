---
tags:
  - rxjs
  - operator
type: reference
created: 2026-09-19
---

# 🔁 distinctUntilChanged — ไม่ยิงซ้ำถ้าค่าเหมือนเดิม

> `distinctUntilChanged` เทียบค่าปัจจุบันกับค่า**ก่อนหน้าทันที** (ไม่ใช่ทุกค่าที่เคยผ่านมา) — ถ้าเหมือนกันจะไม่ปล่อยค่านั้นออกไปซ้ำ

---

## ตัวอย่าง — API call: กันยิงซ้ำตอนค่าไม่ได้เปลี่ยนจริง

```ts
this.form.get('username')!.valueChanges.pipe(
  distinctUntilChanged(),
  switchMap(name => this.http.get(`/api/check-username/${name}`)),
).subscribe(available => this.usernameAvailable = available);
```

กันกรณีค่าไม่ได้เปลี่ยนจริง (เช่น พิมพ์ตัวอักษรแล้วลบออกกลับเป็นค่าเดิม หรือ event ยิงซ้ำโดยค่าเหมือนเดิม) แต่ยังมี event เข้ามา — ไม่ยิง API ตรวจสอบซ้ำโดยไม่จำเป็น

---

## เทียบ object/array ต้องระวัง

```ts
// ❌ default ใช้ === เทียบ — object/array คนละ reference ถือว่า "เปลี่ยน" เสมอแม้ค่าข้างในเหมือนกัน
source$.pipe(distinctUntilChanged())

// ✅ ใส่ comparator เองเทียบค่าข้างในแทน
source$.pipe(distinctUntilChanged((prev, curr) => prev.id === curr.id))
```

---

## กับดัก

- **คาดหวังว่าจะเทียบกับ "ทุกค่าที่เคยผ่านมา"** — ไม่ใช่ เทียบแค่กับค่า**ก่อนหน้าทันที**ตัวเดียว ถ้า A → B → A ค่า A ตัวที่สองจะถูกปล่อยออกไปเพราะต่างจาก B (ค่าก่อนหน้าทันที)
- **ใช้กับ object/array โดยไม่ใส่ comparator** — เทียบด้วย `===` (reference) เสมอ object ที่หน้าตาเหมือนกันแต่คนละ reference ถือว่า "เปลี่ยน" ตลอด ต้องใส่ comparator function เทียบค่าข้างในเอง

---

## Cheat sheet

```ts
source$.pipe(distinctUntilChanged())                                    // primitive (string/number)
source$.pipe(distinctUntilChanged((a, b) => a.id === b.id))             // object — เทียบเอง
```

## 🔗 เกี่ยวข้อง

- [[RxJS Operators]] — สรุปรวม operator ที่ใช้บ่อยทั้งหมด
- [[debounceTime]] — มักใช้คู่กันก่อน switchMap ตอนยิง API จาก input

## 📖 อ่านต่อ

- [RxJS — distinctUntilChanged](https://rxjs.dev/api/operators/distinctUntilChanged)
