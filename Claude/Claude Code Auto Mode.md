---
tags:
  - claude
  - claude-code
  - ai
  - security
type: reference
created: 2026-09-17
---

# 🛡️ Claude Code — Auto Mode ทำงานยังไง

> **ปัญหาที่แก้:** ถามอนุญาตทุกการกระทำที่เสี่ยง (แก้ไฟล์/รัน command) ฟังดูปลอดภัย แต่พอเจอ prompt ขออนุญาตบ่อยๆ คนจะเริ่ม **"approval fatigue"** — กด approve ผ่านๆ โดยไม่อ่านจริง (Anthropic พบว่าคนกด approve ถึง 93% ของ prompt ที่เจอ) กลายเป็นว่าขั้นตอนขออนุญาตไม่ได้ช่วยความปลอดภัยจริง แค่สร้างความรำคาญ

---

## 1. แนวคิด — "จุดกึ่งกลาง" ระหว่าง manual review กับไม่มี guardrail เลย

Auto mode ไม่ได้ปิดการเช็กทั้งหมด (เสี่ยงเกินไป) และไม่ได้ให้คนอนุมัติทุกอย่าง (ช้า/ทำให้ approval fatigue) — แต่ **มอบหน้าที่อนุมัติให้ AI classifier ตัวแยกต่างหาก** แทนคน

---

## 2. โครงสร้างป้องกัน 2 ชั้น

### ชั้น Input — กรองสิ่งที่ Claude "เห็น"

**prompt-injection probe** สแกนผลลัพธ์จาก tool (อ่านไฟล์, fetch เว็บ, รัน shell) **ก่อน** ที่จะถึง Claude — ถ้าเจอเนื้อหาน่าสงสัย จะแจ้งเตือนให้โมเดลยึดกับ "ความตั้งใจจริงของผู้ใช้" ไม่ใช่คำสั่งที่แฝงมาในข้อมูลที่อ่านเข้ามา

### ชั้น Output — กรองสิ่งที่ Claude "จะทำ"

**transcript classifier** ประเมิน**ทุก action ก่อนรันจริง** — ทำงาน 2 stage:
1. filter เร็วๆ ที่เอนเอียงไปทาง "บล็อกไว้ก่อน"
2. ถ้าโดน flag ค่อยให้เหตุผลละเอียดอีกที

**จุดสำคัญ: classifier ไม่เห็น reasoning หรือ tool output ของ Claude เอง** — ป้องกันไม่ให้ Claude "โน้มน้าวตัวเอง" จนเชื่อว่าสิ่งที่กำลังจะทำโอเค (self-justification) — ประเมินจาก**ผลกระทบจริงที่จะเกิด**ไม่ใช่แค่ข้อความผิวเผิน จับ operation อันตรายที่ห่อด้วยคำพูดดูไม่มีพิษภัยได้ด้วย

---

## 3. ขอบเขตที่ block

มี rule ที่ปรับแต่งได้ครอบคลุม: data exfiltration, การลด security ของระบบลง, การสำรวจหา credential, การเลี่ยง infrastructure guardrail

---

## 4. ความจริงที่ยอมรับตรงๆ

**classifier มี false-negative rate 17%** กับ action ที่ overeager จริง — Anthropic ยอมรับตรงๆ ว่านี่คือการ **แลกความเสี่ยงบางส่วนเพื่อได้ autonomy เพิ่มขึ้น** เทียบกับการรันแบบไม่มี guardrail เลย ไม่ได้อ้างว่าปลอดภัย 100%

---

## 5. เอาไปใช้ยังไง

```bash
claude --permission-mode auto -p "fix all lint errors"
```

ใช้ auto mode เมื่ออยากให้ Claude ทำงานต่อเนื่องโดยไม่ต้องนั่งกด approve ตลอด แต่ยังอยากมี safety net อยู่ — ต่างจาก manual mode ที่ต้องอนุมัติทุกจุด และต่างจากการปิด permission ไปเลยซึ่งไม่มี guardrail ใดๆ เหลืออยู่

---

## 🔗 เกี่ยวข้อง

- [[Claude Code Best Practices]] — auto mode คือหนึ่งในวิธี "automate และ scale" ที่พูดถึงในหัวข้อ 7
- [[Claude Agent Harness Design]] — แนวคิด evaluator แยกจาก generator ที่คล้ายกับ classifier แยกจาก Claude ในบทความนี้

## 📖 อ่านต่อ

- [How we built Claude Code auto mode: a safer way to skip permissions](https://www.anthropic.com/engineering/claude-code-auto-mode)
