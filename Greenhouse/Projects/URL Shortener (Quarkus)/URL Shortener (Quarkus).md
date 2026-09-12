---
tags:
  - research
  - research/sprouting
  - java
  - quarkus
type: project
status: sprouting
area: backend
created: 2026-08-17
updated: 2026-09-12
---

# 🔗 URL Shortener (Java / Quarkus)

## 🎯 ปัญหา

อยากมีบริการย่อ URL ของตัวเอง แทนการพึ่ง bit.ly / tinyurl
เหตุผลที่คนทำเองมักจะเป็นข้อใดข้อหนึ่ง — **ควรเลือกให้ชัดว่าเป็นข้อไหน เพราะมันเปลี่ยน design ทั้งหมด**

1. ไม่อยากให้ link ภายในองค์กรวิ่งผ่าน third party (privacy / compliance)
2. อยากได้ domain ตัวเอง + custom alias
3. อยากได้ analytics ที่ควบคุมเอง
4. ~~แค่อยากลองทำ เพื่อเรียนรู้ Quarkus~~ **← เลือกข้อนี้ (ยืนยัน 2026-09-12)** ไม่ต้องแคร์ scale, ไม่ต้องมี auth, ไม่เปิด public

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

### ที่มา & หลักการพื้นฐาน (ทำไมย่อ URL แล้วยังเปิดเว็บได้)

**ที่มา:** จุดเริ่มต้นคือ **TinyURL** เปิดตัวมกราคม 2002 โดย Kevin Gilbertson เพื่อแก้ปัญหา URL ยาว ๆ ที่พังเวลาโพสต์ใน newsgroup (บรรทัดตัดคำ, ระบบ auto-link พังถ้า URL มี query string ยาวเกิน) เป็น shortener เจ้าแรกที่โด่งดังและยังเปิดใช้อยู่จนถึงตอนนี้

จุดเปลี่ยนที่ทำให้ URL shortener กลายเป็นของที่ใครก็ต้องมีคือ **Twitter** — เปิดตัวปี 2006 ด้วยข้อจำกัด 140 ตัวอักษร/ทวีต ถ้า URL ยาว 100 ตัวอักษรก็แทบไม่เหลือที่ให้เขียนอะไรอีกเลย **Bitly** เปิดตัวปี 2008 มาแก้ปัญหานี้โดยเฉพาะ พร้อมเพิ่ม click tracking/analytics ที่ TinyURL ไม่มี จนพฤษภาคม 2009 Twitter เปลี่ยนจาก TinyURL มาใช้ Bitly เป็น default shortener — นี่คือจุดที่ทำให้ URL shortener กลายเป็น infrastructure ระดับโลก (แรงจูงใจข้อ 3 "อยากได้ analytics ที่ควบคุมเอง" ในหัวข้อปัญหาด้านบน ก็มาจากจุดนี้พอดี)

**หลักการพื้นฐาน:** ความเข้าใจผิดที่พบบ่อยที่สุดคือคิดว่า short URL คือการ "บีบอัด" (compress) URL เดิมให้สั้นลงเหมือน zip — **ไม่ใช่** สิ่งที่เกิดขึ้นจริงคือการสร้าง **"ป้ายชี้ทาง" (indirection/pointer)** ตัวใหม่ที่ไม่มีความสัมพันธ์เชิงโครงสร้างกับ URL เดิมเลย แล้วเก็บคู่ (`code` → `original_url`) ไว้ในฐานข้อมูลของผู้ให้บริการ เทียบง่าย ๆ ได้กับเบอร์ฉุกเฉิน "191" — ตัวเลขไม่ได้เข้ารหัสเบอร์จริงของสถานีตำรวจไว้ในตัวมันเอง มันคือรหัสอ้างอิงที่ต้อง lookup กับสมุดของบริษัทโทรศัพท์ถึงจะรู้ปลายทางจริง

**ทำไมกดลิงก์สั้นแล้วยังไปเว็บจริงได้ — flow ที่เกิดขึ้นจริง (ยกตัวอย่าง `https://amzn.to/3xY2zAb`):**
1. Browser ยิง `GET` ไปที่ short URL
2. DNS พาไปเจอ server ของผู้ให้บริการ shortener (โดเมน `amzn.to` เป็นของ Amazon เอง)
3. Server lookup `3xY2zAb` ในฐานข้อมูล → เจอว่า map กับ `https://www.amazon.com/gp/product/B08N5WRWNW/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1`
4. Server **ไม่ได้ส่งหน้าเว็บกลับมา** แต่ตอบเป็น **HTTP redirect response** — status `302 Found` พร้อม header `Location: https://www.amazon.com/gp/product/...`
5. Browser เห็น status 3xx กับ header `Location` แล้ว**ยิง request ใหม่เองอัตโนมัติ**ไปที่ URL นั้น — เป็นพฤติกรรมมาตรฐานของ HTTP ตาม RFC 7231 ไม่มี JavaScript หรือกลไกพิเศษอะไรเลย
6. ผู้ใช้เห็นหน้าเว็บจริงของ Amazon โดยแทบไม่รู้ตัวว่าผ่าน redirect มา

ตรงนี้แหละคือเหตุผลที่ endpoint `GET /{code}` ในโปรเจกต์นี้ถูกเรียกว่า **hot path** — เพราะมันทำแค่ข้อ 3-4 ด้านบน (lookup + ตอบ redirect) เท่านั้น ไม่มี business logic หนักกว่านี้เลย

**ผลที่ตามมาที่คนมักไม่ทันคิด:** เพราะ mapping อยู่ในฐานข้อมูลของผู้ให้บริการที่เดียว **ไม่มีทางคำนวณย้อนจาก code กลับไปหา URL จริงได้เองถ้าไม่มีฐานข้อมูลนั้น** — ถ้าผู้ให้บริการปิดตัวหรือฐานข้อมูลหาย ลิงก์สั้นที่แปะกระจายอยู่ทั่วอินเทอร์เน็ตจะพังพร้อมกันทันที (เคยเกิดขึ้นจริงกับหลายเจ้าที่ปิดตัวไปแล้ว) เป็น single point of failure ที่แถมมาพร้อมความสะดวกของ indirection แบบนี้

### การสร้าง short code — 4 ทาง

| วิธี                                  | ข้อดี                                               | ข้อเสีย                                                                                                  |
| ------------------------------------- | --------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| **Random base62 + unique constraint** | เดาไม่ได้, ไม่ต้องมี state กลาง, scale แนวนอนได้เลย | ต้อง handle การชนแล้ว retry                                                                              |
| **Redis `INCR` + base62**             | ไม่มีวันชนเลย, atomic                               | code เรียงกัน → **เดา link คนอื่นได้ + คนนอกรู้ว่าเราสร้างไปกี่ link แล้ว**, Redis กลายเป็น single point |
| **Snowflake ID**                      | แต่ละ node สร้างเองได้ ไม่ต้องคุยกัน                | ID ยาว, ซับซ้อนเกินความจำเป็นที่ scale นี้                                                               |
| **Hash URL (MD5/SHA1) แล้วตัดสั้น**    | URL เดิมได้ code เดิมเสมอ (deterministic) → dedupe ในตัวโดยไม่ต้องมี state กลาง | ตัดสั้นเหลือ ~7 ตัวอักษรแล้ว collision เกิดง่ายกว่าที่คิด (birthday paradox) ต้องมี fallback อยู่ดี และผูก code เข้ากับเนื้อหา URL ตายตัว — **ขัดกับที่ตัดสินใจไว้แล้วว่าไม่ dedupe** (ด้านล่าง) |

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
POST /api/links     สร้าง (ไม่มี auth — ทำเล่น ไม่เปิด public)
GET  /{code}        redirect 302 ← hot path
GET  /{code}+       preview page ดูปลายทางก่อนกด
DELETE /api/links/{code}   (ไม่มี auth เช่นกัน — ใครมี code ลบได้)
```

**ตาราง `short_urls`** — `code` VARCHAR **primary key** (คือ short code ที่อยู่ใน URL เช่น `aB3xK9z` ในลิงก์ `.../aB3xK9z` ใช้ query ทุกครั้งที่ redirect/delete จึงเป็น PK ตรง ๆ ไม่ต้องมี id แยก), `original_url` VARCHAR(2048), `created_date`, `expires_at`, `deleted_at`
(ตัด `created_by` ออก — ไม่มี auth เลยไม่มี identity ให้ผูก)

**ตาราง `forbidden_words`** — `word` (primary key) เก็บคำสงวนไว้ใน DB (ตัดสินใจ 2026-09-12) แทน hardcode เพื่อเพิ่ม/ลบได้โดยไม่ต้อง redeploy — รายละเอียดการเช็คดูที่ [[API Flow Spec]]

**DB — เลือก SQLite** (ตัดสินใจ 2026-09-12) เหมาะกับสเกลของโปรเจกต์นี้ (ทำเล่น, single instance) ไฟล์เดียวจบ ไม่ต้องตั้ง DB server แยก

**Cache** — เริ่มด้วย `quarkus-cache` (Caffeine) ในเครื่องก่อน เพราะ traffic ของ shortener เป็น Zipf distribution แรงมาก (link ไม่กี่อันกินคลิกเกือบทั้งหมด) local cache เล็ก ๆ ก็ hit rate สูงแล้ว
ค่อยขยับไป Redis เมื่อมีหลาย instance แล้วต้องการให้การลบ link มีผลทันทีทุกเครื่อง (สำหรับ SQLite ยิ่งจำเป็น เพราะ SQLite ไม่เหมาะกับหลาย instance เขียนพร้อมกันอยู่แล้ว)

### ⚠️ กับดักเรื่อง dependency

ชื่อ extension ของ REST เปลี่ยนที่ Quarkus **3.9** — ก่อนหน้านั้นคือ `quarkus-resteasy-reactive` หลังจากนั้นเปลี่ยนเป็น `quarkus-rest`
ถ้าเริ่มโปรเจกต์ใหม่ด้วยเวอร์ชันล่าสุดก็ใช้ชื่อใหม่ได้เลย แต่ถ้าอ้างอิงตัวอย่างเก่า ๆ จะเจอชื่อเดิม **ต้องดูก่อนว่าตัวอย่างนั้นเขียนด้วยเวอร์ชันไหน**
ดู [[Quarkus REST Client]] ประกอบ

**SQLite ไม่ใช่ DB ที่ Quarkus รองรับเป็นทางการ** — ไม่มี `quarkus-jdbc-sqlite` จาก Red Hat ต้องพึ่ง extension ชุมชน (Quarkiverse) หรือประกอบเองด้วย `org.xerial:sqlite-jdbc` + `hibernate-community-dialects` (SQLiteDialect)
**จุดเสี่ยงที่สุดของโปรเจกต์นี้:** `sqlite-jdbc` ห่อ native library (JNI) ไว้ข้างใน ตอน build native image ด้วย GraalVM มีประวัติต้องเพิ่ม reflection/resource config เอง ไม่ใช่ plug-and-play เหมือน H2/Postgres — **ควรลองต่อ SQLite + native build ให้ได้ตั้งแต่การทดลองเล็กที่สุด** ก่อนเขียน business logic ต่อ เพราะถ้าตรงนี้ไม่ผ่าน สมมติฐานหลักของโปรเจกต์ (native image) ก็ไปต่อไม่ได้เลย

extension ที่ต้องใช้: REST + Jackson, `hibernate-orm-panache`, SQLite JDBC driver (ชุมชน) + `hibernate-community-dialects`, `cache`

---

## 🛠️ Spec: API Flow

**การสร้าง short code: ยืนยันแล้วว่าใช้ random base62 + unique constraint** (ตัดคำถามที่ค้างไว้ตั้งแต่ prior art ออกไป, ไม่ใช้ Redis counter / Snowflake)

รายละเอียด request/response/validate ของแต่ละ endpoint (`POST /api/links`, `GET /{code}`, `GET /{code}+`, `DELETE /api/links/{code}`) พร้อม sequence diagram แยกไว้ที่ [[API Flow Spec]]

---

## ✅ รู้ได้ยังไงว่าสำเร็จ

- redirect p99 latency **< 10ms** ตอน cache hit
- native binary กิน RSS **< 100MB** ตอน idle
- สร้าง link 10,000 อันติดกันโดยไม่ชนสักครั้ง
- ยิง 1,000 req/s ใส่ code เดิมแล้ว DB ไม่โดน query เพิ่ม (พิสูจน์ว่า cache ทำงาน)

## 🧪 การทดลองที่เล็กที่สุด

Panache entity 1 ตัว + endpoint 2 ตัว + SQLite + native build ให้ผ่านก่อน ไม่ต้องมี auth (ตัดสินใจแล้วว่าไม่ทำ) ไม่ต้องมี cache จริง ไม่ต้องมี analytics
**เป้าหมายคือได้ตัวเลข latency ของ native build จริง บน SQLite** ถ้า native build ไม่ผ่านเพราะ sqlite-jdbc หรือถ้าตัวเลขไม่ต่างจาก JVM mode อย่างมีนัยสำคัญ สมมติฐานหลักของโปรเจกต์นี้ก็ตกไป — ประมาณ 2-3 ชั่วโมง

## ⚠️ ความเสี่ยง

- **ไม่มี auth = ห้าม deploy ให้คนอื่นเข้าถึงได้** ถ้าจะเปิดสาธารณะเมื่อไหร่ ต้องกลับมาคิดเรื่อง auth ใหม่ก่อน (ไม่งั้นโดนใช้ทำฟิชชิ่ง → โดเมนติด blocklist, หรือใครก็ลบ link ของคนอื่นได้เพราะไม่เช็คความเป็นเจ้าของ)
- `sqlite-jdbc` + GraalVM native image — ยังไม่ยืนยันว่า build native ผ่านลื่น ๆ (ดู กับดักเรื่อง dependency ด้านบน)
- SQLite เขียนพร้อมกันจากหลาย connection ได้จำกัด (single-writer) — ไม่กระทบ redirect (read-only) แต่กระทบ throughput ตอนสร้าง/ลบ link จำนวนมากพร้อมกัน
- SecureRandom บน native image เคยมีปัญหาใน GraalVM รุ่นเก่า ต้องเช็คกับ toolchain ที่ใช้อยู่
- ถ้าทำเพื่อเรียนรู้ ระวังหลงไปทำ analytics dashboard จนลืมว่าแก่นคือ redirect

## 📚 ต้องไปเรียนรู้เพิ่ม

- `sqlite-jdbc` + GraalVM native image — ต้อง config reflection/resource อะไรเพิ่มบ้าง
- `hibernate-community-dialects` — ใช้ SQLiteDialect กับ Panache ยังไง
- `quarkus-cache` + Caffeine tuning
- pattern การ handle unique constraint violation แล้ว retry ใน Panache — ต่อยอดจาก [[Quarkus hibernate]]
- native image + SecureRandom

## 🔗 เกี่ยวข้องกับ

- [[API Flow Spec]] — spec ละเอียดของแต่ละ endpoint
- [[Quarkus hibernate]]
- [[Greenhouse]]

## 📝 Log

### 2026-08-17
- สร้างโน้ต, รวบรวม prior art
- ตัดสินใจเบื้องต้น: **random base62 + unique index** (ไม่ใช้ Redis counter), **302 ไม่ใช่ 301**
- ยังไม่ได้ตอบ: ทำไปเพื่ออะไรกันแน่ (ข้อ 1-4 ด้านบน) ← ต้องตอบก่อนขยับเป็น `sprouting`

### 2026-09-11
- เขียน spec flow API ละเอียดของทั้ง 4 endpoint (validate → business logic → response code) พร้อม sequence diagram
- ที่ยังไม่ปิด: กลไก auth, รายการคำสงวนแบบเต็ม, จำนวน retry สูงสุด, แยก error ลบ vs หมดอายุหรือไม่

### 2026-09-12
- ย้ายโน้ตเข้า folder `Greenhouse/Projects/URL Shortener (Quarkus)/` แยก spec ออกเป็นไฟล์ [[API Flow Spec]] ต่างหาก โน้ตนี้เหลือ prior art / design / risk
- **ยืนยันวิธีสร้าง short code: random base62 + unique constraint** (ปิดคำถามที่ค้างจาก prior art)
- **ตอบคำถามที่ค้างจาก 2026-08-17 ได้แล้ว: เลือกข้อ 4 — ทำเล่นเพื่อเรียนรู้ Quarkus** → ตัดสินใจไม่ทำ auth เลย, ไม่เปิด public
- **เลือก DB: SQLite** — สเกลเล็กพอ ไฟล์เดียวจบ แต่เพิ่มความเสี่ยงใหม่เรื่อง native image build (ดู กับดักเรื่อง dependency)
- ปิดคำถามที่เหลือทั้งหมดใน [[API Flow Spec]]: คำสงวนเก็บใน DB (ตาราง `forbidden_words`), retry สูงสุด **3 ครั้ง**, แยก deleted/expired ด้วย `reason` field (คง HTTP `410` เดิมทั้งคู่ตาม RFC 7231)
- เปลี่ยนชื่อตาราง: `link` → `short_urls`, `reserved_code` → `forbidden_words`
- เพิ่ม research เรื่องที่มา (TinyURL 2002, Bitly/Twitter 2008-2009) และหลักการพื้นฐาน (indirection + HTTP redirect ไม่ใช่การบีบอัด) พร้อมเพิ่มเทคนิคที่ 4 (hash-based) ในตารางเปรียบเทียบ
- สถานะขยับจาก `seed` → `sprouting`

## 🪦 ถ้าเลิกทำ

เลิกถ้า: ทดลองแล้วพบว่า native ไม่ได้เร็วกว่า JVM อย่างมีนัยสำคัญ **และ** ไม่มีความต้องการใช้จริง
จะกลับมาคิดใหม่เมื่อ: มี requirement จริงเรื่อง link ภายในองค์กรที่ห้ามวิ่งผ่าน third party

---

## 📖 แหล่งอ้างอิง

- [TinyURL — Wikipedia](https://en.wikipedia.org/wiki/TinyURL) — ที่มาและปีเปิดตัว
- [History of Link Shorteners: Evolution & Impact](https://blog.linko.me/history-of-link-shorteners/) — ไทม์ไลน์ TinyURL → Bitly → Twitter
- [Shrink That Link: Craft a Java URL Shortener with Quarkus](https://www.the-main-thread.com/p/java-quarkus-url-shortener-tutorial)
- [URL Shortening System Design](https://systemdesign.one/url-shortening-system-design/)
- [Design a URL Shortener Like Bitly — Hello Interview](https://www.hellointerview.com/learn/system-design/problem-breakdowns/bitly)
- [Hashing Strategies: Base62, Counters, and Collision Avoidance](https://wittycoder.in/courses/url-shortener/url-shortener-hashing)
- [Quarkus 3.9 — Big Reactive Rename](https://quarkus.io/blog/quarkus-3-9-1-released/)
