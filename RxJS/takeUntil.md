---
tags:
  - rxjs
  - operator
type: reference
created: 2026-09-19
---

# 🛑 takeUntil — เลิกฟังอัตโนมัติ กัน memory leak

> `takeUntil(notifier$)` ฟังค่าจาก source ต่อไปเรื่อย ๆ จนกว่า `notifier$` จะปล่อยค่าออกมาสักครั้ง — พอนั้นแล้ว **complete ทันที** เลิกฟัง source ต่อ

---

## ตัวอย่าง — UI action: เลิกฟัง event ตอน component ถูกทำลาย

```ts
private destroy$ = new Subject<void>();

ngOnInit() {
  fromEvent(window, 'resize').pipe(
    takeUntil(this.destroy$),
  ).subscribe(() => this.recalculateLayout());
}

ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

`takeUntil` คู่กับ `Subject` ที่ยิงค่าตอน `ngOnDestroy` เป็น pattern มาตรฐานกัน **memory leak** จาก Observable ที่ไม่ complete เอง (event listener, `interval`, WebSocket) — ถ้าไม่เลิกฟัง component ที่ถูกทำลายไปแล้วจะยังมี callback ค้างอยู่เบื้องหลังทุกครั้งที่สร้าง component ใหม่ ยิ่งสร้าง-ทำลายบ่อยยิ่งสะสม

---

## ทำไม HTTP call ปกติไม่ต้องกังวลเรื่องนี้

```ts
this.http.get('/api/config').subscribe();   // ไม่ต้อง takeUntil ก็ได้ เพราะ complete เองอยู่แล้วหลังได้ response
```

`HttpClient` เป็น cold observable ที่ **complete เองทันทีหลังได้ response** ต่างจาก event/interval/WebSocket ที่ไม่มีจุดจบตามธรรมชาติ — ใส่ `takeUntil`/`take(1)` กับ HTTP call เพิ่มความชัดเจนได้แต่ไม่ได้จำเป็นเชิงป้องกัน memory leak เหมือนกรณี stream ที่ไม่จบเอง (ดู [[Observable]] ข้อ 8)

---

## กับดัก

- **ลืม `takeUntil` กับ Observable ที่ไม่ complete เอง** — memory leak สะสมทุกครั้งที่ component ถูกสร้างใหม่ อาการที่เจอบ่อยคือแอปช้าลงเรื่อย ๆ หลังใช้งานไปนาน โดยเฉพาะหน้าที่เข้า-ออกบ่อย
- **วาง `takeUntil` ผิดตำแหน่งใน `.pipe()`** — ต้องอยู่**ท้ายสุด**ของ pipe เกือบทุกครั้ง ถ้าวางไว้กลาง pipe operator ที่ตามมาหลังจากนั้นจะไม่โดน complete ไปด้วย
- **ลืม `.complete()` ที่ `destroy$` เอง** — เรียกแค่ `.next()` โดยไม่ `.complete()` ทำให้ `destroy$` เองก็เป็น Subject ที่ไม่จบ อาจเป็น leak เล็ก ๆ สะสมเช่นกัน

---

## Cheat sheet

```ts
private destroy$ = new Subject<void>();

source$.pipe(takeUntil(this.destroy$)).subscribe();

ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

## 🔗 เกี่ยวข้อง

- [[RxJS Operators]] — สรุปรวม operator ที่ใช้บ่อยทั้งหมด
- [[Observable]] — ทำไม event/interval ไม่ complete เอง ต่างจาก HTTP call

## 📖 อ่านต่อ

- [RxJS — takeUntil](https://rxjs.dev/api/operators/takeUntil)
