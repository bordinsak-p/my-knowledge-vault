---
tags:
  - typescript
  - record
  - utility-types
type: reference
created: 2026-08-18
---

# 📦 TypeScript `Record<K, V>`

> `Record<K, V>` คือ **utility type** ที่บอกว่า "นี่คือ object ที่ทุก key เป็นชนิด `K` และทุก value เป็นชนิด `V`"
> มันไม่ใช่ runtime object พิเศษอะไร — ข้างใต้ก็คือ `object` ธรรมดา เพียงแต่ TypeScript บังคับรูปร่างให้เข้มขึ้น

**อย่าสับสนกับ `record` ของ Java** — คนละภาษา คนละแนวคิดเลย
Java `record` (ดู [[Java Record]]) คือคลาสข้อมูล immutable ที่มี field ตายตัว
TypeScript `Record<K, V>` คือ**รูปร่างของ dictionary/map** ที่ key เปลี่ยนได้ตามที่ประกาศ

---

## 1. นิยามของมัน

```ts
type Record<K extends string | number | symbol, V> = {
  [P in K]: V;
};
```

อ่านว่า "สำหรับทุก key `P` ที่อยู่ใน `K` ให้มี value เป็นชนิด `V`" — เป็น mapped type ที่ built-in มากับ TypeScript ไม่ต้อง import

---

## 2. ใช้พื้นฐาน

```ts
type Scores = Record<string, number>;

const scores: Scores = {
  math: 90,
  english: 85,
  science: 78,
};

console.log(scores.math);       // 90
console.log(scores["english"]); // 85
```

**output ที่ TypeScript ให้จริง — ลองใส่ key ผิดชนิด**

```ts
const bad: Scores = {
  math: "เก้าสิบ",   // ❌ ห้าม
};
```

```
Type 'string' is not assignable to type 'number'.
  Types of property 'math' are incompatible.
```

TypeScript จับได้ตั้งแต่ตอนเขียน ไม่ต้องรอรัน — นี่คือประโยชน์หลักเมื่อเทียบกับการประกาศ `{}` เฉย ๆ

---

## 3. คุมทั้ง key และ value ให้แคบลง

จุดที่ `Record` เก่งกว่า `{ [key: string]: V }` แบบเดิมคือ **`K` ระบุเป็น union literal ได้** แล้ว TypeScript จะบังคับให้มี key ครบทุกตัว ห้ามขาด ห้ามเกิน

```ts
type Day = "Mon" | "Tue" | "Wed" | "Thu" | "Fri" | "Sat" | "Sun";

const openHours: Record<Day, string> = {
  Mon: "09:00-18:00",
  Tue: "09:00-18:00",
  Wed: "09:00-18:00",
  Thu: "09:00-18:00",
  Fri: "09:00-18:00",
  Sat: "10:00-16:00",
  Sun: "ปิด",
};
```

**ลองลบ `Sun` ออก** — TypeScript จับทันที

```ts
const bad: Record<Day, string> = {
  Mon: "09:00-18:00",
  Tue: "09:00-18:00",
  Wed: "09:00-18:00",
  Thu: "09:00-18:00",
  Fri: "09:00-18:00",
  Sat: "10:00-16:00",
  // ลืม Sun
};
```

```
Property 'Sun' is missing in type
'{ Mon: string; Tue: string; Wed: string; Thu: string; Fri: string; Sat: string; }'
but required in type 'Record<Day, string>'.
```

**นี่คือเหตุผลหลักที่ควรใช้ `Record` กับ union แทน `{ [key: string]: V }`** — ตัวหลังไม่รู้ว่ามี key ไหนบ้าง ปล่อยให้ขาดได้เงียบ ๆ ส่วน `Record<Day, V>` บังคับให้ครบทุก key ตั้งแต่ compile time

---

## 4. เคสที่ใช้บ่อยจริง — map สถานะเป็น label/สี

```ts
type OrderStatus = "PENDING" | "PAID" | "SHIPPED" | "CANCELLED";

const statusLabel: Record<OrderStatus, string> = {
  PENDING: "รอชำระเงิน",
  PAID: "ชำระแล้ว",
  SHIPPED: "จัดส่งแล้ว",
  CANCELLED: "ยกเลิก",
};

const statusColor: Record<OrderStatus, string> = {
  PENDING: "#f59e0b",
  PAID: "#3b82f6",
  SHIPPED: "#10b981",
  CANCELLED: "#ef4444",
};

function render(status: OrderStatus) {
  console.log(`${statusLabel[status]} (${statusColor[status]})`);
}

render("PAID");
```

**output:**
```
ชำระแล้ว (#3b82f6)
```

**ทำไมดีกว่า `if/else` หรือ `switch`:** ถ้าเพิ่ม status ใหม่เข้า union แล้วลืมเติมใน `statusLabel` — TypeScript จะฟ้อง "missing property" ทันที ไม่ต้องรอไปเจอตอน UI ขึ้น `undefined` บนหน้าจอจริง

เทียบกับ `switch` ที่ลืม `case` ใหม่แล้ว**เงียบ ไม่มีอะไรเตือนเลย**

```ts
// ❌ เพิ่ม "REFUNDED" เข้า OrderStatus แล้วลืมเติม case — compile ผ่านสบาย ๆ
function label(s: OrderStatus): string {
  switch (s) {
    case "PENDING": return "รอชำระเงิน";
    case "PAID": return "ชำระแล้ว";
    case "SHIPPED": return "จัดส่งแล้ว";
    case "CANCELLED": return "ยกเลิก";
  }
  return "";   // ← ไม่มีใครเตือนว่า REFUNDED หลุดมาตกที่นี่
}
```

---

## 5. ใช้เป็น lookup table แทน `if/else` ยาว ๆ

```ts
type Role = "admin" | "editor" | "viewer";

const canEdit: Record<Role, boolean> = {
  admin: true,
  editor: true,
  viewer: false,
};

const canDelete: Record<Role, boolean> = {
  admin: true,
  editor: false,
  viewer: false,
};

function checkPermission(role: Role) {
  console.log(`role=${role} edit=${canEdit[role]} delete=${canDelete[role]}`);
}

checkPermission("editor");
```

**output:**
```
role=editor edit=true delete=false
```

อ่านง่ายกว่า `if (role === "admin" || role === "editor") { ... }` ที่กระจายอยู่หลายจุด และแก้สิทธิ์แค่แก้ตารางที่เดียว

---

## 6. index signature แบบ dynamic — เมื่อ key ไม่รู้ล่วงหน้า

ถ้า key ไม่ใช่ union ตายตัว แต่เป็น `string` ทั่วไป (เช่น รหัสสินค้าที่ไม่รู้ล่วงหน้า) `Record<string, V>` ยังใช้ได้ แต่ TypeScript ไม่รู้ว่า key ไหนมีจริง

```ts
const inventory: Record<string, number> = {
  P001: 10,
  P002: 25,
};

console.log(inventory.P001);      // 10  (type: number)
console.log(inventory.P999);      // undefined ตอนรัน — แต่ type บอกว่าเป็น number!
```

```
10
undefined
```

**นี่คือกับดัก** — TypeScript บอก type ของ `inventory.P999` เป็น `number` ทั้งที่จริง ๆ รันแล้วได้ `undefined` เพราะ index signature ปกติ**ไม่เผื่อ key ที่ไม่มีอยู่จริง**

### แก้ด้วย `noUncheckedIndexedAccess`

เปิด flag นี้ใน `tsconfig.json`

```json
{
  "compilerOptions": {
    "noUncheckedIndexedAccess": true
  }
}
```

พอเปิดแล้ว type ของ `inventory.P999` จะกลายเป็น `number | undefined` บังคับให้เช็คก่อนใช้

```ts
const qty = inventory["P999"];
console.log(qty.toFixed(0));   // ❌ error ทันที: 'qty' is possibly 'undefined'
```

```
'qty' is possibly 'undefined'.
```

ต้องเช็คก่อนเสมอ

```ts
const qty = inventory["P999"] ?? 0;
console.log(qty.toFixed(0));   // ✅ ผ่าน
```

**แนะนำเปิด flag นี้ไว้ทุกโปรเจกต์ที่ใช้ `Record<string, V>` กับ key ที่ไม่รู้ล่วงหน้า** — ไม่งั้น `undefined` จะหลุดเข้าไปในโค้ดที่คิดว่าปลอดภัยแล้ว

---

## 7. สร้าง `Record` จาก array — `Object.fromEntries`

```ts
interface User { id: string; name: string; }

const users: User[] = [
  { id: "u1", name: "Somchai" },
  { id: "u2", name: "Malee" },
];

const byId: Record<string, User> = Object.fromEntries(
  users.map(u => [u.id, u])
);

console.log(byId["u1"]);
console.log(byId["u2"].name);
```

**output:**
```
{ id: 'u1', name: 'Somchai' }
Malee
```

แปลง array เป็น lookup table แบบนี้บ่อยมากตอนโหลดข้อมูลมาจาก API แล้วอยากค้นด้วย id เร็ว ๆ (O(1) แทนที่จะ `.find()` ทุกครั้งซึ่งเป็น O(n))

---

## 8. `Partial<Record<K, V>>` — เมื่อไม่อยากบังคับให้ครบทุก key

บางทีอยากได้ type ที่ **อนุญาตให้ key ขาดได้** แต่ยัง type-safe เรื่อง key ที่มี — ห่อด้วย `Partial`

```ts
type Day = "Mon" | "Tue" | "Wed" | "Thu" | "Fri" | "Sat" | "Sun";

// อาจมีแค่บางวัน ไม่บังคับครบ
const specialHours: Partial<Record<Day, string>> = {
  Sat: "10:00-14:00",   // เปิดแค่วันเสาร์วันเดียว ไม่ต้องประกาศวันอื่น
};

console.log(specialHours.Sat);   // "10:00-14:00"
console.log(specialHours.Mon);   // undefined
```

```
10:00-14:00
undefined
```

**เทียบความต่างระหว่างสามแบบ**

| type | key ต้องครบไหม | value เป็น `| undefined` ไหม |
|---|---|---|
| `Record<Day, string>` | ✅ ต้องครบทุก key | ❌ ไม่ |
| `Partial<Record<Day, string>>` | ❌ ขาดได้ | ✅ ทุก key เป็น `string \| undefined` |
| `Record<string, string>` | ไม่เกี่ยว (key ไม่รู้ล่วงหน้า) | ขึ้นกับ `noUncheckedIndexedAccess` |

---

## 9. `Record` vs `interface` vs `Map` — เลือกยังไง

| | `Record<K, V>` | `interface` / `type {}` | `Map<K, V>` |
|---|---|---|---|
| ทุก property type เดียวกัน | ✅ ใช่พอดี | ถ้า property มีชนิดต่างกัน ใช้ไม่ได้ | ใช่ก็ได้ |
| รู้ชื่อ property ล่วงหน้า | ใช้ได้เมื่อ key เป็น union | ✅ เหมาะที่สุด | ไม่ต้องรู้ล่วงหน้า |
| key เป็นอะไรก็ได้ (object, ตัวเลขจริง) | ❌ ต้องเป็น string/number/symbol | ❌ | ✅ key เป็นอะไรก็ได้ |
| serialize เป็น JSON ตรง ๆ | ✅ | ✅ | ❌ ต้องแปลงเอง |
| มี method `.has()` `.delete()` built-in | ❌ | ❌ | ✅ |
| รักษาลำดับการใส่ (insertion order) | ไม่การันตี | ไม่การันตี | ✅ การันตี |

```ts
// ✅ ใช้ Record — property ทุกตัวเป็นตัวเลขเหมือนกันหมด รู้ key ล่วงหน้า
type Scores = Record<"math" | "eng" | "sci", number>;

// ✅ ใช้ interface — แต่ละ property คนละชนิด
interface Student {
  name: string;
  age: number;
  active: boolean;
}

// ✅ ใช้ Map — key เป็น object หรือไม่รู้ล่วงหน้าและต้องการ .has()/.delete()
const cache = new Map<string, User>();
cache.set("u1", { id: "u1", name: "Somchai" });
console.log(cache.has("u1"));   // true
```

```
true
```

---

## 10. ต่อยอด — `Record` ผสมกับ utility type อื่น

```ts
type Status = "active" | "inactive" | "pending";

// ทุก value เป็น readonly ห้ามแก้หลังประกาศ
const config: Readonly<Record<Status, number>> = {
  active: 1,
  inactive: 0,
  pending: 2,
};

config.active = 5;   // ❌
```

```
Cannot assign to 'active' because it is a read-only property.
```

```ts
// Record ซ้อน Record — เช่น matrix สิทธิ์ role × action
type Role = "admin" | "user";
type Action = "read" | "write" | "delete";

const permissions: Record<Role, Record<Action, boolean>> = {
  admin: { read: true, write: true, delete: true },
  user:  { read: true, write: false, delete: false },
};

console.log(permissions.admin.delete);   // true
console.log(permissions.user.write);     // false
```

```
true
false
```

---

## 11. กับดักที่เจอบ่อย

- **`Record<string, V>` ไม่การันตีว่า key นั้นมีจริง** — เปิด `noUncheckedIndexedAccess` เสมอ (ข้อ 6)
- **ลืมว่า `Record` เป็นแค่ type ไม่ใช่ runtime validator** — `JSON.parse(text) as Record<Day, string>` ไม่ได้เช็คจริงว่าข้อมูลตรงตามที่บอก TypeScript เชื่อไปตามที่ `as` บอกเฉย ๆ ถ้าข้อมูลจาก API ไม่ตรงจริง จะพังตอนรันไม่ใช่ตอน compile
- **สร้าง `Record<UnionใหญยาวยากๆV>` มือแล้วลืม key** — ให้ TypeScript ช่วยจับ (ข้อ 3) แทนที่จะเขียนแล้วมานั่งไล่ตาเช็คเอง
- **ใช้ `Record<string, any>` แทน type ที่ควรรู้ล่วงหน้า** — เสียประโยชน์ของ TypeScript ไปเกือบหมด เท่ากับเขียน JavaScript ที่ใส่ type annotation หลอก ๆ

---

## 🔗 เกี่ยวข้อง

- [[Java Record]] — คนละแนวคิด อย่าสับสนชื่อ

## 📖 อ่านต่อ

- [TypeScript Handbook — Record\<Keys, Type\>](https://www.typescriptlang.org/docs/handbook/utility-types.html#recordkeys-type)
- [TypeScript — noUncheckedIndexedAccess](https://www.typescriptlang.org/tsconfig#noUncheckedIndexedAccess)
