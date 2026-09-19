---
tags:
  - claude
  - ai
  - agent-engineering
type: reference
created: 2026-09-17
---

# 🏗️ Harness Design — ออกแบบระบบให้ agent ทำงานยาวๆ ได้จริง

> "Harness" คือระบบ/โครงสร้างที่ห่อ agent ไว้ (ไม่ใช่ตัวโมเดลเอง) — บทความนี้พูดถึงการออกแบบ harness สำหรับงานที่ agent ต้องทำงาน**ต่อเนื่องยาวๆ โดยไม่มีคนคอยดูตลอด** เช่น สร้างแอปทั้งแอปให้เสร็จเอง

---

## 1. ปัญหาหลัก 2 อย่างของ agent ที่ทำงานยาว

### Context เสื่อมลงเรื่อยๆ

โมเดลเริ่มคุมงานไม่อยู่เมื่อ context window เต็มขึ้นเรื่อยๆ บางกรณีถึงขั้น **"context anxiety"** — รีบสรุปงานให้จบก่อนเวลาอันควร เพราะรู้สึกว่าใกล้ถึง limit ที่ (คิดไปเอง) ว่ามี

### Agent มักชมงานตัวเองเกินจริง

agent มักจะ "มั่นใจและชมผลงานตัวเอง" แม้คุณภาพจะแค่กลางๆ โดยเฉพาะงานที่ประเมินแบบ subjective (ไม่มีคำตอบถูกผิดชัดเจนแบบเทสอัตโนมัติเช็กได้)

---

## 2. ทางแก้ — แยก "คนทำงาน" ออกจาก "คนตรวจงาน"

### Generator-Evaluator Architecture (แนวคิดคล้าย GAN)

**แยกการ "ผลิตงาน" ออกจากการ "ประเมินคุณภาพ" เป็นคนละ agent กัน** — ดีกว่าให้ agent ตัวเดียวกันประเมินงานตัวเอง เพราะ **evaluator ที่แยกต่างหากปรับให้ "เข้มงวด/ขี้สงสัย" ได้ง่ายกว่า** การพยายามฝึกให้ generator วิจารณ์ตัวเองอย่างตรงไปตรงมา (แนวคิดเดียวกับที่ [[Claude Code Auto Mode]] แยก classifier ออกจาก Claude ที่กำลังทำงาน)

### Multi-Agent Specialization

- **Planner** — แปลง prompt สั้นๆ ให้เป็น spec ละเอียด
- **Generator** — implement ฟีเจอร์จริง
- **Evaluator** — เทสการทำงานจริงผ่านการโต้ตอบแบบผู้ใช้จริง (ใช้ [[Playwright]] ขับเคลื่อน)

### Sprint Contracts

ก่อนเริ่ม implement generator กับ evaluator **ต่อรองกันก่อนว่า "เสร็จ" หน้าตาเป็นยังไง** — เชื่อมช่องว่างระหว่าง spec ระดับสูงกับ requirement ที่เทสได้จริง โดยไม่ล็อก implementation detail เร็วเกินไป

### Measurable Criteria

แปลงการตัดสินแบบ subjective ให้เป็นเกณฑ์ที่ให้คะแนนได้จริง — ตัวอย่างในบทความใช้ 4 มิติ: **design quality, originality, craft, functionality**

---

## 3. ข้อคิดสำคัญ

งานที่ท้าทายของ AI engineer **ไม่ได้ลดลงเมื่อโมเดลเก่งขึ้น** — กลายเป็นว่าต้องคอยหา "combination ใหม่ๆ" ของ pattern พวกนี้ไปเรื่อยๆ แทน

---

## 🔗 เกี่ยวข้อง

- [[Claude Code Auto Mode]] — แนวคิด evaluator แยกจากตัวที่ทำงาน เพื่อไม่ให้ agent "โน้มน้าวตัวเอง"
- [[Claude Code Best Practices]] — หัวข้อ "ให้ Claude มีทางตรวจสอบงานตัวเอง" คือหลักการเดียวกันแบบใช้งานจริงระดับ session เดียว
- [[Playwright]] — เครื่องมือที่ evaluator ในบทความนี้ใช้ทดสอบผ่านการโต้ตอบจริง

## 📖 อ่านต่อ

- [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)
