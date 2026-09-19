---
tags:
  - claude
  - ai
  - agent-engineering
  - case-study
type: reference
created: 2026-09-17
---

# 🧪 เคสทดลอง — สร้าง C compiler ด้วย Claude 16 ตัวพร้อมกัน

> Case study ของนักวิจัย Nicholas Carlini — ให้ Claude Opus 4.6 จำนวน **16 instance** ร่วมกันสร้าง C compiler (เขียนด้วย Rust) ที่ compile Linux kernel ได้จริง โดยแทบไม่มีคนเข้าไปแตะเลย

---

## 1. วิธีทำงาน

Agent ทั้ง 16 ตัวรันเป็น **infinite loop** — หยิบงาน ทำงาน แล้ว push เข้า shared repository วนไปเรื่อยๆ โดยไม่ต้องรอคนสั่ง

**ป้องกันชนกันด้วย git-based synchronization** — แต่ละ agent "ล็อก" งานที่หยิบมาทำด้วยการสร้างไฟล์ (claim งาน) ทำงานแยกกันในสภาพแวดล้อมแบบ container แล้วค่อย merge + push ขึ้นไปทีหลัง

**ตัวเลขรวม:** เกือบ 2,000 code session, ค่าใช้จ่าย API **$20,000**, ได้ compiler ขนาด **100,000 บรรทัด** รองรับ x86, ARM, RISC-V

---

## 2. สิ่งที่ทำให้สำเร็จ — คุณภาพของตัวตรวจสอบสำคัญที่สุด

> **"ตัว verifier ของ task ต้องเกือบสมบูรณ์แบบ ไม่งั้น Claude จะไปแก้ปัญหาผิดข้อแทน"**

ทีมใช้ **CI pipeline ต่อเนื่อง** และใช้ **GCC เป็น "oracle" (ตัวเทียบผลลัพธ์ที่รู้ว่าถูก)** — compile ด้วยตัวที่เขียนเองแล้วเทียบผลกับ GCC ทุกครั้ง

**การแบ่งความเชี่ยวชาญ (specialization)** ช่วยได้มาก — agent แยกกันรับผิดชอบ code deduplication, performance optimization, documentation พร้อมกัน

---

## 3. ข้อจำกัดที่ยังเจอ

- ไม่มี compiler สำหรับ x86 แบบ 16-bit (ต้องเรียก GCC แทนในกรณีนั้น)
- โค้ดที่ได้ไม่มีประสิทธิภาพเท่า GCC จริง
- **เพิ่มฟีเจอร์ใหม่แล้วทำของเดิมพังบ่อย** — สิ่งที่ต้องเฝ้าระวังมากที่สุดในงานแบบนี้

แม้จะมีข้อจำกัด แต่ compiler ที่ได้ **build Linux 6.9 ได้จริง** และ compile โปรเจกต์ดังอย่าง SQLite, Redis ได้ด้วย

---

## 4. ข้อคิดที่ generalize ได้

> **"ใส่ตัวเองอยู่ในมุมมองของ Claude"** — ออกแบบ interface โดยคิดถึงวิธีที่ LLM ประมวลผลข้อมูลโดยเฉพาะ

Carlini เน้นย้ำ 2 เรื่อง:
- **หลีกเลี่ยง context pollution** — อย่าให้ agent ต้องแบกข้อมูลที่ไม่จำเป็นติดตัวไปเรื่อยๆ
- **รับมือกับ "time blindness"** — agent ไม่รู้ตัวว่าใช้เวลาไปนานแค่ไหน ต้องมีทางเลือก "sample เทสแบบเร็ว" ให้เลือกใช้แทนการรันเทสเต็มทุกครั้ง

---

## 🔗 เกี่ยวข้อง

- [[Claude Agent Harness Design]] — หลักการ generator/evaluator แยกกันที่ใช้จริงในเคสนี้ (verifier เกือบสมบูรณ์แบบ)
- [[Claude Code Best Practices]] — หัวข้อ parallel session/fan-out คือเวอร์ชันเล็กกว่าของสิ่งที่เคสนี้ทำในสเกลใหญ่

## 📖 อ่านต่อ

- [Building a C compiler with a team of parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler)
