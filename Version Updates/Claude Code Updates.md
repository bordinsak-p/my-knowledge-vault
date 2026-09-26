---
tags:
  - claude
  - version-updates
  - changelog
type: reference
created: 2026-09-23
---

# 🆕 Claude Code — Version Updates ล่าสุด

> โน้ตนี้ติดตาม **"อะไรใหม่"** ของตัว Claude Code (CLI) เอง — ไม่ใช่สรุปบทความ/แนวคิดเบื้องหลัง agent (ไปดู [[Claude]] สำหรับสรุปบทความ Anthropic Engineering/Blog) เช็กวันที่ด้านล่างก่อนเชื่อ 100% เสมอ

> **อัปเดตล่าสุด: 2026-09-23 — เวอร์ชันล่าสุดคือ v2.1.280 (22 ก.ย. 2026)**

---

## 1. Claude Code นับเวอร์ชันยังไง — ถี่กว่าทุกตัวในโฟลเดอร์นี้

**ไม่มี LTS ไม่มี major/minor cadence ตายตัวแบบ Java/Quarkus** — ออก patch แทบทุกวัน (เช่นจาก v2.1.269 ถึง v2.1.280 ใช้เวลาแค่ ~2 สัปดาห์) เป็น continuous delivery ของ CLI tool ไม่ใช่ framework ที่ทีมต้อง pin เวอร์ชันตายตัวแบบ backend

**วิธีติดตั้งมีผลโดยตรงว่าจะได้ของใหม่เร็วแค่ไหน:**

| วิธีติดตั้ง | auto-update? |
|---|---|
| **Native installer** (`curl -fsSL https://claude.ai/install.sh \| bash` หรือสคริปต์ PowerShell เทียบเท่าบน Windows) — วิธีแนะนำปัจจุบัน ไม่ต้องมี Node.js/npm เลย | ✅ อัปเดตอัตโนมัติเบื้องหลัง |
| npm (`npm install -g @anthropic-ai/claude-code`) | ❌ ต้อง `npm update -g` เอง |
| Homebrew / WinGet | ❌ ต้องอัปเดตเอง |

```bash
claude --version   # เช็คเวอร์ชันที่ใช้อยู่จริง
claude update      # สั่งอัปเดตเองตรงๆ (ใช้ได้แม้ปิด auto-update ไว้)
```

---

## 2. Milestone ฟีเจอร์ใหญ่ — ไทม์ไลน์คร่าวๆ

| เมื่อไหร่ | ฟีเจอร์ |
|---|---|
| พ.ย. 2024 | MCP (Model Context Protocol) |
| ก.ค. 2025 | Subagents |
| ก.ย. 2025 | Hooks |
| ต.ค. 2025 | Plugins + Skills + Native installer |
| ก.พ. 2026 | Agent Teams |
| ก.ย. 2026 | Opus 5.5 เป็น default model, auto mode ใช้ server-side classifier เป็นค่าเริ่มต้น |

**มีประโยชน์เวลาอ่านบทความ/tutorial เก่า** — บทความที่เขียนก่อน ต.ค. 2025 จะไม่มี plugins/skills พูดถึงเลย เพราะยังไม่มีฟีเจอร์นี้ตอนนั้น

---

## 3. ล่าสุด (ก.ย. 2026) — headline ที่ควรรู้

- **Opus 5.5 เป็น default model ใหม่** (v2.1.280) — context window 1M, ราคาลดลง เปลี่ยน default บน Pro/Team Standard จาก Sonnet เป็น Opus
- **Auto mode ใช้ server-side classifier เป็นค่าเริ่มต้น** (v2.1.278) สำหรับ Claude API/Enterprise/Bedrock/Vertex/Foundry — ไม่เสียค่า classifier overhead เพิ่มแล้ว (ปิดกลับไปใช้ local classifier ได้ผ่าน `CLAUDE_CODE_AUTO_MODE_SERVER=0`)
- **อ่าน `AGENTS.md` แทน `CLAUDE.md` ได้** (v2.1.277) ถ้าโปรเจกต์ไม่มี `CLAUDE.md`
- **`/output-style [name]`** (v2.1.269) — list/สลับ output style ได้ตรงๆ
- **`claude plugin eval`** (v2.1.269) — รัน test suite ของ plugin แบบ reproducible พร้อมคะแนน (JSON/HTML report)
- **Agent Map ใน VS Code extension** — เห็น subagent แต่ละตัวเป็นการ์ด กด Stop แยกตัวได้, ดู transcript แบบ read-only
- **`/tasks`** — ดู background shell/task ที่รันอยู่เบื้องหลังรวมกันเป็นลิสต์เดียว

---

## กับดัก

- **ติดตั้งผ่าน npm/Homebrew/WinGet แล้วคิดว่าอัปเดตให้อัตโนมัติเหมือน native installer** — ไม่จริง (ข้อ 1) ใช้เวอร์ชันเก่าค้างโดยไม่รู้ตัวได้ง่ายมากถ้าไม่สั่ง `claude update`/`npm update -g` เอง
- **อ่าน blog/tutorial เก่าแล้วเชื่อว่าฟีเจอร์นั้นมีในเวอร์ชันที่ใช้อยู่** — เช็ค milestone ข้อ 2 ก่อนเสมอ (เช่นบทความก่อน ต.ค. 2025 จะไม่มี plugins/skills เลย)
- **เวอร์ชันเปลี่ยนถี่มาก (เกือบทุกวัน)** — ถ้าทีม/CI pin เวอร์ชันไว้ตายตัวนานเกินไปจะพลาด security fix สะสม ต่างจาก Java/Quarkus ที่ pin เวอร์ชันได้นานกว่าอย่างสมเหตุสมผล
- **จำ config/flag จากเวอร์ชันเก่ามาใช้** — env var และ config key ใหม่โผล่บ่อย (เช่น `CLAUDE_CODE_MAX_MCP_DESCRIPTION_LENGTH`, `CLAUDE_CODE_WORKFLOW_MAX_CONCURRENT_AGENTS`) เช็ค `claude --version` ก่อนเทียบกับ doc ทุกครั้งถ้าพฤติกรรมไม่ตรงกับที่เคยเจอ

---

## Cheat sheet

```bash
claude --version                 # เช็คเวอร์ชันที่ใช้อยู่
claude update                    # อัปเดตเอง (เผื่อ auto-update ปิดอยู่)
```

| ต้องการรู้ | ดูที่ |
|---|---|
| changelog เต็มทุกเวอร์ชัน | [Claude Code Changelog (official)](https://code.claude.com/docs/en/changelog) |
| ฟีเจอร์ตัวไหนมาตอนไหน | ข้อ 2 ในโน้ตนี้ |
| ติดตั้ง/ย้ายวิธีติดตั้ง | [Claude Code — Setup](https://code.claude.com/docs/en/setup) |

---

## 🔗 เกี่ยวข้อง

- [[Claude]] — สรุปบทความ Anthropic Engineering/Blog (แนวคิดเบื้องหลัง ไม่ใช่ version-by-version)
- [[Claude Agent Skills]] — รายละเอียดฟีเจอร์ Skills ที่มาจาก milestone ต.ค. 2025 ในข้อ 2
- [[Claude Code Hooks]] — รายละเอียดฟีเจอร์ Hooks ที่มาจาก milestone ก.ย. 2025 ในข้อ 2
- [[Claude Code Auto Mode]] — permission mode ที่เพิ่งเปลี่ยน default เป็น server-side classifier (ข้อ 3)

## 📖 อ่านต่อ

- [Claude Code Changelog (official)](https://code.claude.com/docs/en/changelog)
- [Claude Code — Setup](https://code.claude.com/docs/en/setup)
