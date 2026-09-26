---
tags:
  - rxjs
  - operator
type: reference
created: 2026-09-19
---

# ⏱️ debounceTime — รอให้ "หยุด" ก่อนค่อยทำงาน

> `debounceTime(ms)` รอจนกว่าจะไม่มีค่าใหม่เข้ามาอีกภายในเวลาที่กำหนด **แล้วค่อยปล่อยค่าล่าสุดออกไป** — ทุกครั้งที่มีค่าใหม่เข้ามาก่อนครบเวลา จับเวลาใหม่ทั้งหมด (reset timer)

---

## ตัวอย่าง — API call: ไม่ยิงทุกตัวอักษรที่พิมพ์

```ts
searchInput$.pipe(
  debounceTime(300),              // รอ 300ms หลังพิมพ์หยุด
  distinctUntilChanged(),
  switchMap(keyword => this.http.get<Result[]>(`/api/search?q=${keyword}`)),
).subscribe(results => this.results = results);
```

ใช้กับช่องค้นหา/autocomplete — ผู้ใช้พิมพ์เร็ว ๆ ต่อเนื่องกัน 10 ตัวอักษร จะยิง API แค่ **1 ครั้ง** (หลังพิมพ์หยุดจริง 300ms) แทนที่จะยิง 10 ครั้งตามทุกตัวอักษร

---

## ต่างจาก [[throttleTime]] ตรงไหน

| | `debounceTime` | `throttleTime` |
|---|---|---|
| ทำงานเมื่อไหร่ | รอจน "หยุด" แล้วค่อยยิง (นับเวลาใหม่ทุก event) | ยิงทันทีตัวแรก แล้วปิดรับชั่วคราว |
| เหมาะกับ | search box, validate ฟอร์มขณะพิมพ์ | ปุ่มกดรัว ๆ ที่อยากให้ทำงานทันทีครั้งแรก (auto-save, scroll) |

**เลือกผิดตัว = UX ผิดทันที** — ใช้ `debounceTime` กับปุ่มที่อยากให้ตอบสนองทันที จะรู้สึกหน่วง/ดีเลย์ผิดที่

---

## กับดัก

- **ลืมใส่ `debounceTime` กับ input ที่ผูก API call** — ยิง request ทุกตัวอักษรที่พิมพ์ เปลืองทั้ง network และ backend โดยไม่จำเป็น
- **ตั้งเวลานานเกินไป (เช่น 1000ms+)** — ผู้ใช้รู้สึกว่าแอปหน่วง ค่าทั่วไปที่ใช้บ่อยคือ 200-400ms
- **ใช้ `debounceTime` กับปุ่มคลิกที่ต้องการ response ทันที** — ผิดโอกาส ควรใช้ [[throttleTime]] หรือ [[exhaustMap]] แทนแล้วแต่สถานการณ์

---

## Cheat sheet

```ts
source$.pipe(debounceTime(300))
```

## 🔗 เกี่ยวข้อง

- [[RxJS Operators]] — สรุปรวม operator ที่ใช้บ่อยทั้งหมด
- [[throttleTime]] — คู่เทียบที่สับสนกันบ่อยที่สุด
- [[distinctUntilChanged]] — มักใช้คู่กันหลัง debounce เพื่อกันยิงซ้ำถ้าค่าไม่เปลี่ยน
- [[switchMap]] — operator ที่ต่อท้าย debounce บ่อยที่สุดตอนยิง API

## 📖 อ่านต่อ

- [RxJS — debounceTime](https://rxjs.dev/api/operators/debounceTime)
