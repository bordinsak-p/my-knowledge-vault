---
tags:
  - playwright
  - testing
  - e2e
type: reference
created: 2026-09-17
---

# 🎭 Playwright — E2E Testing

> **Playwright คือ framework ทดสอบแบบ end-to-end (E2E)** — เปิด browser จริง (Chromium/Firefox/WebKit) แล้วจำลองผู้ใช้จริงคลิก/พิมพ์/เช็กผลลัพธ์ ต่างจาก unit test ที่เทสฟังก์ชัน/component แยกส่วน E2E เทส **ทั้ง flow** ตั้งแต่ UI จริงไปจนถึง backend จริง

---

## 1. ติดตั้ง

```bash
npm init playwright@latest
```

ได้ config + example test + ติดตั้ง browser ให้ครบอัตโนมัติ (ไม่ต้องลง Chromium/Firefox เอง)

---

## 2. โครงสร้าง test พื้นฐาน

```typescript
import { test, expect } from '@playwright/test';

test('login แล้วเห็นหน้า dashboard', async ({ page }) => {
  await page.goto('https://example.com/login');
  await page.getByLabel('อีเมล').fill('user@example.com');
  await page.getByLabel('รหัสผ่าน').fill('password123');
  await page.getByRole('button', { name: 'เข้าสู่ระบบ' }).click();

  await expect(page.getByText('ยินดีต้อนรับ')).toBeVisible();
});
```

`page` คือ fixture ที่ Playwright inject ให้อัตโนมัติ — แต่ละ test ได้ browser context ใหม่ทุกครั้ง แยกจาก test อื่นโดยสมบูรณ์ (cookie/localStorage ไม่ปนกัน)

---

## 3. Locator — หัวใจของ Playwright

**Auto-waiting:** ทุก action (`click`/`fill`/...) รอเองจนกว่า element จะ attached + visible + stable + enabled ถึงจะทำ ไม่ต้องเขียน sleep/wait เอง

**ลำดับ locator ที่ Playwright แนะนำ (บนสุด = ควรใช้ก่อน):**

| ลำดับ | Locator | ใช้เมื่อ |
|---|---|---|
| 1 | `getByRole('button', { name: '...' })` | อิง ARIA role — ทนต่อการเปลี่ยน DOM/CSS มากที่สุด |
| 2 | `getByLabel('...')` | form field ที่มี label |
| 3 | `getByPlaceholder('...')` | input ที่มีแค่ placeholder |
| 4 | `getByText('...')` | เนื้อหาข้อความ |
| 5 | `getByTestId('...')` | ใส่ `data-testid` เองตอนไม่มีทางอื่นจริง ๆ |
| ❌ | CSS/XPath selector | หลีกเลี่ยง — พังง่ายสุดเมื่อ UI เปลี่ยน |

**ทำไม role-based ดีที่สุด:** อิงสิ่งที่ผู้ใช้จริง (และ screen reader) มองเห็น ไม่ใช่ implementation detail อย่าง class name ที่เปลี่ยนบ่อยตอน refactor CSS

---

## 4. Assertion แบบ web-first

```typescript
await expect(page.getByText('บันทึกสำเร็จ')).toBeVisible();
await expect(page.getByRole('button', { name: 'ส่ง' })).toBeDisabled();
await expect(page).toHaveURL(/dashboard/);
```

`expect(locator).toXxx()` **auto-retry เองจนกว่าจะ timeout** (default 5 วิ) — ต่างจาก assertion ทั่วไปที่เช็กครั้งเดียวแล้วจบ เหมาะกับ UI ที่ update แบบ async (เช่นรอ API ตอบก่อนข้อความจะโผล่มา)

---

## 5. เครื่องมือ debug ที่มากับตัว

| เครื่องมือ | ใช้ทำอะไร |
|---|---|
| `npx playwright codegen <url>` | เปิด browser ให้คลิกเอง แล้ว **generate โค้ด test ให้อัตโนมัติ** จากสิ่งที่คลิก |
| `npx playwright test --ui` | UI mode — รัน test แบบเห็นทุกขั้นตอน time-travel ดูแต่ละ step ได้ |
| `npx playwright show-trace trace.zip` | Trace Viewer — ดู network/console/DOM snapshot ก่อน-หลังทุก action ตอน test fail |
| `npx playwright test --debug` | เปิด Playwright Inspector หยุดทีละบรรทัด |

---

## 6. Cross-browser + parallel — ตั้งใน config เดียว

```typescript
// playwright.config.ts
export default defineConfig({
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
  ],
  fullyParallel: true,
});
```

test ชุดเดียว รันได้ทั้ง 3 browser engine พร้อมกันโดยไม่ต้องแก้โค้ด test เลย

---

## 7. รันใน CI — GitHub Actions

```yaml
- uses: actions/checkout@v4
- uses: actions/setup-node@v4
- run: npm ci
- run: npx playwright install --with-deps
- run: npx playwright test
- uses: actions/upload-artifact@v4
  if: always()
  with: { name: playwright-report, path: playwright-report/ }
```

ดู syntax เพิ่มที่ [[GitHub Actions Workflow Syntax]] — `if: always()` สำคัญ เพื่อให้ได้ report แม้ test fail

---

## 8. Playwright ต่างจาก unit test ยังไง — เทสคนละชั้น

| | Unit test (เช่น `TestBed` ของ Angular) | Playwright (E2E) |
|---|---|---|
| เทสอะไร | component/service แยกส่วน (mock dependency) | flow เต็มผ่าน UI จริง + backend จริง |
| เร็วแค่ไหน | เร็วมาก (ms) | ช้ากว่ามาก (วินาที ต้องเปิด browser จริง) |
| จำนวนที่ควรมี | เยอะที่สุด (ฐานของพีระมิด) | น้อยกว่า เฉพาะ flow สำคัญ |
| เจอบั๊กแบบไหน | logic ผิดในฟังก์ชัน/component เดียว | integration พัง (frontend เรียก backend ไม่ตรง, routing ผิด) |

**ใช้คู่กัน ไม่ใช่เลือกอย่างใดอย่างหนึ่ง** — unit test เยอะ ๆ คุม logic รายจุด, Playwright คุมเฉพาะ flow สำคัญจริง ๆ ไม่กี่ทาง (login, checkout, ฟีเจอร์หลัก) ไม่ต้อง E2E ทุกหน้าเพราะช้าและเปราะกว่า unit test มาก

---

## 🎯 แนวทางการเริ่มศึกษา

1. **เริ่มต้น** — เข้าใจว่า Playwright เทสคนละชั้นกับ unit test (ข้อ 8) และ auto-waiting ทำงานยังไง (ข้อ 3) ก่อนเขียน test เส้นแรก จะได้ไม่หลงไปใช้ pattern ของ unit test มาผสม
2. **ลงมือทำจริง** — รัน `npm init playwright@latest` แล้วใช้ `npx playwright codegen <url>` คลิกในแอปจริงของตัวเอง 1 flow (เช่น login) ดูโค้ดที่ generate ให้ก่อนค่อยแก้เอง — เร็วกว่าเขียน test เปล่าตั้งแต่ศูนย์มาก
3. **ใช้งานได้คล่อง** — เปลี่ยน locator ที่ codegen generate มาเป็นแบบ CSS/XPath ให้เป็น role-based locator เอง (ข้อ 3), ต่อเข้า CI พร้อม upload trace ตอน fail (ข้อ 7), และเลือกเฉพาะ critical flow มาทำ E2E ไม่พยายามครอบคลุมทุกหน้า

---

## 9. กับดัก

- **ใช้ CSS selector/XPath ที่ผูกกับ implementation** — พังทันทีที่ refactor class name แม้ UI จะเหมือนเดิมทุกอย่าง ใช้ role-based locator แทน (ข้อ 3)
- **เขียน `page.waitForTimeout(3000)` เอง** — Playwright auto-wait อยู่แล้ว ใส่ sleep เองทำให้ test ช้าโดยไม่จำเป็นและไม่ reliable ไปกว่าเดิม
- **แชร์ state (เช่น login session) ข้าม test โดยไม่ตั้งใจ** — แต่ละ test ควร independent กันได้ สลับลำดับรันแล้วผลต้องเหมือนเดิม
- **เขียน E2E ครอบคลุมทุกหน้า ทุก edge case** — ช้าและเปราะ เลือกเฉพาะ critical flow เท่านั้น (ข้อ 8)
- **ไม่ดู Trace ตอน test fail ใน CI** — เดางมโดยไม่มีหลักฐาน ทั้งที่ trace file มี network/console/DOM ให้ดูครบ (ข้อ 5)

---

## 10. Cheat sheet

```bash
npm init playwright@latest
npx playwright test                  # รันทั้งหมด
npx playwright test --ui             # UI mode
npx playwright codegen <url>         # generate test จากการคลิกเอง
npx playwright show-trace trace.zip  # ดู trace
```

```typescript
await page.goto(url);
await page.getByRole('button', { name: 'X' }).click();
await page.getByLabel('X').fill('value');
await expect(page.getByText('X')).toBeVisible();
```

## 🔗 เกี่ยวข้อง

- [[Playwright MCP]] — ใช้ Playwright ผ่าน AI agent แบบภาษาธรรมชาติ แทนเขียนโค้ดเอง
- [[GitHub Actions Workflow Syntax]] — รัน Playwright ใน CI
- [[Angular Outline]] — `TestBed` (unit test) คนละชั้นกับ E2E นี้ (ข้อ 8)

## 📖 อ่านต่อ

- [Playwright — Getting Started](https://playwright.dev/docs/intro)
- [Playwright — Locators](https://playwright.dev/docs/locators)
- [Playwright — Best Practices](https://playwright.dev/docs/best-practices)
