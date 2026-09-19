---
tags:
  - claude
  - ai
  - agent-engineering
type: reference
created: 2026-09-17
---

# 🎓 Agent Skills — แปลง Claude ทั่วไปให้เป็นผู้เชี่ยวชาญเฉพาะทาง

> Skills คือ **"วิธีสร้าง agent เฉพาะทางด้วยไฟล์และโฟลเดอร์"** — Claude เก่งแบบทั่วไป แต่งานจริงมักต้องการความรู้เฉพาะโดเมน/บริบทองค์กรที่โมเดล general-purpose ไม่มี Skills คือการห่อ "ความรู้เชิงกระบวนการ" ให้เป็นชิ้นที่ใช้ซ้ำได้ — เหมือนทำ **"คู่มือ onboarding ให้พนักงานใหม่"**

---

## 1. โครงสร้างเทคนิค — ไฟล์เดียวเป็นแกนกลาง

ทุก skill มี `SKILL.md` เป็นแกน ประกอบด้วย YAML metadata (`name`, `description`) บวกคำแนะนำละเอียด

```yaml
---
name: api-conventions
description: REST API design conventions for our services
---
# API Conventions
- ใช้ kebab-case สำหรับ URL path
- ใช้ camelCase สำหรับ JSON property
- ...
```

---

## 2. Progressive Disclosure — โหลดความรู้เป็นชั้นๆ

นี่คือกลไกที่ทำให้ skill ไม่ถ่วง context ทุก session ทั้งที่มีเนื้อหาข้างในเยอะแค่ไหนก็ได้:

| ชั้น | โหลดตอนไหน |
|---|---|
| **Metadata** | ชื่อ+description โหลดเข้า system prompt ล่วงหน้า ใช้แค่ตัดสินใจว่า "ควรใช้ skill นี้ไหม" |
| **Core** | เนื้อหาเต็มของ `SKILL.md` โหลดเฉพาะตอน Claude ตัดสินใจว่าเกี่ยวข้องจริง |
| **Extended** | ไฟล์/โค้ดที่แนบมาเพิ่ม รันเฉพาะตอนจำเป็นจริงๆ เท่านั้น |

**ผลคือ:** เนื้อหาที่ bundle เข้า skill หนึ่งตัว **แทบไม่มีขีดจำกัด** ตราบใดที่ agent เข้าถึง filesystem และรันโค้ดได้ — เพราะไม่ได้โหลดทั้งหมดเข้า context พร้อมกันตั้งแต่แรก

---

## 3. ความสามารถสำคัญ — สคริปต์ที่รันได้จริง

Skill ใส่สคริปต์ที่ Claude รันเป็น tool ได้ — ทำงาน **deterministic และประหยัด token** เช่นจัดการไฟล์ PDF โดยไม่ต้องประมวลผลแบบ token-based ทั้งหมด (ต่างจากให้โมเดลอ่าน/เขียนทุกอย่างเป็นข้อความ)

---

## 4. ทำไมเรื่องนี้ใกล้ตัวกว่าที่คิด

Skill ที่กล่าวถึงในบทความนี้ **คือกลไกเดียวกับ skill ที่ปรากฏในลิสต์ของ session นี้เอง** (เช่น `/code-review`, `/diagram-design`, `/git-commit`) — ตอนเปิด session ใหม่ ระบบโหลดแค่ชื่อ+description ของ skill ทั้งหมดไว้ก่อน (ชั้น metadata) แล้วค่อยโหลดเนื้อหาเต็มเฉพาะตัวที่ถูกเรียกใช้จริง

**ไม่ใช่แค่ Anthropic ทำ — บริษัทอื่นก็เริ่มทำ skill ของตัวเองแล้ว** เช่น [[Supabase]] ออก `supabase/agent-skills` อย่างเป็นทางการ เพราะเจอปัญหาว่า agent ทั่วไปมักลืมเปิด RLS หรือมโนคำสั่งที่ไม่มีจริงเวลาช่วยเขียนโค้ดเชื่อม Supabase — สะท้อนว่า **Agent Skills กำลังกลายเป็นมาตรฐานเปิด (Agent Skills Open Standard)** ที่ agent เครื่องมืออื่น (Cursor, GitHub Copilot, Cline) ใช้ format เดียวกันได้ ไม่ผูกกับ Claude เท่านั้น

---

## 🔗 เกี่ยวข้อง

- [[Claude Code Best Practices]] — หัวข้อ "Create skills" อธิบายวิธีสร้าง skill ของตัวเองแบบใช้งานจริง
- [[Claude MCP Code Execution]] — แนวคิด "โหลดเฉพาะที่จำเป็น" แบบเดียวกัน ใช้กับ MCP tool แทนที่จะเป็น skill

## 📖 อ่านต่อ

- [Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
