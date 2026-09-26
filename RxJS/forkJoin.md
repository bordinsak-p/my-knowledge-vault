---
tags:
  - rxjs
  - operator
type: reference
created: 2026-09-19
---

# 🤝 forkJoin — รอทุกตัว complete แล้วยิงครั้งเดียว

> `forkJoin` คือ **`Promise.all()` เวอร์ชัน Observable** — ยิงหลาย Observable พร้อมกัน แล้ว**รอจนทุกตัว complete**ก่อนถึงจะปล่อยค่าออกมาครั้งเดียว (เอาค่าสุดท้ายของแต่ละตัวมารวมกัน)

---

## ตัวอย่าง — API call: โหลดข้อมูลหลายก้อนก่อนแสดงผลหน้าเดียว

```ts
forkJoin({
  user: this.http.get<User>('/api/user'),
  orders: this.http.get<Order[]>('/api/orders'),
}).subscribe(({ user, orders }) => {
  console.log(user, orders);   // ได้ทั้งคู่พร้อมกันตอนที่ทั้งสอง request เสร็จแล้ว
});
```

เหมาะกับหน้าที่ต้องมีข้อมูลครบทุกก้อนก่อนถึงจะ render ได้ — ยิง `/api/user` กับ `/api/orders` พร้อมกัน ไม่ต้องรอทีละตัว แต่ก็ไม่แสดงผลจนกว่าจะได้ครบทั้งคู่

---

## ข้อควรรู้ — ถ้าตัวใดตัวหนึ่ง error หรือไม่ complete

**ถ้ามีตัวใดตัวหนึ่ง error → `forkJoin` ทั้งก้อน error ทันที** ตัวอื่นที่ยัง pending อยู่จะถูกยกเลิกไปด้วย — ต้องใส่ `catchError` ในแต่ละ Observable ย่อยเองถ้าอยากให้ตัวที่ error ไม่ทำให้ตัวอื่นพังไปด้วย:

```ts
forkJoin({
  user: this.http.get<User>('/api/user').pipe(catchError(() => of(null))),
  orders: this.http.get<Order[]>('/api/orders').pipe(catchError(() => of([]))),
}).subscribe(({ user, orders }) => { /* user อาจเป็น null ถ้า request แรก fail แต่ orders ยังได้ค่า */ });
```

**ถ้า Observable ตัวไหนไม่เคย complete เลย (เช่น `interval` เปล่า ๆ)** — `forkJoin` จะไม่ยิงอะไรออกมาเลยตลอดไป เพราะรอให้ครบ complete ทุกตัวก่อนเสมอ ใช้กับ HTTP call (ที่ complete เองตามธรรมชาติ) เท่านั้น ไม่ใช้กับ stream ที่ไม่จบเอง

---

## เทียบกับ [[combineLatest]] และ [[zip]]

| operator | ยิงค่าออกมาตอนไหน | เหมาะกับ |
|---|---|---|
| **forkJoin** | รอทุกตัว **complete** แล้วยิงครั้งเดียว (ค่าสุดท้ายของแต่ละตัว) | เรียก API หลายตัวพร้อมกัน รอครบทุกตัว |
| [[combineLatest]] | ยิงทุกครั้งที่ตัวใดตัวหนึ่งปล่อยค่าใหม่ (หลังทุกตัวปล่อยค่าแรกแล้ว) | ค่าที่ derive จากหลาย stream ที่ยังเปลี่ยนต่อเนื่อง |
| [[zip]] | จับคู่ตามลำดับ (ค่าที่ 1 คู่กับค่าที่ 1, ค่าที่ 2 คู่กับค่าที่ 2 ...) | ต้องเรียงคู่ค่าจากหลาย source ให้ตรงกันตามลำดับ |

---

## กับดัก

- **ใช้กับ Observable ที่ไม่ complete เอง** — ไม่มีอะไรเกิดขึ้นเลยตลอดไป (ดูข้อควรรู้ด้านบน)
- **ไม่ใส่ `catchError` ในแต่ละตัวย่อย** — request หนึ่งพังแล้วทำให้ request อื่นที่โหลดสำเร็จแล้วถูกทิ้งไปด้วย ทั้งที่เอาไปใช้ต่อได้
- **ใช้ `forkJoin` ตอนต้องการ progressive/ทยอยแสดงผล** — `forkJoin` ให้ผลลัพธ์แค่ครั้งเดียวตอนจบ ถ้าต้องการเห็นผลลัพธ์ทีละตัวที่โหลดเสร็จก่อน ควรใช้ [[mergeMap]] หรือ `combineLatest` แทน

---

## Cheat sheet

```ts
forkJoin({ user: obsA$, orders: obsB$ }).subscribe(({ user, orders }) => { ... })
forkJoin([obsA$, obsB$]).subscribe(([a, b]) => { ... })   // แบบ array ก็ได้
```

## 🔗 เกี่ยวข้อง

- [[RxJS]] — use case แบบเต็มที่ forkJoin ปรากฏอยู่
- [[combineLatest]] · [[zip]] — operator รวม Observable ตัวอื่นที่ต้องเลือกให้ถูก
- [[catchError]] — ต้องใส่ในแต่ละ Observable ย่อยกัน error ตัวเดียวทำพังทั้งก้อน

## 📖 อ่านต่อ

- [RxJS — forkJoin](https://rxjs.dev/api/index/function/forkJoin)
