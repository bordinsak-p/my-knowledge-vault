---
tags:
  - java
  - quarkus
  - config
type: reference
created: 2026-09-16
---

# ⚙️ Quarkus Config & Profiles

> **หลักการเดียวที่ต้องเข้าใจก่อนอย่างอื่น:** ค่า config หนึ่งตัวสามารถถูกตั้งได้จาก**หลายแหล่งพร้อมกัน** (ไฟล์, env var, system property, ...) — Quarkus ไม่ error ไม่งง มันมี**ลำดับความสำคัญตายตัว**ว่าแหล่งไหนชนะแหล่งไหนเสมอ
> ปัญหา "ตั้ง config แล้วทำไมไม่มีผล" เกือบทั้งหมดมาจากการตั้งค่าไว้ในแหล่งที่ **ลำดับต่ำกว่า** แหล่งที่มีค่าอยู่แล้ว

---

## 1. Config มาจากไหนได้บ้าง — เรียงตามลำดับความสำคัญ

**ค่าที่ ordinal สูงกว่าชนะเสมอ ไม่ว่าจะตั้งไว้กี่แหล่งก็ตาม**

| ลำดับ (ordinal) | แหล่ง | ใช้ตอนไหน |
|---|---|---|
| **400** | System property (`-Dkey=value`) | สั่งตรงตอนรัน `java -jar` |
| **300** | Environment variable | **มาตรฐานสำหรับ container / Kubernetes** |
| **295** | ไฟล์ `.env` ใน working directory | สะดวกตอน dev บนเครื่องตัวเอง |
| **260** | `config/application.properties` ข้าง ๆ ไฟล์ที่รัน (`$PWD/config/`) | แก้ config หลัง deploy โดยไม่แตะ image/jar เดิม |
| **250** | `application.properties` ที่อยู่ใน artifact (classpath) | ค่าตั้งต้นที่ build ติดไปด้วย |
| **100** | `META-INF/microprofile-config.properties` | มาตรฐาน MicroProfile Config (ใช้น้อยกว่า `application.properties`) |

**กฎการค้นหา: ไล่จาก ordinal สูงสุดลงมา เจอค่าแรกที่ไหนหยุดที่นั่นทันที** ไม่ได้รวมค่าจากทุกแหล่งเข้าด้วยกัน — ตั้งไว้ 5 ที่พร้อมกัน ใช้แค่ค่าจากที่ ordinal สูงสุดที่มีค่าเท่านั้น

```properties
# application.properties (ordinal 250)
app.timeout=30
```
```bash
# ถ้ามี env var นี้ด้วย (ordinal 300) — env var ชนะเสมอ ไม่ว่า application.properties จะเขียนว่าอะไร
APP_TIMEOUT=60
```

---

## 2. Profile — ค่าที่ต่างกันตามสภาพแวดล้อม

Quarkus มีให้ 3 profile มาตรฐาน: **`dev`**, **`test`**, **`prod`** — ใช้ prefix `%profile-name.` หน้าชื่อ property

```properties
# ไม่มี prefix = ใช้ทุก profile รวม prod ด้วย
quarkus.http.port=8080

# เจาะจงเฉพาะ profile
%dev.quarkus.log.level=DEBUG
%test.quarkus.datasource.jdbc.url=jdbc:h2:mem:test
%prod.quarkus.log.console.json.enabled=true
```

**⚠️ property ที่ไม่มี prefix มีผลกับทุก profile รวม prod** — ของที่ตั้งใจให้เปิดเฉพาะ dev (เช่น log สี, SQL logging, ข้อมูลตัวอย่าง) **ต้องใส่ `%dev.` เสมอ** ไม่งั้นหลุดไป prod โดยไม่ตั้งใจ

### profile ไหน active ตอนไหน — ตัดสินอัตโนมัติจากคำสั่งที่รัน

| รันด้วยคำสั่ง | active profile |
|---|---|
| `mvn quarkus:dev` | `dev` |
| `mvn test` | `test` |
| `mvn clean package` (แล้วรัน jar/native ปกติ) | **`prod`** |

**Quarkus build เป็น `prod` profile โดย default เสมอ** ไม่ว่าจะสั่ง build ตอนไหนก็ตาม (เว้นแต่ระบุ `-Dquarkus.profile=xxx` เอง) — เชื่อมกับ [[Quarkus Build]] ที่อธิบายไว้แล้วว่าทำไมต้องระบุ profile ให้ชัดตอน build จริง

### ตั้ง profile เองตอนรัน

```bash
java -Dquarkus.profile=prod -jar quarkus-run.jar
```

### Custom profile — ไม่ได้จำกัดแค่ 3 ตัว

```properties
%staging.quarkus.datasource.jdbc.url=jdbc:mariadb://staging-db:3306/app
%staging.app.feature.new-checkout=true
```

```bash
java -Dquarkus.profile=staging -jar quarkus-run.jar
```

ตั้งชื่อ profile อะไรก็ได้ตามต้องการ (`staging`, `qa`, `uat`, ...) — ใช้ prefix แบบเดียวกับ 3 ตัวมาตรฐาน

---

## 3. `@ConfigProperty` vs `@ConfigMapping` — inject ทีละตัว vs รวมเป็นกลุ่ม

### แบบกระจาย — `@ConfigProperty`

```java
@ConfigProperty(name = "app.timeout")
int timeout;

@ConfigProperty(name = "app.retry.max-attempts", defaultValue = "3")
int maxAttempts;
```

ใช้ได้เร็ว แต่พอมี config หลายตัวที่เกี่ยวข้องกัน จะกระจัดกระจายไปทั่วหลายคลาส ไม่มีที่รวมให้ดูภาพรวม

### แบบรวมกลุ่ม — `@ConfigMapping` (แนะนำเมื่อ config มีมากกว่า 2-3 ตัว)

```java
@ConfigMapping(prefix = "app.retry")
public interface RetryConfig {
    @WithDefault("3")
    int maxAttempts();

    @WithDefault("PT0.2S")
    Duration delay();

    Optional<String> strategy();     // ไม่มีค่า default = Optional
}
```

```properties
app.retry.max-attempts=5
app.retry.delay=PT1S
```

```java
@Inject RetryConfig retryConfig;

void doSomething() {
    int attempts = retryConfig.maxAttempts();
}
```

**ชื่อ method ในตัว interface ถูก map เป็นชื่อ property ต่อจาก prefix โดยอัตโนมัติ** (`maxAttempts()` → `app.retry.max-attempts`, แปลง camelCase เป็น kebab-case ให้เอง) — รองรับ nested interface สำหรับ config ที่ซ้อนกันเป็นชั้น ๆ ได้ด้วย

**ข้อดีเหนือ `@ConfigProperty`:** เห็น config ทั้งกลุ่มในที่เดียว, type-safe ตรวจตอน compile, validation annotation (`@Min`, `@NotBlank`) ใช้ได้ตรง ๆ บน method

---

## 4. Build-time vs Runtime config — คนละโลกกันเลย

Quarkus แบ่ง config เป็น 3 เฟส (`ConfigPhase`) ที่พฤติกรรมต่างกันโดยสิ้นเชิง:

| Phase | เปลี่ยนตอน runtime ได้ไหม | ตัวอย่าง |
|---|---|---|
| **`BUILD_TIME`** | ❌ ไม่ได้เลย — ต้อง build ใหม่ | `quarkus.package.type`, extension ไหนถูกเปิดใช้ |
| **`BUILD_AND_RUN_TIME_FIXED`** | ❌ อ่านได้ตอน runtime แต่ **ค่าที่ใช้จริงถูกตรึงไว้ตั้งแต่ build** | `quarkus.native.*` ส่วนใหญ่ |
| **`RUNTIME`** | ✅ เปลี่ยนได้อิสระหลัง build | `quarkus.datasource.jdbc.url`, `quarkus.log.level` |

**นี่ไม่ใช่แค่ทฤษฎี — เป็นสาเหตุอันดับหนึ่งของ "แก้ config แล้วไม่มีผล"**

```bash
# build ตอนนี้ profile เป็น dev (ไม่ได้ระบุ -Dquarkus.profile)
mvn clean package -Pnative

# พอ deploy จริงแล้วเปลี่ยน env var หวังว่าจะมีผล
QUARKUS_NATIVE_SOME_BUILD_TIME_FLAG=true java ...   # ❌ ไม่มีผล — ค่านี้ตรึงไว้ตั้งแต่ build แล้ว
```

**วิธีเช็คว่า property ตัวไหนเป็นเฟสไหน:** ดูเอกสารของ extension นั้น ๆ (Quarkus guide ทุกตัวจะมีคอลัมน์บอก "Type" ต่อท้ายชื่อ property) หรือลองเปลี่ยนตอน runtime แล้วดูว่า Quarkus เตือน `Build time property cannot be changed at runtime` ไหม (ถ้าเตือนแปลว่าเป็น build-time fixed)

รายละเอียดเรื่อง build-time config ที่ผูกกับ path ไฟล์ในคอนเทนเนอร์ ดูตัวอย่างจริงที่ [[Quarkus Build]] ข้อ 5

---

## 5. กับดัก

- **ตั้งค่าไว้หลายแหล่งแล้วงงว่าทำไมได้ค่าอื่น** — ตรวจ ordinal ก่อนเสมอ (ข้อ 1) โดยเฉพาะ env var (300) ที่ชนะ `application.properties` (250) เสมอ
- **ลืมใส่ `%dev.`** — ค่าที่ตั้งใจให้เฉพาะ dev หลุดไปมีผลบน prod ด้วย เพราะ property ไม่มี prefix ใช้ทุก profile (ข้อ 2)
- **แก้ build-time config แล้วรอผลตอน deploy** — ไม่มีวันเห็นผลจนกว่าจะ build ใหม่ (ข้อ 4) — โดยเฉพาะเจ็บกับ native build ที่ใช้เวลานาน
- **ลืมว่า build ปกติ = prod profile เสมอ** — ทดสอบบนเครื่องด้วย `mvn quarkus:dev` (profile `dev`) ผ่านหมด แต่ artifact จริงที่ build ออกมาใช้ config ของ `prod` ซึ่งอาจไม่เคยถูกทดสอบเลย
- **`.env` มีผลบนเครื่อง dev แต่ไม่มีผลตอน deploy จริง** — `.env` (ordinal 295) เป็นไฟล์ที่ Quarkus อ่านจาก working directory ตอน dev เท่านั้น ไม่ได้ถูก build เข้า artifact ไปด้วย
- **สับสน `@ConfigMapping` ไม่ map ตามที่คิด** — ชื่อ property ต้องเป็น kebab-case (`max-attempts`) ให้ตรงกับ method `maxAttempts()` ที่แปลงให้อัตโนมัติ ถ้าเขียน property เป็น camelCase เองจะหาไม่เจอ

---

## 6. Cheat sheet

```properties
# profile
quarkus.http.port=8080              # ทุก profile
%dev.quarkus.log.level=DEBUG        # เฉพาะ dev
%prod.quarkus.log.console.json.enabled=true

# ตั้งตอนรัน
java -Dquarkus.profile=staging -jar app.jar
```

```java
// กระจาย
@ConfigProperty(name = "app.timeout", defaultValue = "30")
int timeout;

// รวมกลุ่ม
@ConfigMapping(prefix = "app.retry")
public interface RetryConfig {
    @WithDefault("3") int maxAttempts();
    Optional<String> strategy();
}
```

| อาการ | สาเหตุ |
|---|---|
| ตั้ง env var แล้วไม่มีผล ทั้งที่ชื่อถูก | มีค่าจาก system property (`-D`) ซึ่ง ordinal สูงกว่าอยู่ก่อนแล้ว |
| ตั้งค่าใน `application.properties` แล้วไม่มีผล | มี env var ชื่อเดียวกัน (ordinal สูงกว่า) ตั้งอยู่ |
| config เฉพาะ dev หลุดไป prod | ลืมใส่ `%dev.` prefix |
| แก้ config แล้ว deploy ใหม่ยังไม่เห็นผล | เป็น build-time/build-and-runtime-fixed config ต้อง build ใหม่ |
| `@ConfigMapping` หาค่าไม่เจอทั้งที่ประกาศไว้แล้ว | ชื่อ property ใน `.properties` ไม่ใช่ kebab-case ตรงกับชื่อ method |
| `.env` ใช้ได้ตอน dev แต่หายไปตอน deploy | `.env` ไม่ได้ถูก build เข้า artifact — ต้องใช้ env var จริงหรือ `config/application.properties` แทน |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus]] — หน้ารวม
- [[Quarkus Build]] — build-time config ที่ผูกกับ path ไฟล์ในคอนเทนเนอร์ ตัวอย่างจริง
- [[Quarkus Logging]] — ตัวอย่าง config source หลายแหล่งที่ใช้กับ log level โดยเฉพาะ

## 📖 อ่านต่อ

- [Quarkus — Configuring your application](https://quarkus.io/guides/config)
- [Quarkus — Configuration reference guide](https://quarkus.io/guides/config-reference)
- [Quarkus — Mapping configuration to objects](https://quarkus.io/guides/config-mappings)
