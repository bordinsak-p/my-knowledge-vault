---
tags:
  - rxjs
  - operator
type: reference
created: 2026-09-19
---

# 👀 tap — side effect โดยไม่แตะค่าที่ไหลผ่าน

> `tap` "แอบดู" ค่าที่ไหลผ่าน pipeline โดย**ไม่เปลี่ยนแปลงค่านั้นเลย** — ค่าที่เข้าไปหน้าตายังไงก็ออกมาแบบนั้น ใช้ทำ side effect (log, เปิด/ปิด loading, debug) ระหว่างทาง

---

## ตัวอย่าง — UI action: เปิด/ปิด loading spinner

```ts
this.http.get('/api/data').pipe(
  tap(() => this.loading = true),
  finalize(() => this.loading = false),
).subscribe(data => this.data = data);
```

`tap` ตั้ง `loading = true` ตอนเริ่ม, `finalize` ปิด `loading = false` **ไม่ว่าจะจบแบบ success หรือ error** — แยก concern การจัดการ UI state ออกจาก logic หลักของ pipeline ได้ชัดเจน

---

## ต่างจาก `map` ตรงไหน

```ts
// tap — ไม่เปลี่ยนค่า แค่ทำ side effect
source$.pipe(tap(value => console.log(value)))   // subscriber ยังได้ value เดิม

// map — เปลี่ยนค่าจริง ๆ
source$.pipe(map(value => value * 2))            // subscriber ได้ value ที่ถูกแปลงแล้ว
```

**ถ้าต้องการแปลงค่า ใช้ `map`** — `tap` ไม่ได้ออกแบบมาให้แปลงค่า แม้จะเทคนิคจริงแล้วแก้ return value ข้างในได้ (ฟังก์ชันที่ส่งเข้าไปไม่บังคับ return type) แต่ผิดหลักการออกแบบ ทำให้โค้ดอ่านยากเพราะคนอ่านคาดหวังว่า `tap` ไม่เปลี่ยนอะไร

---

## กับดัก

- **ใส่ `tap` แล้วดันไปแก้ค่าที่ไหลผ่านข้างใน** (เช่น mutate object/array ที่รับมา) — ผิดหลักการออกแบบ ทำให้ debug ยากเพราะคนอ่านโค้ดคาดหวังว่า `tap` ไม่เปลี่ยนอะไร ถ้าต้องการแปลงค่าให้ใช้ `map` แทนเสมอ
- **ใช้ `tap` ทำ business logic สำคัญ** — เช่น เขียนข้อมูลลง database ข้าง `tap` ทำให้ side effect นั้นผูกอยู่กับว่ามี subscriber หรือไม่ (cold observable ไม่ subscribe ก็ไม่รัน) ควรแยก logic สำคัญออกมาให้ชัดเจนกว่านี้
- **ลืมว่า `tap` รันทุกครั้งที่มี subscriber ใหม่ (cold observable)** — ถ้า observable นั้นถูก subscribe หลายครั้ง `tap` (เช่น log) จะรันซ้ำหลายครั้งตามไปด้วย ไม่ใช่แค่ครั้งเดียว

---

## Cheat sheet

```ts
source$.pipe(
  tap(value => console.log('ก่อนแปลง:', value)),
  map(value => transform(value)),
  tap(value => console.log('หลังแปลง:', value)),
)
```

## 🔗 เกี่ยวข้อง

- [[RxJS Operators]] — สรุปรวม operator ที่ใช้บ่อยทั้งหมด
- [[exhaustMap]] — ใช้ `tap` คู่กันโชว์ loading state ระหว่างรอ

## 📖 อ่านต่อ

- [RxJS — tap](https://rxjs.dev/api/operators/tap)
