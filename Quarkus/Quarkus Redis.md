---
tags:
  - java
  - quarkus
  - redis
  - cache
type: reference
created: 2026-08-18
---

# 🔴 Quarkus + Redis

> **Redis ไม่ใช่แค่ที่เก็บ cache**
> มันคือ **โครงสร้างข้อมูลที่หลาย process ใช้ร่วมกันได้** ซึ่งเปิดทางให้ทำหลายอย่างที่ทำในหน่วยความจำของ JVM เดียวไม่ได้ — lock ข้ามเครื่อง, ตัวนับ atomic, คิว, สถานะที่แชร์กัน

---

## 1. Extension มีสองตัว เลือกให้ถูก

| | `quarkus-cache` (+ `quarkus-redis-cache`) | `quarkus-redis-client` |
|---|---|---|
| วิธีใช้ | annotation `@CacheResult` | `RedisDataSource` เรียก command ตรง |
| ทำอะไรได้ | cache ผลลัพธ์ method เท่านั้น | ทุก data structure ของ Redis |
| เหมาะกับ | cache ธรรมดา | lock, counter, queue, pub/sub |

ใช้ทั้งคู่พร้อมกันได้ ไม่ขัดกัน

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-redis-client</artifactId>
</dependency>
```

---

## 2. `quarkus-cache` — ทางที่ง่ายที่สุด

```java
@CacheResult(cacheName = "product")
public Product findByCode(String code) { ... }

@CacheInvalidate(cacheName = "product")
public void update(@CacheKey String code, Product p) { ... }

@CacheInvalidateAll(cacheName = "product")
public void reloadAll() { ... }
```

| annotation | ทำอะไร |
|---|---|
| `@CacheResult` | มีใน cache แล้วคืนเลย ไม่เรียก method |
| `@CacheInvalidate` | ลบ key เดียว |
| `@CacheInvalidateAll` | ล้างทั้ง cache |
| `@CacheKey` | เลือกว่าพารามิเตอร์ตัวไหนเป็น key (ถ้าไม่ระบุ ใช้ทุกตัวรวมกัน) |

### จุดขายจริงของมัน: เปลี่ยน backend โดยไม่แตะโค้ด

default เป็น **Caffeine** (in-memory ของ process นั้น) พอต้องการแชร์ข้ามเครื่อง เพิ่ม `quarkus-redis-cache` **โค้ดไม่ต้องแก้เลยสักบรรทัด**

```properties
quarkus.cache.type=redis
quarkus.cache.redis.expire-after-write=10M
```

key ใน Redis จะเป็น `cache:{cache-name}:{cache-key}`

**เริ่มด้วย Caffeine ก่อนเสมอ** — เร็วกว่ามาก (ไม่ต้องข้ามเน็ตเวิร์ก) ย้ายไป Redis เมื่อมีเหตุผลชัดเจนเท่านั้น: ต้องแชร์ข้ามเครื่อง หรือ cache ใหญ่เกิน heap

### ⚠️ กับดักของ annotation cache

- **เรียก method ในคลาสเดียวกันเอง cache ไม่ทำงาน** — มันทำงานผ่าน CDI proxy การเรียกภายในไม่ผ่าน proxy
- method ที่คืน `null` ก็ถูก cache ด้วย ระวังกรณี "ยังไม่มีข้อมูล" ถูกจำไว้
- ค่า default ไม่มี TTL ต้องตั้ง `expire-after-write` เอง

---

## 3. `RedisDataSource` — คุมเองทั้งหมด

```java
@Inject RedisDataSource ds;              // blocking
@Inject ReactiveRedisDataSource rds;     // reactive คืน Uni<T>
```

command แยกเป็นกลุ่มตามชนิดข้อมูล **ไม่ใช่ยิง string command ดิบ** — มี type safety

```java
ds.value(String.class)      // GET / SET / INCR
ds.hash(String.class)       // HSET / HGET
ds.list(String.class)       // LPUSH / BRPOP
ds.set(String.class)        // SADD / SISMEMBER
ds.sortedSet(String.class)  // ZADD / ZRANGE
ds.key()                    // DEL / EXPIRE / TTL
ds.pubsub(String.class)
ds.stream(String.class)
ds.bitmap() / ds.geo() / ds.hyperloglog()
```

---

## 4. Data structure ไหน ใช้ทำอะไร

| structure | เหมาะกับ |
|---|---|
| **String** | cache, counter (`INCR`), lock (`SETNX`) |
| **Hash** | object ที่อัปเดตทีละ field — ไม่ต้องอ่านมาทั้งก้อนแล้วเขียนกลับ |
| **List** | คิวง่าย ๆ `LPUSH` + `BRPOP` (blocking pop) |
| **Set** | เก็บของไม่ซ้ำ เช็คสมาชิก O(1) |
| **Sorted Set** | จัดอันดับ, คิวตามเวลา, sliding-window rate limit |
| **Stream** | event log ย้อนอ่านได้ มี consumer group |

---

## 5. รูปแบบการใช้งานที่เจอบ่อย

### 5.1 Cache-aside

รูปแบบพื้นฐานที่สุด — หาใน cache ก่อน ไม่เจอค่อยไป DB แล้วเก็บกลับ

```java
String v = ds.value(String.class).get(key);
if (v == null) {
    v = loadFromDb(key);
    ds.value(String.class).setex(key, 300, v);   // ← TTL ตั้งแต่แรก
}
```

**ใช้ `setex` ไม่ใช่ `set`** ตั้ง TTL ตอนเขียนเลย อย่าเขียนก่อนแล้วค่อย `expire` ทีหลัง เพราะถ้าโปรแกรมตายคั่นกลาง key นั้นจะไม่มีวันหมดอายุ

### 5.2 Distributed lock

`synchronized` กันได้แค่ภายใน JVM เดียว **ข้ามเครื่องไม่ได้** — งานที่รันหลาย instance ต้องใช้ lock กลาง

```java
// SET key value NX EX 300 — สำเร็จแค่คนแรก
boolean got = ds.value(String.class).setnx("lock:job:" + id, ownerId);
if (!got) return;      // คนอื่นทำอยู่
try {
    ds.key().expire("lock:job:" + id, 300);   // กันตายคาล็อก
    doWork();
} finally {
    ds.key().del("lock:job:" + id);
}
```

**ต้องมี TTL เสมอ** ไม่งั้น process ตายคาล็อกแล้วงานนั้นค้างตลอดกาล

> lock แบบนี้ **ไม่ปลอดภัย 100%** ในกรณีที่งานทำนานเกิน TTL แล้ว lock หมดอายุระหว่างทาง ถ้าต้องการความถูกต้องระดับสูงต้องดู Redlock หรือใช้ fencing token — แต่สำหรับงานส่วนใหญ่ที่แค่ "อย่าให้ทำซ้ำ" เท่านี้พอ

### 5.3 Rate limiting

```java
String k = "rate:" + userId + ":" + (epochSecond / 60);
long n = ds.value(Long.class).incr(k);
if (n == 1) ds.key().expire(k, 60);
if (n > 100) throw new TooManyRequestsException();
```

key ผูกกับนาทีปัจจุบัน พอขึ้นนาทีใหม่ key ใหม่เกิดเอง ของเก่าหมดอายุเอง ไม่ต้องมีงานล้าง

### 5.4 Idempotency key

กันการทำงานซ้ำจากการกดปุ่มรัวหรือ client retry

```java
if (!ds.value(String.class).setnx("idem:" + requestKey, "1")) {
    return previousResult;    // เคยทำไปแล้ว
}
ds.key().expire("idem:" + requestKey, 86400);
```

### 5.5 สถานะงาน background ที่แชร์ข้ามเครื่อง

งานที่รันใน worker thread คนละตัวกับ request → เก็บ progress ไว้ตรงกลาง เครื่องไหนรับ request ก็อ่านได้

```java
ds.hash(String.class).hset("task:" + id, Map.of(
    "status", "PROCESSING", "total", "5000", "done", "1200"));
```

### 5.6 Pub/Sub vs Streams vs message broker

| | Redis Pub/Sub | Redis Streams | RabbitMQ / Kafka |
|---|---|---|---|
| ไม่มีคนฟัง = ข้อความหาย | ✅ หาย | ❌ ไม่หาย | ❌ ไม่หาย |
| ย้อนอ่านย้อนหลัง | ไม่ได้ | ได้ | ได้ (Kafka) |
| routing ซับซ้อน | ไม่มี | ไม่มี | ✅ |

**ใช้ Pub/Sub เฉพาะงานที่ข้อความหายได้** เช่น broadcast ให้ทุก instance ล้าง cache พร้อมกัน
ถ้ามี message broker อยู่แล้ว **อย่าเอา Redis มาทำงานทับ**

---

## 6. Config

```properties
quarkus.redis.hosts=redis://localhost:6379
quarkus.redis.password=secret
quarkus.redis.timeout=10s
quarkus.redis.max-pool-size=6

# หลาย client
quarkus.redis.cache-store.hosts=redis://host-a:6379
quarkus.redis.session-store.hosts=redis://host-b:6379
```

```java
@Inject @RedisClientName("cache-store") RedisDataSource cacheDs;
```

รองรับ standalone / cluster / sentinel ผ่านรูปแบบ URL

### Dev Services — ของดีที่คนมักไม่รู้

**ถ้าไม่ตั้ง `quarkus.redis.hosts` ใน dev/test เลย Quarkus จะสตาร์ต Redis ในคอนเทนเนอร์ให้อัตโนมัติ** ไม่ต้องลง Redis บนเครื่อง ไม่ต้องเขียน docker-compose

ถึงต้องตั้งค่าสำหรับ prod ก็ใส่ prefix profile ไว้ แล้วปล่อย dev ให้ Dev Services จัดการ

```properties
%prod.quarkus.redis.hosts=redis://prod-host:6379
```

---

## 7. กับดักที่ต้องรู้

- **ไม่ตั้ง TTL = memory leak ช้า ๆ** Redis อยู่ใน RAM ล้วน key ที่ไม่มีวันหมดอายุจะกองจน OOM
  ตั้ง `maxmemory` + `maxmemory-policy allkeys-lru` ไว้เป็นตาข่ายกันตก แต่อย่าใช้แทนการตั้ง TTL
- **cache invalidation ยากกว่าที่คิด** — แก้ DB แล้วลืมล้าง cache = ผู้ใช้เห็นของเก่า
- **Redis ล่มแล้วแอปต้องไม่ล่มตาม** — ห่อด้วย try/catch แล้ว fallback ไป DB เสมอ ถือว่า Redis เป็น optimization ไม่ใช่ source of truth
- **serialization เปลี่ยนแล้วของเก่าพัง** — เปลี่ยนโครงสร้าง class แล้ว deserialize ของเดิมไม่ออก ใส่เวอร์ชันในชื่อ cache (`product-v2`) ตอน deploy ที่มีการเปลี่ยนโครงสร้าง
- **`KEYS *` ห้ามใช้บน prod** — มัน block ทั้ง server เพราะ Redis เป็น single-thread ใช้ `SCAN` แทน
- **Redis single-threaded** — คำสั่งช้าตัวเดียวบล็อกทุกคน ระวัง Lua script ยาว ๆ หรือ operation บน collection ใหญ่มาก
- **native image** — deserialize ด้วย reflection อาจต้องประกาศคลาสใน reflection config ดู [[Quarkus Build]]

---

## 8. Cheat sheet

```java
@Inject RedisDataSource ds;

// string
ds.value(String.class).set("k", "v");
ds.value(String.class).setex("k", 300, "v");    // + TTL
ds.value(String.class).get("k");
ds.value(Long.class).incr("counter");

// lock
boolean got = ds.value(String.class).setnx("lock:x", "owner");

// hash
ds.hash(String.class).hset("task:1", "status", "DONE");
ds.hash(String.class).hgetall("task:1");

// list เป็นคิว
ds.list(String.class).lpush("queue", "job1");
ds.list(String.class).brpop(Duration.ofSeconds(5), "queue");

// sorted set
ds.sortedSet(String.class).zadd("rank", 100, "user1");
ds.sortedSet(String.class).zrangeWithScores("rank", 0, 9);

// key
ds.key().expire("k", 60);
ds.key().ttl("k");
ds.key().del("k");
```

| อาการ                       | สาเหตุ                                             |
| --------------------------- | -------------------------------------------------- |
| Redis กิน RAM ขึ้นเรื่อย ๆ  | มี key ที่ไม่ได้ตั้ง TTL                           |
| เห็นข้อมูลเก่าหลังแก้ DB    | ลืม invalidate                                     |
| `@CacheResult` ไม่ทำงาน     | เรียก method จากในคลาสเดียวกัน ไม่ผ่าน CDI proxy   |
| lock ค้างหลัง process ตาย   | `SETNX` ไม่ได้ตั้ง TTL                             |
| deserialize พังหลัง deploy  | เปลี่ยนโครงสร้าง class โดยไม่เปลี่ยนชื่อ cache     |
| Redis ค้างทั้งตัวเป็นช่วง ๆ | มีคนรัน `KEYS *` หรือ operation บน collection ใหญ่ |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus]] — หน้ารวม
- [[Quarkus Build]] — reflection config ตอน native
- [[URL Shortener (Quarkus)]] — เคสที่พิจารณา Redis counter แล้วสรุปว่าไม่ใช้

## 📖 อ่านต่อ

- [Quarkus — Using Redis](https://quarkus.io/guides/redis)
- [Quarkus — Redis Reference](https://quarkus.io/guides/redis-reference)
- [Quarkus — Cache Guide](https://quarkus.io/guides/cache)
- [Quarkus — Redis Cache Reference](https://quarkus.io/guides/cache-redis-reference)
