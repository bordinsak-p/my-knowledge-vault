---
tags:
  - rxjs
  - reactive-programming
  - javascript
  - typescript
  - rxjs/index
type: moc
created: 2026-09-15
---

# 🌊 RxJS — ใช้ทำอะไรได้บ้าง

> **RxJS ไม่ใช่แค่ "Observable"** — Observable เป็นแค่ตัวข้อมูล ส่วน RxJS คือ **toolkit เต็มรูป**: operator กว่า 100 ตัวสำหรับแปลง/กรอง/รวม stream, `Subject` หลายแบบสำหรับสร้าง stream เอง, และฟังก์ชันสร้าง Observable จากแหล่งข้อมูลแทบทุกชนิด (event, HTTP, timer, promise, array)
> ก่อนอ่านโน้ตนี้ควรผ่าน [[Observable]] มาก่อน — อันนั้นคือ "หลักคิด" อันนี้คือ "เอาไปทำอะไรได้บ้างจริง ๆ"

---

## โน้ตในนี้

| โน้ต | ว่าด้วย | สถานะ |
|---|---|---|
| [[Observable]] | หลักคิดเบื้องหลัง, push vs pull, cold/hot, next/error/complete contract | ✅ |
| [[shareReplay]] | operator แปลง cold → hot + replay ค่าล่าสุด | ✅ |

### Operator เดี่ยว ๆ ที่ใช้บ่อย — แยกไฟล์ตามตัว

| โน้ต | ทำอะไร |
|---|---|
| [[switchMap]] | ยกเลิกตัวเก่า สลับไปตัวใหม่ — search |
| [[mergeMap]] | ทำพร้อมกันหมด ไม่สนลำดับ — like/independent action |
| [[concatMap]] | เข้าคิว ทีละตัวตามลำดับ — save ที่ต้องเรียง |
| [[exhaustMap]] | เมินตัวใหม่ถ้าตัวเก่ายังไม่เสร็จ — submit/login กันกดซ้ำ |
| [[debounceTime]] | รอจนหยุดพิมพ์/หยุดกดก่อนค่อยทำงาน |
| [[throttleTime]] | ทำทันทีตัวแรก แล้วปิดรับชั่วคราว |
| [[distinctUntilChanged]] | ไม่ยิงซ้ำถ้าค่าเหมือนเดิม |
| [[filter]] | กรองก่อนทำงานต่อ |
| [[takeUntil]] | เลิกฟังอัตโนมัติ กัน memory leak |
| [[tap]] | side effect ไม่แตะค่า (loading spinner, log) |
| [[catchError]] | จับ error กัน stream ตาย |
| [[forkJoin]] | รอทุกตัว complete แล้วยิงครั้งเดียว — เรียก API หลายตัวพร้อมกัน |
| [[combineLatest]] | ยิงใหม่ทุกครั้งที่มีตัวเปลี่ยน — validate ฟอร์มหลายฟิลด์ |
| [[zip]] | จับคู่ค่าตามลำดับที่มาถึง |

---

## 1. ใช้ทำอะไรได้บ้าง — ตามสถานการณ์จริง

### 1.1 ยิง HTTP แบบ debounce — ไม่ยิงทุกตัวอักษรที่พิมพ์

ใช้กับช่องค้นหา/autocomplete — `debounceTime` รอให้หยุดพิมพ์ก่อน แล้ว `switchMap` ยกเลิก request เก่าอัตโนมัติถ้าพิมพ์คำใหม่ก่อน request เก่าตอบ ดูโค้ดเต็มและเหตุผลที่ [[debounceTime]] และ [[switchMap]] (ดูเหตุผลเรื่อง cancel เต็ม ๆ ที่ [[Observable]] ข้อ 7)

### 1.2 ยิงหลาย API พร้อมกัน รอครบทุกตัว

ใช้ [[forkJoin]] — เหมือน `Promise.all()` แต่เป็น Observable เหมาะกับหน้าที่ต้องโหลดข้อมูลหลายก้อนก่อนแสดงผล ดูโค้ดเต็มที่ [[forkJoin]]

### 1.3 Retry อัตโนมัติเมื่อ request ล้มเหลว (พร้อม exponential backoff)

```ts
this.http.get('/api/data').pipe(
  retry({
    count: 3,
    delay: (error, retryCount) => timer(1000 * 2 ** retryCount), // รอ 2s, 4s, 8s
  }),
  catchError(err => { console.error('ลองครบ 3 ครั้งแล้วยังพัง', err); return of(null); }),
).subscribe();
```

เหมาะกับ network ที่ไม่เสถียร — ลองใหม่เองโดยไม่ต้องเขียน loop/setTimeout เอง

### 1.4 Form validation ที่ต้องดูหลายฟิลด์พร้อมกัน

ใช้ [[combineLatest]] — `valueChanges` ของ Angular reactive form เป็น Observable อยู่แล้วในตัว รวมหลายฟิลด์แล้ว derive ค่าใหม่ทุกครั้งที่ฟิลด์ใดฟิลด์หนึ่งเปลี่ยน ดูโค้ดเต็มที่ [[combineLatest]]

### 1.5 Real-time ผ่าน WebSocket

```ts
const messages$ = webSocket<ChatMessage>('ws://localhost:8080/chat/Somchai');

messages$.subscribe(msg => this.addMessage(msg));
messages$.next({ type: 'MESSAGE', text: 'สวัสดี' });   // ส่งออกก็ผ่าน Observable เดียวกัน
```

`webSocket()` ห่อ connection เป็น Observable สองทาง (ทั้งรับและส่ง) — ต่อ operator ได้เหมือน HTTP call ปกติ ฝั่ง server ดูได้ที่ [[Quarkus WebSocket]]

### 1.6 Polling — ดึงข้อมูลซ้ำเป็นช่วง ๆ

```ts
interval(5000).pipe(
  switchMap(() => this.http.get('/api/status')),
).subscribe(status => this.status = status);
```

### 1.7 คุมจำนวนงานที่ทำพร้อมกัน

ใช้ [[mergeMap]] พร้อม parameter ตัวที่สองจำกัด concurrency — กันไม่ให้ยิง request/อัปโหลดพร้อมกันหมดจนเซิร์ฟเวอร์ล่ม ดูโค้ดเต็มที่ [[mergeMap]]

### 1.8 State เบา ๆ ที่หลาย component ใช้ร่วมกัน

```ts
@Injectable({ providedIn: 'root' })
export class CartStore {
  private items$ = new BehaviorSubject<Item[]>([]);
  readonly items = this.items$.asObservable();

  add(item: Item) {
    this.items$.next([...this.items$.value, item]);
  }
}
```

`BehaviorSubject` เก็บค่าล่าสุดไว้เสมอ (subscriber ใหม่เห็นค่าปัจจุบันทันที ไม่ต้องรอ event ถัดไป) — ใช้แทน state management library เต็มรูปได้ถ้า state ไม่ซับซ้อนมาก

---

## 2. Operator แบ่งตามหมวด

| หมวด | ตัวอย่าง | ทำอะไร |
|---|---|---|
| **Creation** | `of`, `from`, `fromEvent`, `interval`, `timer`, `ajax`, `webSocket` | สร้าง Observable จากแหล่งข้อมูล |
| **Transformation** | `map`, `scan`, `switchMap`, `mergeMap`, `concatMap`, `exhaustMap` | แปลงค่า / จัดการ Observable ที่ซ้อนกัน |
| **Filtering** | `filter`, `debounceTime`, `distinctUntilChanged`, `take`, `takeUntil`, `skip` | กรอง/จำกัดว่าค่าไหนผ่านออกไป |
| **Combination** | `combineLatest`, `forkJoin`, `zip`, `merge`, `withLatestFrom` | รวมหลาย Observable เข้าด้วยกัน (ข้อ 3) |
| **Error handling** | `catchError`, `retry`, `retryWhen` | จับ/จัดการ error โดยไม่ทำให้ stream ตายไปเฉย ๆ |
| **Multicasting** | `share`, `shareReplay`, `Subject`, `BehaviorSubject`, `ReplaySubject` | แชร์ execution ให้หลาย subscriber (แปลง cold → hot) |
| **Utility** | `tap`, `delay`, `finalize`, `timeout` | side effect / debug โดยไม่เปลี่ยนค่าที่ไหลผ่าน |

### `forkJoin` vs `combineLatest` vs `zip` — สามตัวที่สับสนกันบ่อยที่สุด

แต่ละตัวแยกเป็นโน้ตของตัวเองพร้อมตารางเทียบและตัวอย่างเต็ม: [[forkJoin]] (รอ complete ครบทุกตัว), [[combineLatest]] (ยิงใหม่ทุกครั้งที่มีตัวเปลี่ยน), [[zip]] (จับคู่ตามลำดับ)

---

## 3. ไม่ได้ผูกกับ Angular

RxJS เป็น library แยก ใช้ได้ใน **vanilla JavaScript, Node.js backend (เช่น NestJS), React** ก็ได้ — Angular แค่เลือกใช้มันเป็น default สำหรับ HTTP/forms/router event เท่านั้น

**แนวคิดเดียวกันมีอยู่ในภาษาอื่นด้วย** — เป็นแพทเทิร์นข้ามภาษา ไม่ใช่ของเฉพาะ JS

| ภาษา/แพลตฟอร์ม | ชื่อ library |
|---|---|
| Java / Android / Quarkus reactive | RxJava (หรือ Mutiny ใน Quarkus) |
| .NET | Rx.NET |
| Swift / iOS | RxSwift |
| Python | RxPY |

---

## 4. กับดัก

- **สับสน `forkJoin`/`combineLatest`/`zip`** — เลือกผิดตัวได้ผลลัพธ์ไม่ตรงกับที่ต้องการ (ข้อ 2)
- **ยิง HTTP ทุกตัวอักษรที่พิมพ์เพราะลืม `debounceTime`** — เปลืองทั้ง network และ backend โดยไม่จำเป็น
- **`mergeMap` ไม่จำกัด concurrency ตอนอัปโหลด/ยิง request จำนวนมาก** — ยิงพร้อมกันหมดทุกตัว เซิร์ฟเวอร์รับไม่ไหว (ข้อ 1.7 แก้ด้วยพารามิเตอร์ตัวที่สอง)
- **ใช้ `Subject` ธรรมดาแทน `BehaviorSubject`** แล้วงงว่าทำไม subscriber ใหม่ไม่เห็นค่าปัจจุบัน — `Subject` ไม่เก็บค่าไว้ให้เลย
- **retry ไม่มี backoff** — ยิงซ้ำรัว ๆ ทันทีตอน server กำลังมีปัญหาอยู่แล้ว ยิ่งซ้ำเติม ควรมี delay ที่เพิ่มขึ้นเรื่อย ๆ เสมอ (ข้อ 1.3)

---

## Cheat sheet

```ts
// retry พร้อม backoff
obs$.pipe(retry({ count: 3, delay: (e, i) => timer(1000 * 2 ** i) }))

// state เบา ๆ
new BehaviorSubject(initialValue)
```

operator เดี่ยว ๆ (debounce, switchMap, forkJoin, combineLatest, zip, mergeMap concurrency ฯลฯ) มี cheat sheet ของตัวเองในแต่ละโน้ต — ดูตารางด้านบน

---

## 🔗 เกี่ยวข้อง

- [[Observable]] — หลักคิดเบื้องหลังทั้งหมดในโน้ตนี้
- [[switchMap]], [[mergeMap]], [[concatMap]], [[exhaustMap]], [[debounceTime]], [[throttleTime]], [[distinctUntilChanged]], [[filter]], [[takeUntil]], [[tap]], [[catchError]], [[forkJoin]], [[combineLatest]], [[zip]] — operator เดี่ยว ๆ แยกไฟล์ตามตัว ต่างจากโน้ตนี้ที่จัดตามสถานการณ์
- [[shareReplay]] — ใช้ตอนอยากแชร์ผลลัพธ์ HTTP call เดียวกันให้หลาย subscriber
- [[Quarkus WebSocket]] — ฝั่ง server ของ use case ข้อ 1.5
- [[Java Stream]] — mental model แบบ lazy pipeline ที่คล้ายกันฝั่ง Java

## 📖 อ่านต่อ

- [RxJS — Operators overview](https://rxjs.dev/guide/operators)
- [Learn RxJS — Operators by category](https://www.learnrxjs.io/learn-rxjs/operators)
- [RxJS — RetryConfig](https://rxjs.dev/api/operators/RetryConfig)
