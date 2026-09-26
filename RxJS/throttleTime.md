---
tags:
  - rxjs
  - operator
type: reference
created: 2026-09-19
---

# ⏱️ throttleTime — ทำทันทีตัวแรก แล้วปิดรับชั่วคราว

> `throttleTime(ms)` ปล่อยค่าแรกที่เข้ามาออกไป**ทันที** แล้ว "ปิดรับ" ค่าถัดไปทั้งหมดเป็นเวลาตามที่กำหนด — ต่างจาก [[debounceTime]] ที่รอให้หยุดก่อนค่อยทำงาน `throttleTime` ทำงานทันทีโดยไม่ต้องรอ

---

## ตัวอย่าง — UI action: จำกัดความถี่ของ auto-save

```ts
formChanges$.pipe(
  throttleTime(2000),
).subscribe(() => this.autoSave());
```

ผู้ใช้พิมพ์ต่อเนื่องในฟอร์ม — auto-save ทำงาน**ทันที**ตอนพิมพ์ตัวแรก แล้วปิดรับการเปลี่ยนแปลงเป็นเวลา 2 วินาที ก่อนจะรับค่าถัดไปมา save อีกครั้ง กันไม่ให้ save ถี่เกินไปจนกระทบ performance/quota ของ API

---

## ต่างจาก [[debounceTime]] ตรงไหน

| | `throttleTime` | `debounceTime` |
|---|---|---|
| ทำงานเมื่อไหร่ | ยิงทันทีตัวแรก แล้วปิดรับชั่วคราว | รอจน "หยุด" แล้วค่อยยิง (นับเวลาใหม่ทุก event) |
| เหมาะกับ | ปุ่มกดรัว ๆ ที่อยากให้ทำงานทันทีครั้งแรก (auto-save, scroll) | search box, validate ฟอร์มขณะพิมพ์ |

---

## กับดัก

- **สับสนกับ `debounceTime`** — เลือกผิดตัวได้ UX ที่ไม่ตรงกับที่ตั้งใจ (ตารางด้านบน)
- **ค่าสุดท้ายก่อนหมดช่วงเวลาหายไป** — `throttleTime` แบบ default ปล่อยแค่ค่าแรกของแต่ละช่วง ถ้าต้องการค่าล่าสุดของช่วงด้วยต้องตั้ง `{ trailing: true }` เพิ่ม
- **ตั้งเวลาสั้นเกินไปจนแทบไม่ต่างจากไม่มี throttle** — ต้องเลือกเวลาที่สอดคล้องกับสิ่งที่พยายามป้องกันจริง (เช่น scroll event ควรสั้นกว่า auto-save มาก)

---

## Cheat sheet

```ts
source$.pipe(throttleTime(2000))
source$.pipe(throttleTime(2000, undefined, { leading: true, trailing: true }))   // เอาทั้งค่าแรกและค่าสุดท้ายของช่วง
```

## 🔗 เกี่ยวข้อง

- [[RxJS Operators]] — สรุปรวม operator ที่ใช้บ่อยทั้งหมด
- [[debounceTime]] — คู่เทียบที่สับสนกันบ่อยที่สุด

## 📖 อ่านต่อ

- [RxJS — throttleTime](https://rxjs.dev/api/operators/throttleTime)
