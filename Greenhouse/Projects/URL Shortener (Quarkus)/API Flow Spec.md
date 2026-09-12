---
tags:
  - spec
  - java
  - quarkus
type: spec
status: ready
parent: "[[URL Shortener (Quarkus)]]"
created: 2026-09-11
updated: 2026-09-12
---

# 🛠️ API Flow Spec — URL Shortener

ต่อยอดจาก decision ใน [[URL Shortener (Quarkus)]]: **random base62 + unique constraint** (ยืนยันแล้ว 2026-09-12, ไม่ใช้ Redis counter), **302 ไม่ใช่ 301**, ไม่ dedupe, **ไม่มี auth** (ทำเล่น ไม่เปิด public), **DB = SQLite**

## 📋 Data Model

### ตาราง `short_urls`

| column         | type            | null ได้ไหม | หมายเหตุ                                                                                                 |
| -------------- | --------------- | ----------- | -------------------------------------------------------------------------------------------------------- |
| `code`         | `VARCHAR(7)`    | ❌ (PK)      | short code — random base62 หรือ `custom_alias` ที่ผู้ใช้ตั้งเอง เป็น **primary key** โดยตรง ไม่มี id แยก |
| `original_url` | `VARCHAR(2048)` | ❌           | ปลายทางที่จะ redirect ไป ผ่าน validate scheme (`http`/`https`) มาแล้ว                                    |
| `created_date` | `TIMESTAMP`     | ❌           | เวลาที่สร้าง, ตั้งจาก server ตอน insert                                                                  |
| `expires_at`   | `TIMESTAMP`     | ✅           | ไม่ใส่ = ไม่หมดอายุ ถ้าใส่ต้องเป็นเวลาอนาคต ณ ตอนสร้าง                                                   |
| `deleted_at`   | `TIMESTAMP`     | ✅           | null = ยังใช้งานอยู่, ไม่ null = ถูกลบแล้ว (soft delete)                                                 |

ไม่มี `created_by` (ไม่มี auth เลยไม่มี identity ให้ผูก) ไม่มี `id` แยกจาก `code`

### ตาราง `forbidden_words`

| column | type | null ได้ไหม | หมายเหตุ |
|---|---|---|---|
| `word` | `VARCHAR` | ❌ (PK) | คำที่ห้ามใช้เป็น `custom_alias` เช่น `api`, `admin`, `health`, `q` — seed ไว้ตอน migration แรก เพิ่ม/ลบทีหลังได้โดยไม่ redeploy |

---

### `POST /api/links` — สร้าง short link (ไม่มี auth)

**Request**
```json
{
  "original_url": "https://example.com/very/long/path",
  "custom_alias": "my-link",      // optional
  "expires_at": "2026-12-31T00:00:00Z"  // optional
}
```

**ขั้นตอน validate (ก่อนแตะ DB)**
1. `original_url` ไม่ว่าง และยาว ≤ 2048 ตัวอักษร
2. scheme ต้องเป็น `http`/`https` เท่านั้น — ปฏิเสธ `javascript:` `data:` `file:` (ป้องกัน open redirect)
3. ถ้ามี `custom_alias`: เช็ค charset ให้ตรงชุดที่เลือก (base62), query ตาราง `forbidden_words` ว่าไม่ตรงกับคำสงวน
4. ถ้ามี `expires_at`: ต้องเป็นเวลาที่ยังไม่ผ่านไป

**คำสงวน — เก็บใน DB** (ตัดสินใจ 2026-09-12) ตาราง `forbidden_words(word VARCHAR PRIMARY KEY)` seed ค่าเริ่มต้นไว้ (`api`, `admin`, `health`, `q`, …) เพิ่ม/ลบได้โดยไม่ต้อง redeploy
เช็คเฉพาะตอน validate `custom_alias` เท่านั้น (ไม่ใช่ hot path ไม่ต้อง cache) — โค้ดสุ่มไม่ต้องเช็คตารางนี้ เพราะความยาว fix ไว้ที่ 7 ตัวอักษรอยู่แล้ว ยาวกว่าคำสงวนทุกคำ ชนกันไม่ได้อยู่แล้วโดยโครงสร้าง

**business logic**
1. มี `custom_alias` → insert ตรง ๆ, ให้ unique constraint ของ DB เป็นตัวกันชนซ้ำ
2. ไม่มี `custom_alias` → generate random base62 code แล้ว insert, ถ้าชน unique constraint หรือ SQLite `SQLITE_BUSY` ให้ retry **สูงสุด 3 ครั้ง** (ตัดสินใจ 2026-09-12) เกินนั้นถือว่า fail
3. บันทึก `created_date = now`, `deleted_at = null` (ไม่มี `created_by` เพราะไม่มี auth ไม่มี identity ให้ผูก)
4. ไม่ query หา `original_url` เดิมเพื่อ dedupe (ตามที่สรุปไว้แล้วในโน้ตหลัก)
5. บน SQLite: insert แต่ละครั้งแย่งไฟล์ lock กับ writer อื่น (single-writer) — ไม่กระทบตอนใช้งานคนเดียว แต่ถ้าทดสอบสร้าง link รัว ๆ พร้อมกันหลาย thread จะเห็นการ serialize/`SQLITE_BUSY` ได้ (นับรวมอยู่ใน retry 3 ครั้งด้านบน)

**Response**
| status | เมื่อไหร่ |
|---|---|
| `201` | สำเร็จ → `{ "code", "short_url", "expires_at" }` |
| `400` | url ไม่ผ่าน validate / scheme ไม่ผ่าน / alias ผิด format |
| `409` | `custom_alias` ชนของเดิม หรือชนคำสงวน |
| `500` | retry ครบ 3 ครั้งแล้วยังไม่สำเร็จ (ชน unique constraint ต่อเนื่อง หรือ SQLite ยัง busy) |

```mermaid
sequenceDiagram
    participant C as Client
    participant API as POST /api/links
    participant DB as DB

    C->>API: original_url, custom_alias?, expires_at?
    API->>API: validate (scheme, length)
    alt มี custom_alias
        API->>DB: query forbidden_words
        DB-->>API: 409 ถ้าตรงคำสงวน
        API->>DB: insert พร้อม alias
        DB-->>API: 409 ถ้าชน
    else ไม่มี custom_alias
        loop retry สูงสุด 3 ครั้ง
            API->>API: generate random base62 code
            API->>DB: insert
        end
    end
    API-->>C: 201 { code, short_url }
```

### `GET /{code}` — redirect (hot path, ไม่ auth)

1. route คำสงวน (`/api/*`, `/health`, …) ต้องกันไว้ก่อนตกมาถึง handler นี้ ไม่ใช่เช็คทีหลัง
2. lookup cache (Caffeine) → hit ข้ามไปข้อ 4
3. cache miss → query DB ด้วย `code`
   - ไม่เจอแถวเลย → `404`
   - เจอแต่ `deleted_at` ไม่ null → `410 Gone` body `{ "reason": "deleted" }`
   - เจอแต่ `expires_at` ผ่านไปแล้ว → `410 Gone` body `{ "reason": "expired", "expired_at": ... }`
   - เจอและยังใช้ได้ → populate cache แล้วไปข้อ 4
4. ยิง click event เข้าคิว **แบบ async / fire-and-forget** (ห้าม synchronous UPDATE บน hot path) ดู [[Quarkus Redis]]
5. ตอบ `302` พร้อม header `Location: original_url`

**เรื่อง deleted vs expired (ตัดสินใจ 2026-09-12):** แยกกันแน่นอน แต่ทั้งคู่ยังใช้ HTTP status `410` เหมือนเดิม เพราะทั้งสองเคสเข้าเงื่อนไข RFC 7231 ของ 410 พอดี (เคยมี, ไม่กลับมาแล้วแน่นอน) — สิ่งที่แยกจริง ๆ คือ **response body** (`reason` field) ให้ client เอาไปโชว์ข้อความที่ต่างกันได้ ("ลิงก์นี้ถูกลบไปแล้ว" vs "ลิงก์นี้หมดอายุแล้ว") โดยไม่ต้องเสีย semantic ของ HTTP status ไป

```mermaid
sequenceDiagram
    participant C as Client
    participant R as GET /{code}
    participant Cache as Caffeine
    participant DB as DB
    participant Q as Queue

    C->>R: GET /{code}
    R->>Cache: lookup
    alt cache hit
        Cache-->>R: original_url
    else cache miss
        R->>DB: SELECT by code
        DB-->>R: row หรือไม่เจอ
        R->>Cache: populate
    end
    R->>Q: emit click event (async)
    R-->>C: 302 Location: original_url
```

### `GET /{code}+` — preview page (ไม่ auth)

flow เหมือน redirect ทุกขั้น (reserved word → cache → DB → 404/410 พร้อมแยก `reason: deleted/expired`) แต่ต่างตรงที่:
- ตอบ `200` + HTML page แสดง `original_url` ให้ผู้ใช้เห็นก่อนกด แทนการ redirect ทันที
- ไม่นับเป็น click จริง (หรือถ้าจะนับ ให้แยก event type เป็น `preview` ไม่ปนกับ `click`)

### `DELETE /api/links/{code}` — ลบ (ไม่มี auth — ใครรู้ code ลบได้)

1. หาแถวด้วย `code` → ไม่เจอ → `404`
2. เจอแต่ `deleted_at` ไม่ null อยู่แล้ว → `410` body `{ "reason": "deleted" }` (idempotent)
3. soft delete: `UPDATE ... SET deleted_at = now` (ไม่ hard delete เพื่อให้ preview/redirect ตอบ `410` แทน `404` ได้)
4. invalidate cache entry ทันที (local cache ก่อน, ต้องคิดเรื่อง multi-instance ตอนย้ายไป Redis)

**Response:** `204` สำเร็จ, `404` ไม่เจอ, `410` ถูกลบไปแล้ว (ถึงแม้ link จะหมดอายุด้วยก็ตาม ยัง soft-delete สำเร็จได้ปกติ — DELETE เช็คแค่ `deleted_at` ไม่สนใจ `expires_at`)

**⚠️ trade-off ที่รับรู้แล้ว:** ไม่มี auth แปลว่าไม่มีการเช็คความเป็นเจ้าของ ใครก็ตามที่รู้ `code` เรียก `DELETE` ได้เลย รับได้เพราะเป็นโปรเจกต์ทดลองส่วนตัวไม่เปิด public — **ถ้าจะเปิดให้คนอื่นใช้เมื่อไหร่ ต้องกลับมาเพิ่ม auth + ownership check ก่อน**

### ✅ ปิดครบแล้ว (2026-09-12)

ทุกจุดที่เคยค้างใน spec นี้ตัดสินใจครบแล้ว: คำสงวนเก็บใน DB (`forbidden_words`), retry สูงสุด 3 ครั้ง, แยก deleted/expired ผ่าน `reason` field — พร้อมเริ่ม implement ได้

## 🔗 เกี่ยวข้องกับ

- [[URL Shortener (Quarkus)]] — โน้ตหลัก (prior art, design, risk)
- [[Quarkus Redis]]
