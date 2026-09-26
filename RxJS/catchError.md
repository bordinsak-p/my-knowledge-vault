---
tags:
  - rxjs
  - operator
type: reference
created: 2026-09-19
---

# 🧯 catchError — จับ error โดยไม่ให้ stream ตาย

> เมื่อ Observable ปล่อย error ออกมา **stream นั้นจะ "ตาย" ทันที** ไม่มีค่าอะไรไหลผ่านได้อีกเลย — `catchError` ดักจับ error นั้นไว้แล้วให้เรา**คืน Observable ใหม่** (เช่น ค่า fallback) แทนที่ไปต่อได้ แทนที่จะปล่อยให้ subscriber โดน error ตรง ๆ

---

## ตัวอย่าง — API call: ให้ค่า fallback แทน error

```ts
this.http.get<Product[]>('/api/products').pipe(
  catchError(err => {
    console.error('โหลดสินค้าไม่สำเร็จ', err.status, err.message);
    return of([]);   // ให้ array ว่างแทนที่จะปล่อย error หลุดไปหา subscriber
  }),
).subscribe(products => this.products = products);
```

**ถ้าไม่มี `catchError`** — `subscribe` จะเข้า error callback ไปเลย (ถ้ามี) หรือโยน error ที่ไม่มีใครจับขึ้นมา และ**จะไม่ได้รับค่าอะไรอีกเลย** แม้ backend จะกลับมาทำงานปกติทีหลัง เพราะ stream ตายไปแล้ว

---

## ต้อง `return` Observable เสมอ ไม่ใช่ค่าธรรมดา

```ts
// ❌ ผิด — return ค่าธรรมดา ไม่ใช่ Observable
catchError(err => [])

// ✅ ถูก — ห่อด้วย of() ให้เป็น Observable
catchError(err => of([]))
```

`catchError` คาดหวังว่า callback จะ**คืน Observable** เสมอ (ปกติใช้ `of(fallbackValue)` หรือ `throwError(() => newError)` ถ้าอยากโยน error ใหม่ต่อ)

---

## กับดัก

- **ลืมว่าไม่มี `catchError` = stream ตายถาวร** — subscriber จะไม่ได้รับค่าอะไรอีกเลยแม้จะพยายาม retry ทีหลัง (การ retry เต็มรูปพร้อม backoff ดูที่ [[RxJS]] ข้อ 1.3)
- **`return` ค่าธรรมดาแทนที่จะ `return` Observable** — TypeScript อาจ error หรือพฤติกรรมไม่ตรงที่คาด ต้องห่อด้วย `of(...)` เสมอ
- **วาง `catchError` ผิดตำแหน่งใน pipe แบบ higher-order** (เช่นข้างใน `switchMap`) — error ที่เกิดข้างใน inner Observable ถ้าไม่จับตรงนั้น จะทำให้ **outer stream ตายไปด้วย** ไม่ใช่แค่ inner ตัวเดียว ต้องวาง `catchError` ให้ครอบคลุมจุดที่ error อาจเกิดจริง ๆ
- **จับ error แล้วเงียบไปเลยไม่แจ้งผู้ใช้** — ให้ fallback เงียบ ๆ โดยไม่มี log/แจ้งเตือน ทำให้ debug ยากตอนมีปัญหาจริงเพราะไม่มีร่องรอยอะไรเลย

---

## Cheat sheet

```ts
source$.pipe(
  catchError(err => {
    console.error(err);
    return of(fallbackValue);
  }),
)
```

## 🔗 เกี่ยวข้อง

- [[RxJS Operators]] — สรุปรวม operator ที่ใช้บ่อยทั้งหมด
- [[RxJS]] — retry พร้อม exponential backoff (ข้อ 1.3) ก่อนจะยอม `catchError` fallback

## 📖 อ่านต่อ

- [RxJS — catchError](https://rxjs.dev/api/operators/catchError)
