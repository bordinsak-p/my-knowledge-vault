---
tags:
  - rxjs
  - operator
type: reference
created: 2026-09-19
---

# 🔀 mergeMap — ทำพร้อมกันหมด ไม่สนใจลำดับ

> `mergeMap` คือ flattening operator ที่ **ปล่อยให้ inner Observable ทุกตัวทำงานพร้อมกัน** ไม่ยกเลิกตัวไหนเลย แม้ตัวใหม่จะเข้ามาก่อนตัวเก่าจะเสร็จ — เหมาะกับงานที่ **แต่ละตัวเป็นอิสระต่อกัน** ไม่เกี่ยวข้องกัน

---

## ตัวอย่าง — UI action: กดปุ่ม "like" หลายอันพร้อมกันได้

```ts
likeButtonClicks$.pipe(
  mergeMap(postId => this.http.post(`/api/posts/${postId}/like`, {})),
).subscribe();
```

กด like โพสต์ A แล้วกด like โพสต์ B ทันทีโดยไม่รอ A เสร็จก่อน — **ทั้งสอง request ยิงพร้อมกันได้เลย** เพราะเป็นคนละ resource ไม่เกี่ยวข้องกัน ไม่มีเหตุผลต้องรอ

---

## จำกัด concurrency ด้วย parameter ตัวที่สอง

```ts
from(fileList).pipe(
  mergeMap(file => uploadFile(file), 3),   // อัปโหลดพร้อมกันได้สูงสุด 3 ไฟล์ ที่เหลือรอคิว
).subscribe();
```

ไม่ใส่ parameter นี้ = ไม่จำกัด ยิงทุกตัวพร้อมกันหมด — งานที่มีจำนวนมาก (อัปโหลดหลายสิบไฟล์, ยิงหลายร้อย request) ควรใส่เลขจำกัดเสมอ กันเซิร์ฟเวอร์รับไม่ไหว

---

## เทียบกับ flattening operator ตัวอื่น

| operator | ตัวใหม่เข้ามาก่อนตัวเก่าเสร็จ ทำยังไง |
|---|---|
| **mergeMap** | ปล่อยให้ทำงานพร้อมกันทั้งคู่ |
| [[switchMap]] | ยกเลิกตัวเก่าทันที สลับไปตัวใหม่ |
| [[concatMap]] | เข้าคิวรอตัวเก่าเสร็จก่อน |
| [[exhaustMap]] | เมินตัวใหม่ทิ้งไปเลย |

---

## กับดัก

- **ไม่จำกัด concurrency ตอนงานมีจำนวนมาก** — ยิง request/อัปโหลดพร้อมกันหมดทุกตัว เซิร์ฟเวอร์รับไม่ไหว ควรใส่พารามิเตอร์ตัวที่สองเสมอเมื่อ source อาจมีจำนวนมาก
- **ใช้ `mergeMap` ตอนที่ต้องการลำดับ** — เช่น บันทึกหลายรายการที่ต้องเรียงตามลำดับที่ผู้ใช้ทำ `mergeMap` ไม่การันตีลำดับผลลัพธ์เลย (ยิงพร้อมกัน ใครเสร็จก่อนก็มาก่อน) ใช้ [[concatMap]] แทน
- **ใช้ `mergeMap` กับปุ่มที่ห้ามกดซ้ำ** — ทุกคลิกกลายเป็น request ใหม่พร้อมกันหมด ไม่มีอะไรกันการกดซ้ำเลย ใช้ [[exhaustMap]] แทน

---

## Cheat sheet

```ts
source$.pipe(mergeMap(value => innerObservable(value)))
source$.pipe(mergeMap(value => innerObservable(value), 3))   // จำกัด concurrency ที่ 3
```

## 🔗 เกี่ยวข้อง

- [[RxJS Operators]] — สรุปรวม operator ที่ใช้บ่อยทั้งหมด
- [[switchMap]] · [[concatMap]] · [[exhaustMap]] — flattening operator ตัวอื่นที่ต้องเลือกให้ถูก

## 📖 อ่านต่อ

- [RxJS — mergeMap](https://rxjs.dev/api/operators/mergeMap)
