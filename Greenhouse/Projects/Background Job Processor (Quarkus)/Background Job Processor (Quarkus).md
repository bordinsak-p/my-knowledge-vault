---
tags:
  - research
  - research/seed
  - java
  - quarkus
type: project
status: seed
area: backend
created: 2026-09-21
updated: 2026-09-28
---

# ⚙️ Background Job Processor (Quarkus)

## 🎯 ปัญหา

อยากฝึกทักษะ **"background task"** โดยเฉพาะ — pattern ที่เจอบ่อยในงานประจำ (API รับข้อมูลเข้ามาเยอะๆ แล้วมีอะไรบางอย่างประมวลผลทีหลังแบบ async) แต่ไม่เคยลงมือทำเองตั้งแต่ต้นจนจบ ไม่รู้จริงว่ากลไกเบื้องหลัง (claim งานไม่ให้ชนกัน, retry แบบไม่ซ้ำงาน, backpressure) ทำงานยังไง

รูปแบบกว้างๆ ที่คุยกันไว้: **API insert ข้อมูลจำนวนมาก → background task แยกออกไปประมวลผลทีหลัง** — ส่วนนี้ปิดแล้ว ตัวเลือก use case ที่คุยกันไว้ตอนแรก:

1. ~~Bulk import + validate/enrich~~ — insert ข้อมูลดิบเยอะๆ worker ทีหลัง validate + enrich เขียนผลลง table ที่สอง
2. ~~Job/report generation~~ — insert "job" worker ทำงานหนักจริงแล้วอัปเดตสถานะ+path ผลลัพธ์
3. **Notification fan-out** ← **เลือกข้อนี้ (ยืนยัน 2026-09-28)** — insert "campaign" พร้อมรายชื่อผู้รับเยอะๆ worker ไล่ส่ง (mock) ทีละคน ต้องมี rate limit + retry + สถานะต่อรายคน
4. ~~Event rollup/analytics~~ — insert raw event เข้ามาต่อเนื่อง worker รันเป็นรอบๆ สรุปยอดลง summary table

**เหตุผลที่เลือกข้อ 3:** ให้ทั้ง concurrency (หลาย worker แย่งกันหยิบผู้รับ), retry ต่อรายคน, rate limit, และสถานะย่อยที่ต้อง track ทีละแถว — ครบทุก component ที่อยากฝึกในข้อ 🎯 ด้านบนในโดเมนเดียว โดยตัวงานจริง (ส่งข้อความ) mock ได้เต็มที่ไม่ต้องต่อ provider จริงเลย

**รายละเอียด endpoint/schema/worker algorithm เต็มๆ อยู่ที่ [[API Flow Spec]]**

## 💭 สมมติฐาน

เชื่อว่า pattern หลักของ background task (claim แถวไม่ให้ worker หลายตัวแย่งกัน, retry แบบ idempotent ไม่ประมวลผลซ้ำ, สังเกต backpressure ตอน insert เร็วกว่า process) เรียนรู้ได้จากโปรเจกต์ที่ตัว "งาน" ของ worker เองไม่จำเป็นต้องซับซ้อนมาก — แค่ mechanism ที่ล้อมรอบมันต้องแน่นจริง ถ้าสมมติฐานนี้ผิด (คือต้องมี use case ที่ธุรกิจซับซ้อนจริงถึงจะฝึกอะไรได้) จะรู้ตอนทำการทดลองเล็กที่สุดด้านล่างแล้วรู้สึกว่างเปล่าไม่ได้อะไร

## ❓ ทำไมต้องตอนนี้

ไม่มีเหตุผลเชิงเวลาเฉพาะ — เจอ pattern "insert เยอะๆ แล้วประมวลผลทีหลัง" ในงานประจำอยู่บ่อย แต่ไม่เคยได้ลงมือสร้างกลไกนี้เองตั้งแต่ต้นจนจบสักที เป็นช่องว่างทักษะที่รู้ตัวมานานแล้ว

## 🔍 Prior art

(ยังไม่ได้ค้น — ทำตอน `sprouting`)

## 🧱 Design ที่เสนอ

**DB: แนะนำ PostgreSQL** (ยังไม่ยืนยัน — เป็นคำแนะนำ ไม่ใช่มติร่วม เปลี่ยนได้ถ้าไม่อยากลงเครื่อง Postgres) เหตุผล: การ "claim" งานไม่ให้ worker สองตัวหยิบแถวเดียวกัน — pattern มาตรฐานคือ `SELECT ... FOR UPDATE SKIP LOCKED` ซึ่ง Postgres รองรับเต็มรูปแบบ (SQLite ไม่มี, H2 รองรับจำกัด) เป็น concept ที่**อยากฝึกโดยตรง**ในโปรเจกต์นี้ ใช้ตัวที่รองรับดีที่สุดคุ้มกว่า

**Trigger กลไก — แบ่งเป็น 2 เฟส:**

| เฟส | กลไก | เมื่อไหร่ |
|---|---|---|
| **1 (ทำตอนนี้)** | Quarkus `@Scheduled` polling ทุก 2 วินาที | เริ่มจากง่ายที่สุด ไม่ต้องมี message broker |
| 2 (ต่อยอดทีหลัง ถ้าอยากลอง) | Message queue จริง ([[RabbitMQ]]) | หลังกลไก polling+claim ใช้งานได้จริงแล้ว ค่อยเทียบว่าเปลี่ยนเป็น queue ได้อะไรเพิ่ม |

**Rate limit — แบ่งเป็น 2 เฟสเช่นกัน:** เฟส 1 คุมง่ายๆ ด้วยขนาด batch ต่อ tick (ไม่ต้องพึ่ง Redis) เฟส 2 ค่อยเปลี่ยนเป็น token bucket จริงจาก [[Quarkus Redis]] เมื่ออยากลองจำกัด throughput ให้แม่นยำกว่านี้

**Retry — เฟส 1 ทำแบบง่ายสุดก่อน:** retry ทันทีในรอบถัดไป ไม่มี backoff ยัง**ไม่**ทำ exponential backoff (เพิ่ม schema/ความซับซ้อนทีหลังได้ถ้าจำเป็นจริง)

รายละเอียดเต็มของทุกจุดข้างบน (schema, endpoint, worker algorithm, ตัวอย่างเดินเทรซจริง) → [[API Flow Spec]]

## ✅ รู้ได้ยังไงว่าสำเร็จ

- Insert ผู้รับจำนวนมาก (เช่น 10,000 รายชื่อ) ผ่าน API แล้ว background task ประมวลผลครบทุกรายชื่อ **โดยไม่มีรายชื่อไหนถูกส่งซ้ำ** (พิสูจน์ว่า claim/idempotency ทำงานถูก)
- ปิด worker กลางคันตอนกำลังมีงานค้าง (สถานะ `SENDING`) แล้วเปิดใหม่ — งานที่ค้างต้องถูกหยิบไปทำต่อ ไม่หายไปไม่ทำซ้ำ (ทดสอบ reclaim-stuck ใน [[API Flow Spec]])
- วัด throughput ได้จริงเป็นตัวเลข (records/second) และรู้ว่าคอขวดอยู่ที่ insert หรือที่ process

## 🧪 การทดลองที่เล็กที่สุด

**ข้อสำคัญ: ไม่ทำ "API ให้ครบก่อน" แล้วค่อยไปทำ background task ทีหลัง** — เพราะ CRUD API เป็นส่วนที่คุ้นมืออยู่แล้ว (เห็นได้จากโน้ต Quarkus อื่นๆ ในวอลต์นี้) ความเสี่ยงจริงคือถ้าทำ API เต็มก่อน จะหมดแรง/เวลาไปกับส่วนที่ไม่ใช่เป้าหมายจริงของโปรเจกต์นี้ (ดู ⚠️ ความเสี่ยง ข้อแรก)

แนะนำ **vertical slice บางที่สุดที่แตะทั้งสองฝั่งตั้งแต่รอบแรก** แทน (เวอร์ชันย่อของ [[API Flow Spec]]):

1. `POST /api/campaigns` insert campaign 1 ตัว + ผู้รับ 1 คน สถานะ `PENDING` (ไม่ต้อง validate อะไรเลย)
2. `@Scheduled` job เดียวหยิบ 1 แถว `PENDING` มา "ส่ง" แบบ mock ง่ายสุด (log + sleep) แล้วเปลี่ยนเป็น `SENT`
3. ยิง insert รัวๆ 2-3 ครั้งพร้อมกัน เช็คว่าไม่มีแถวไหนถูกหยิบซ้ำ

ทำให้จบภายในไม่กี่ชั่วโมง — ถ้าผ่านค่อยขยับไปตาม spec เต็มใน [[API Flow Spec]] (retry, reclaim-stuck, rate limit แบบ batch)

## ⚠️ ความเสี่ยง / สิ่งที่อาจทำให้พัง

- **หลงทำ CRUD API เต็มรูปแบบก่อน แล้วไม่เหลือเวลา/แรงจูงใจมาทำ background task จริง** — ความเสี่ยงอันดับหนึ่งของโปรเจกต์นี้ (ตรงกับคำถามตอนเริ่ม ว่าจะทำแค่ API ก่อนดีไหม)
- ลืมเรื่อง concurrency (หลาย worker/thread แย่งกันหยิบแถวเดียวกัน) จนกว่าจะเจอบั๊กจริงตอน insert เร็วๆ พร้อมกัน
- ไม่ทำ idempotency ตั้งแต่แรก พอเพิ่ม retry เข้ามาทีหลัง โครงสร้างเดิมไม่รองรับ ต้องรื้อใหม่
- **หลงทำ mock notification ให้ realistic เกินไปจนกลายเป็นต่อ SMS/email provider จริง** — เป้าหมายคือฝึก mechanism รอบๆ ไม่ใช่ integration กับผู้ให้บริการข้อความ mock (sleep + สุ่ม fail) พอแล้วตลอดโปรเจกต์นี้

## 📚 ต้องไปเรียนรู้เพิ่ม

- Quarkus `@Scheduled` (มีพื้นฐานอยู่แล้วที่ [[Quarkus Thread Pool]]) — โดยเฉพาะ `concurrentExecution` (default อนุญาตให้ tick ซ้อนกันได้ ตั้งใจใช้ประโยชน์จากจุดนี้ ดู [[API Flow Spec]])
- pattern "claim" ด้วย `FOR UPDATE SKIP LOCKED` ผ่าน native query ใน [[Quarkus EntityManager]] — ยังไม่เคยลองมือจริง
- idempotency key / claim pattern — หลักการทั่วไปเขียนไว้แล้วที่ [[Idempotency]] เหลือแค่ implement จริงตาม spec
- ถ้าไปถึงเฟส 2: message queue ([[RabbitMQ]]) เทียบกับ polling ว่าต่างกันจริงตรงไหน, rate limit จริงจาก [[Quarkus Redis]]

## 🔗 เกี่ยวข้องกับ

- [[API Flow Spec]] — schema, endpoint, worker algorithm เต็ม
- [[Quarkus Thread Pool]] — `@Scheduled`/thread pool ที่เป็นกลไกหลักเฟส 1
- [[Quarkus EntityManager]] — native query สำหรับ claim pattern (`FOR UPDATE SKIP LOCKED`)
- [[Quarkus Redis]] — rate limit แบบ token bucket (เฟส 2)
- [[RabbitMQ]] — ทางเลือกกลไก trigger แบบ message queue (เฟส 2)
- [[Idempotency]] — หลักการเบื้องหลัง claim/retry ที่โปรเจกต์นี้ฝึกตรงๆ
- [[Java OOP]] — แนวคิด notification channel (interface + polymorphism) ที่ `MockNotificationClient` ใช้
- [[Greenhouse]]

## 📝 Log

### 2026-09-21
- สร้างโน้ต จาก brainstorm เรื่องอยากฝึก background task โดยเฉพาะ
- คุยกันเรื่อง component ที่โปรเจกต์แบบนี้ควรมี: claim/concurrency, retry+idempotency, backpressure, observability
- เสนอ use case candidate 4 แบบ (ยังไม่เลือก): bulk import+enrich, job/report generation, notification fan-out+rate-limit, event rollup/analytics
- ตัดสินใจเรื่อง scope: **ไม่ทำ API ให้ครบก่อน** — เสี่ยงเสียเวลา/แรงจูงใจไปกับส่วนที่คุ้นมืออยู่แล้วทั้งที่เป้าหมายจริงคือ background task เลือกทำ vertical slice บางที่สุด (insert 1 แถว + `@Scheduled` หยิบไปประมวลผล mock) ตั้งแต่รอบแรกแทน
- ยังไม่ได้ตอบ: เลือก use case ไหน (ข้อ 1-4), กลไก trigger แบบไหน (polling/`@Scheduled`/queue) ← ต้องตอบก่อนขยับเป็น `sprouting`

### 2026-09-28
- **ตอบคำถามที่ค้างจาก 2026-09-21 ได้แล้ว: เลือกข้อ 3 — Notification fan-out** (ครบทุก component ที่อยากฝึกในโดเมนเดียว)
- ย้ายโน้ตเข้า folder `Greenhouse/Projects/Background Job Processor (Quarkus)/` แยก spec ออกเป็นไฟล์ [[API Flow Spec]] ต่างหาก (ตามแพทเทิร์นเดียวกับ [[URL Shortener (Quarkus)]]) — โน้ตนี้เหลือปัญหา/สมมติฐาน/design ภาพรวม/risk
- เขียน spec เต็ม: schema (`campaigns`/`campaign_recipients`), `POST /api/campaigns`, `GET /api/campaigns/{id}`, worker algorithm (claim ด้วย `FOR UPDATE SKIP LOCKED`, reclaim-stuck safety net, retry ไม่มี backoff), ตัวอย่างเดินเทรซจริง
- แนะนำ (ยังไม่ยืนยันร่วมกัน): **DB = PostgreSQL** เพราะต้องใช้ `SKIP LOCKED` เป็น concept หลักที่อยากฝึก
- ยังไม่ได้ตอบ: เห็นด้วยกับ Postgres ไหม, ขยับ status เป็น `sprouting` หรือยัง

## 🪦 ถ้าเลิกทำ

เลิกถ้า: ลองทำ vertical slice เล็กสุดแล้วรู้สึกว่า pattern นี้เจอในงานประจำเป็นปกติอยู่แล้ว ไม่ได้ให้ทักษะใหม่พอจะเรียกว่าฝึกเพิ่ม
