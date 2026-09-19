---
tags:
  - claude
  - claude/index
type: moc
created: 2026-09-17
---

# 🧡 Claude — สรุปบทความจาก Anthropic Engineering/Blog

หน้ารวมโน้ตสรุปบทความที่เกี่ยวกับ Claude/Anthropic ที่น่าสนใจสำหรับคนใช้ Claude Code ทำงานจริง — คัดมาจาก [anthropic.com/engineering](https://www.anthropic.com/engineering) และ [claude.com/blog](https://claude.com/blog) เขียนสรุปเป็นภาษาไทยแบบเข้าใจง่าย ไม่ใช่แปลตรงตัว

> ⚠️ เนื้อหาเป็นสรุปตามความเข้าใจ ณ วันที่เขียน ไม่ใช่ต้นฉบับเต็ม — อยากได้รายละเอียดครบ ให้ตามลิงก์ "อ่านต่อ" ท้ายแต่ละโน้ต

---

## โน้ตในนี้

### ใช้งาน Claude Code จริง

| โน้ต | ว่าด้วย |
|---|---|
| [[Claude Code Best Practices]] | best practice รวมทุกอย่าง — verification loop, plan mode, CLAUDE.md, session management, automate/scale |
| [[Claude Code Auto Mode]] | permission mode ที่ใช้ AI classifier แทนคนอนุมัติทุก action |
| [[Claude Agent Lifecycle]] | agentic loop คืออะไร — 3 phase, tool 5 หมวด, session/context window ทำงานยังไง |
| [[Claude Code Hooks]] | รันสคริปต์แบบ deterministic ที่จุดต่างๆ ใน lifecycle — event ที่ใช้บ่อย, exit code, ตัวอย่างจริง |

### แนวคิดเบื้องหลัง agent (ใช้ได้กว้างกว่าแค่ CLI)

| โน้ต | ว่าด้วย |
|---|---|
| [[Claude Agent Harness Design]] | สถาปัตยกรรม generator/evaluator สำหรับ agent ที่ทำงานยาวๆ ไม่มีคนคอยดู |
| [[Claude Agent Tool Design]] | หลักการออกแบบ tool ให้ agent ใช้ได้ดี (ต่างจากออกแบบ API ให้คนเรียก) |
| [[Claude Agent Skills]] | Skills คืออะไร ทำงานยังไง (progressive disclosure) |
| [[Claude MCP Code Execution]] | ให้ agent เขียนโค้ดเรียก MCP แทน tool call ตรงๆ ประหยัด token ได้มาก |

### Case study

| โน้ต | ว่าด้วย |
|---|---|
| [[Claude Code CI at Scale]] | Anthropic ปรับ CI ของตัวเองยังไงตอนโค้ดจาก agent เพิ่มขึ้นก้าวกระโดด |
| [[Claude Parallel Agents C Compiler]] | ทดลองให้ Claude 16 ตัวช่วยกันสร้าง C compiler จริง |

---

## 🔗 ที่อื่นใน vault

- [[Playwright MCP]] — MCP server ตัวอย่างที่เกี่ยวกับหัวข้อ MCP ในนี้โดยตรง

## 📖 อ้างอิงหลัก

- [Anthropic Engineering](https://www.anthropic.com/engineering)
- [Claude Blog](https://claude.com/blog)
