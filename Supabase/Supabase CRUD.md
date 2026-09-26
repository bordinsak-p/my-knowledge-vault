---
tags:
  - supabase
  - javascript
  - typescript
type: reference
created: 2026-09-19
---

# 🗄️ Supabase JS Client — CRUD

> โน้ตนี้คือ reference คำสั่ง CRUD ของ `@supabase/supabase-js` (query builder ที่คุยกับ Postgres ผ่าน REST ให้อัตโนมัติ — ดูภาพรวมทั้งระบบที่ [[Supabase]]) ยังไม่รวม Auth/Storage/Realtime — แยกเป็นโน้ตอื่นทีหลัง

---

## 0. กฎที่ต้องรู้ก่อนอย่างอื่น — insert/update/upsert/delete ไม่ return แถวให้เป็นค่า default

**`insert()`/`update()`/`upsert()`/`delete()` ไม่คืนแถวที่ถูกแก้ไขมาให้เองแล้ว** — ต้องต่อ `.select()` ท้ายสุดเสมอถ้าต้องการข้อมูลกลับมา (ตัวอย่างเก่าบนเว็บจำนวนมากยังเขียนไม่มี `.select()` ต่อท้าย ใช้ไม่ได้กับพฤติกรรมปัจจุบัน)

```ts
// ❌ ได้แค่ error กลับมา data เป็น undefined เสมอ
const { error } = await supabase.from('products').insert({ name: 'Laptop' });

// ✅ ต่อ .select() ถึงจะได้แถวที่เพิ่ง insert กลับมาจริง
const { data, error } = await supabase.from('products').insert({ name: 'Laptop' }).select();
```

---

## 1. Create — `insert()`

```ts
// เพิ่มแถวเดียว
const { data, error } = await supabase
  .from('products')
  .insert({ name: 'Laptop', price: 999 })
  .select();

// เพิ่มหลายแถวพร้อมกันในคำสั่งเดียว
const { data, error } = await supabase
  .from('products')
  .insert([
    { name: 'Laptop', price: 999 },
    { name: 'Mouse', price: 19 },
  ])
  .select();
```

`insert()` รับได้ทั้ง object เดียว (แถวเดียว) หรือ array ของ object (หลายแถวพร้อมกัน — เร็วกว่าวน insert ทีละแถวมาก เพราะเป็น request เดียว)

---

## 2. Read — `select()`

```ts
const { data, error } = await supabase.from('products').select('*');                    // ทุกคอลัมน์
const { data, error } = await supabase.from('products').select('id, name, price');      // เฉพาะบางคอลัมน์

// join ตารางที่เกี่ยวข้อง (foreign key) ได้ในคำสั่งเดียว ไม่ต้อง query แยก
const { data, error } = await supabase
  .from('orders')
  .select(`
    id,
    total,
    products ( name, price )
  `);
```

### Filter — ต่อท้าย `select()` ได้เหมือน SQL `WHERE`

```ts
await supabase.from('products').select('*').eq('category', 'electronics');   // =
await supabase.from('products').select('*').neq('status', 'archived');       // !=
await supabase.from('products').select('*').gt('price', 1000);               // >
await supabase.from('products').select('*').lt('price', 1000);               // <
await supabase.from('products').select('*').like('name', '%phone%');         // pattern match
await supabase.from('products').select('*').in('category', ['electronics', 'books']);
await supabase.from('products').select('*').is('deleted_at', null);          // เช็ค NULL ต้องใช้ .is() ไม่ใช่ .eq()
```

### Modifier — เรียง/จำกัดผลลัพธ์

```ts
await supabase.from('products').select('*').order('price', { ascending: false });
await supabase.from('products').select('*').limit(10);
await supabase.from('products').select('*').range(0, 9);   // pagination — แถวที่ 0-9

const { data } = await supabase.from('products').select('*').eq('id', 1).single();       // ต้องเจอ 1 แถวพอดี ไม่งั้น error
const { data } = await supabase.from('products').select('*').eq('id', 1).maybeSingle();  // ไม่เจอเลย = คืน null ไม่ error
```

**`single()` vs `maybeSingle()`:** `single()` error ถ้าเจอ 0 แถวหรือมากกว่า 1 แถว — ใช้ตอนมั่นใจว่าต้องมีแถวเดียวเป๊ะ (query ด้วย primary key) ส่วน `maybeSingle()` ไม่ error ถ้าเจอ 0 แถว (คืน `null` เฉย ๆ) error แค่ตอนเจอมากกว่า 1 — เหมาะกับกรณี "อาจจะไม่เจอก็ได้" เช่นเช็คว่า username ถูกใช้ไปแล้วหรือยัง

### นับจำนวนแถวโดยไม่ต้องดึงข้อมูลจริงมาทั้งหมด

```ts
const { count } = await supabase.from('products').select('*', { count: 'exact', head: true });
```

`head: true` บอกว่าไม่ต้องส่งข้อมูลจริงกลับมาเลย เอาแค่ตัวเลข count — เร็วกว่า select ข้อมูลมาทั้งหมดแล้วนับเอง

---

## 3. Update — `update()`

```ts
const { data, error } = await supabase
  .from('products')
  .update({ price: 899 })
  .eq('id', 1)
  .select();
```

**ต้องมี filter เสมอ** (เช่น `.eq('id', 1)`) — ถ้าไม่ใส่ filter เลย `update()` จะแก้**ทุกแถวในตาราง** ทันที

---

## 4. Upsert — `insert()` หรือ `update()` ในคำสั่งเดียว

```ts
const { data, error } = await supabase
  .from('products')
  .upsert({ id: 1, name: 'Laptop', price: 999 }, { onConflict: 'id' })
  .select();
```

ถ้ายังไม่มีแถวที่ `id = 1` → insert ใหม่ ถ้ามีอยู่แล้ว → update ทับ — `onConflict` บอกว่าคอลัมน์ไหนใช้เช็คว่า "ซ้ำ" (ปกติคือ primary key หรือ unique column)

---

## 5. Delete — `delete()`

```ts
const { data, error } = await supabase
  .from('products')
  .delete()
  .eq('id', 1)
  .select();
```

**ต้องมี filter เสมอเช่นกัน** — ไม่ใส่ filter = ลบ**ทุกแถวในตาราง**ทันที ไม่มี prompt เตือนใด ๆ ทั้งสิ้น (ต่างจากลบผ่าน SQL editor ใน Studio ที่มักมีขั้นตอนยืนยัน)

---

## กับดัก

- **ลืมต่อ `.select()` แล้วงงว่าทำไม `data` เป็น `undefined` ตลอด** — insert/update/upsert/delete ไม่คืนแถวให้เป็นค่า default (ข้อ 0) ตัวอย่างเก่าบนเว็บจำนวนมากยังเขียนแบบไม่มี `.select()` ต่อท้าย
- **`update()`/`delete()` ไม่ใส่ filter เลย** — แก้/ลบทุกแถวในตารางทันที เป็นความผิดพลาดที่แก้คืนยากที่สุดในลิสต์นี้ ตรวจ filter ให้ครบทุกครั้งก่อนรัน โดยเฉพาะใน production
- **ใช้ `.eq('column', null)` เช็คค่า NULL** — ไม่ทำงานตามที่คิด (`= NULL` ใน SQL ไม่มีความหมายเชิงตรรกะ) ต้องใช้ `.is('column', null)` แทนเสมอ
- **ใช้ `single()` กับ query ที่อาจไม่เจอแถวเลย** — ได้ error ทันทีถ้าไม่เจอ ทั้งที่ "ไม่เจอ" อาจเป็นผลลัพธ์ปกติของ flow นั้น (เช่น เช็คว่า username ซ้ำไหม) ควรใช้ `maybeSingle()` แทนในกรณีนี้
- **`insert()` หลายแถวแล้ว error ตัวเดียวทำให้ทั้งชุดไม่เข้าเลย** — เป็น transaction เดียวกัน ถ้าแถวใดแถวหนึ่งผิด schema/ละเมิด constraint ทั้งชุดจะไม่ถูกเพิ่มเลยสักแถว ไม่ใช่แค่แถวที่ผิด
- **ลืมว่าทุก query ถูกกรองด้วย RLS อยู่ดี** — ต่อให้เขียน syntax ถูกเป๊ะ แต่ถ้า RLS policy ไม่อนุญาต จะได้ผลลัพธ์เป็น array ว่างหรือ error สิทธิ์ ไม่ใช่บั๊กที่โค้ด (ดู [[Supabase]] ข้อ 4)

---

## Cheat sheet

```ts
// Create
await supabase.from('t').insert({ ... }).select()
await supabase.from('t').insert([{ ... }, { ... }]).select()

// Read
await supabase.from('t').select('*')
await supabase.from('t').select('col1, col2')
await supabase.from('t').select('*').eq('col', v)
await supabase.from('t').select('*').single()        // ต้องเจอ 1 แถวพอดี ไม่งั้น error
await supabase.from('t').select('*').maybeSingle()    // 0 แถว = null ไม่ error

// Update
await supabase.from('t').update({ col: v }).eq('id', 1).select()

// Upsert
await supabase.from('t').upsert({ id: 1, ... }, { onConflict: 'id' }).select()

// Delete
await supabase.from('t').delete().eq('id', 1).select()
```

| filter | ความหมาย |
|---|---|
| `.eq(col, v)` / `.neq(col, v)` | = / != |
| `.gt(col, v)` / `.lt(col, v)` | > / < |
| `.like(col, pattern)` | pattern match |
| `.in(col, [v1, v2])` | อยู่ใน list |
| `.is(col, null)` | เช็ค NULL (ห้ามใช้ `.eq()`) |

## 🔗 เกี่ยวข้อง

- [[Supabase]] — ภาพรวม Supabase ทั้งระบบ, RLS ที่ query ทุกตัวโดนกรองด้วยเสมอ

## 📖 อ่านต่อ

- [Supabase JS — insert](https://supabase.com/docs/reference/javascript/insert)
- [Supabase JS — select](https://supabase.com/docs/reference/javascript/select)
- [Supabase JS — update](https://supabase.com/docs/reference/javascript/update)
- [Supabase JS — upsert](https://supabase.com/docs/reference/javascript/upsert)
- [Supabase JS — delete](https://supabase.com/docs/reference/javascript/delete)
- [Supabase JS — Using filters](https://supabase.com/docs/reference/javascript/using-filters)
