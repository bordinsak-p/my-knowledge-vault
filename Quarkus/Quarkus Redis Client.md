---
tags:
  - java
  - quarkus
  - redis
type: reference
created: 2026-09-29
---

# 🔌 Quarkus Redis Client — `RedisDataSource` API เต็มรูปแบบ

> **โน้ตนี้เจาะ API ของตัว `quarkus-redis-client` เอง** — command group มีอะไรบ้าง, reactive vs blocking ต่างกันตรงไหน, transaction, raw command, serialization, testing
> ถ้าหาสูตรใช้งานจริง (cache-aside, distributed lock, rate limit, idempotency key, pub/sub vs streams) ไปที่ [[Quarkus Redis]] แทน — สองโน้ตนี้เสริมกัน ไม่ทับกัน โน้ตนั้นตอบ "จะใช้ Redis ทำอะไรได้" โน้ตนี้ตอบ "API หน้าตาเป็นยังไง"

---

## 1. Dependency + inject สองแบบ

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-redis-client</artifactId>
</dependency>
```

```java
@Inject RedisDataSource ds;                  // blocking — เรียกแล้วรอผลตรงๆ
@Inject ReactiveRedisDataSource reactiveDs;   // reactive — คืน Uni<T>/Multi<T>
```

**ทั้งสองตัว inject พร้อมกันในคลาสเดียวกันได้** ไม่ขัดกัน เลือกใช้ตามบริบท (ข้อ 4)

---

## 2. Command group — ครบทุกตัว

`RedisDataSource`/`ReactiveRedisDataSource` ไม่มี method ยิง string command ดิบแบบ `jedis`/`lettuce` ทั่วไป — แบ่งเป็น**กลุ่มตามชนิดข้อมูล** แต่ละกลุ่ม type-safe ตาม type parameter ที่ระบุตอนเรียก

| accessor                  | คืน interface                     | ตัวอย่าง method                                          | ใช้กับ Redis type   |
| ------------------------- | --------------------------------- | -------------------------------------------------------- | ------------------- |
| `ds.value(V.class)`       | `ValueCommands<String, V>`        | `get`/`set`/`setex`/`incr`/`append`/`mget`/`mset`        | String              |
| `ds.hash(V.class)`        | `HashCommands<String, String, V>` | `hset`/`hget`/`hgetall`/`hdel`/`hincrby`                 | Hash                |
| `ds.list(V.class)`        | `ListCommands<String, V>`         | `lpush`/`rpush`/`lpop`/`brpop`/`lrange`                  | List                |
| `ds.set(V.class)`         | `SetCommands<String, V>`          | `sadd`/`smembers`/`sismember`/`sinter`                   | Set                 |
| `ds.sortedSet(V.class)`   | `SortedSetCommands<String, V>`    | `zadd`/`zrange`/`zscore`/`zrangeWithScores`              | Sorted Set          |
| `ds.key()`                | `KeyCommands<String>`             | `expire`/`ttl`/`del`/`exists`/`rename`/`type`            | ทุก key             |
| `ds.pubsub(V.class)`      | `PubSubCommands<V>`               | `publish`/ดู [[Quarkus Redis]] ข้อ 5.6                   | Pub/Sub             |
| `ds.stream(V.class)`      | `StreamCommands<String, V>`       | `xadd`/`xrange`/`xreadgroup`                             | Stream              |
| `ds.bitmap()`             | `BitMapCommands<String>`          | `setbit`/`getbit`/`bitcount`                             | Bitmap              |
| `ds.geo(V.class)`         | `GeoCommands<String, V>`          | `geoadd`/`geodist`/`geosearch`                           | Geo (บน sorted set) |
| `ds.hyperloglog(V.class)` | `HyperLogLogCommands<String, V>`  | `pfadd`/`pfcount`                                        | HyperLogLog         |
| `ds.json()`               | `JsonCommands<String>`            | RedisJSON module (ต้องมี module ติดตั้งที่ Redis server) | JSON                |

**แต่ละ type ใน Redis เอาไปใช้ทำอะไรจริง (พร้อมตัวอย่าง) ดู [[Redis]] ข้อ 2 และ 4** — โน้ตนี้บอกแค่ว่า Java เรียกยังไง ไม่สอน "ใช้ทำอะไร" ซ้ำอีกที

**ฟอร์มด้านบนคือ convenience overload ที่ตรึง key เป็น `String` ให้อัตโนมัติ** — ถ้า key ไม่ใช่ `String` (เช่นเป็น `Long`/enum) แต่ละ group มี overload เต็มที่รับ type ของ key เพิ่ม เช่น `ds.hash(K.class, F.class, V.class)` — ใช้ฟอร์มสั้นไปก่อนจนกว่าจะมีเหตุผลจริงที่ key ต้องไม่ใช่ String

```java
ValueCommands<String, Long> counters = ds.value(Long.class);
counters.incr("visits");

HashCommands<String, String, Person> people = ds.hash(Person.class);
people.hset("user:1", "profile", new Person("Somchai", 30));
```

---

## 3. Type parameter คือ serialization ไม่ใช่แค่ syntax

`ds.value(Person.class)` ไม่ได้แค่บอก compiler ว่าเป็น type ไหน — มันบอก client ว่า**ต้อง (de)serialize ยังไง** ก่อนเก็บ/หลังอ่านจาก Redis (ซึ่งเก็บได้แค่ byte/string ล้วนๆ อยู่แล้ว)

| type ที่ระบุ | serialize ยังไง |
|---|---|
| `String.class`, `Long.class`, `Integer.class`, primitive wrapper อื่นๆ | แปลงตรงไปตรงมา ไม่ผ่าน JSON |
| class ธรรมดา/record (เช่น `Person.class`) | **แปลงเป็น JSON ให้อัตโนมัติผ่าน `quarkus-jackson`** ไม่ต้องตั้งค่าเพิ่ม |
| generic collection (เช่น `List<Person>`) | ใช้ `new TypeReference<List<Person>>(){}` แทน `.class` ตรงๆ (ลบ type parameter ไม่ได้ตอน runtime) |

```java
HashCommands<String, String, List<Person>> h =
    ds.hash(new TypeReference<List<Person>>() {});
```

**เปลี่ยน serialization เองได้** — implement `io.quarkus.redis.datasource.codecs.Codec` แล้วประกาศเป็น CDI bean (`@ApplicationScoped`) เมื่ออยากคุม format เอง (เช่น Protobuf, หรือ backward-compat format ตอน migrate) แทน JSON default

> **กับดักเรื่อง JSON default:** เปลี่ยน field ของ class ที่เคยเก็บไว้ใน Redis แล้ว (เพิ่ม/ลบ/เปลี่ยน type field) deserialize ของเก่าที่เก็บไว้ก่อนหน้าอาจพังหรือได้ค่า default ผิดเงียบๆ — วิธีแก้เดียวกับที่ [[Quarkus Redis]] ข้อ 7 เขียนไว้: ใส่เวอร์ชันในชื่อ key/cache ตอน deploy ที่เปลี่ยนโครงสร้าง

---

## 4. Reactive vs Blocking — เลือกให้ตรงบริบท thread

```java
// blocking — เรียกแล้วรอผลตรงๆ บล็อก thread ปัจจุบันจนกว่าจะได้คำตอบ
String v = ds.value(String.class).get("key");

// reactive — คืน Uni ทันที ไม่บล็อก ต้อง subscribe/return ต่อเป็น pipeline
Uni<String> uni = reactiveDs.value(String.class).get("key");
```

| | Blocking (`RedisDataSource`) | Reactive (`ReactiveRedisDataSource`) |
|---|---|---|
| ใช้ตอนไหน | REST endpoint ปกติที่รันบน worker thread (`@Blocking` โดย default หรือ signature ไม่คืน `Uni`/`Multi`) | endpoint ที่เขียนเป็น reactive pipeline (คืน `Uni`/`Multi`) รันบน event loop |
| บล็อก event loop ไหมถ้าเรียกผิดที่ | ✅ เรียก blocking API บน event loop thread ได้ผลเสียจริง (ดู [[Quarkus REST Layer]] เรื่องทำไม block event loop ร้ายแรง) | ไม่บล็อก แต่ต้องเขียนเป็น chain ของ `Uni`/`Multi` ให้ถูก ไม่ใช่ subscribe แล้วรอเอง |

**กฎง่ายที่สุด:** endpoint/service เดิมเป็น blocking อยู่แล้ว → ใช้ `RedisDataSource` ต่อได้ตรงๆ ไม่ต้องเปลี่ยนสไตล์ทั้งคลาส แค่เพราะ Redis call เดียว

---

## 5. Transaction — `withTransaction` (MULTI/EXEC)

```java
TransactionResult result = ds.withTransaction(tx -> {
    HashCommands<String, String, String> hash = tx.hash(String.class);
    hash.hset("user:1", "status", "ACTIVE");
    hash.hset("user:1", "updated_at", Instant.now().toString());
});
```

**คำสั่งข้างในไม่ execute ทันทีทีละบรรทัด** — ถูก queue ไว้ทั้งหมดก่อน แล้วส่งเป็น `MULTI ... EXEC` ก้อนเดียวตอนจบ block (atomic — ไม่มี command อื่นแซกกลางได้) **ผลลัพธ์ของคำสั่งก่อนหน้าอ่านไม่ได้ทันทีข้างใน block เดียวกัน** เพราะยังไม่ execute จริง

ถ้าต้องอ่านค่าก่อนแล้วเอาไปตัดสินใจต่อในทรานแซกชันเดียวกัน (optimistic locking) ใช้ overload ที่มี pre-block:

```java
OptimisticLockingTransactionResult<Boolean> result = ds.withTransaction(
    // อ่านค่าปัจจุบันก่อน (นอก MULTI)
    preTx -> preTx.value(Integer.class).get("stock:sku1"),   
    
    (stock, tx) -> {   // ใช้ค่านั้นตัดสินใจ ใน MULTI จริง
        if (stock > 0) tx.value(Integer.class).decr("stock:sku1");
    },
    "stock:sku1"   // watched key — ถ้าค่านี้ถูกแก้ระหว่างทาง transaction ยกเลิกอัตโนมัติ
);
```

`watchedKey` ทำงานแบบเดียวกับ optimistic lock ทั่วไป — ถ้ามีใครแก้ key นั้นระหว่างช่วง "อ่าน" กับ "commit" ธุรกรรมจะถูกยกเลิกให้เอง (ไม่ commit ค่าที่อาจจะเพี้ยนไปแล้ว)

---

## 6. Pipelining — เปิดอยู่แล้ว ไม่ต้องเขียนอะไรเพิ่ม

client ของ Quarkus (สร้างจาก Vert.x Redis client) **ส่งคำสั่งแบบ pipeline บน connection เดียวกันโดย default อยู่แล้ว** ไม่มี API แยกให้ "เปิด/ปิด" pipelining เหมือน client บางตัว (เช่น Jedis ที่ต้องเปิด `Pipeline` object เอง) — เขียนเรียกคำสั่งต่อๆกันตามปกติ ก็ได้ประโยชน์เรื่อง round-trip ลดลงอยู่แล้วโดยไม่ต้องรู้ตัว

---

## 7. Raw command — escape hatch เมื่อไม่มี typed group ให้ใช้

```java
Response response = ds.execute("MY.CUSTOMCOMMAND", "arg1", "arg2");
```

**ทุก argument ต่อจากชื่อคำสั่งต้องเป็น `String`** เท่านั้น (ไม่มี overload รับ type อื่น) ใช้เมื่อ:
- Redis module ที่ยังไม่มี command group แบบ typed ให้ (module ใหม่/เฉพาะทาง)
- คำสั่งใหม่ของ Redis เวอร์ชันล่าสุดที่ client ยังไม่ได้ห่อ typed API ให้ทัน

---

## 8. Client หลายตัว — `@RedisClientName`

```properties
quarkus.redis.cache-store.hosts=redis://host-a:6379
quarkus.redis.session-store.hosts=redis://host-b:6379
```

```java
@Inject @RedisClientName("cache-store") RedisDataSource cacheDs;
@Inject @RedisClientName("session-store") RedisDataSource sessionDs;
```

รายละเอียด config เต็ม (standalone/cluster/sentinel, Dev Services) ดู [[Quarkus Redis]] ข้อ 6

---

## 9. Testing — Dev Services โหลดข้อมูลตั้งต้นให้ได้

```properties
%test.quarkus.redis.load-script=import.redis      # รันตอนเริ่มแอปใน dev/test
%test.quarkus.redis.flush-before-load=true         # ล้าง DB ก่อน import ทุกครั้ง (default อยู่แล้ว)
```

ไฟล์ `import.redis` เป็น command ดิบ 1 คำสั่งต่อบรรทัด (ห่อด้วย `MULTI`/`EXEC` ให้อัตโนมัติเพื่อความ atomic) — เหมาะกับ seed ข้อมูลตั้งต้นก่อนรันเทสที่ต้องมีของอยู่ก่อนแล้ว พื้นฐาน Dev Services (auto-start Redis ในคอนเทนเนอร์) ดู [[Quarkus Redis]] ข้อ 6

---

## กับดัก

- **ใช้ `.execute()` ส่ง argument ที่ไม่ใช่ `String`** — compile ไม่ผ่านหรือต้อง `.toString()` เองทุกตัว (ข้อ 7)
- **อ่านผลลัพธ์ของคำสั่งก่อนหน้าทันทีข้างใน `withTransaction`** — ยังไม่ execute จริงจนกว่า block จะจบ ได้ค่าผิด/ไม่มีค่า ต้องใช้ overload ที่มี pre-block + watched key ถ้าต้องอ่านมาตัดสินใจก่อน (ข้อ 5)
- **เปลี่ยน field ของ class ที่ serialize เป็น JSON เก็บไว้แล้ว** — ของเก่าใน Redis deserialize พังหรือผิดเงียบๆ (ข้อ 3, เทคนิคแก้เดียวกับ [[Quarkus Redis]] ข้อ 7)
- **เรียก `RedisDataSource` (blocking) จาก endpoint ที่เป็น reactive/รันบน event loop** — บล็อก event loop ทั้งเส้น กระทบ request อื่นที่ไม่เกี่ยวข้องเลย (ข้อ 4)
- **ใช้ generic collection (`List<Person>`) กับ `.class` ตรงๆ** — compile ไม่ผ่านหรือ type ไม่ตรง ต้องใช้ `TypeReference` แทน (ข้อ 3)
- **native image** — custom `Codec` หรือ class ที่ serialize เป็น JSON อาจต้องประกาศ reflection config เพิ่ม ดู [[Quarkus Build]]

---

## Cheat sheet

```java
@Inject RedisDataSource ds;
@Inject ReactiveRedisDataSource reactiveDs;

// command group (key = String โดย default)
ds.value(V.class)      ds.hash(V.class)             ds.list(V.class)
ds.set(V.class)        ds.sortedSet(V.class)        ds.key()
ds.pubsub(V.class)     ds.stream(V.class)           ds.bitmap()
ds.geo(V.class)        ds.hyperloglog(V.class)      ds.json()

// generic collection
ds.hash(new TypeReference<List<Person>>() {});

// transaction
ds.withTransaction(tx -> tx.value(String.class).set("k", "v"));

// raw command
ds.execute("MY.CMD", "arg1", "arg2");

// client หลายตัว
@Inject @RedisClientName("cache-store") RedisDataSource cacheDs;
```

| อาการ | สาเหตุ |
|---|---|
| อ่านค่าใน `withTransaction` แล้วได้ผลผิด/ไม่มีค่า | คำสั่งก่อนหน้ายังไม่ execute จริงจนกว่า block จะจบ (ข้อ 5) |
| deserialize object เก่าจาก Redis พังหลัง deploy | เปลี่ยนโครงสร้าง class ที่เคย serialize เป็น JSON ไว้ (ข้อ 3) |
| `execute()` compile ไม่ผ่าน | ส่ง argument ที่ไม่ใช่ `String` (ข้อ 7) |
| event loop ช้าทั้งระบบตอนเรียก Redis | ใช้ `RedisDataSource` (blocking) ผิดที่ ควรใช้ตัว reactive (ข้อ 4) |
| `List<Person>` เป็น type parameter compile ไม่ผ่าน | ต้องใช้ `TypeReference` แทน `.class` กับ generic type (ข้อ 3) |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus Redis]] — สูตรใช้งานจริง (cache-aside, distributed lock, rate limit, idempotency, pub/sub vs streams vs message broker) ต่อยอดจาก client API ในโน้ตนี้
- [[Quarkus REST Layer]] — reactive vs blocking ในบริบท request thread/event loop
- [[Quarkus Build]] — reflection config ที่อาจต้องเพิ่มสำหรับ native image
- [[Quarkus]] — หน้ารวม

## 📖 อ่านต่อ

- [Quarkus — Using the Redis Client](https://quarkus.io/guides/redis)
- [Quarkus — Redis Extension Reference Guide](https://quarkus.io/guides/redis-reference)
- [Quarkus Redis Client — Javadoc](https://javadoc.io/doc/io.quarkus/quarkus-redis-client/latest/index.html)
