---
tags:
  - angular
  - version-updates
  - changelog
type: reference
created: 2026-09-17
---

# 🆕 Angular — Version Updates ล่าสุด

> โน้ตนี้ติดตาม **"อะไรใหม่/อะไรถูกเลิกใช้"** ของ Angular แต่ละเวอร์ชัน ไม่ใช่โน้ตอธิบายฟีเจอร์เชิงลึก (ไปดู [[Angular Outline]] สำหรับโน้ตแยกตามหัวข้อ) — **โน้ตนี้ล้าสมัยเร็วกว่าโน้ตอื่นในวอลต์มาก เพราะ Angular ออก major version ใหม่ทุก ~6 เดือน** เช็กวันที่ด้านล่างก่อนเชื่อ 100% เสมอ

> **อัปเดตล่าสุด: 2026-09-17 — เวอร์ชันล่าสุดคือ Angular 22 (ออก 3 มิ.ย. 2026)**

---

## 1. สถานะแต่ละเวอร์ชัน ณ วันที่บันทึก

| เวอร์ชัน | ออกเมื่อ | สถานะ (ณ 2026-09-17) |
|---|---|---|
| **22** | 3 มิ.ย. 2026 | ✅ เวอร์ชันล่าสุด (current) |
| 21 | 20 พ.ย. 2025 | Active support |
| 20 | พ.ค. 2025 | LTS (ถึง ~พ.ย. 2026) |
| 19 | พ.ย. 2024 | LTS หมดอายุแล้ว (19 พ.ค. 2026) |

Angular ออก major release ทุก **~6 เดือน** ตายตัว (พฤศจิกายน/พฤษภาคม) — โปรเจกต์ production ควรอยู่ในช่วง LTS อย่างน้อย โน้ตอื่นในวอลต์นี้ส่วนใหญ่เขียนอ้างอิงยุค "17+" (ดู [[Angular Outline]]) ซึ่งยังใช้ได้ปกติ ไม่มีอะไรพัง

---

## 2. Angular 22 (มิ.ย. 2026) — เวอร์ชันล่าสุด

**ธีมหลัก: consolidation release** — ไม่ได้ออกฟีเจอร์ใหม่กองใหญ่ แต่เอาของที่ค้างสถานะ experimental/developer-preview มานานผลักให้ stable รวดเดียวหลายตัว

### ใหม่ / เพิ่งเป็น stable

- **Signal Forms → stable** (เคย experimental ใน v21) — ระบบฟอร์มใหม่ทั้งหมด **ไม่ใช่ส่วนต่อยอดของ** [[Angular Reactive Forms]] แต่เป็นสถาปัตยกรรมใหม่บน signal ทั้งก้อน (data model เป็น source of truth ฟอร์มเป็นแค่ projection ของมัน) — ยังไม่มีโน้ตแยกอธิบายในวอลต์นี้
- **Angular Aria (`@angular/aria`) → stable** — accessible UI pattern สำเร็จรูป (keyboard/screen reader/mouse) พร้อม test harness
- **Resource API → stable** — `resource()`, `rxResource()`, `httpResource()` ใช้ใน production ได้แล้ว (ดึงข้อมูล async แบบผูกกับ signal โดยตรง)
- **`OnPush` กลายเป็นค่า default ของ change detection strategy** — ผลกระทบกว้างสุดของเวอร์ชันนี้ เพราะเปลี่ยนพฤติกรรม default ของทุกโปรเจกต์ใหม่ (ยังไม่มีโน้ตอธิบายเรื่อง change detection ในวอลต์นี้เลย)
- Template syntax เพิ่ม: spread/rest syntax, `@switch` จับหลาย case พร้อมกันได้ + เช็ก `never` แบบ exhaustive, inline arrow function ใน template, ใส่ comment ในแท็ก element ได้
- Angular MCP Server (เครื่องมือช่วย AI coding ใน VS Code/Cursor) พัฒนาต่อเนื่องจาก v21

---

## 3. Angular 21 (พ.ย. 2025)

### ใหม่

- **Signal Forms** เปิดตัว (ตอนนั้น experimental, เพิ่ง stable ใน v22 — ข้อ 2)
- **Zoneless change detection กลายเป็น default สำหรับโปรเจกต์ใหม่** — สร้างแอปใหม่แล้วไม่มี `zone.js` มาด้วยอัตโนมัติ (เดิมต้องเปิดเองแบบ opt-in)
- **Vitest แทน Karma เป็น test runner default** และเป็น stable เต็มตัว
- **`HttpClient` provide เข้า root injector โดย default** — เรียก `provideHttpClient()` ธรรมดา (ดู [[Angular HttpClient]]) ไม่ต้องตั้งอะไรเพิ่มเหมือนก่อน
- Angular Aria เปิดตัวครั้งแรก (ตอนนั้น preview, stable ใน v22)
- Angular MCP Server เปิดตัวครั้งแรก

### Deprecated / Removed

- **HammerJS ถูกถอดออกจริง** (เคย deprecated ตั้งแต่ v20) — component ที่ยัง import `HammerModule` จะ error ทันทีหลังอัปเกรด
- **Jest และ Web Test Runner ถูก deprecate** (มีกำหนดถอดจริงใน v22) — ใครยังใช้ Karma/Jasmine อยู่ใช้ได้ปกติ ไม่ต้องรีบย้าย

---

## 4. Angular 20 (พ.ค. 2025)

### ใหม่ / เพิ่งเป็น stable

- **`computed()`/`effect()`/`afterEveryRender`/`afterNextRender` → stable** (ดู [[Angular Signal Primitives]] ที่เขียนอ้างอิงฟีเจอร์กลุ่มนี้)
- Zoneless เลื่อนจาก experimental → developer preview
- Template syntax: exponentiation operator (`**`), tagged template literal, `void` ใช้ใน template expression ได้

### Deprecated / Removed

- **View Engine ถูกถอดออกทั้งหมด** — เหลือแค่ Ivy renderer เท่านั้น (โปรเจกต์เก่ามากที่ยังลาก View Engine มา อัปเกรดข้ามมาตรงนี้พังแน่นอน)
- **`InjectFlags` API ถูกถอดออก** (เคย deprecated มาก่อนแล้ว)
- **HammerJS ถูก deprecate** (ถอดออกจริงใน v21 — ข้อ 3)
- **`@angular/platform-browser-dynamic` deprecate** — ฟังก์ชันย้ายไปรวมกับ `@angular/platform-browser` แล้ว
- **`ng-reflect-*` attribute ไม่ถูก generate ให้อัตโนมัติอีกต่อไป** — breaking change ที่กระทบ e2e test เก่าที่เคยอิง attribute พวกนี้

---

## 5. ผลกระทบต่อโน้ตอื่นในวอลต์นี้

| โน้ตเดิม | อาจต้องปรับปรุงยังไง |
|---|---|
| [[Angular Reactive Forms]] | ยังใช้ได้ปกติ ไม่ถูกแทนที่ — แต่ควรเพิ่มหมายเหตุว่ามี **Signal Forms** เป็นทางเลือกใหม่คู่ขนานแล้ว |
| [[Angular Signal Primitives]] | เนื้อหายังถูกต้องทั้งหมด `computed()`/`effect()` แค่ตอนนี้ "stable" เป็นทางการมากขึ้น (ตั้งแต่ v20) |
| [[Angular Outline]] | ทุกโน้ตที่เขียนด้วยรูปแบบ decorator (`@Input`/`@Output`) ยังใช้ได้ปกติ — Angular ไม่เคยถอด backward compat ส่วนนี้ |

**ยังไม่มีโน้ตแยกสำหรับ:** Signal Forms, Angular Aria, `OnPush`/change detection — ตัวสุดท้ายสำคัญขึ้นมากเพราะกลายเป็น **default** ไปแล้วใน v22

---

## 6. กับดัก

- **เชื่อตัวอย่างโค้ดเก่าที่ตั้งใจใส่ `zone.js`/`NgModule` เอง** — โปรเจกต์ที่สร้างด้วย `ng new` ตั้งแต่ v21 ขึ้นไปไม่มีของสองอย่างนี้มาให้โดย default อีกแล้ว
- **อัปเกรดข้ามหลาย major version ทีเดียว** (เช่น v17 ตรงไป v22) — เสี่ยงชนกับ breaking change สะสมหลายตัวพร้อมกัน ควรใช้ `ng update` ทีละ major version เสมอ ไม่ข้าม
- **ใช้ `HammerJS` อยู่แล้วอัปเกรดเป็น v21+ โดยไม่เตรียมตัว** — component ที่ import `HammerModule` error ทันที (ข้อ 3)
- **คิดว่า Signal Forms จะมาแทน Reactive Forms ทันที** — ปัจจุบันเป็นแค่ "ทางเลือกใหม่คู่ขนาน" ไม่ใช่การ deprecate reactive forms เดิม โปรเจกต์ที่ใช้ reactive forms อยู่ไม่จำเป็นต้องรีบย้าย

---

## 7. Cheat sheet — เช็กเวอร์ชันตัวเอง

```bash
ng version                             # ดูเวอร์ชัน Angular ของโปรเจกต์ปัจจุบัน
ng update @angular/core @angular/cli   # อัปเกรดทีละ major version พร้อม migration script อัตโนมัติ
```

| ต้องการรู้ | ดูที่ |
|---|---|
| เวอร์ชันไหนยัง LTS อยู่ | [angular.dev/reference/releases](https://angular.dev/reference/releases) |
| รายการ deprecation ทั้งหมด | Angular release notes ต่อเวอร์ชันบน GitHub |
| migrate ข้าม major version | `ng update` (รันทีละเวอร์ชัน ไม่ข้าม) |

## 🔗 เกี่ยวข้อง

- [[Angular Outline]] — โน้ตเชิงลึกแยกตามหัวข้อ (อ้างอิงยุค 17+ ส่วนใหญ่ยังใช้ได้ปกติ)
- [[Angular Signal Primitives]] — `computed()`/`effect()` ที่เพิ่งเป็น stable ทางการใน v20

## 📖 อ่านต่อ

- [Angular — Versioning and releases](https://angular.dev/reference/releases)
- [Angular v22 Release](https://angular.dev/events/v22)
- [Angular 21 – What's New — Angular.love](https://angular.love/angular-21-whats-new)
