# RxJS: shareReplay()

## 💡 คืออะไร
`shareReplay` เป็น **RxJS operator** ที่ใช้ทำให้ Observable:
1. **แชร์ค่า (multicast)** ให้หลาย subscriber ใช้ร่วมกันได้ โดยไม่ต้องสร้าง stream ใหม่
2. **เก็บค่า (replay)** ล่าสุดไว้ให้ subscriber ใหม่ที่เข้ามาภายหลัง

---

## ปัญหาที่มันแก้

ปกติแล้ว **Observable ใน RxJS เป็น cold** → คือถ้ามี 2 subscriber มาต่อกับ observable ตัวเดียวกัน มันจะ **ยิง source ใหม่ 2 รอบ** (เช่น request API ซ้ำ 2 ครั้ง)

## 🔎 ทำไมต้องใช้
ปกติ **Observable เป็น cold** → ทุกครั้งที่ subscribe จะสร้าง execution ใหม่  
```ts
const obs$ = ajax.getJSON('/api/data');

obs$.subscribe(console.log); // 🔥 ยิง API รอบที่ 1
obs$.subscribe(console.log); // 🔥 ยิง API รอบที่ 2
```

## ตัวอย่างการใช้งาน

```ts
import { ajax } from 'rxjs/ajax';
import { shareReplay } from 'rxjs/operators';

const obs$ = ajax.getJSON('/api/data').pipe(
  shareReplay(1) // เก็บค่าไว้ 1 ค่า
);

obs$.subscribe(console.log); // 🔥 call API รอบเดียว
obs$.subscribe(console.log); // ✅ ใช้ cache ที่เก็บไว้
```


## 🎯 ใช้บ่อยตรงไหน

- **Cache API result** → ไม่ยิง API ซ้ำเมื่อมีหลาย subscriber
- **State management** → เก็บค่าล่าสุดไว้ให้ component ใหม่ ๆ ใช้
- **UI / Form events** → subscriber ใหม่เห็นค่าล่าสุดได้ทันที
## ⚠️ ข้อควรระวัง

- อย่าใช้ `shareReplay(Infinity)` กับ stream ที่ไม่สิ้นสุด (เช่น websocket) → เสี่ยง memory leak
- ใช้กับ one-time stream (HTTP request) → เหมาะมาก
- ถ้าเป็น long-living stream → ต้องจัดการ unsubscribe ให้ดี