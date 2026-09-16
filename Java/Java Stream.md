---
tags:
  - java
  - stream
  - collectors
  - functional
type: reference
created: 2026-09-14
---

# 🌊 Java Stream API

> ก่อนอ่านโน้ตนี้ควรผ่าน [[Java Lambda]] มาก่อน — Stream แทบทุกเมธอดรับ lambda/method reference เป็นพารามิเตอร์
> **Stream ไม่ใช่ data structure** ไม่เก็บข้อมูล ไม่มี index — มันคือ **สายพานประมวลผล** ที่ไล่ข้อมูลผ่านขั้นตอนต่าง ๆ ทีละตัว

---

## 1. ภาพรวม — สามส่วนที่ต้องมีเสมอ

```java
List<String> names = List.of("bob", "alice", "carol", "dan");

List<String> result = names.stream()          // ① source
    .filter(n -> n.length() > 3)               // ② intermediate operation (ทำได้หลายอัน)
    .map(String::toUpperCase)                   // ② intermediate operation
    .sorted()                                    // ② intermediate operation
    .collect(Collectors.toList());               // ③ terminal operation

System.out.println(result);
```
```
[ALICE, CAROL]
```

| ① Source         | ② Intermediate (0 อันขึ้นไป)          | ③ Terminal (พอดี 1 อัน)   |
| ---------------- | ------------------------------------- | ------------------------- |
| `list.stream()`  | `.filter()` `.map()` `.sorted()`      | `.collect()` `.forEach()` |
| `array`          | `.distinct()` `.limit()` `.flatMap()` | `.count()` `.reduce()`    |
| `Stream.of(...)` |                                       | `.findFirst()` ...        |

**กฎที่ต้องจำ: intermediate operation คืน Stream ใหม่เสมอ (เชื่อมต่อกันได้เรื่อย ๆ), terminal operation คืนค่าอื่นที่ไม่ใช่ Stream (จบสาย)**

---

## 2. ⭐ Lazy Evaluation — เรื่องที่ทำให้ Stream ต่างจาก loop จริง ๆ

```java
List<String> names = List.of("bob", "alice", "carol");

Stream<String> s = names.stream()
    .filter(n -> {
        System.out.println("filtering: " + n);
        return n.length() > 3;
    })
    .map(n -> {
        System.out.println("mapping: " + n);
        return n.toUpperCase();
    });

System.out.println("ยังไม่ทำอะไรเลยจนถึงตรงนี้");
s.forEach(System.out::println);
```

**output จริง:**
```
ยังไม่ทำอะไรเลยจนถึงตรงนี้
filtering: bob
filtering: alice
mapping: alice
filtering: carol
mapping: carol
ALICE
CAROL
```

**สังเกตสองอย่างที่ขัดกับสัญชาตญาณ:**

1. **ไม่มีอะไรรันเลยจนกว่าจะเจอ terminal operation** (`forEach`) — บรรทัด `filter`/`map` แค่ "วางแผน" ไว้ ไม่ได้ execute ทันที
2. **แต่ละ element วิ่งผ่าน "ทุกขั้นตอน" ก่อนจะไปตัวถัดไป** ไม่ใช่ filter ทุกตัวก่อนแล้วค่อย map ทุกตัว — `"bob"` ถูก filter ทิ้งจึงไม่ถูก map เลย

| loop ปกติ (ทำทีละขั้นตอน ทุก element) | Stream (ทำทุกขั้นตอน ทีละ element) |
|---|---|
| filter ทุกตัวก่อน → ได้ list ใหม่ | `bob`: filter → (ตกรอบ) |
| map ทุกตัวจาก list ที่เหลือ | `alice`: filter → map → เก็บผล |
| | `carol`: filter → map → เก็บผล |

**ประโยชน์จริงของ lazy evaluation คือ short-circuit**

```java
Optional<String> first = names.stream()
    .filter(n -> n.length() > 3)
    .map(n -> { System.out.println("mapping: " + n); return n.toUpperCase(); })
    .findFirst();                    // ← เจอตัวแรกแล้วหยุดทันที ไม่ทำต่อ
```
```
mapping: alice
```

**`carol` ไม่ถูก map เลย** เพราะ `findFirst()` พอใจกับ `alice` แล้วสั่งหยุดทั้ง pipeline — ถ้าเขียนเป็น loop ธรรมดาต้อง `break` เองด้วยมือ แต่ stream ทำให้อัตโนมัติ

---

## 3. Terminal operation ที่ใช้บ่อย

```java
List<Integer> nums = List.of(3, 1, 4, 1, 5, 9, 2, 6);

nums.stream().count();                        // 8
nums.stream().anyMatch(n -> n > 8);            // true (สั้นวงจร เจอ 9 แล้วหยุด)
nums.stream().allMatch(n -> n > 0);            // true
nums.stream().noneMatch(n -> n > 100);         // true
nums.stream().max(Integer::compareTo);         // Optional[9]
nums.stream().reduce(0, Integer::sum);         // 45
nums.stream().forEach(System.out::println);    // พิมพ์ทีละตัว ไม่คืนค่า
```

**`anyMatch`/`allMatch`/`noneMatch`/`findFirst`/`findAny` เป็น short-circuit ทั้งหมด** — หยุดทันทีที่รู้คำตอบ ไม่ต้องไล่จนครบ ต่างจาก `count()`/`collect()` ที่ต้องไล่ทุกตัวเสมอ

### `reduce` — รวมค่าทั้งหมดเป็นค่าเดียว

```java
int sum = nums.stream().reduce(0, (acc, n) -> acc + n);
//                        │        │
//                        │        └── ทำอะไรกับแต่ละตัว: (ค่าที่สะสมไว้, ตัวปัจจุบัน) -> ค่าใหม่
//                        └── ค่าเริ่มต้น (identity)
```
```
28
```

`reduce` คือรูปแบบทั่วไปของ `sum`/`max`/`min` — ใช้เมื่อต้องการรวมค่าแบบที่ไม่มีเมธอดสำเร็จรูปให้

---

## 4. `collect()` และ `Collectors` — จุดที่ใช้บ่อยที่สุดในงานจริง

```java
record Person(String name, String city, int age) {}

List<Person> people = List.of(
    new Person("สมชาย", "กรุงเทพ", 30),
    new Person("มาลี", "เชียงใหม่", 25),
    new Person("สมหญิง", "กรุงเทพ", 35));
```

### แปลงกลับเป็น Collection

```java
List<String> names = people.stream().map(Person::name).collect(Collectors.toList());
Set<String>  cities = people.stream().map(Person::city).collect(Collectors.toSet());

// Java 16+ ทางลัดสำหรับ toList() (immutable, สั้นกว่า)
List<String> names2 = people.stream().map(Person::name).toList();
```
```
[สมชาย, มาลี, สมหญิง]
```

**`toList()` (ตัวเล็ก ไม่มี `Collectors.`) คืน list ที่แก้ไขไม่ได้** ต่างจาก `Collectors.toList()` ที่คืน list แก้ไขได้ — เลือกให้ตรงกับที่ต้องการ

### จัดกลุ่ม — `groupingBy`

```java
Map<String, List<Person>> byCity = people.stream()
    .collect(Collectors.groupingBy(Person::city));
```
```
{กรุงเทพ=[Person[name=สมชาย, ...], Person[name=สมหญิง, ...]], เชียงใหม่=[Person[name=มาลี, ...]]}
```

**`groupingBy` มี downstream collector ต่อได้** — ไม่ต้องแค่เก็บเป็น list เฉย ๆ

```java
Map<String, Long> countByCity = people.stream()
    .collect(Collectors.groupingBy(Person::city, Collectors.counting()));

Map<String, Double> avgAgeByCity = people.stream()
    .collect(Collectors.groupingBy(Person::city, Collectors.averagingInt(Person::age)));

Map<String, List<String>> namesByCity = people.stream()
    .collect(Collectors.groupingBy(Person::city,
             Collectors.mapping(Person::name, Collectors.toList())));
```
```
{กรุงเทพ=2, เชียงใหม่=1}
{กรุงเทพ=32.5, เชียงใหม่=25.0}
{กรุงเทพ=[สมชาย, สมหญิง], เชียงใหม่=[มาลี]}
```

### แบ่งสอง — `partitioningBy`

```java
Map<Boolean, List<Person>> partitioned = people.stream()
    .collect(Collectors.partitioningBy(p -> p.age() >= 30));
```
```
{false=[Person[name=มาลี, ...]], true=[Person[name=สมชาย, ...], Person[name=สมหญิง, ...]]}
```

**ต่างจาก `groupingBy` ตรงที่ผลลัพธ์มีแค่ key `true`/`false` เสมอ** แม้จะไม่มีสมาชิกฝั่งไหนเลยก็ยังได้ key นั้นกลับมา (list ว่าง) เหมาะกับเงื่อนไขที่แบ่งเป็นสองฝั่งจริง ๆ

### รวมเป็น String

```java
String csv = people.stream().map(Person::name).collect(Collectors.joining(", "));
```
```
สมชาย, มาลี, สมหญิง
```

### สรุปสถิติ

```java
IntSummaryStatistics stats = people.stream()
    .collect(Collectors.summarizingInt(Person::age));

System.out.println(stats.getMin() + " " + stats.getMax() + " " + stats.getAverage());
```
```
25 35 30.0
```

---

## 5. `map` vs `flatMap`

```java
List<List<Integer>> nested = List.of(List.of(1, 2), List.of(3, 4), List.of(5));

nested.stream().map(l -> l.size()).forEach(System.out::println);
```
```
2
2
1
```

```java
nested.stream().flatMap(List::stream).forEach(System.out::println);
```
```
1
2
3
4
5
```

```
map      : List<List<Integer>>  →  Stream<Integer>   (ทำงานกับ list ย่อยแต่ละก้อน)
flatMap  : List<List<Integer>>  →  Stream<Integer>   (แกะทุกชั้นออกมาเป็นเส้นเดียว)
```

**ใช้ `flatMap` เมื่อแต่ละ element กลายเป็นหลาย element** (หรือ "ไม่มี element" ก็ได้) — เช่น หนึ่ง order มีหลาย item อยากได้ item ทั้งหมดของทุก order รวมกันเป็นเส้นเดียว

```java
record Order(String id, List<String> items) {}
List<Order> orders = List.of(new Order("O1", List.of("A", "B")), new Order("O2", List.of("C")));

List<String> allItems = orders.stream()
    .flatMap(o -> o.items().stream())
    .toList();
```
```
[A, B, C]
```

---

## 6. Primitive Stream — `IntStream`/`LongStream`/`DoubleStream`

```java
IntStream.range(1, 5).forEach(System.out::println);       // 1 2 3 4 (ไม่รวม 5)
IntStream.rangeClosed(1, 5).forEach(System.out::println);  // 1 2 3 4 5 (รวม 5)

int sum = IntStream.rangeClosed(1, 100).sum();
System.out.println(sum);
```
```
5050
```

**ทำไมมี stream แยกสำหรับ primitive** — `Stream<Integer>` ต้อง box ค่า `int` เป็น `Integer` ทุกตัว (เสีย memory + CPU) `IntStream` เก็บเป็น `int` ดิบไม่ต้อง box ประหยัดกว่ามากเมื่อมีข้อมูลจำนวนมาก

```java
List<Person> people2 = people;
double avgAge = people2.stream()
    .mapToInt(Person::age)   // Stream<Person> → IntStream
    .average()
    .orElse(0);
```
```
30.0
```

**`mapToInt`/`mapToLong`/`mapToDouble` สลับไป primitive stream, `.boxed()` สลับกลับ** เวลาต้องการทำอะไรที่ primitive stream ไม่มีให้ (เช่น `.collect()`)

---

## 7. ⚠️ Stream ใช้ได้ครั้งเดียว

```java
Stream<String> s = names.stream();
s.forEach(System.out::println);
s.forEach(System.out::println);   // ❌ throw ตอนรัน
```
```
java.lang.IllegalStateException: stream has already been operated upon or closed
```

**ทันทีที่ terminal operation ทำงาน stream นั้น "ถูกใช้ไปแล้ว"** เรียกซ้ำไม่ได้ ต้องสร้าง stream ใหม่จาก source เดิมทุกครั้งที่ต้องการวนซ้ำ

```java
// ✅ สร้างใหม่ทุกครั้ง
names.stream().forEach(System.out::println);
names.stream().forEach(System.out::println);
```

---

## 8. Parallel Stream — เร็วขึ้นได้ แต่ไม่ใช่เสมอไป

```java
long count = hugeList.parallelStream().filter(x -> isSlowCheck(x)).count();
```

**`parallelStream()` แบ่งงานให้หลาย thread ทำพร้อมกัน** ผ่าน common `ForkJoinPool` ของ JVM — ฟังดูเร็วขึ้นเสมอ แต่ในทางปฏิบัติ**เร็วขึ้นจริงเฉพาะบางเงื่อนไข**

| เร็วขึ้นจริง | ช้าลงหรือไม่ต่าง |
|---|---|
| ข้อมูลเยอะมาก (หลักหมื่นขึ้นไป) | ข้อมูลน้อย (overhead การแบ่งงานแพงกว่าที่ประหยัดได้) |
| งานต่อ element หนัก (CPU-bound) | งานเบา ๆ อย่าง `map`/`filter` ธรรมดา |
| source แบ่งงานได้ดี (array, `ArrayList`) | source แบ่งงานยาก (`LinkedList`, I/O) |

**กฎที่ใช้ได้เสมอ: วัดผลจริงก่อนใช้ อย่าเดาว่า parallel เร็วกว่า** — เขียนโค้ดสองแบบแล้ว benchmark เทียบเวลาจริงบนข้อมูลจริง ไม่ใช่เปลี่ยนเป็น `parallelStream()` เพราะชื่อฟังดูดี

**อันตรายที่แท้จริงของ parallel stream ไม่ใช่ความเร็ว แต่คือ thread-safety**

```java
// ❌ อันตราย — list ปกติไม่ thread-safe
List<String> result = new ArrayList<>();
hugeList.parallelStream().forEach(x -> result.add(transform(x)));   // race condition
```

```java
// ✅ ให้ stream จัดการรวมผลลัพธ์ให้เอง
List<String> result = hugeList.parallelStream()
    .map(this::transform)
    .collect(Collectors.toList());   // thread-safe โดยกลไกของ collect เอง
```

**อย่าเขียนผลลงตัวแปรร่วม (shared mutable state) จากใน `forEach`ของ parallel stream เด็ดขาด** ใช้ `collect()`/`reduce()` ที่ออกแบบมาให้รวมผลจากหลาย thread อย่างปลอดภัยแทนเสมอ

---

## 9. Checked Exception ใน stream — ปัญหาเดียวกับใน lambda

```java
files.stream().map(f -> Files.readString(Path.of(f)));   // ❌ compile error เหมือนใน Consumer
```

ดูวิธีแก้เต็ม ๆ ที่ [[Java Lambda]] ข้อ 7 — หลักการเดียวกัน เพราะ `map()` รับ `Function<T,R>` ซึ่งไม่ประกาศ `throws`

---

## 10. กับดัก

- **คิดว่า intermediate operation ทำงานทันที** — ไม่มีอะไรเกิดขึ้นจนกว่าจะเจอ terminal operation (ข้อ 2) เขียน `filter`/`map` เฉย ๆ โดยไม่มี terminal จะไม่มีอะไรเกิดขึ้นเลย ไม่ error ด้วยซ้ำ
- **ใช้ stream ซ้ำ** — `IllegalStateException` เพราะ stream ถูกใช้ไปแล้ว (ข้อ 7)
- **เปลี่ยนเป็น `parallelStream()` โดยไม่วัดผล** — บางทีช้าลงกว่าเดิมเพราะ overhead การแบ่งงาน (ข้อ 8)
- **เขียนผลลงตัวแปรร่วมใน `forEach` ของ parallel stream** — race condition ที่หาสาเหตุยากเพราะเกิดแบบสุ่ม (ข้อ 8)
- **ใช้ stream ทำสิ่งที่ loop ธรรมดาอ่านง่ายกว่า** — บาง logic (มีหลาย early return, ต้องแก้ตัวแปรหลายตัวพร้อมกัน) เขียนเป็น for-loop อ่านง่ายกว่า stream chain ที่ยาวและซ้อนกันเยอะ ไม่ต้องฝืนใช้ stream ทุกที่
- **สับสน `Collectors.toList()` กับ `.toList()`** — ตัวหลัง (Java 16+) คืน immutable list เพิ่มเข้าไปทีหลังจะ throw `UnsupportedOperationException`
- **นับ element ด้วย `collect(Collectors.toList()).size()`** — เปลืองโดยไม่จำเป็น ใช้ `count()` ตรง ๆ

---

## 11. Cheat sheet

```java
// สร้าง stream
list.stream()
Arrays.stream(array)
Stream.of(a, b, c)
IntStream.range(0, 10)

// intermediate
.filter(predicate)   .map(function)   .flatMap(function)
.distinct()           .sorted()         .sorted(comparator)
.limit(n)             .skip(n)          .peek(consumer)   // debug เท่านั้น

// terminal
.forEach(consumer)    .collect(collector)   .toList()
.count()              .reduce(identity, accumulator)
.anyMatch(p) .allMatch(p) .noneMatch(p)
.findFirst()  .findAny()
.min(comparator) .max(comparator)

// Collectors ที่ใช้บ่อย
Collectors.toList() / toSet() / toMap(keyFn, valFn)
Collectors.joining(", ")
Collectors.groupingBy(fn)
Collectors.partitioningBy(predicate)
Collectors.counting()
Collectors.summarizingInt(fn)
```

| อาการ | สาเหตุ |
|---|---|
| `IllegalStateException: stream has already been operated upon` | เรียก terminal operation ซ้ำบน stream เดิม |
| เขียนโค้ดไม่ error แต่ไม่มีอะไรเกิดขึ้น | ลืมใส่ terminal operation |
| `UnsupportedOperationException` ตอนแก้ list | ใช้ `.toList()` (immutable) แทน `Collectors.toList()` |
| parallel stream ให้ผลลัพธ์ไม่คงที่ (สุ่ม) | เขียนผลลง collection ปกติใน `forEach` แทนที่จะใช้ `collect()` |
| ข้อมูลเยอะแต่ parallel ช้ากว่าเดิม | ไม่ได้วัดผลก่อนใช้ — parallel ไม่ได้เร็วกว่าเสมอ |
| `unhandled exception` ใน `map()`/`filter()` | functional interface มาตรฐานไม่ประกาศ `throws` — ต้องจับเองข้างใน |

---

## 🔗 เกี่ยวข้อง

- [[Java Lambda]] — พื้นฐานที่ Stream ใช้ทุกเมธอด
- [[Java]] — หน้ารวม
- [[Java Record]] — ใช้เป็น element ของ stream และ projection ได้ดี (`.map(Row::new)`)

## 📖 อ่านต่อ

- [Oracle — Aggregate Operations (Streams)](https://docs.oracle.com/javase/tutorial/collections/streams/)
- [Collectors — Java API docs](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/Collectors.html)
- [Oracle — Parallelism](https://docs.oracle.com/javase/tutorial/collections/streams/parallelism.html)
