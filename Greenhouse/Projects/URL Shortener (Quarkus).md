---
tags:
  - research
  - research/seed
  - java
  - quarkus
type: project
status: seed
area: backend
created: 2026-08-17
updated: 2026-08-17
---

# 🔗 URL Shortener (Java / Quarkus)

## 🎯 ปัญหา

อยากมีบริการย่อ URL ของตัวเอง แทนการพึ่ง bit.ly / tinyurl
เหตุผลที่คนทำเองมักจะเป็นข้อใดข้อหนึ่ง — **ควรเลือกให้ชัดว่าเป็นข้อไหน เพราะมันเปลี่ยน design ทั้งหมด**

1. ไม่อยากให้ link ภายในองค์กรวิ่งผ่าน third party (privacy / compliance)
2. อยากได้ domain ตัวเอง + custom alias
3. อยากได้ analytics ที่ควบคุมเอง
4. แค่อยากลองทำ เพื่อเรียนรู้ Quarkus (**ถ้าเป็นข้อนี้ ไม่ต้องแคร์ scale เลย**)

## 💭 สมมติฐาน

Quarkus + native image เหมาะกับงานนี้ผิดปกติ เพราะ workload มันคือ "รับ request → lookup 1 ครั้ง → ตอบ 302"
ไม่มี business logic หนัก แต่ต้อง **latency ต่ำและ memory ต่ำ** ซึ่งตรงกับจุดแข็งของ native image พอดี
เป็นโปรเจกต์ที่เล็กพอจะจบได้ แต่แตะครบทั้ง REST / DB / cache / native build

## ❓ ทำไมต้องตอนนี้

- เป็นโจทย์ที่คุ้นเครื่องมืออยู่แล้ว (Quarkus, DB เชิงสัมพันธ์, cache) — ดู [[Quarkus Hibernate]]
- ขอบเขตเล็กพอจะทำจบได้จริง แต่แตะครบทั้ง REST / DB / cache / native build
- เริ่มจากศูนย์ ไม่ติดข้อจำกัดของโค้ดเบสเดิม จะลองอะไรก็ได้

---

## 🔍 Prior art — สรุปที่หาได้

### การสร้าง short code — 3 ทาง

| วิธี | ข้อดี | ข้อเสีย |
|---|---|---|
| **Random base62 + unique constraint** | เดาไม่ได้, ไม่ต้องมี state กลาง, scale แนวนอนได้เลย | ต้อง handle การชนแล้ว retry |
| **Redis `INCR` + base62** | ไม่มีวันชนเลย, atomic | code เรียงกัน → **เดา link คนอื่นได้ + คนนอกรู้ว่าเราสร้างไปกี่ link แล้ว**, Redis กลายเป็น single point |
| **Snowflake ID** | แต่ละ node สร้างเองได้ ไม่ต้องคุยกัน | ID ยาว, ซับซ้อนเกินความจำเป็นที่ scale นี้ |

**ข้อสรุปที่ได้จากการคำนวณ:** base62 7 ตัว = 62⁷ ≈ **3.5 ล้านล้าน** ความเป็นไปได้
ถ้ามี link อยู่ 1 ล้านอัน โอกาสชนต่อการ insert หนึ่งครั้ง ≈ 1M / 3.5e12 ≈ **0.00003%**

→ **random + unique index + retry ก็พอแล้ว** ความซับซ้อนของ counter/Snowflake ไม่คุ้ม
(หลาย ๆ บทความเชียร์ Redis counter เพราะมองในมุม system design interview ที่สมมติ scale ระดับ Bitly ซึ่งไม่ใช่เคสนี้)

### เรื่องที่สำคัญที่สุดแต่คนมักพลาด: **status code**

| code | ผลที่ตามมา |
|---|---|
| **301** Moved Permanently | browser **cache ถาวร** → แก้ปลายทางไม่ได้อีกเลย, ลบ link แล้วคนเดิมยังเข้าได้, นับคลิกไม่ได้ |
| **302** Found | browser ถามใหม่ทุกครั้ง → แก้/ลบ/นับคลิกได้ |

**ต้องใช้ 302** (หรือ 307) — เร็วกว่านิดหน่อยไม่คุ้มกับการเสียความสามารถในการแก้ link
อันนี้เป็นการตัดสินใจที่ย้อนกลับยากที่สุดในโปรเจกต์ เพราะ 301 ที่ถูก cache ไปแล้วเรียกคืนไม่ได้

### สิ่งที่พบว่าคนทำ URL shortener มักลืม

- **Open redirect** — ต้อง whitelist scheme เฉพาะ `http`/`https` ปิด `javascript:` `data:` `file:`
- **โดน block ทั้งโดเมน** — ถ้าใครเอาไปย่อ link ฟิชชิ่ง โดเมนเราติด blocklist ทั้งอัน กระทบ link ที่เหลือทั้งหมด → ต้องมี auth ตอนสร้าง ไม่เปิดสาธารณะ
- **ชนกับ route ตัวเอง** — ต้องกันคำสงวน `api`, `admin`, `health`, `q` ไม่ให้ถูกสุ่มเป็น code
- **case-sensitive** — base62 แยก `aB` กับ `Ab` ถ้า link จะถูกพิมพ์ลงกระดาษหรือบอกกันทางโทรศัพท์ ควรใช้ **base32 (Crockford)** ที่ตัด `I O U 1 0` ทิ้งแทน สั้นกว่านิดหน่อยแต่ไม่ผิดพลาด
- **ลบแล้วตอบอะไร** — `410 Gone` (เคยมี แต่ถูกลบ) สื่อความหมายกว่า `404`
- **นับคลิกบน hot path** — ห้าม `UPDATE ... SET clicks = clicks + 1` แบบ synchronous จะกลายเป็น bottleneck ทันที → ยิง event เข้าคิว (message broker หรือ Redis) แล้วค่อยประมวลผลทีหลัง ดู [[Quarkus Redis]]
- **dedupe หรือไม่** — URL เดิมควรได้ code เดิมไหม? ประหยัดพื้นที่ แต่ **พังทันทีถ้าอยากได้ analytics แยกต่อ link หรือ expiry ต่างกัน** → แนะนำไม่ dedupe

---

## 🧱 Design ที่เสนอ

```
POST /api/links     สร้าง (ต้อง auth)
GET  /{code}        redirect 302 ← hot path
GET  /{code}+       preview page ดูปลายทางก่อนกด
DELETE /api/links/{code}
```

**ตาราง** — `code` (unique index), `original_url` VARCHAR(2048), `created_by`, `created_date`, `expires_at`, `deleted_at`

**Cache** — เริ่มด้วย `quarkus-cache` (Caffeine) ในเครื่องก่อน เพราะ traffic ของ shortener เป็น Zipf distribution แรงมาก (link ไม่กี่อันกินคลิกเกือบทั้งหมด) local cache เล็ก ๆ ก็ hit rate สูงแล้ว
ค่อยขยับไป Redis เมื่อมีหลาย instance แล้วต้องการให้การลบ link มีผลทันทีทุกเครื่อง

### ⚠️ กับดักเรื่อง dependency

ชื่อ extension ของ REST เปลี่ยนที่ Quarkus **3.9** — ก่อนหน้านั้นคือ `quarkus-resteasy-reactive` หลังจากนั้นเปลี่ยนเป็น `quarkus-rest`
ถ้าเริ่มโปรเจกต์ใหม่ด้วยเวอร์ชันล่าสุดก็ใช้ชื่อใหม่ได้เลย แต่ถ้าอ้างอิงตัวอย่างเก่า ๆ จะเจอชื่อเดิม **ต้องดูก่อนว่าตัวอย่างนั้นเขียนด้วยเวอร์ชันไหน**
ดู [[Quarkus REST Client]] ประกอบ

extension ที่ต้องใช้: REST + Jackson, `hibernate-orm-panache`, JDBC driver ของ DB ที่เลือก, `cache`

---

## ✅ รู้ได้ยังไงว่าสำเร็จ

- redirect p99 latency **< 10ms** ตอน cache hit
- native binary กิน RSS **< 100MB** ตอน idle
- สร้าง link 10,000 อันติดกันโดยไม่ชนสักครั้ง
- ยิง 1,000 req/s ใส่ code เดิมแล้ว DB ไม่โดน query เพิ่ม (พิสูจน์ว่า cache ทำงาน)

## 🧪 การทดลองที่เล็กที่สุด

Panache entity 1 ตัว + endpoint 2 ตัว + `ConcurrentHashMap` แทน cache จริง ยังไม่ต้องมี auth ไม่ต้องมี analytics
**เป้าหมายคือได้ตัวเลข latency ของ native build จริง** ถ้าตัวเลขไม่ต่างจาก JVM mode อย่างมีนัยสำคัญ สมมติฐานหลักของโปรเจกต์นี้ก็ตกไป — ประมาณ 2-3 ชั่วโมง

## ⚠️ ความเสี่ยง

- ถ้าเปิดสาธารณะโดยไม่มี auth → โดนใช้ทำฟิชชิ่ง → โดเมนติด blocklist
- SecureRandom บน native image เคยมีปัญหาใน GraalVM รุ่นเก่า ต้องเช็คกับ toolchain ที่ใช้อยู่
- ถ้าทำเพื่อเรียนรู้ ระวังหลงไปทำ analytics dashboard จนลืมว่าแก่นคือ redirect

## 📚 ต้องไปเรียนรู้เพิ่ม

- `quarkus-cache` + Caffeine tuning
- pattern การ handle unique constraint violation แล้ว retry ใน Panache — ต่อยอดจาก [[Quarkus hibernate]]
- native image + SecureRandom

## 🔗 เกี่ยวข้องกับ

- [[Quarkus hibernate]]
- [[Greenhouse]]

## 📝 Log

### 2026-08-17
- สร้างโน้ต, รวบรวม prior art
- ตัดสินใจเบื้องต้น: **random base62 + unique index** (ไม่ใช้ Redis counter), **302 ไม่ใช่ 301**
- ยังไม่ได้ตอบ: ทำไปเพื่ออะไรกันแน่ (ข้อ 1-4 ด้านบน) ← ต้องตอบก่อนขยับเป็น `sprouting`

## 🪦 ถ้าเลิกทำ

เลิกถ้า: ทดลองแล้วพบว่า native ไม่ได้เร็วกว่า JVM อย่างมีนัยสำคัญ **และ** ไม่มีความต้องการใช้จริง
จะกลับมาคิดใหม่เมื่อ: มี requirement จริงเรื่อง link ภายในองค์กรที่ห้ามวิ่งผ่าน third party

---

## 📖 แหล่งอ้างอิง

- [Shrink That Link: Craft a Java URL Shortener with Quarkus](https://www.the-main-thread.com/p/java-quarkus-url-shortener-tutorial)
- [URL Shortening System Design](https://systemdesign.one/url-shortening-system-design/)
- [Design a URL Shortener Like Bitly — Hello Interview](https://www.hellointerview.com/learn/system-design/problem-breakdowns/bitly)
- [Hashing Strategies: Base62, Counters, and Collision Avoidance](https://wittycoder.in/courses/url-shortener/url-shortener-hashing)
- [Quarkus 3.9 — Big Reactive Rename](https://quarkus.io/blog/quarkus-3-9-1-released/)
