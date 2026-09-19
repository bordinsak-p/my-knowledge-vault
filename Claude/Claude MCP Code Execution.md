---
tags:
  - claude
  - mcp
  - ai
type: reference
created: 2026-09-17
---

# ⚡ Code Execution with MCP — ประหยัด token ได้ถึง 98.7%

> แนวทางใหม่ในการต่อ agent เข้ากับ [[Playwright MCP|MCP]] server — แทนที่จะให้ agent เรียก tool ตรงๆ ทีละตัว ให้ agent **เขียนโค้ดเรียก MCP เหมือนเรียก API ปกติ** แล้วรันในสภาพแวดล้อม execution แยกต่างหาก

---

## 1. ปัญหาของการเรียก tool ตรงๆ แบบเดิม

### Context ล้นตั้งแต่ยังไม่เริ่มทำงาน

วิธีเดิมโหลด **definition ของทุก tool เข้า context window ตั้งแต่แรก** — ถ้ามี tool เป็นร้อยเป็นพันตัว แค่คำอธิบาย tool ก็กิน token มหาศาลก่อนที่โมเดลจะได้อ่าน request จริงด้วยซ้ำ

### ผลลัพธ์กลางทางถูกส่งผ่านโมเดลซ้ำๆ

**"ทุกผลลัพธ์ระหว่างทางต้องผ่านโมเดล"** — เอกสารยาวๆ ถูกส่งเข้า-ออกโมเดลซ้ำทุกครั้งที่มี tool call ต่อกัน กินเป็นหมื่น token โดยไม่จำเป็น

---

## 2. ทางแก้ — ให้ MCP server เป็น code API แทน tool call ตรงๆ

Agent **เขียนโค้ดจริง** เรียกใช้ MCP ผ่านสภาพแวดล้อม execution ของมันเอง:

- **ค้นหา tool ตามต้องการ (on-demand)** — สำรวจโครงสร้างไฟล์เพื่อโหลดเฉพาะ tool definition ที่จำเป็นสำหรับงานนั้นจริงๆ ไม่โหลดหมดตั้งแต่แรก
- **ประมวลผลข้อมูลใน execution environment ก่อนส่งกลับ** — filter/transform/aggregate ทำในโค้ดเลย ไม่ต้องส่งข้อมูลดิบผ่านโมเดล
- **ใช้ loop/condition แทนการเรียง tool call ทีละตัว** — workflow ซับซ้อนเขียนเป็นโค้ดปกติได้ ไม่ต้อง chain การเรียกทีละขั้น

---

## 3. ตัวเลขจริงจากบทความ

**ประหยัด token/cost ได้ 98.7%** ในตัวอย่างที่ยกมา — จาก 150,000 token เหลือแค่ 2,000 token สำหรับงานเดียวกัน และผลประหยัดนี้ยิ่งเห็นชัดขึ้นเมื่อ workflow ซับซ้อนขึ้น (ต้องใช้หลาย tool ต่อกันหลายขั้น)

---

## 4. ทำไมเรื่องนี้สำคัญกับที่เพิ่งเขียนเรื่อง MCP ไป

หลักการนี้ตรงข้ามกับภาพที่คุ้นเคยจาก [[Playwright MCP]] ที่ agent เรียก tool call ทีละตัว (`click`, `fill`, `navigate`) — **code execution คือวิวัฒนาการขั้นต่อไปของแนวคิดเดียวกัน** เหมาะกับสถานการณ์ที่มี MCP server หลายตัว/tool เยอะมาก จนการโหลด definition ทั้งหมดเข้า context กลายเป็นคอขวดจริงๆ

---

## 🔗 เกี่ยวข้อง

- [[Playwright MCP]] — ตัวอย่าง MCP แบบเรียก tool ตรงๆ ที่แนวทางนี้พยายามแก้ข้อจำกัด
- [[Claude Agent Tool Design]] — หลักการออกแบบ tool ให้ agent ใช้ได้ดี ซึ่งเกี่ยวโยงกับปัญหา context ที่บทความนี้แก้

## 📖 อ่านต่อ

- [Code execution with MCP: Building more efficient agents](https://www.anthropic.com/engineering/code-execution-with-mcp)
