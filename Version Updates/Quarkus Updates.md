---
tags:
  - quarkus
  - version-updates
  - changelog
type: reference
created: 2026-09-17
---

# 🆕 Quarkus — Version Updates ล่าสุด

> โน้ตนี้ติดตาม **"อะไรใหม่/อะไรถูกเลิกใช้"** ของ Quarkus แต่ละเวอร์ชัน ไม่ใช่โน้ตอธิบายฟีเจอร์เชิงลึก (ไปดู [[Quarkus]] สำหรับโน้ตแยกตามหัวข้อ) — Quarkus ออกเวอร์ชันถี่กว่า Angular มาก ไม่ใช่ทุก 6 เดือนตายตัว เช็กวันที่ด้านล่างก่อนเชื่อ 100% เสมอ

> **อัปเดตล่าสุด: 2026-09-17 — เวอร์ชันล่าสุดคือ 3.39.2 (8 ก.ย. 2026), LTS ล่าสุดคือ 3.33 (25 มี.ค. 2026)**

---

## 1. Quarkus นับเวอร์ชันยังไง — ต่างจาก Angular ตรงไหน

Quarkus ไม่ได้ออก major version ใหม่ตามรอบตายตัวแบบ Angular (17 → 18 → 19 → ...) — อยู่ในสาย **3.x** มาตลอดตั้งแต่ปี 2023 แล้วออก **minor version ถี่มาก** (3.9, 3.15, 3.20, 3.27, 3.33, 3.39, ...) โดยเลือกบาง minor version มาประกาศเป็น **LTS** (long-term support) เป็นระยะ ๆ

| ประเภท | เวอร์ชันปัจจุบัน | ใช้ตอนไหน |
|---|---|---|
| **LTS** | 3.33 (25 มี.ค. 2026 → รองรับถึง 25 มี.ค. 2027) | โปรเจกต์ production ที่อยากอัปเดตน้อยที่สุด |
| **Latest (community)** | 3.39.2 (8 ก.ย. 2026) | อยากได้ฟีเจอร์ใหม่สุด ยอมอัปเดตถี่กว่า |

**ถ้าไม่รู้จะเลือกอันไหน — เลือก LTS เสมอสำหรับ production** เหตุผลเดียวกับที่เลือกใช้ Java LTS (ดู [[Java]])

---

## 2. Quarkus 3.33 (LTS, มี.ค. 2026) — baseline ที่ควรใช้ตอนนี้

- **Cache หลาย backend พร้อมกันได้ในแอปเดียว** — เลือก backend ต่างกันต่อ cache แต่ละตัว เช่น Caffeine (in-memory, เร็ว) สำหรับข้อมูล hot + Redis (distributed) สำหรับข้อมูลที่ต้องแชร์ข้าม instance (ดู [[Redis]])
- **OIDC filter เลือก scope ได้ว่าจะรันกับ flow ไหน** — แยก filter สำหรับ bearer token flow ออกจาก authorization code flow ได้ชัดเจน (ดู [[Quarkus Keycloak]]) เดิมถ้าเปิดทั้งสอง flow พร้อมกัน filter จะรันปนกันคาดเดายาก
- Dev Services อัปเดต default image ของ Elasticsearch/OpenSearch เป็นเวอร์ชันใหม่กว่า

---

## 3. Quarkus 3.39 (ก.ย. 2026) — ล่าสุด (community stream)

เพิ่งออกไม่นาน รายละเอียดยังไม่สรุปละเอียดเท่า 3.33 — จุดที่เห็นชัดคือ **Camel Quarkus 3.39.0 ตามมาให้พร้อมกัน** (ตาม Camel 4.22.0) สำหรับใครที่ใช้ Apache Camel ร่วมกับ Quarkus

---

## 4. สิ่งที่ต้องระวังต่อเนื่อง — เปลี่ยนชื่อ extension/API บ่อย

Quarkus มีประวัติเปลี่ยนชื่อ extension/API บ่อยกว่าเฟรมเวิร์กอื่นทั่วไป — ที่เจอไปแล้วและเขียนไว้ใน [[Quarkus]] คือ `quarkus-resteasy-reactive` → `quarkus-rest` (3.9) แนวโน้มนี้ยังต่อเนื่อง **ก่อนก๊อปตัวอย่างจากเว็บมาใช้ ให้เช็กเสมอว่าตัวอย่างเขียนด้วยเวอร์ชันไหน**

---

## 5. กับดัก

- **อัปเดตแบบข้าม LTS ไปหลายเวอร์ชันทีเดียว** — Quarkus ออกถี่มาก ยิ่งค้างนานยิ่งมีของเปลี่ยนสะสมเยอะ ควรอัปเดตตามรอบ LTS ทุกครั้งที่มีใหม่ ไม่ปล่อยค้างข้ามหลายรอบ
- **ใช้ community stream (non-LTS) ใน production โดยไม่ตั้งใจ** — ได้ฟีเจอร์ใหม่เร็วแต่ support สั้นกว่า ต้องอัปเดตถี่กว่าที่วางแผนไว้
- **ก๊อปตัวอย่างจากเว็บที่เขียนด้วยชื่อ extension เก่า** — เช่น `quarkus-resteasy-reactive` ที่ปัจจุบันคือ `quarkus-rest` แล้ว (ข้อ 4) — build ไม่ผ่านทันที
- **เปิดทั้ง bearer flow และ authorization code flow พร้อมกันโดยไม่ scope filter** — ก่อน 3.33 filter รันปนกันทั้งสอง flow คาดเดาพฤติกรรมยาก (ข้อ 2)

---

## 6. Cheat sheet

```bash
./mvnw quarkus:info          # ดูเวอร์ชัน Quarkus + extension ที่ใช้อยู่ในโปรเจกต์
```

| ต้องการรู้ | ดูที่ |
|---|---|
| เวอร์ชันไหนเป็น LTS | [quarkus.io/releases](https://quarkus.io/releases/) |
| migration guide ข้ามเวอร์ชัน | [Quarkus Migration Guides (wiki)](https://github.com/quarkusio/quarkus/wiki) |

## 🔗 เกี่ยวข้อง

- [[Quarkus]] — หน้ารวมโน้ตเชิงลึก
- [[Quarkus Keycloak]] — ตัวอย่าง OIDC filter scoping ที่มาใน 3.33

## 📖 อ่านต่อ

- [Quarkus Releases](https://quarkus.io/releases/)
- [Red Hat build of Quarkus 3.33 — Learn what's new](https://docs.redhat.com/en/documentation/red_hat_build_of_quarkus/3.33/html/learn_whats_new_in_3.33/assembly_release-notes-quarkus)
