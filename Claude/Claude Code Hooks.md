---
tags:
  - claude
  - claude-code
  - ai
type: reference
created: 2026-09-17
---

# 🪝 Claude Code — Hooks

> **ปัญหาที่แก้:** `CLAUDE.md` เป็นแค่ "คำแนะนำ" — Claude *อาจ* ทำตามหรือไม่ก็ได้ ถ้าอยากได้ผลลัพธ์ที่ **เกิดขึ้นแน่นอนทุกครั้งไม่มีข้อยกเว้น** (format โค้ดหลังแก้, ห้ามแก้ไฟล์บางไฟล์, แจ้งเตือนตอนรอ input) ต้องใช้ **hook** — shell command ที่ Claude Code เรียกเองตรงจุดที่กำหนดไว้ใน lifecycle ไม่ผ่านการตัดสินใจของโมเดลเลย

---

## 1. Config พื้นฐาน — ใส่ใน `settings.json`

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write" }
        ]
      }
    ]
  }
}
```

**โครงสร้าง:** ชื่อ event → array ของ `{matcher, hooks: [{type, command}]}` — `matcher` กรองว่าจะรันตอนไหน (เช่น เฉพาะ tool `Edit`/`Write`) ปล่อยว่างไว้ = รันทุกครั้งที่ event นั้นเกิด

**ตำแหน่งไฟล์กำหนด scope:**

| ตำแหน่ง | ขอบเขต | แชร์ได้ไหม |
|---|---|---|
| `~/.claude/settings.json` | ทุกโปรเจกต์ของเรา | ไม่ (เฉพาะเครื่องเรา) |
| `.claude/settings.json` | โปรเจกต์เดียว | ได้ — commit ลง repo ได้ |
| `.claude/settings.local.json` | โปรเจกต์เดียว | ไม่ (gitignore อัตโนมัติ) |
| Managed policy settings | ทั้งองค์กร | ได้ — แอดมินคุม |

รัน `/hooks` ดู hook ที่ตั้งไว้ทั้งหมดแยกตาม event (เป็น read-only แก้ผ่านเมนูไม่ได้ ต้องแก้ไฟล์ settings ตรงๆ)

---

## 2. Event ที่ใช้บ่อยที่สุด (จากทั้งหมด 30+ event)

| Event | เกิดตอนไหน | บล็อกได้ไหม |
|---|---|---|
| **`PreToolUse`** | ก่อน tool call จะรัน | ✅ (exit 2) |
| **`PostToolUse`** | หลัง tool call สำเร็จ | ❌ (ทำไปแล้ว แก้คืนไม่ได้) |
| **`UserPromptSubmit`** | ทันทีที่ส่ง prompt ก่อน Claude ประมวลผล | ✅ (exit 2 ลบ prompt ทิ้ง) |
| **`Stop`** | Claude ตอบจบ turn | ✅ (exit 2 บังคับทำต่อ) |
| **`SessionStart`** | session เริ่ม/resume/หลัง compact | ❌ |
| **`Notification`** | Claude Code ส่ง notification (รอ input, permission prompt ฯลฯ) | ❌ |
| **`SubagentStop`** | subagent ทำงานเสร็จ | ❌ |
| **`PreCompact`/`PostCompact`** | ก่อน/หลัง context compaction | ❌ |

ที่เหลืออีกเยอะ (`WorktreeCreate`, `ConfigChange`, `FileChanged`, `PreModelSwitch` ฯลฯ) ดูรายการเต็มที่ [Hooks reference](https://code.claude.com/docs/en/hooks#hook-lifecycle)

---

## 3. กลไก exit code — หัวใจของการสื่อสารกับ Claude Code

Hook คุยกับ Claude Code ผ่าน **stdin (รับข้อมูล) + exit code/stdout (ส่งคำตอบกลับ)**

| Exit code | ผลลัพธ์ |
|---|---|
| **0** | ไม่มีข้อคัดค้าน — ปล่อยผ่านตามปกติ (หรือถ้า stdout เป็น JSON ที่ valid ใช้ JSON นั้นตัดสินแทน) |
| **2** | **บล็อก action นั้น** — เขียนเหตุผลลง stderr ส่งกลับให้ Claude ปรับตัว |
| อื่นๆ | ถือเป็น non-blocking error ปกติปล่อยผ่าน |

**ตัวอย่าง `PreToolUse` ที่บล็อกคำสั่งอันตราย:**

```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')

if echo "$COMMAND" | grep -q "drop table"; then
  echo "Blocked: dropping tables is not allowed" >&2
  exit 2
fi
exit 0
```

**อยากควบคุมละเอียดกว่า block/allow เฉยๆ** ใช้ exit 0 พร้อม print JSON แทน:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "ใช้ rg แทน grep เพื่อความเร็ว"
  }
}
```

`permissionDecision` มี 3 ค่า: **`allow`** (ข้าม permission prompt ไปเลย), **`deny`** (ยกเลิก + ส่งเหตุผลกลับ), **`ask`** (โชว์ prompt ให้คนตัดสินตามปกติ)

---

## 4. ตัวอย่างที่ใช้งานจริงบ่อยที่สุด

### Auto-format หลังแก้ไฟล์

```json
{
  "hooks": {
    "PostToolUse": [
      { "matcher": "Edit|Write", "hooks": [
        { "type": "command", "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write" }
      ]}
    ]
  }
}
```

### ป้องกันไฟล์สำคัญโดนแก้ (`.env`, `package-lock.json`, `.git/`)

```bash
#!/bin/bash
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
for pattern in ".env" "package-lock.json" ".git/"; do
  if [[ "$FILE_PATH" == *"$pattern"* ]]; then
    echo "Blocked: $FILE_PATH matches protected pattern '$pattern'" >&2
    exit 2
  fi
done
exit 0
```

ผูกกับ `PreToolUse` matcher `Edit|Write` — Claude ได้ feedback ตรงๆ ว่าทำไมโดนบล็อก ปรับวิธีต่อได้เอง

### แจ้งเตือนตอน Claude รอ input (จะได้ไม่ต้องเฝ้าจอ)

```json
{
  "hooks": {
    "Notification": [
      { "matcher": "", "hooks": [
        { "type": "command", "command": "notify-send 'Claude Code' 'ต้องการ input แล้ว'" }
      ]}
    ]
  }
}
```

---

## 5. เมื่อกฎตายตัวไม่พอ — Prompt-based / Agent-based hooks

นอกจาก `"type": "command"` (shell command ธรรมดา) ยังมี:

- **`"type": "prompt"`** — ส่ง input ไปให้โมเดล (default Haiku) ตัดสินแทน เหมาะกับเงื่อนไขที่ต้องใช้วิจารณญาณ ไม่ใช่กฎ if/else ตายตัว
- **`"type": "agent"`** — spawn subagent ที่อ่านไฟล์/รันคำสั่งเพื่อตรวจสอบสภาพจริงก่อนตัดสิน (experimental)

**ตัวอย่าง:** ใช้ `Stop` hook แบบ prompt เช็กว่างานเสร็จจริงหรือยังก่อนปล่อยให้ Claude หยุด:

```json
{
  "hooks": {
    "Stop": [{ "hooks": [
      { "type": "prompt", "prompt": "เช็กว่างานที่ขอทั้งหมดเสร็จหรือยัง ถ้ายังตอบ {\"ok\": false, \"reason\": \"สิ่งที่เหลือ\"}" }
    ]}]
  }
}
```

---

## 6. กับดัก

- **`Stop` hook บล็อกเกิน 8 ครั้งติดกัน → Claude Code overrides แล้วปล่อยให้จบ turn เอง** — ถ้า hook ต้องวนหลายรอบจริงๆ ต้องเช็ก field `stop_hook_active` ใน input แล้ว exit 0 ทันทีถ้าเป็น `true` ไม่งั้นชนเพดานเร็วเกินไป
- **`PostToolUse` แก้คืนไม่ได้** — tool รันไปแล้วจริง hook นี้ใช้ตรวจ/แจ้งเตือนได้เท่านั้น ไม่ใช่จุดป้องกัน ถ้าต้องกันไว้ก่อนต้องใช้ `PreToolUse`
- **หลาย hook รันพร้อมกันบน event เดียวกัน (ขนาน ไม่ใช่เรียงลำดับ)** — ถ้าตัวหนึ่ง `deny` อีกตัว `allow` → **`deny` ชนะเสมอ** แต่ side effect ของแต่ละ hook (เช่น log) ยังรันครบทุกตัวไม่ถูกยกเลิก
- **ลืม `chmod +x` ไฟล์ script** — hook ไม่รัน หรือ error "command not found" เงียบๆ
- **ใส่ field ผิดตำแหน่ง (top-level แทนที่จะอยู่ใน `hookSpecificOutput`)** — Claude Code parse JSON ผ่านแต่เมิน field ที่วางผิดที่แบบไม่มี error ให้เห็น เสียเวลา debug นาน
- **`echo` แบบไม่มีเงื่อนไขใน `.bashrc`/`.zshrc`** — ข้อความหลุดมาปนหน้า JSON output ทำให้ parse ไม่ผ่านทั้งที่ hook เขียนถูก ต้องครอบด้วยเช็ก `[[ $- == *i* ]]` (interactive shell เท่านั้น)

---

## 7. Cheat sheet

```json
{
  "hooks": {
    "<EventName>": [
      { "matcher": "ToolName|OtherTool", "hooks": [
        { "type": "command", "command": "your-script.sh" }
      ]}
    ]
  }
}
```

| exit code | ผล |
|---|---|
| 0 | ผ่าน (หรือใช้ JSON ตัดสินแทนถ้ามี) |
| 2 | บล็อก + ส่ง stderr กลับเป็น feedback |
| อื่นๆ | non-blocking error |

```
/hooks          → ดู hook ที่ตั้งไว้ทั้งหมด
chmod +x <script>  → ต้องทำเสมอก่อน hook จะรันได้
```

## 🔗 เกี่ยวข้อง

- [[Claude Code Best Practices]] — hooks คือหนึ่งในเครื่องมือ "ตั้งค่าสภาพแวดล้อมให้ Claude ทำงานได้ดีขึ้น"
- [[Claude Agent Lifecycle]] — จุดที่ hook เกาะอยู่ (`PreToolUse`/`PostToolUse` อยู่ใน tool-use loop, `PreCompact`/`PostCompact` อยู่ในการจัดการ context window)
- [[Claude Code Auto Mode]] — `PreToolUse` hook ทำงานคู่กับ permission mode ได้ (บล็อกได้แม้อยู่ใน bypass mode)

## 📖 อ่านต่อ

- [Automate actions with hooks](https://code.claude.com/docs/en/hooks-guide)
- [Hooks reference](https://code.claude.com/docs/en/hooks)
