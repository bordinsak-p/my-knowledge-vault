---
tags:
  - keycloak
  - version-updates
  - changelog
type: reference
created: 2026-09-17
---

# 🆕 Keycloak — Version Updates ล่าสุด

> โน้ตนี้ติดตาม **"อะไรใหม่/อะไรถูกเลิกใช้"** ของ Keycloak แต่ละเวอร์ชัน ไม่ใช่โน้ตอธิบายแนวคิด (ไปดู [[Keycloak]] และ [[Quarkus Keycloak]]) เช็กวันที่ด้านล่างก่อนเชื่อ 100% เสมอ

> **อัปเดตล่าสุด: 2026-09-17 — เวอร์ชันล่าสุดคือสาย 26.7 (26.7.0 ออก 9 ก.ค. 2026, 26.7.3 patch ความปลอดภัย 31 ส.ค. 2026)**

---

## 1. Keycloak 26.7 (ก.ค. 2026) — ล่าสุด

- **SCIM API (preview)** — automate การสร้าง/ลบ/แก้ user จากระบบภายนอกด้วยมาตรฐาน SCIM แทนการเขียนสคริปต์เรียก Admin REST API เอง
- **Multi-cluster High Availability แบบไม่ต้องมี external cache (preview)** — ง่ายกว่าการตั้ง HA แบบเดิมที่ต้องพึ่ง cache cluster แยกต่างหาก (เช่น Infinispan)
- **Identity Brokering API ดึง token จริงจาก external IdP ได้** — แอปขอ token ตัวจริงที่ผู้ใช้ได้จาก provider ภายนอก (เช่น Google) ตอน login ผ่าน brokering ได้แล้ว ไม่ใช่แค่ token ของ Keycloak เอง
- **SAML step-up authentication** — บังคับยืนยันตัวตนเพิ่มเติมสำหรับ SAML flow ในบาง action ที่ sensitive กว่าปกติ
- คู่มือ reverse proxy เพิ่ม blueprint สำหรับ HAProxy/Traefik (นอกเหนือจาก nginx เดิม)

---

## 2. Keycloak 26.6 (ก่อนหน้า) — Deprecated

- **Fine-Grained Admin Permissions (FGAP) v1 → deprecated** จะถูกถอดออกในอนาคต ต้อง migrate ไป FGAP v2
- **`tracing-service-name`/`tracing-resource-attributes` → deprecated** เปลี่ยนไปใช้ `telemetry-service-name`/`telemetry-resource-attributes` แทน (ชื่อ config เปลี่ยนตามการรวม observability เข้ากับมาตรฐาน OpenTelemetry)
- **รัน Keycloak บน database ที่ไม่ใช่ UTF-8 charset → deprecated** เตรียม migrate database charset ถ้ายังไม่ได้ใช้ UTF-8
- method บางตัวใน `AuthenticationManager.AuthResult` deprecated เปลี่ยนไปใช้ `session()` accessor แทน

---

## 3. ผลกระทบต่อโน้ตอื่นในวอลต์นี้

| โน้ตเดิม | อ้างอิงเวอร์ชันไหน | ปรับปรุงยังไง |
|---|---|---|
| [[Quarkus Keycloak]] | ตัวอย่าง `docker-compose.yml` ใช้ image `26.5` | ยังใช้ได้ปกติ (breaking change ไม่กระทบตัวอย่างพื้นฐาน) แต่ถ้าจะ deploy จริงควรขยับเป็น 26.7.x ล่าสุดแทน โดยเฉพาะเพื่อได้ security patch |

---

## 4. กับดัก

- **ยังใช้ FGAP v1 อยู่โดยไม่รู้ว่า deprecated แล้ว** — จะถูกถอดในอนาคต ควรเริ่มวางแผน migrate ไป v2 ตั้งแต่ตอนนี้ถ้าใช้ authorization services แบบ fine-grained อยู่
- **ใช้ image เวอร์ชันเก่ามาก (เช่น 26.5) แล้วไม่เคยอัปเดต** — พลาด security patch สะสม (26.7.3 เป็น security update ล้วน แก้ 20 ช่องโหว่)
- **ไม่รู้ว่า `tracing-*` config ถูกเปลี่ยนชื่อ** — config เก่ายังทำงานได้ตอนนี้ (deprecated ≠ removed ทันที) แต่ควรย้ายไป `telemetry-*` ก่อนเวอร์ชันที่ถอดออกจริงมาถึง

---

## 5. Cheat sheet

```bash
docker run quay.io/keycloak/keycloak:26.7 --version   # เช็กเวอร์ชัน image
```

| ต้องการรู้ | ดูที่ |
|---|---|
| release note ทุกเวอร์ชัน | [keycloak.org — Release Notes](https://www.keycloak.org/docs/latest/release_notes/index.html) |
| EOL/support timeline | [endoflife.date/keycloak](https://endoflife.date/keycloak) |

## 🔗 เกี่ยวข้อง

- [[Keycloak]] — Keycloak คืออะไร, protocol เบื้องหลัง
- [[Quarkus Keycloak]] — ตัวอย่าง implement เต็มรูปแบบ (ใช้ image เวอร์ชันเก่ากว่านี้เล็กน้อย)

## 📖 อ่านต่อ

- [Keycloak 26.7.0 released](https://www.keycloak.org/2026/07/keycloak-2670-released)
- [Keycloak — Release Notes](https://www.keycloak.org/docs/latest/release_notes/index.html)
