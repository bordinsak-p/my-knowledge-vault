---
tags:
  - claude
  - ai
  - agent-engineering
type: reference
created: 2026-09-17
---

# 🔧 Writing Effective Tools — ออกแบบ tool ให้ agent ใช้ได้ดี

> **"Tool คือสัญญาระหว่างระบบที่ deterministic กับ agent ที่ non-deterministic"** — การออกแบบ tool ให้ agent ใช้ ต้องคิดต่างจากการออกแบบ API ทั่วไปที่คนเขียนโค้ดเรียกตรงๆ

---

## 1. ออกแบบอย่างตั้งใจ (Intentional Design)

- **โฟกัส workflow ที่มี impact สูง** ไม่ใช่ห่อทุก endpoint ที่มีให้ครบ
- **รวมหลาย operation เข้า tool เดียว** — เช่น `schedule_event` ที่หาเวลาว่างให้เองในตัว ดีกว่าแยกเป็น tool ค้นหา + tool สร้าง คนละตัว
- ให้ agent **"แบ่งงานแล้วแก้ปัญหา"** ได้เหมือนที่คนจะทำถ้ามี resource เดียวกัน

## 2. ประหยัด context

- คืนแค่ข้อมูลที่ "signal สูง" — **ความเกี่ยวข้องกับบริบทสำคัญกว่าความยืดหยุ่น**
- ใช้ชื่อที่มีความหมาย (semantic identifier) แทน UUID ที่อ่านไม่รู้เรื่อง
- ทำ pagination/filter/truncate ให้มี default ที่สมเหตุสมผล
- ตัด field ทางเทคนิคที่ไม่จำเป็น (เช่น MIME type) ออก

## 3. ขอบเขตชัดเจน

- ใช้ namespace (prefix/suffix) แยกจุดประสงค์ของแต่ละ tool ให้ชัด
- แต่ละ tool ต้องมีจุดประสงค์เดียวที่ชัดเจน — **tool ที่ทำหน้าที่ซ้อนทับกันทำให้ agent สับสน**
- หลีกเลี่ยงการมี tool ซ้ำซ้อนกันเป็นสิบตัว

## 4. เอกสารประกอบที่ดี

- เขียน description เหมือนอธิบายให้ **เพื่อนร่วมทีมคนใหม่** เข้าใจ
- ทำ context ที่โดยนัย (implicit) ให้ชัดเจนขึ้นมาตรงๆ
- ตั้งชื่อ parameter ไม่กำกวม (`user_id` ไม่ใช่แค่ `user`)
- ให้ตัวอย่างและรูปแบบ input/output ที่คาดหวังชัดเจน

---

## 5. หน้าตา tool ที่ไม่ดี

Tool ที่แค่ห่อ functionality เดิมโดยไม่คิดถึงข้อจำกัดของ agent — เปลือง context ไปกับข้อมูลที่ไม่เกี่ยวข้อง, ไม่มีจุดประสงค์ชัดเจน, หรือคืนผลลัพธ์ยืดยาวพร้อม metadata ที่ไม่จำเป็น

---

## 🔗 เกี่ยวข้อง

- [[Claude MCP Code Execution]] — ปัญหา context ที่การออกแบบ tool ดีๆ ช่วยลดได้ตั้งแต่ต้นทาง
- [[Playwright MCP]] — ตัวอย่าง MCP server จริงที่หลักการออกแบบ tool พวกนี้นำไปใช้ได้

## 📖 อ่านต่อ

- [Writing effective tools for agents — with agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
