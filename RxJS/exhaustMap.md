---
tags:
  - rxjs
  - operator
type: reference
created: 2026-09-19
---

# 🔀 exhaustMap — เมินตัวใหม่ ถ้าตัวเก่ายังไม่เสร็จ

> `exhaustMap` คือ flattening operator ที่ **ทิ้งค่าใหม่จาก source ไปเลยถ้า inner Observable ตัวก่อนหน้ายังไม่เสร็จ** — ต่างจาก [[switchMap]] ที่ยกเลิกตัวเก่าไปหาตัวใหม่ `exhaustMap` เลือก**อยู่กับตัวเก่าต่อ**และเมินตัวใหม่ทั้งหมด เหมาะกับปุ่มที่ **ห้ามกดซ้ำระหว่างรอ**

---

## ตัวอย่าง — UI action: ปุ่ม login กันกดซ้ำ

```ts
loginButtonClicks$.pipe(
  exhaustMap(() => this.http.post('/api/login', credentials)),
).subscribe(result => this.handleLogin(result));
```

ผู้ใช้กดปุ่ม login รัว ๆ ระหว่างรอ response แรก — **คลิกที่เกิดขึ้นระหว่างรอถูกเมินไปเลย** ไม่ยิง request ซ้ำซ้อน ไม่ต้องเขียนโค้ดปิดปุ่ม (`disabled = true`) แยกต่างหาก operator จัดการให้ในตัวเสร็จ

---

## เทียบกับ flattening operator ตัวอื่น

| operator | ตัวใหม่เข้ามาก่อนตัวเก่าเสร็จ ทำยังไง |
|---|---|
| **exhaustMap** | เมิน/ทิ้งตัวใหม่ไปเลย |
| [[switchMap]] | ยกเลิกตัวเก่าทันที สลับไปตัวใหม่ |
| [[mergeMap]] | ปล่อยให้ทำงานพร้อมกันทั้งคู่ |
| [[concatMap]] | เข้าคิวรอตัวเก่าเสร็จก่อน |

---

## กับดัก

- **ใช้ `exhaustMap` กับ search box** — คลิก/พิมพ์ระหว่างรอ request แรกจะถูกเมินไปเลย ทั้งที่ผู้ใช้อาจตั้งใจค้นหาคำใหม่จริง ๆ ใช้ [[switchMap]] แทนสำหรับ search
- **เข้าใจผิดว่า `exhaustMap` "คิว" คลิกที่โดนเมินไว้ทำทีหลัง** — ไม่ใช่ คลิกที่โดนเมินคือหายไปเลย ไม่ถูกเก็บไว้ทำต่อ ถ้าต้องการเก็บไว้ทำทีหลังต้องใช้ [[concatMap]] แทน
- **ไม่มี feedback ให้ผู้ใช้รู้ว่ากำลังทำงานอยู่** — ผู้ใช้กดปุ่มแล้วไม่เห็นอะไรเกิดขึ้น (เพราะคลิกซ้ำโดนเมิน) อาจงงว่าปุ่มพังไหม ควรโชว์ loading state ระหว่างรอด้วยเสมอ (`tap`/`finalize`)

---

## Cheat sheet

```ts
source$.pipe(exhaustMap(value => innerObservable(value)))
```

## 🔗 เกี่ยวข้อง

- [[RxJS Operators]] — สรุปรวม operator ที่ใช้บ่อยทั้งหมด
- [[switchMap]] · [[mergeMap]] · [[concatMap]] — flattening operator ตัวอื่นที่ต้องเลือกให้ถูก
- [[tap]] — ใช้คู่กันโชว์ loading state ระหว่างรอ

## 📖 อ่านต่อ

- [RxJS — exhaustMap](https://rxjs.dev/api/operators/exhaustMap)
