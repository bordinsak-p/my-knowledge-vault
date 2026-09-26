---
tags:
  - rxjs
  - operator
type: reference
created: 2026-09-19
---

# 🔀 switchMap — ยกเลิกตัวเก่า สลับไปตัวใหม่เสมอ

> `switchMap` คือ flattening operator ที่ **ยกเลิก inner Observable ตัวก่อนหน้าทันที** ทุกครั้งที่มีค่าใหม่จาก source เข้ามา แล้วสลับไปทำตัวใหม่แทน — เหมาะกับสถานการณ์ที่ **ค่าล่าสุดสำคัญที่สุดเสมอ** ค่าเก่าไม่มีความหมายอีกต่อไป

---

## ตัวอย่าง — API call: ค้นหาแบบ real-time

```ts
searchInput$.pipe(
  debounceTime(300),
  switchMap(keyword => this.http.get<Result[]>(`/api/search?q=${keyword}`)),
).subscribe(results => this.results = results);
```

พิมพ์คำใหม่ก่อน request เก่าตอบ → **request เก่าถูกยกเลิกทันที** (HTTP request จริงถูก unsubscribe/abort) ไม่ต้องกลัวผลลัพธ์เก่ามาทับผลลัพธ์ใหม่ทีหลัง — นี่คือ race condition คลาสสิกที่ `switchMap` แก้ให้ฟรีโดยไม่ต้องเช็ค timestamp/id เอง

---

## ทำไมยกเลิกได้จริง ๆ

`switchMap` unsubscribe จาก inner Observable ตัวเก่าก่อนเสมอเมื่อมีตัวใหม่มา — ถ้า inner Observable นั้นเป็น cold observable (เช่น `HttpClient` ที่ทุก subscribe = request ใหม่) การ unsubscribe จะ **ยกเลิก request จริงที่ browser ยิงออกไปด้วย** (ไม่ใช่แค่เมินผลลัพธ์ตอนกลับมา) ดูหลักการ cold observable เต็ม ๆ ที่ [[Observable]]

---

## เทียบกับ flattening operator ตัวอื่น

| operator | ตัวใหม่เข้ามาก่อนตัวเก่าเสร็จ ทำยังไง |
|---|---|
| **switchMap** | ยกเลิกตัวเก่าทันที สลับไปตัวใหม่ |
| [[mergeMap]] | ปล่อยให้ทำงานพร้อมกันทั้งคู่ |
| [[concatMap]] | เข้าคิวรอตัวเก่าเสร็จก่อน |
| [[exhaustMap]] | เมินตัวใหม่ทิ้งไปเลย |

---

## กับดัก

- **ใช้ `switchMap` กับปุ่ม submit/login** — กดซ้ำแล้ว "ยกเลิก" request เดิมทิ้งแทนที่จะเมินการกดซ้ำ เสี่ยง transaction ค้างครึ่ง ๆ กลาง ๆ ถ้า backend ทำงานไปแล้วบางส่วนก่อนโดน abort ใช้ [[exhaustMap]] แทนสำหรับกรณีนี้
- **ใช้ `switchMap` ตอนที่ต้องการให้ทุก request เสร็จจริง** — เช่น ยิงหลาย analytics event ที่อยากให้ส่งครบทุกตัว `switchMap` จะทำให้ event เก่าถูกยกเลิกถ้ามี event ใหม่เข้ามาเร็ว ควรใช้ [[mergeMap]] แทน

---

## Cheat sheet

```ts
source$.pipe(switchMap(value => innerObservable(value)))
```

## 🔗 เกี่ยวข้อง

- [[RxJS Operators]] — สรุปรวม operator ที่ใช้บ่อยทั้งหมด
- [[mergeMap]] · [[concatMap]] · [[exhaustMap]] — flattening operator ตัวอื่นที่ต้องเลือกให้ถูก
- [[Observable]] — cold observable, ทำไม unsubscribe ถึงยกเลิก request จริงได้

## 📖 อ่านต่อ

- [RxJS — switchMap](https://rxjs.dev/api/operators/switchMap)
