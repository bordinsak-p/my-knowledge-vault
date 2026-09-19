---
tags:
  - claude
  - claude-code
  - ai
type: reference
created: 2026-09-17
---

# 🧭 Claude Code — Best Practices

> สรุปจากเอกสารทางการของ Anthropic เรื่องวิธีใช้ Claude Code ให้ได้ผลดีที่สุด — **ใจความหลักทั้งหมดวนกลับมาที่ข้อจำกัดเดียว: context window เต็มเร็ว และ performance แย่ลงเมื่อ context เต็ม** ทุก best practice ในนี้คือวิธีบริหาร context ให้ฉลาดขึ้น ไม่ใช่ trick แยกกันคนละเรื่อง

---

## 1. ให้ Claude มีทางตรวจสอบงานตัวเอง (ข้อสำคัญที่สุด)

**ถ้าไม่มี check ให้รัน "ดูเหมือนเสร็จแล้ว" คือสัญญาณเดียวที่ Claude มี** — กลายเป็นว่าคนต้องเป็นคนตรวจทุกจุดเอง แต่ถ้าให้ check ที่ได้ผล pass/fail ชัดเจน (test suite, build, screenshot เทียบ design) Claude จะรันเอง อ่านผลเอง แล้ววนแก้เองจนผ่าน

| แบบเดิม | แบบที่ดีกว่า |
|---|---|
| "เขียนฟังก์ชันเช็ก email" | "เขียน `validateEmail` เทสเคสตัวอย่าง: `user@example.com` = true, `invalid` = false รันเทสหลัง implement" |
| "ทำ dashboard ให้สวยขึ้น" | "\[แปะ screenshot] implement ตามนี้ แล้ว screenshot ผลลัพธ์มาเทียบกับต้นฉบับ ระบุจุดต่าง แล้วแก้" |
| "build มันพัง" | "build fail ด้วย error นี้ \[แปะ error] แก้ที่ root cause ไม่ใช่แค่ suppress error" |

ระดับความเข้มงวดของ check เลือกได้ตามงาน: บอกในพรอมต์เดียว → ตั้งเป็น `/goal` ให้ evaluator เช็กซ้ำทุก turn → ตั้งเป็น Stop hook (deterministic บังคับจริง) → ให้ subagent อีกตัวมาช่วย "หักล้าง" ผลลัพธ์ (fresh context ไม่มี bias จากที่เพิ่งเขียนเอง)

---

## 2. Explore → Plan → Code — อย่าให้กระโดดเขียนโค้ดทันที

ปล่อยให้ Claude เขียนโค้ดทันทีมีโอกาสแก้ผิดปัญหา ใช้ **plan mode** (`Shift+Tab` จนขึ้น `⏸ plan mode on`) แยกขั้น "สำรวจ" ออกจาก "ลงมือทำ":

1. **Explore** — ให้ Claude อ่านโค้ด ตอบคำถาม ยังไม่แก้อะไร
2. **Plan** — ขอแผน implementation แบบละเอียด (`Ctrl+G` เปิดแผนไปแก้ในตัว editor เองได้ก่อนอนุมัติ)
3. **Implement** — ออกจาก plan mode แล้วให้ลงมือ พร้อมเช็กกับแผนที่วางไว้
4. **Commit** — ให้ commit message ที่ดีและเปิด PR

**เมื่อไหร่ไม่ต้อง plan:** งานที่ scope ชัดและเล็ก (แก้ typo, เพิ่ม log) ถ้าบรรยาย diff จบในประโยคเดียวได้ ข้าม plan ไปเลย — planning คุ้มค่าตอนไม่แน่ใจ approach, แก้หลายไฟล์พร้อมกัน, หรือไม่คุ้นโค้ดส่วนนั้น

---

## 3. ให้ context เจาะจง ไม่ใช่แค่บอกสิ่งที่อยากได้

| แบบเดิม | แบบที่ดีกว่า |
|---|---|
| "เพิ่มเทสให้ foo.py" | "เขียนเทส foo.py คลุม edge case ตอน user logout แล้ว อย่าใช้ mock" |
| "ทำไม API นี้แปลก" | "ไล่ git history ของ `ExecutionFactory` แล้วสรุปว่า API หน้าตาแบบนี้มาจากไหน" |
| "เพิ่ม calendar widget" | "ดูว่า widget อื่นในหน้า home ทำยังไง `HotDogWidget.php` เป็นตัวอย่างดี ทำตาม pattern เดียวกัน" |
| "แก้บั๊ก login" | "user รายงานว่า login fail หลัง session timeout เช็ก auth flow ใน `src/auth/` โดยเฉพาะ token refresh เขียนเทสที่ reproduce บั๊กได้ก่อน แล้วค่อยแก้" |

**ใช้ `@` reference ไฟล์**, แปะ screenshot ตรงๆ, ให้ URL เอกสารอ้างอิง, หรือ `cat error.log | claude` ส่งข้อมูลเข้าไปตรงๆ — ยิ่งเจาะจง ยิ่งแก้กลับน้อยรอบ

---

## 4. ตั้งค่าสภาพแวดล้อมให้ Claude ทำงานได้ดีขึ้นทุก session

### `CLAUDE.md` — context ที่ Claude เดาเองไม่ได้

รัน `/init` สร้างไฟล์เริ่มต้น แล้วปรับไปเรื่อยๆ — **ยิ่งสั้นยิ่งดี** กฎแต่ละบรรทัดถามตัวเองว่า "ถ้าลบบรรทัดนี้ Claude จะพลาดไหม" ถ้าไม่ ให้ตัดทิ้ง

| ✅ ควรใส่ | ❌ ไม่ควรใส่ |
|---|---|
| Bash command ที่ Claude เดาไม่ได้ | อะไรก็ตามที่อ่านโค้ดแล้วรู้เองได้ |
| กฎ code style ที่ต่างจาก default | convention มาตรฐานของภาษาที่ Claude รู้อยู่แล้ว |
| คำสั่งเทส/test runner ที่ใช้ | เอกสาร API แบบละเอียด (ใส่ลิงก์แทน) |
| ข้อควรระวัง/พฤติกรรมที่ไม่ obvious | อธิบายยาวๆ แบบ tutorial |

CLAUDE.md ที่ยาวเกินไป = Claude เมินครึ่งหนึ่งเพราะกฎสำคัญจมหายไปในสิ่งที่ไม่จำเป็น

### เครื่องมืออื่นที่ตั้งไว้แล้วช่วยได้มาก

| เครื่องมือ | ใช้ทำอะไร |
|---|---|
| **Permission allowlist / Sandbox** | ลดจำนวนครั้งที่ต้องกด approve โดยไม่เสียการควบคุม |
| **CLI tools** (`gh`, `aws`, `gcloud`) | วิธีที่ context-efficient สุดในการคุยกับ service ภายนอก |
| **MCP servers** | ต่อ Notion/Figma/database ฯลฯ เข้ามาใช้ตรงๆ |
| **Hooks** | รันสคริปต์แบบ deterministic ("ทำทุกครั้งไม่มีข้อยกเว้น") ต่างจาก CLAUDE.md ที่เป็นแค่คำแนะนำ |
| **Skills** (`.claude/skills/`) | ความรู้เฉพาะทาง/workflow ที่โหลดเฉพาะตอนเกี่ยวข้อง ไม่ถ่วง context ทุก session |
| **Subagents** (`.claude/agents/`) | ผู้ช่วยเฉพาะทางที่รันแยก context ของตัวเอง |

---

## 5. สื่อสารแบบมีประสิทธิภาพ

- **ถามเหมือนถาม senior engineer** — "logging ทำงานยังไง", "endpoint ใหม่ทำยังไง", "บรรทัด 134 ทำอะไร" — ไม่ต้อง prompt พิเศษอะไร ถามตรงๆ ได้เลย
- **ให้ Claude สัมภาษณ์กลับ** สำหรับ feature ใหญ่ — ใช้ `AskUserQuestion` tool ให้ Claude ถามละเอียด (technical, UX, edge case) จนครบแล้วเขียนเป็น `SPEC.md` ก่อนเริ่ม implement — spec ที่ดีคือระบุไฟล์/interface ที่เกี่ยวข้อง บอกว่าอะไร out of scope และจบด้วยขั้นตอน verify แบบ end-to-end

---

## 6. บริหาร session

- **แก้ไขเร็วตั้งแต่เห็นว่าออกนอกทาง** — `Esc` หยุดกลางคัน (context ยังอยู่), `Esc Esc`/`/rewind` ย้อนกลับไปจุดก่อนหน้า, พูด "undo that" ตรงๆ ได้
- **แก้ผิดจุดเดิมเกิน 2 ครั้งในเซสชันเดียว → `/clear` แล้วเริ่มใหม่** ด้วย prompt ที่เจาะจงกว่าเดิม (context ที่เต็มไปด้วยความพยายามที่ผิดพลาดทำให้แย่ลงเรื่อยๆ ไม่ใช่ดีขึ้น)
- **`/clear` ระหว่างงานที่ไม่เกี่ยวกัน** เสมอ — auto-compact จะสรุปให้เองตอน context ใกล้เต็ม แต่เชิงรุกดีกว่ารอ
- **ใช้ subagent สำหรับงานสำรวจ** — "use subagents to investigate X" กันไม่ให้การอ่านไฟล์เยอะๆ ไปกิน context หลัก
- **ตั้งชื่อ session** (`/rename`) แล้วปฏิบัติเหมือน branch — แต่ละ workstream มี context ของตัวเอง

---

## 7. Automate และ scale ขึ้น

| แนวทาง | ใช้เมื่อ |
|---|---|
| `claude -p "prompt"` (non-interactive) | รันใน CI, pre-commit hook, script |
| **Worktrees** | หลาย session พร้อมกันแบบไม่ชนกัน (แก้ไฟล์คนละชุด) |
| **Writer/Reviewer pattern** | session หนึ่งเขียนโค้ด อีก session (context ใหม่ ไม่ bias) มารีวิว |
| **Fan-out** (`/batch` หรือ loop `claude -p`) | migration ขนาดใหญ่ กระจายงานเป็นสิบ subagent พร้อมกัน |
| **Auto mode** (`--permission-mode auto`) | รันแบบไม่ถูกขัดจังหวะ มี classifier คอยเช็กความปลอดภัยแทนคน |

---

## 8. กับดัก (จาก "Avoid common failure patterns")

- **Kitchen sink session** — เริ่มงานหนึ่ง แทรกงานอื่นที่ไม่เกี่ยวกลางคัน context ปนกันหมด → แก้ด้วย `/clear` ระหว่างงานที่ไม่เกี่ยวกันเสมอ
- **แก้ผิดจุดเดิมซ้ำๆ** — correction ซ้อน correction ทำให้ context เน่า → หลัง 2 ครั้งให้ `/clear` เริ่มใหม่พร้อม prompt ที่ดีกว่าเดิม
- **CLAUDE.md ยาวเกินไป** — กฎสำคัญจมหายในสิ่งที่ไม่จำเป็น → ตัดทิ้งอย่างไม่ปรานี
- **Trust-then-verify gap** — โค้ดดูดีแต่ไม่ handle edge case จริง → ต้องมี verification (เทส/สคริปต์/screenshot) เสมอ ตรวจไม่ได้ = ไม่ควร ship
- **Infinite exploration** — สั่ง "investigate" แบบไม่จำกัด scope Claude อ่านเป็นร้อยไฟล์จน context เต็ม → scope การสำรวจให้แคบ หรือใช้ subagent

---

## 9. Cheat sheet

```
Shift+Tab              → เข้า/ออก plan mode
Esc                    → หยุดกลางคัน (context ยังอยู่)
Esc Esc / /rewind       → ย้อนกลับไป checkpoint ก่อนหน้า
/clear                  → รีเซ็ต context (ระหว่างงานที่ไม่เกี่ยวกัน)
/compact <instruction>  → สรุป context แบบระบุเงื่อนไขเอง
/init                   → generate CLAUDE.md เริ่มต้น
claude -p "..."         → non-interactive (CI/script)
claude --permission-mode auto -p "..."  → รันไม่ถูกขัดจังหวะ
```

## 🔗 เกี่ยวข้อง

- [[Claude Agent Harness Design]] — สถาปัตยกรรมเบื้องหลัง agent ที่ทำงานยาวๆ แบบมี verification loop
- [[Claude Code Auto Mode]] — รายละเอียดว่า auto mode ที่กล่าวถึงข้อ 7 ทำงานยังไงจริงๆ
- [[Claude Agent Skills]] — รายละเอียดเต็มของ Skills ที่พูดถึงในข้อ 4

## 📖 อ่านต่อ

- [Claude Code — Best practices](https://code.claude.com/docs/en/best-practices)
