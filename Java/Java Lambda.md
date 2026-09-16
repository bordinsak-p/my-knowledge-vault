---
tags:
  - java
  - lambda
  - functional-interface
type: reference
created: 2026-09-14
---

# λ Java Lambda Expression

> **Lambda คือวิธีส่ง "พฤติกรรม" เป็นพารามิเตอร์** — ก่อนหน้านี้ถ้าอยากส่งโค้ดที่จะรันทีหลังเข้าไปในเมธอด ต้องสร้าง anonymous class ทั้งก้อน lambda ย่อของเดิมที่ยาวเป็นสิบบรรทัดให้เหลือบรรทัดเดียว

---

## 1. ปัญหาที่ lambda แก้ — เทียบให้เห็นภาพ

```java
// ก่อน Java 8 — ต้องสร้าง anonymous class เพื่อส่ง "การเทียบ" เข้าไป
Collections.sort(names, new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.length() - b.length();
    }
});
```

```java
// ตั้งแต่ Java 8 — lambda ย่อทั้งก้อนเหลือบรรทัดเดียว
Collections.sort(names, (a, b) -> a.length() - b.length());
```

**สิ่งที่หายไป:** ชื่อคลาส, ชื่อเมธอด, `@Override`, ชนิดพารามิเตอร์ (compiler เดาให้จาก context) — เหลือแค่ **"รับอะไรเข้ามา แล้วทำอะไร"** ซึ่งเป็นส่วนเดียวที่มีความหมายจริง

---

## 2. รูปแบบไวยากรณ์

```java
(พารามิเตอร์) -> เนื้อหา
```

```java
() -> System.out.println("hi")              // ไม่รับพารามิเตอร์
x -> x * 2                                    // รับตัวเดียว ไม่ต้องมีวงเล็บก็ได้
(x, y) -> x + y                               // รับหลายตัว ต้องมีวงเล็บ
(int x, int y) -> x + y                       // ระบุชนิดเองก็ได้ (ปกติไม่จำเป็น)
x -> { System.out.println(x); return x * 2; } // หลายบรรทัด ต้องมี {} และ return
```

**output จริงเมื่อลองรัน:**

```java
Function<Integer, Integer> doubleIt = x -> x * 2;
System.out.println(doubleIt.apply(5));
```
```
10
```

### กฎเรื่อง `{}` และ `return`

| รูปแบบ | ต้องมี `{}` | ต้องมี `return` |
|---|---|---|
| expression เดียว (`x -> x * 2`) | ❌ ไม่ต้อง | ❌ ไม่ต้อง (คืนค่าอัตโนมัติ) |
| หลาย statement | ✅ ต้องมี | ✅ ต้องมีถ้าจะคืนค่า |

```java
// ❌ ผิด — มี {} แล้วต้องมี return
x -> { x * 2 }

// ✅ ถูก
x -> { return x * 2; }
// หรือกลับไปแบบ expression เดียวไปเลย
x -> x * 2
```

---

## 3. Lambda ต้องมี "แบบฟอร์ม" ให้ลง — Functional Interface

**Lambda เดี่ยว ๆ ไม่มีชนิด** มันต้องมี interface ที่บอกว่า "รับกี่ตัว คืนอะไร" ให้ compiler รู้ว่าจะแปลง lambda เป็นอะไร — interface แบบนี้เรียกว่า **functional interface**: อินเทอร์เฟซที่มี **abstract method เดียว**

```java
@FunctionalInterface
interface Greeter {
    String greet(String name);
}

Greeter g = name -> "สวัสดี " + name;
System.out.println(g.greet("สมชาย"));
```
```
สวัสดี สมชาย
```

**`@FunctionalInterface` ไม่บังคับ แต่ควรใส่เสมอ** — ให้ compiler ช่วยเช็คว่ามี abstract method เดียวจริง ถ้ามีคนมาเพิ่ม method ที่สองทีหลังโดยไม่ทันสังเกต compiler จะ error ทันทีแทนที่จะไปพังตอนมีคนพยายามใช้เป็น lambda

**`default` และ `static` method ไม่นับ** — interface มี default/static method กี่ตัวก็ได้ ยังเป็น functional interface ได้ตราบใดที่มี **abstract** method แค่ตัวเดียว

### ไม่ต้องประกาศเอง — มีสำเร็จรูปให้ใช้ (`java.util.function`)

| interface | รับ | คืน | ใช้ทำ |
|---|---|---|---|
| `Function<T,R>` | `T` | `R` | แปลงค่า |
| `Predicate<T>` | `T` | `boolean` | ทดสอบเงื่อนไข |
| `Consumer<T>` | `T` | ไม่คืน (`void`) | ทำอะไรกับค่า |
| `Supplier<T>` | ไม่รับ | `T` | สร้าง/คืนค่า |
| `BiFunction<T,U,R>` | `T`, `U` | `R` | แปลงค่าจากสองตัว |
| `UnaryOperator<T>` | `T` | `T` | แปลงแล้วคืนชนิดเดิม |
| `BinaryOperator<T>` | `T`, `T` | `T` | รวมสองค่าชนิดเดิม |

```java
Function<String, Integer>  length  = s -> s.length();
Predicate<Integer>         isEven  = n -> n % 2 == 0;
Consumer<String>           print   = s -> System.out.println(s);
Supplier<Double>           random  = () -> Math.random();

System.out.println(length.apply("hello"));
System.out.println(isEven.test(4));
print.accept("ทดสอบ");
```
```
5
true
ทดสอบ
```

**เกือบทุกกรณีใช้ตัวสำเร็จรูปพวกนี้พอ** — ประกาศ functional interface เองเฉพาะตอนต้องการชื่อ method ที่สื่อความหมายเฉพาะทาง (เช่น `Greeter.greet` อ่านง่ายกว่า `Function.apply` ในบริบทนั้น)

---

## 4. Method Reference — lambda ที่สั้นลงไปอีก

**เมื่อ lambda ทำแค่ "เรียก method ที่มีอยู่แล้วตรง ๆ"** เขียนย่อด้วย `::` ได้

```java
names.forEach(n -> System.out.println(n));   // lambda ธรรมดา
names.forEach(System.out::println);           // method reference — ความหมายเดียวกัน
```

### สี่แบบ

| แบบ                                       | ไวยากรณ์              | ตัวอย่าง              | เทียบเท่า lambda             |
| ----------------------------------------- | --------------------- | --------------------- | ---------------------------- |
| Static method                             | `Class::staticMethod` | `Integer::parseInt`   | `s -> Integer.parseInt(s)`   |
| Instance method ของ object ที่มีอยู่แล้ว  | `object::method`      | `System.out::println` | `x -> System.out.println(x)` |
| Instance method ของ "ใครก็ได้" ในชนิดนั้น | `Class::method`       | `String::toUpperCase` | `s -> s.toUpperCase()`       |
| Constructor                               | `Class::new`          | `ArrayList::new`      | `() -> new ArrayList<>()`    |

```java
List<String> names = List.of("bob", "alice", "carol");

names.stream().map(String::toUpperCase).forEach(System.out::println);
```
```
BOB
ALICE
CAROL
```

**แบบที่ 3 คือตัวที่งงที่สุดตอนแรกเห็น** — `String::toUpperCase` ไม่มี object นำหน้า เพราะ object **จะถูกส่งเข้ามาเป็นพารามิเตอร์ตัวแรกให้เอง** ตอนใช้จริงใน `map()` แต่ละ element ของ stream (`"bob"`, `"alice"`, ...) กลายเป็นตัว `this` ที่ถูกเรียก `.toUpperCase()`

---

## 5. Effectively Final — กฎที่งงที่สุดตอนเริ่มต้น

```java
int count = 0;
Runnable r = () -> System.out.println(count);   // ✅ อ่านได้ เพราะ count ไม่เคยถูกแก้ค่าอีกเลย
```

```java
int count = 0;
count = 5;                                        // ← แก้ค่า
Runnable r = () -> System.out.println(count);    // ❌ Compile error!
```

```
error: local variables referenced from a lambda expression must be final or effectively final
```

**"effectively final" แปลว่า:** ตัวแปรนั้นไม่เคยถูกกำหนดค่าใหม่เลยหลัง initialize ครั้งแรก ต่อให้ไม่มีคำว่า `final` เขียนไว้ก็ถือว่า final โดยพฤตินัย

### ทำไมต้องมีกฎนี้

```
Method ปกติ                          Lambda
──────────                          ──────
ตัวแปร local อยู่บน stack             lambda อาจถูกเก็บไว้ใช้ทีหลัง (เช่นส่งไป thread อื่น)
หาย/reset เมื่อ method จบ             ตอนนั้น stack ของ method เดิมอาจไม่อยู่แล้ว
```

**Lambda ไม่ได้จับ "ตัวแปร" แต่จับ "ค่า ณ ตอนนั้น" (capture by value)** — ถ้ายอมให้ตัวแปรเปลี่ยนค่าได้ระหว่างทาง lambda ที่ถูกเก็บไว้เรียกทีหลังจะเห็นค่าไหนกันแน่? กฎ effectively final ตัดปัญหานี้ทิ้งไปตั้งแต่ compile time

### ทางแก้เมื่อจำเป็นต้อง "สะสมค่า"

```java
// ❌ ห้าม — count ไม่ effectively final
int count = 0;
list.forEach(x -> count++);   // compile error
```

```java
// ✅ ใช้ตัวที่ออกแบบมาสำหรับข้ามเธรด/สะสมค่าได้
AtomicInteger count = new AtomicInteger(0);
list.forEach(x -> count.incrementAndGet());
System.out.println(count.get());
```
```
(จำนวนสมาชิกใน list)
```

**อย่าใช้ array ขนาด 1 (`int[] count = {0}`) เป็นทางลัด** — แก้ผ่านได้เพราะตัว array เองไม่ได้ถูก reassign (ตัวแปรอ้างอิง array ยัง effectively final) แต่เป็น code smell ที่บอกว่ากำลังฝืนธรรมชาติของ lambda อยู่ ใช้ `AtomicInteger`/`AtomicReference` หรือปรับให้ใช้ `Stream`/`reduce` แทนจะสื่อความหมายชัดกว่า

---

## 6. `this` ใน lambda — ต่างจาก anonymous class

```java
class Counter {
    int value = 10;

    void run() {
        // lambda — this ยังหมายถึง Counter (ไม่ใช่ lambda เอง)
        Runnable r1 = () -> System.out.println(this.value);

        // anonymous class — this หมายถึงตัว anonymous class เอง
        Runnable r2 = new Runnable() {
            int value = 20;
            public void run() { System.out.println(this.value); }
        };

        r1.run();
        r2.run();
    }
}
new Counter().run();
```
```
10
20
```

**Lambda ไม่สร้าง scope ของ `this` ใหม่** มันใช้ `this` ของ class ที่ล้อมรอบอยู่ (**lexical scoping**) ต่างจาก anonymous class ที่ `this` หมายถึงตัวมันเอง — นี่คือกับดักตอนย้ายโค้ดจาก anonymous class เก่ามาเป็น lambda แล้วพฤติกรรมเปลี่ยนไปเงียบ ๆ

---

## 7. Checked Exception ใน lambda — ปัญหาที่เจอบ่อยตอนใช้จริง

```java
List<String> files = List.of("a.txt", "b.txt");

files.forEach(f -> {
    Files.readString(Path.of(f));   // ❌ Compile error
});
```
```
error: unhandled exception: java.io.IOException
```

**`Consumer<T>` ไม่ได้ประกาศว่าโยน checked exception ได้** เพราะ interface มาตรฐานใน `java.util.function` ไม่มีตัวไหนประกาศ `throws` ไว้เลย — lambda ที่ implement มันเลยโยน checked exception ไม่ได้ตามไปด้วย

### ทางแก้ — ห่อเป็น unchecked หรือจับในนั้นเลย

```java
files.forEach(f -> {
    try {
        Files.readString(Path.of(f));
    } catch (IOException e) {
        throw new UncheckedIOException(e);   // แปลงเป็น unchecked แล้วโยนต่อ
    }
});
```

**อย่ากลืน exception เงียบ ๆ ด้วย `catch (Exception e) {}` เปล่า ๆ** เพียงเพราะอยากให้ compile ผ่าน — เลือกระหว่างแปลงเป็น unchecked exception (ให้ยังเห็น error) หรือจัดการจริงตรงนั้น

---

## 8. กับดัก

- **สร้าง object ใหม่ทุกครั้งที่เรียก method ที่มี lambda เป็นค่าเริ่มต้น** — lambda ที่ capture ตัวแปรจะสร้าง instance ใหม่ทุกครั้งที่ expression นั้นถูก evaluate ถ้าอยู่ใน loop ที่รันบ่อยมาก อาจมีผลด้าน performance เล็กน้อย (lambda ที่ไม่ capture อะไรเลย JVM cache ให้ได้)
- **แก้ตัวแปรนอก lambda แล้วงงว่าทำไม compile ไม่ผ่าน** — ลืมกฎ effectively final (ข้อ 5)
- **ย้ายจาก anonymous class เป็น lambda แล้ว `this` เปลี่ยนความหมาย** — โดยเฉพาะโค้ดที่เคยพึ่ง `this` ของ anonymous class เอง (ข้อ 6)
- **lambda ยาวเกินไปจนอ่านไม่รู้เรื่อง** — ถ้า lambda ต้องมีหลาย statement และซับซ้อน ควรแยกเป็น named method แล้วใช้ method reference แทน อ่านง่ายกว่าและ debug ง่ายกว่า (stack trace ของ lambda นิรนามอ่านยาก)
- **ใช้ lambda จับ exception แบบ generic เกินไป** — `catch (Exception e)` ในทุก lambda ทำให้ error ที่ควรเห็นหายไปเงียบ ๆ
- **สับสน `Function<T,R>` กับ `Consumer<T>`** — `Function` ต้อง `return` ค่า `Consumer` ไม่คืนอะไรเลย ใช้ผิดจะ compile error ทันที ไม่ใช่บั๊กที่มองไม่เห็น

---

## 9. Cheat sheet

```java
() -> expr                    // ไม่รับพารามิเตอร์
x -> expr                     // รับตัวเดียว
(x, y) -> expr                // รับหลายตัว
x -> { stmt; return val; }    // หลาย statement

Function<T,R>       f = x -> ...;   f.apply(x)
Predicate<T>         p = x -> ...;   p.test(x)
Consumer<T>          c = x -> ...;   c.accept(x)
Supplier<T>           s = () -> ...; s.get()
BiFunction<T,U,R>    bf = (x,y) -> ...; bf.apply(x,y)

Class::staticMethod    // Integer::parseInt
object::method          // System.out::println
Class::instanceMethod   // String::toUpperCase
Class::new               // ArrayList::new
```

| อาการ | สาเหตุ |
|---|---|
| `local variables referenced from a lambda expression must be final or effectively final` | ตัวแปรถูก reassign หลัง lambda จับไปใช้ |
| `unhandled exception` ใน lambda | interface มาตรฐานไม่ประกาศ `throws` — ต้อง try/catch ในนั้น |
| `this` ใน lambda ไม่ใช่ตัวที่คาดไว้ | lambda ใช้ `this` ของ class ล้อมรอบ ไม่ใช่ของตัวเอง (ต่างจาก anonymous class) |
| lambda ไม่ compile บอกว่า target type ไม่ตรง | functional interface มี abstract method มากกว่า 1 หรือ signature ไม่ตรงกับ lambda |
| stack trace อ่านไม่รู้เรื่องเวลา error ใน lambda | lambda ซับซ้อนเกินไป — แยกเป็น named method + method reference |

---

## 🔗 เกี่ยวข้อง

- [[Java]] — หน้ารวม
- [[Java Record]] — record มักใช้คู่กับ lambda ใน stream (เช่น `map(Row::new)`)

## 📖 อ่านต่อ

- [Oracle — Lambda Expressions](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html)
- [Oracle — Method References](https://docs.oracle.com/javase/tutorial/java/javaOO/methodreferences.html)
- [java.util.function package](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/function/package-summary.html)
