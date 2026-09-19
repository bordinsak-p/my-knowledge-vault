---
tags:
  - playwright
  - mcp
  - ai
type: reference
created: 2026-09-17
---

# 🤖 Playwright MCP — ให้ AI agent คุม browser จริง

> **MCP (Model Context Protocol)** คือมาตรฐานกลางที่ให้ AI agent (Claude, Cursor, VS Code ฯลฯ) เรียกใช้ "เครื่องมือ" ภายนอกได้แบบเป็นระบบ — **Playwright MCP คือ MCP server ทางการของ Microsoft** ที่ห่อความสามารถของ [[Playwright]] ไว้เป็นเครื่องมือให้ AI agent เรียกใช้ควบคุม browser จริงได้ ไม่ต้องเขียนโค้ด Playwright เอง

---

## 1. ทำงานต่างจาก Playwright ปกติยังไง

| | Playwright (เขียนเทสเอง) | Playwright MCP |
|---|---|---|
| ใครเขียนโค้ด | คนเขียน `.spec.ts` เอง | AI agent สั่งงานผ่านภาษาธรรมชาติ แล้วแปลงเป็นคำสั่ง Playwright เอง |
| รับรู้หน้าเว็บยังไง | ตามโค้ดที่เขียนไว้ตายตัว | อ่าน **accessibility tree** ของหน้าจริงแบบ real-time (ไม่ใช้ screenshot/pixel) |
| เหมาะกับ | test suite ที่ต้อง maintain ระยะยาว รันซ้ำใน CI | สำรวจแอป, debug แบบโต้ตอบ, generate test เริ่มต้นให้ |

**จุดสำคัญ: ไม่ใช้ vision model/screenshot เป็นหลัก** — อ่านโครงสร้างหน้าเว็บจาก accessibility tree ตรง ๆ (เหมือนที่ screen reader ใช้) ทำให้ AI "เห็น" ว่าปุ่มไหนชื่ออะไรจริง ๆ ไม่ต้องเดาจากภาพ แม่นกว่าและเร็วกว่า

---

## 2. ติดตั้ง

```bash
npx @playwright/mcp@latest
```

รองรับ MCP client แทบทุกตัว: Claude Code, Claude Desktop, Cursor, VS Code, Windsurf — config ผ่านไฟล์ MCP config ของแต่ละ client (เช่น `.vscode/mcp.json`, `claude_desktop_config.json`)

---

## 3. ใช้ทำอะไรได้บ้าง

- **สำรวจแอปแบบไม่ต้องเขียนโค้ด** — บอก AI เป็นภาษาธรรมชาติว่า "ไป login แล้วลองเพิ่มสินค้าลงตะกร้าดู" agent คุม browser จริงทำตามให้เห็นผลจริง
- **Generate Playwright test จาก natural language** — อธิบาย flow ที่อยากเทส agent สำรวจหน้าเว็บจริงผ่าน MCP แล้วเขียน `.spec.ts` ให้ พร้อม locator แบบ `getByRole`/`getByTestId` ที่แม่นยำ (เพราะอ่านจาก accessibility tree จริง ไม่ได้เดา)
- **Reproduce บั๊ก** — บอก step ที่ทำให้เกิดบั๊ก agent ทำตามจริงในเบราว์เซอร์ ช่วยยืนยันว่าบั๊กเกิดจริงก่อนเริ่มแก้
- **Autonomous exploration** — ให้ agent สำรวจแอปเองแล้วเสนอ test case ที่ยังไม่ครอบคลุม

---

## 4. ตัวอย่างการสั่งงาน (ผ่าน chat ของ AI agent ที่ต่อ MCP นี้ไว้)

```
"เปิด localhost:4200 ไปหน้า login กรอก user/pass ทดสอบ
 แล้วดูว่า redirect ไปหน้า dashboard จริงไหม เขียนเป็น Playwright test ให้ด้วย"
```

agent จะ: เปิด browser จริง → อ่าน accessibility tree ของหน้า login → กรอกฟอร์มจริง → เช็กผล → สรุปเป็นโค้ด `.spec.ts` ที่รันซ้ำได้ภายหลัง (เอาไปใช้ต่อกับ [[Playwright]] ปกติ)

---

## 5. กับดัก

- **ให้ agent ทำงานกับ production/ข้อมูลจริงโดยไม่ระวัง** — agent คุม browser จริง คลิกจริง ส่ง form จริง ถ้าไม่ได้จำกัดขอบเขต (env ไหน, URL ไหน) เสี่ยงกระทบข้อมูลจริงโดยไม่ตั้งใจ ควรทดสอบกับ staging/local เสมอ
- **คาดหวังว่า test ที่ agent generate ให้จะสมบูรณ์แบบทันที** — เป็นจุดเริ่มต้นที่ดีกว่าเขียนเองจากศูนย์ แต่ควรอ่านทวน/ปรับ assertion ให้ตรงกับสิ่งที่อยากเทสจริง ไม่ใช่ copy ไปใช้เลยโดยไม่ดู
- **สับสนระหว่าง Playwright MCP กับ Playwright ปกติ** — MCP เหมาะกับ "สำรวจ/สร้างต้นแบบ" ส่วน test suite จริงที่ต้องรันซ้ำใน CI ทุกวันยังควรเป็นไฟล์ `.spec.ts` ที่ commit ไว้ (ดู [[Playwright]]) ไม่ใช่พึ่ง agent รันสดทุกครั้ง
- **เปิด MCP server แบบไม่จำกัดสิทธิ์ในเครื่องที่มีข้อมูล sensitive** — agent เข้าถึง browser ได้เต็มรูปแบบ (เห็นทุกอย่างที่เปิดอยู่) ควรรู้ว่ากำลังให้สิทธิ์อะไรอยู่

---

## 6. Cheat sheet

```bash
npx @playwright/mcp@latest        # รัน MCP server (ปกติ client เป็นคนเรียกให้เองตาม config)
```

| ต้องการ | ใช้ |
|---|---|
| เขียน test suite ถาวรที่รันใน CI | [[Playwright]] ปกติ เขียนโค้ดเอง/commit |
| สำรวจแอป/debug แบบโต้ตอบ | Playwright MCP ผ่าน AI agent |
| ให้ AI ช่วยร่าง test เริ่มต้นจาก flow ที่อธิบาย | Playwright MCP → เอาผลลัพธ์ไปขัดต่อใน [[Playwright]] |

## 🔗 เกี่ยวข้อง

- [[Playwright]] — พื้นฐาน E2E test แบบเขียนโค้ดเอง ที่ MCP นี้ generate โค้ดออกมาให้ในรูปแบบเดียวกัน
- [[Claude MCP Code Execution]] — แนวทางที่ประหยัด token กว่า เรียก MCP ผ่านโค้ดแทน tool call ตรงๆ

## 📖 อ่านต่อ

- [Playwright MCP — Introduction](https://playwright.dev/mcp/introduction)
- [Playwright — Getting Started with MCP](https://playwright.dev/docs/getting-started-mcp)
