---
tags:
  - java
  - version-updates
  - changelog
type: reference
created: 2026-09-21
---

# 🆕 Java — Version Updates ล่าสุด

> โน้ตนี้ติดตาม **"อะไรใหม่/อะไรถูกเลิกใช้"** ของแต่ละเวอร์ชัน Java ไม่ใช่โน้ตอธิบายฟีเจอร์ภาษาเชิงลึก (ไปดู [[Java]] สำหรับโน้ตแยกตามหัวข้อ + ตาราง feature-by-version) — เช็กวันที่ด้านล่างก่อนเชื่อ 100% เสมอ

> **อัปเดตล่าสุด: 2026-09-21 — เวอร์ชันล่าสุดคือ Java 27 (15 ก.ย. 2026, non-LTS), LTS ล่าสุดคือ Java 25 (ก.ย. 2025)**

---

## 1. Java นับเวอร์ชันยังไง

ออกเวอร์ชันใหม่ทุก 6 เดือนตายตัว (มี.ค./ก.ย.) — ต่างจาก Quarkus ที่ minor version ถี่ไม่ตายตัว (ดู [[Quarkus Updates]]) แล้วเลือกเวอร์ชันหนึ่งเป็น **LTS ทุก 2 ปี**: 17 (2021) → 21 (2023) → 25 (2025) → 29 (คาดว่า 2027)

| ประเภท               | เวอร์ชันปัจจุบัน  | ใช้ตอนไหน                                                                                    |
| -------------------- | ----------------- | -------------------------------------------------------------------------------------------- |
| **LTS**              | 25 (ก.ย. 2025)    | production ทั่วไป — support ยาวหลายปี                                                        |
| **Latest (non-LTS)** | 27 (15 ก.ย. 2026) | อยากได้ฟีเจอร์ใหม่สุด/ทดลอง — support สั้นแค่ 6 เดือนถึงตัวถัดไป ไม่แนะนำ production จริงจัง |

**วอลต์นี้ใช้ Java 17 (LTS) เป็นเวอร์ชันอ้างอิง** (ดู [[Java]]) — **ถ้าจะอัปเกรด เป้าหมายที่ควรมองคือ LTS ถัดไป (21 หรือ 25) ไม่ใช่เวอร์ชันล่าสุดที่เป็น non-LTS**

---

## 2. Java 21 (LTS, ก.ย. 2023)

JEP หลักที่ **finalize** (พ้น preview แล้ว ใช้ใน production ได้เต็มตัว):

- **Virtual Threads (JEP 444)** — thread แบบเบาที่ JVM จัดการเองแทน OS สร้างได้เป็นล้าน thread โดยไม่ชน OS resource limit เหมาะกับงาน blocking I/O เยอะ ๆ (เช่น เรียก REST API ต่อเนื่องหลายที่) โดยไม่ต้องเปลี่ยนไปเขียนสไตล์ reactive/async ให้ซับซ้อนขึ้น — รายละเอียดเต็ม/กับดัก (pinning ฯลฯ) แยกไว้ที่ [[Java Virtual Threads]]

  ```java
  try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
      executor.submit(() -> callExternalApi());   // 1 task = 1 virtual thread ใหม่
  }
  ```

- **Record Patterns (JEP 440)** — แกะ component ของ record ออกมาใช้ตรง ๆ ตอน pattern matching เช่น `if (obj instanceof Point(int x, int y))` ได้ตัวแปร `x`/`y` ทันทีไม่ต้องเรียก getter เอง (รายละเอียด/ตัวอย่างเต็มดู [[Java Record]] ข้อ 7)

  ```java
  String describe(PaymentResult r) {
      return switch (r) {
          case Success(String id, Money m) -> "ok " + id + " " + m.amount();
          case Declined(String reason)     -> "declined: " + reason;
          default -> "unknown";
      };
  }
  ```

- **Pattern Matching for switch (JEP 441)** — `switch` เช็ค type พร้อม destructure ในเงื่อนไขเดียวกันได้ และมี `case null` แยกเงื่อนไข null ได้ตรง ๆ (ไม่ต้องเช็ค null แยกก่อนเข้า switch)

  ```java
  String format(Object obj) {
      return switch (obj) {
          case null       -> "ไม่มีค่า";
          case Integer i  -> "จำนวนเต็ม: " + i;
          case String s when s.isBlank() -> "string ว่าง";
          case String s   -> "ข้อความ: " + s;
          default         -> "ไม่รู้จัก";
      };
  }
  ```

- **Sequenced Collections (JEP 431)** — interface ใหม่ `SequencedCollection`/`SequencedSet`/`SequencedMap` ให้ `getFirst()`/`getLast()`/`reversed()` ตรง ๆ เลิกเขียน workaround แบบเดิม (`list.get(list.size() - 1)`)

  ```java
  List<String> names = new ArrayList<>(List.of("a", "b", "c"));
  names.addFirst("start");        // ["start", "a", "b", "c"]
  names.getLast();                // "c"
  List<String> rev = names.reversed();   // view กลับด้าน แก้ต้นฉบับก็สะท้อนที่นี่ด้วย
  ```

- **Generational ZGC (JEP 439)** — garbage collector หยุดโลกสั้นลงมาก แยก generation young/old ให้ ZGC ทำงานมีประสิทธิภาพขึ้น (ตั้งค่าผ่าน JVM flag ไม่มีโค้ดให้เขียน)

ยังเป็น **preview** ในเวอร์ชันนี้ (ต้องเปิด `--enable-preview` ถึงใช้ได้ ไม่ควรเอาเข้า production):

- **Unnamed Patterns and Variables** — ใช้ `_` แทนตัวแปรที่ไม่ได้ใช้จริงใน pattern (JEP 443) — **finalize แล้วตั้งแต่ Java 22** (เร็วกว่าฟีเจอร์ preview ตัวอื่นในกลุ่มนี้มาก)

  ```java
  // ไม่สนใจ y เลย ใช้ _ แทนชื่อได้
  if (obj instanceof Point(int x, var _)) {
      System.out.println("x = " + x);
  }
  ```

- **Unnamed Classes and Instance Main Methods** — เขียนโปรแกรมแรกสั้นลง ไม่ต้อง `public class`/`public static void main(String[] args)` เต็ม ๆ (JEP 445) — ⚠️ **finalize ใน Java 25 แต่เปลี่ยนชื่อ/รายละเอียดไปจากตอน preview ใน 21 ดูข้อ 3**

  ```java
  // ทั้งไฟล์มีแค่นี้ก็รันได้ (ตอน preview 21-24)
  void main() {
      System.out.println("Hello, World!");
  }
  ```

- **Scoped Values** — ทางเลือกใหม่แทน `ThreadLocal` ที่เข้ากับ virtual thread ได้ดีกว่า (JEP 446) — **finalize แล้วใน Java 25 ดูตัวอย่างเต็มข้อ 3**
- **String Templates** — ⚠️ **ดูข้อ 4 ก่อนใช้จริง — ไม่ได้ finalize แล้วถูกถอนออกไปเลย**

---

## 3. Java 25 (LTS ปัจจุบัน, ก.ย. 2025) — ปลายทางที่ควรมองถ้าจะอัปเกรดตอนนี้

- **Scoped Values (JEP 506) — finalize แล้วจริง** (preview ตั้งแต่ 21) ทางเลือกแทน `ThreadLocal` ที่เข้ากับ virtual thread ได้ดีกว่า (immutable, ไม่มีปัญหา leak ข้าม thread)

  ```java
  private static final ScopedValue<String> USER_ID = ScopedValue.newInstance();

  ScopedValue.where(USER_ID, "u123").run(() -> {
      process();   // เรียก USER_ID.get() ได้จากในนี้ รวมถึง method ที่เรียกต่อจากนี้
  });
  ```

- ⚠️ **Structured Concurrency (JEP 505) — ยังเป็น preview อยู่ ไม่ได้ finalize** แม้จะ preview มาตั้งแต่ 21 แล้วก็ตาม (ถึง 25 นับเป็นรอบ preview ที่ 5) API เปลี่ยนรายละเอียดแทบทุกรอบ preview — **อย่าเพิ่งอ้างอิง syntax ที่แน่นอนจนกว่าจะ finalize จริง**
- **Unnamed Classes and Instance Main Methods (JEP 445 เดิม) — finalize แล้ว แต่เปลี่ยนชื่อเป็น "Compact Source Files and Instance Main Methods" (JEP 512)** พร้อมรายละเอียดเปลี่ยนจากตอน preview:

  ```java
  // ไฟล์ .java ทั้งไฟล์มีแค่นี้ — รันได้ตรง ๆ ไม่ต้อง public class
  void main() {
      IO.println("Hello, World!");   // IO อยู่ package java.lang (ย้ายจาก java.io)
  }
  ```

  **โค้ดตัวอย่างเก่าที่ลองตอน preview (21-24) อาจใช้ต่อบน 25 ไม่ได้ตรง ๆ** — คลาส `IO` ย้ายจาก `java.io` ไป **`java.lang`** และ static method ของ `IO` **ไม่ได้ auto-import ให้แล้ว** (ต้องเรียกผ่าน `IO.println(...)` ไม่ใช่ `println(...)` เฉย ๆ แบบที่ตัวอย่างเก่าเคยโชว์)
- **Compact Object Headers (JEP 519) — เป็น stable/product feature แล้ว** (จากที่เคยเป็น experimental ใน JDK 24) ลด header size ของทุก object ในฮีปจาก 12 ไบต์เหลือ 8 ไบต์ **แต่ยังต้องเปิดเองผ่าน flag `-XX:+UseCompactObjectHeaders`** ไม่ได้เปิด default ให้ — Java 27 ถึงจะทำให้เป็นค่า default (ไม่ต้องเปิดเองอีกต่อไป)

---

## 4. กับดัก

- **String Templates (JEP 430) — preview ใน 21 และ 22 แล้วถูกถอนออกไปเลยตั้งแต่ 23** ไม่ใช่แค่ยืด preview ต่อ — ทีม Java เองยอมรับว่า design แบบ processor-centric ที่เสนอไว้สับสนและไม่ compositional พอ เลยถอนกลับไปคิดใหม่ทั้งหมด **โค้ดที่เขียนพึ่ง `STR."..."` ไว้ตอน 21/22 ใช้ต่อบน 23 ขึ้นไปไม่ได้เลย**
- **เอาฟีเจอร์ preview ไปใช้ใน production** — ต้องเปิด `--enable-preview` ตอน compile และ **ผูกกับ JDK เวอร์ชันนั้นตายตัว** (bytecode ที่ compile ด้วย preview ของเวอร์ชันหนึ่งรันข้าม minor version อื่นไม่ได้เลย) รอให้ finalize ก่อนค่อยใช้จริง
- **ก๊อปตัวอย่างจากเว็บที่เขียนด้วย 21+ มาใช้ทั้งที่โปรเจกต์ยังอยู่ 17** — `SequencedCollection`/record pattern/pattern matching switch ยังใช้ไม่ได้เลยบน 17 (เช็คตาราง feature-by-version ที่ [[Java]] ก่อนก๊อปเสมอ)
- **อัปเกรดข้าม LTS ไปหลายรอบทีเดียว** (เช่น 17 → 25 ตรง ๆ ไม่ผ่าน 21) — เช็ค migration guide ทุกครั้ง เพราะ removed API/deprecation สะสมมากกว่าอัปทีละ LTS
- **เชื่อว่าฟีเจอร์ preview ทุกตัว finalize เร็ว-ช้าเท่ากัน** — ไม่จริงเลย: Unnamed Patterns/Variables preview ตัวเดียวจบใน 1 รอบ (22), Virtual Threads/Record Patterns ใช้เวลาหลายรอบกว่าจะ finalize (21), Structured Concurrency ผ่านมาแล้ว 5 รอบ preview ยังไม่จบ (ถึง 25) และ String Templates ไม่จบเลยเพราะถูกถอน — **ต้องเช็คสถานะจริงของแต่ละตัวแยกกัน อย่าคาดเดาจากเวอร์ชันที่เห็นครั้งแรก**
- **ก๊อปตัวอย่าง "Unnamed Classes" จากตอน preview (21-24) มาใช้บน 25** — ชื่อเปลี่ยนเป็น "Compact Source Files", `IO` ย้ายจาก `java.io` ไป `java.lang`, ไม่ auto-import static method ให้แล้ว (ข้อ 3) โค้ดเก่าที่เคยรันได้อาจ compile ไม่ผ่าน

---

## 5. Cheat sheet

```bash
java -version                 # ดูเวอร์ชัน JDK ที่ใช้อยู่จริง
```

| ต้องการรู้ | ดูที่ |
|---|---|
| เวอร์ชันไหนเป็น LTS / support ถึงเมื่อไหร่ | [Oracle Java SE Support Roadmap](https://www.oracle.com/java/technologies/java-se-support-roadmap.html) |
| JEP ทั้งหมดของแต่ละเวอร์ชัน | [OpenJDK JEP Index](https://openjdk.org/jeps/0) |
| migration guide ข้ามเวอร์ชัน | [Oracle — Java Language Updates](https://docs.oracle.com/en/java/javase/25/migrate/) |

---

## 🔗 เกี่ยวข้อง

- [[Java]] — โน้ตหลัก, ตาราง feature-by-version, เวอร์ชันที่วอลต์นี้ใช้อ้างอิง (17)
- [[Java Record]] — record pattern (JEP 440) ต่อยอดจาก record พื้นฐาน
- [[Java Virtual Threads]] — รายละเอียดเต็มของ JEP 444 (carrier thread, pinning, กับดัก)
- [[Quarkus Updates]] — เทียบ pattern การนับเวอร์ชันแบบ minor ถี่ของ Quarkus

## 📖 อ่านต่อ

- [OpenJDK JEP Index](https://openjdk.org/jeps/0)
- [Inside.java — The Arrival of Java 27](https://inside.java/2026/09/15/jdk-27-available/)
- [JEP 512: Compact Source Files and Instance Main Methods](https://openjdk.org/jeps/512)
- [JEP 506: Scoped Values](https://openjdk.org/jeps/506)
- [Oracle Java SE Support Roadmap](https://www.oracle.com/java/technologies/java-se-support-roadmap.html)
