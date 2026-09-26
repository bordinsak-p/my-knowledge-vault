---
tags:
  - design-patterns
  - api-design
  - distributed-systems
type: reference
created: 2026-09-23
---

# 🔁 Idempotency

> **ไม่ใช่ GoF design pattern** แต่เป็น**คุณสมบัติของ operation** ที่ถูกถามคู่กับ design pattern บ่อยมากในสัมภาษณ์สาย backend/system design — เก็บไว้ในหมวดนี้เพราะเป็นหลักการทั่วไปเหมือนกัน ไม่ผูกภาษา/framework
>
> **Idempotent = ทำ operation เดิมซ้ำกี่ครั้งก็ได้ผลลัพธ์สุดท้ายเหมือนกับทำครั้งเดียว** — คำจำกัดความทางการจาก [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html) (HTTP Semantics ฉบับปัจจุบัน แทนที่ RFC 7231 เดิม): *"the intended effect on the server of multiple identical requests... is the same as the effect for a single such request"*

---

## 1. ความเข้าใจผิดที่พบบ่อยที่สุด

### 1.1 Idempotent ≠ Safe (ไม่มีผลข้างเคียง)

```
DELETE /orders/5   → มีผลข้างเคียงจริง (ลบ order 5 ทิ้ง)
                   → แต่ยิงซ้ำ 5 ครั้ง state สุดท้ายเหมือนกัน (order 5 หายไปแล้ว)
                   → จึงเป็น idempotent ได้ทั้งที่ "ไม่ safe"
```

**Safe** (ไม่มีผลข้างเคียงเลย เช่น `GET`) กับ **Idempotent** (ยิงซ้ำแล้ว state เหมือนเดิม) เป็นคุณสมบัติคนละอย่าง — `GET` เป็นทั้งคู่, `DELETE`/`PUT` idempotent แต่ไม่ safe, `POST` ธรรมดาไม่เป็นทั้งคู่

### 1.2 Idempotent ≠ "ทำได้แค่ครั้งเดียว"

```
SET x = 5        → เรียกกี่ครั้ง x ก็เป็น 5           → idempotent
x = x + 1        → เรียก 3 ครั้ง x เพิ่มไป 3        → ไม่ idempotent (ผลสะสมตามจำนวนครั้ง)
mark_as_read()   → อ่านซ้ำกี่ครั้ง สถานะก็ "อ่านแล้ว"   → idempotent
increment_view() → view ซ้ำกี่ครั้ง นับเพิ่มทุกครั้ง      → ไม่ idempotent
```

Idempotent operation **เรียกซ้ำได้เรื่อยๆ อย่างปลอดภัย** ไม่ใช่ operation ที่ถูกออกแบบให้เรียกได้ครั้งเดียวแล้วบล็อกครั้งต่อไป —ตรงกันข้ามเลย มันคือสิ่งที่ทำให้ **retry ได้อย่างสบายใจ** โดยไม่ต้องกลัวผลซ้ำ

---

## 2. ทำไมเรื่องนี้สำคัญมาก — "exactly-once" เป็นไปไม่ได้จริงในระบบ distributed

เวลาส่ง request/message ข้าม network มี 3 แบบการันตีที่เป็นไปได้:

| การันตี | ความหมาย | ปัญหา |
|---|---|---|
| **At-most-once** | ส่งแค่ครั้งเดียว ไม่ retry | timeout แล้วไม่รู้ว่าปลายทางทำสำเร็จหรือยัง → อาจ**หายไปเงียบๆ** |
| **At-least-once** | retry จนกว่าจะมั่นใจว่าสำเร็จ | ปลายทางอาจ**ได้รับซ้ำ**หลายครั้ง |
| **Exactly-once** | ได้รับและประมวลผลพอดี 1 ครั้งเป๊ะ | **เป็นไปไม่ได้จริงที่ network layer ล้วนๆ** (Two Generals Problem — พิสูจน์ทางคณิตศาสตร์แล้วว่าสอง node คุยกันผ่าน channel ที่ไม่เสถียร ไม่มีทางรู้ 100% ว่าอีกฝั่งได้รับหรือเปล่าด้วยจำนวนข้อความจำกัด) |

**สิ่งที่ระบบจริงทำได้คือ "effectively exactly-once" = At-least-once delivery + Idempotent processing** — ยอมรับว่า message อาจมาซ้ำ (retry เมื่อสงสัยว่า timeout) แต่ทำให้ตัวประมวลผลปลายทาง **idempotent** จนซ้ำกี่ครั้งก็ไม่มีผลต่าง นี่คือเหตุผลที่ทุก background job / message queue consumer / payment API ที่ดีต้องออกแบบให้ idempotent — ไม่มีทางเลี่ยงได้เลยถ้าระบบต้อง retry (ดู [[Background Job Processor (Quarkus)]] ที่ต้องใช้หลักการนี้ตรงๆ)

---

## 3. HTTP method กับ idempotency — สรุปสั้น

| Method | Idempotent? |
|---|---|
| `GET` / `HEAD` / `OPTIONS` | ✅ (เป็น safe ด้วย) |
| `PUT` | ✅ (แทนที่ทั้งก้อนด้วยค่าเดิม) |
| `DELETE` | ✅ (state สุดท้ายเหมือนกันไม่ว่ายิงกี่ครั้ง) |
| `POST` | ❌ โดยธรรมชาติ (ยิงซ้ำ = สร้างซ้ำ) |
| `PATCH` | ⚠️ กำกวม — สเปกไม่บังคับ ขึ้นกับว่า patch นั้นเป็น "set ค่า" หรือ "increment" |

รายละเอียดเต็ม (diagram, กับดักเรื่อง DELETE ซ้ำแล้ว throw 500 ทำให้ไม่ idempotent จริง, ตัวอย่าง Quarkus) อยู่ที่ [[Quarkus HTTP Methods]] — โน้ตนี้เจาะเฉพาะหลักการทั่วไปที่ใช้ได้นอกเหนือจาก HTTP ด้วย (message queue, background job, RPC ก็ต้องคิดเรื่องเดียวกัน)

---

## 4. ทำให้ operation เป็น idempotent จริง — เทคนิคที่ใช้จริง

### 4.1 ออกแบบให้ idempotent โดยธรรมชาติ (ทางที่ดีที่สุด ถ้าทำได้)

```sql
-- ❌ ไม่ idempotent
UPDATE accounts SET balance = balance - 100 WHERE id = 1;

-- ✅ idempotent ถ้าผูกกับ transaction id ที่เจาะจง
UPDATE accounts SET balance = balance - 100
WHERE id = 1 AND NOT EXISTS (
    SELECT 1 FROM processed_transactions WHERE tx_id = 'tx-abc123'
);
INSERT INTO processed_transactions (tx_id) VALUES ('tx-abc123');
```

```sql
-- INSERT ธรรมดาไม่ idempotent (ยิงซ้ำ = แถวซ้ำ)
INSERT INTO short_urls (code, url) VALUES ('abc123', 'https://...');

-- UPSERT idempotent — ยิงซ้ำแล้วได้ผลเหมือนเดิม ไม่มีแถวซ้ำ
INSERT INTO short_urls (code, url) VALUES ('abc123', 'https://...')
ON CONFLICT (code) DO UPDATE SET url = EXCLUDED.url;
```

**เปลี่ยนจาก "สั่งทำ action" (increment, append) เป็น "ประกาศ state สุดท้ายที่ต้องการ" (set, upsert)** คือหัวใจของเทคนิคนี้

### 4.2 Idempotency Key — ใช้เมื่อ operation ไม่มีทาง idempotent โดยธรรมชาติ (เช่น "ตัดเงิน", "ส่ง job")

รูปแบบที่ **Stripe** ใช้จริง (แหล่งอ้างอิงมาตรฐานของวงการ):

1. Client สร้าง key ที่ไม่ซ้ำต่อ "ความตั้งใจ" หนึ่งครั้ง (แนะนำ UUID v4) ส่งมาใน header เช่น `Idempotency-Key: 6f8a...`
2. Server เช็คก่อนว่าเคยเห็น key นี้มาก่อนไหม (เก็บคู่ `(account_id, key) → (status_code, response_body)`)
3. **ถ้าเคยเห็นแล้ว** — ส่ง response เดิมกลับไปเป๊ะๆ (แม้ครั้งแรกจะ error ก็ตอบ error เดิม) **ไม่ทำงานซ้ำ**
4. **ถ้ายังไม่เคยเห็น** — ทำงานจริง บันทึกผลลัพธ์คู่กับ key แล้วค่อยตอบกลับ
5. เก็บ record ไว้มีระยะเวลาจำกัด (Stripe ใช้ 24 ชั่วโมง) ไม่ใช่ตลอดไป

```
Client                          Server
  │──POST /charge─────────────────▶│
  │  Idempotency-Key: abc123       │  ยังไม่เคยเห็น abc123 → ทำงานจริง หักเงิน
  │◀──200 OK {charged: true}───────│  บันทึก (abc123 → 200, {charged:true})
  │
  │  (network timeout, client ไม่รู้ว่าสำเร็จหรือเปล่า retry เอง)
  │
  │──POST /charge─────────────────▶│
  │  Idempotency-Key: abc123       │  เคยเห็น abc123 แล้ว → ไม่หักซ้ำ
  │◀──200 OK {charged: true}───────│  ส่ง response เดิมกลับไปเลย
```

**ตัวอย่างการ implement จริงด้วย Redis** (key หมดอายุเองไม่ต้องมีงานล้าง) อยู่ที่ [[Quarkus Redis]] ข้อ 5.4

### 4.3 Conditional update / Compare-and-Swap

```sql
-- claim งานแบบ idempotent: เปลี่ยนสถานะได้แค่ครั้งเดียวจาก PENDING เท่านั้น
UPDATE jobs SET status = 'PROCESSING', worker_id = 'w1'
WHERE id = 42 AND status = 'PENDING';
-- ยิงซ้ำกี่ครั้งก็ตาม แค่ครั้งแรกที่ status ยัง PENDING เท่านั้นที่เปลี่ยนสำเร็จ (0 แถวถูกอัปเดตในครั้งถัดไป)
```

ใช้ WHERE clause ผูกกับ**สถานะปัจจุบัน** แทนที่จะสั่ง "เปลี่ยนเป็น X" เฉยๆ — ทำให้การเปลี่ยนสถานะเกิดขึ้นได้จริงแค่ครั้งเดียวไม่ว่าจะยิงคำสั่งเดิมซ้ำกี่รอบ (pattern นี้เป็นแกนกลางของการ "claim" งานใน [[Background Job Processor (Quarkus)]])

### 4.4 DB unique constraint เป็นตัวการันตีสุดท้าย

แม้ใช้เทคนิคไหนก็ตาม **ควรมี unique constraint ที่ DB เป็นด่านสุดท้ายเสมอ** เผื่อ race condition หลุดผ่านชั้น application logic มาได้ (เช่นสอง request ที่มี idempotency key เดียวกันมาถึงพร้อมกันเป๊ะ) — DB constraint เป็นสิ่งเดียวที่ atomic จริงในระดับ storage

---

## 5. กับดักตอน implement เอง

- **Race condition แบบ check-then-act** — เช็คว่ามี key แล้ว *แล้วค่อย* insert ทีหลัง มีช่องให้ request สองตัวที่มี key เดียวกันมาถึงพร้อมกันผ่านการเช็คทั้งคู่ก่อนที่ตัวไหนจะ insert ทัน แก้ด้วยการทำให้ "เช็ค+จอง" เป็น atomic operation เดียว (unique constraint แล้ว insert ก่อนเลย ไม่ใช่ select ก่อน insert)
- **Idempotency key เดิม แต่ body ของ request ต่างกัน** — ควร reject (Stripe ทำแบบนี้) ไม่ใช่เงียบๆ ใช้ response เก่าทั้งที่ผู้ใช้ตั้งใจจะทำ operation ที่ต่างออกไป
- **ไม่ตั้ง TTL ให้ record ของ idempotency key** — เก็บถาวรทำให้ storage โตไม่จำกัด (ตั้ง TTL สั้นเกินไปก็แย่พอกัน เพราะ retry ที่มาช้ากว่า TTL จะถูกมองว่าเป็น request ใหม่ ประมวลผลซ้ำได้)
- **คิดว่า idempotent = atomic/transactional** — เป็นคนละเรื่องกัน operation ที่ idempotent ยังต้องมี proper locking/transaction ค้ำอยู่ดี ไม่งั้น concurrent request สองตัวยังชนกันได้ระหว่างทาง แม้ผลลัพธ์สุดท้ายจะ idempotent ก็ตาม
- **สมมติว่า `PATCH` idempotent เสมอเพราะ "มันเป็น HTTP method ที่ดูปลอดภัย"** — ขึ้นกับเนื้อหาจริงของ patch (ดูข้อ 3) `{"op": "increment"}` ไม่ idempotent ทั้งที่ส่งผ่าน `PATCH`
- **ลืมว่า idempotency key ต้องผูกกับ scope ที่ถูกต้อง** — key ซ้ำกันข้าม user/account คนละคนไม่ควรชนกัน (Stripe ผูกกับ `(account_id, key)` ไม่ใช่ `key` เดี่ยวๆ)

---

## Cheat sheet

| สถานการณ์ | เทคนิคที่ใช้ |
|---|---|
| อัปเดตค่าธรรมดา | เปลี่ยนจาก "สั่งทำ action" เป็น "set state สุดท้าย" (4.1) |
| insert ที่อาจซ้ำ | `UPSERT` / `ON CONFLICT DO UPDATE` (4.1) |
| operation ที่ไม่มีทาง idempotent เอง (ตัดเงิน, ส่ง job) | Idempotency key + เก็บ response ไว้ replay (4.2) |
| claim งานไม่ให้ทำซ้ำ | Conditional update ผูกกับสถานะปัจจุบัน (4.3) |
| ด่านสุดท้ายกัน race condition | Unique constraint ที่ DB (4.4) |

```sql
-- pattern ที่ใช้บ่อยที่สุด: claim ด้วย conditional update
UPDATE jobs SET status='PROCESSING' WHERE id=? AND status='PENDING';
```

---

## 🔗 เกี่ยวข้อง

- [[Background Job Processor (Quarkus)]] — โปรเจกต์ที่ต้องใช้หลักการนี้ตรงๆ (claim + retry ไม่ประมวลผลซ้ำ)
- [[Quarkus HTTP Methods]] — idempotency ในบริบท HTTP method โดยเฉพาะ พร้อมตัวอย่างโค้ด Quarkus
- [[Quarkus REST Client]] — ทำไม retry logic ต้องรู้ว่า operation ไหน idempotent ก่อน retry อัตโนมัติ
- [[Quarkus Redis]] — ตัวอย่าง implement idempotency key จริงด้วย Redis (ข้อ 5.4)
- [[Thai QR Payment]] — ตัวอย่างในโลกจริง: webhook แจ้งชำระเงินที่ถูกส่งซ้ำ (`retryFlag`) ต้อง claim ก่อนทำ (ข้อ 6.4)
- [[Design Patterns]] — หน้ารวม

## 📖 อ่านต่อ

- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [Stripe — Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency)
- [Stripe API Reference — Idempotent requests](https://docs.stripe.com/api/idempotent_requests)
- [Implementing Stripe-like Idempotency Keys in Postgres — brandur.org](https://brandur.org/idempotency-keys)
