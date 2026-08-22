---
tags:
  - ci-cd
  - docker
  - cache
  - github-actions
type: reference
created: 2026-08-18
---

# 🗂️ Docker Layer Caching ใน CI

> ก่อนอ่านโน้ตนี้ควรเข้าใจเรื่อง layer/cache ของ Docker พื้นฐานจาก [[Docker Dockerfile]] มาก่อน
> คำถามของโน้ตนี้คือ: **ทำไม cache ที่เร็วมากตอน build บนเครื่องตัวเอง กลับช้าเหมือนไม่มี cache เลยตอน build บน CI**

---

## 1. ทำไม cache หายทุกรอบใน CI

```
เครื่องตัวเอง (local)                    CI runner
──────────────────                     ──────────────────
build วันจันทร์ → cache เก็บไว้           build วันจันทร์ → cache เก็บไว้ใน runner
build วันอังคาร → ใช้ cache เดิม           runner ตัวนั้นถูกทำลายทิ้งหลัง job จบ!
   (เครื่องเดียวกัน ดิสก์เดียวกัน)          build วันอังคาร → runner ใหม่ทั้งหมด
                                              ไม่มี cache จาก "เมื่อวาน" หลงเหลือเลย
```

**runner ของ CI (โดยเฉพาะ cloud) เป็นเครื่องใหม่ทุกครั้ง (ephemeral)** — Docker cache ที่อยู่บนดิสก์ของ runner ตัวเก่าหายไปพร้อมกับ runner ตัวนั้น ไม่ได้ตกทอดไปยัง runner ตัวถัดไปเหมือนตอน build บนเครื่องตัวเองที่ดิสก์ยังอยู่เหมือนเดิมทุกวัน **ต้องบอก CI ให้เก็บ cache ไว้ที่อื่นแล้วดึงกลับมาเองทุกครั้ง**

---

## 2. วิธีแก้ — ส่ง cache ออกไปเก็บที่อื่น แล้วดึงกลับมาก่อน build

```yaml
# GitHub Actions ตัวอย่าง
steps:
  - uses: docker/setup-buildx-action@v3
  - uses: docker/build-push-action@v6
    with:
      push: true
      tags: myapp:latest
      cache-from: type=gha
      cache-to: type=gha,mode=max
```

```
รอบที่ 1 (ยังไม่มี cache)
  build ──► ทุก layer รันใหม่หมด ──► เก็บ cache ไว้ที่ GitHub Actions Cache (type=gha)

รอบที่ 2 (มี cache แล้ว)
  cache-from: type=gha ──► ดึง cache กลับมาก่อน build
  build ──► layer ที่ไม่เปลี่ยน = cache hit ──► เร็วขึ้นมาก
  cache-to: type=gha ──► อัปเดต cache เก็บกลับไปใหม่
```

**`cache-from` = ที่ที่จะไปหา cache มาใช้ก่อน build, `cache-to` = ที่ที่จะเก็บ cache ไว้หลัง build เสร็จ** ต้องมีทั้งสองบรรทัดคู่กันเสมอ ใส่แค่ `cache-from` อย่างเดียวจะไม่มีวันมี cache ใหม่ให้ดึงในรอบถัดไป

### `mode=max` สำคัญกว่าที่คิด

```yaml
cache-to: type=gha,mode=max     # เก็บทุก layer รวม intermediate stage
cache-to: type=gha              # (ไม่มี mode) เก็บแค่ layer ของ stage สุดท้าย
```

**ถ้าใช้ multi-stage build (ดู [[Docker Dockerfile]]) แล้วไม่ใส่ `mode=max`** cache จะเก็บแค่ layer ของ stage สุดท้ายเท่านั้น — พอ stage แรก (เช่น `npm install`) เปลี่ยน จะไม่มี cache ให้ใช้เลยทั้งที่ stage นั้นควร cache ได้

### ข้อจำกัดที่ต้องรู้

**พื้นที่ cache ของ GitHub Actions มีโควตารวมต่อ repo (ใช้ร่วมกันทุก cache ไม่ใช่แค่ Docker)** — repo ที่มี image ใหญ่หรือ layer เยอะมากอาจชนโควตา แล้ว cache เก่าจะถูกไล่ออกอัตโนมัติเพื่อให้พื้นที่ ผลคือบางรอบ cache hit บางรอบ miss ทั้งที่ทำเหมือนเดิม — ถ้าเจอปัญหานี้บ่อย ให้ทำ Dockerfile ให้ layer เล็กลง/น้อยลง หรือพิจารณาใช้ registry เป็นที่เก็บ cache แทน (ข้อ 3)

---

## 3. ทางเลือกอื่นนอกจาก `type=gha`

```yaml
# เก็บ cache ไว้ใน registry เดียวกับที่ push image (ทำงานได้ทุก CI ไม่ผูกกับ GitHub)
cache-from: type=registry,ref=myrepo/myapp:cache
cache-to: type=registry,ref=myrepo/myapp:cache,mode=max
```

| แบบ | ข้อดี | ข้อเสีย |
|---|---|---|
| `type=gha` | ตั้งง่ายที่สุด ไม่ต้องมี registry แยก | ผูกกับ GitHub Actions, มีโควตาจำกัด |
| `type=registry` | ใช้ได้กับทุก CI ไม่ผูกแพลตฟอร์ม, ควบคุมพื้นที่เก็บเอง | ต้องมี registry ที่เขียนได้ ต้อง push cache image แยก |
| `type=local` | ใช้ตอนมี self-hosted runner ที่ดิสก์อยู่ถาวร | ใช้ไม่ได้กับ cloud runner ที่เป็นเครื่องใหม่ทุกครั้ง |

**ถ้าใช้ self-hosted runner (ดู [[Self-Hosted Runners]]) ที่ดิสก์ไม่ได้ถูกล้างทุกรอบ** จริง ๆ แล้ว cache ของ Docker ปกติ (ไม่ต้องส่งออกไปไหน) ก็ใช้งานได้เลย เพราะเครื่องเดิมไม่ได้ถูกทำลายทิ้งเหมือน cloud runner — ปัญหาในข้อ 1 เกิดกับ ephemeral runner เท่านั้น

---

## 4. เชื่อมกับหลักการเดียวกันใน [[Quarkus Build]] — fast-jar

**fast-jar แยก dependency (`lib/`) ออกจากโค้ดแอป (`app/`) เพื่อให้ Docker layer cache ทำงานได้จริง** — เป็นหลักการเดียวกันเป๊ะกับเรื่องในโน้ตนี้ แค่คนละบริบท

```dockerfile
# เรียงแบบนี้ตั้งใจ — ของที่เปลี่ยนน้อยอยู่บน
COPY target/quarkus-app/lib/     /deployments/lib/      ← dependency ไม่ค่อยเปลี่ยน = cache hit บ่อย
COPY target/quarkus-app/*.jar    /deployments/
COPY target/quarkus-app/app/     /deployments/app/       ← โค้ดเราเปลี่ยนบ่อยที่สุด
COPY target/quarkus-app/quarkus/ /deployments/quarkus/
```

```
บน local: layer cache ทำงานได้เองเพราะ Docker daemon ตัวเดียวกันตลอด
บน CI: ต้องมี cache-from/cache-to (ข้อ 2) เพิ่มเข้าไป
       ไม่งั้นการเรียง layer ให้ดีใน Dockerfile จะไม่มีความหมายเลย
       เพราะ CI runner เป็นเครื่องใหม่ทุกครั้งอยู่ดี — ไม่มี "ของเก่า" ให้ cache hit ใส่
```

**นี่คือจุดที่คนพลาดบ่อย:** เขียน Dockerfile เรียง layer ถูกต้องแล้ว (ตามที่ [[Docker Dockerfile]] สอน) แต่ยัง build ช้าใน CI ทุกรอบ — เพราะ "การเรียง layer ให้ดี" กับ "การส่ง cache ข้าม CI run" เป็นคนละเรื่องกัน **ต้องทำทั้งคู่ครบถึงจะได้ผลจริง**

---

## 5. วัดผลว่า cache ทำงานจริงไหม

```yaml
- uses: docker/build-push-action@v6
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

เปิด log ของ step build แล้วดูคำว่า `CACHED` หน้าแต่ละ layer:

```
#5 [2/6] WORKDIR /app
#5 CACHED                              ← layer นี้ใช้ cache จริง

#7 [4/6] RUN npm install
#7 1.2s                                 ← layer นี้รันใหม่ (ไม่มี CACHED)
```

**ถ้าไม่เห็น `CACHED` เลยสักบรรทัดหลังตั้ง `cache-from`/`cache-to` แล้ว** แปลว่า cache ยังไม่ทำงาน ให้เช็คว่า Buildx/BuildKit เวอร์ชันรองรับ backend ที่ใช้อยู่ไหม (บาง backend ต้องการเวอร์ชันใหม่กว่าที่ runner มีให้เป็นค่าเริ่มต้น)

---

## 6. กับดัก

- **ตั้ง `cache-from` แต่ลืม `cache-to`** — ดึง cache ได้แต่ไม่เคยอัปเดต cache ใหม่ ผลคือ cache แช่แข็งอยู่ที่รอบแรกตลอดไป
- **ใช้ multi-stage แต่ไม่ใส่ `mode=max`** — cache เฉพาะ stage สุดท้าย stage build ที่กินเวลานานที่สุดไม่ถูก cache เลย
- **คิดว่าเรียง layer ถูกใน Dockerfile พอแล้ว** — ลืมว่าต้องส่ง cache ออกจาก runner ด้วย ไม่งั้นไม่มีผลกับ ephemeral runner เลย
- **cache โต จนชนโควตาแล้วโดนไล่ที่แบบสุ่ม** — build บาง commit เร็ว บาง commit ช้า ทั้งที่โค้ดเปลี่ยนเท่ากัน เพราะ cache หายไปตามโควตาไม่ใช่ตาม logic ของเรา
- **ใช้ `docker build` ธรรมดาแทน `docker buildx build`** — cache backend อย่าง `type=gha`/`type=registry` เป็นฟีเจอร์ของ Buildx ไม่ใช่ Docker build แบบเก่า

---

## 7. Cheat sheet

```yaml
steps:
  - uses: docker/setup-buildx-action@v3
  - uses: docker/build-push-action@v6
    with:
      push: true
      tags: ${{ steps.meta.outputs.tags }}
      cache-from: type=gha
      cache-to: type=gha,mode=max
```

| อาการ | สาเหตุ |
|---|---|
| local เร็ว, CI ช้าเหมือนไม่มี cache | ephemeral runner ไม่มี cache-from/cache-to ให้ |
| ตั้ง cache แล้วก็ยังไม่เร็วขึ้น | ลืม `cache-to` หรือลืม `mode=max` ตอนใช้ multi-stage |
| บาง build เร็วบางครั้งช้า สลับกันไป | cache โดนไล่ที่เพราะชนโควตาพื้นที่ |
| ไม่เห็น `CACHED` ใน log เลย | ไม่ได้ใช้ `docker buildx build`, หรือ BuildKit เวอร์ชันเก่าเกินไป |

---

## 🔗 เกี่ยวข้อง

- [[CI-CD]] — หน้ารวม
- [[Docker Dockerfile]] — layer/cache พื้นฐาน ก่อนอ่านโน้ตนี้
- [[Quarkus Build]] — fast-jar ใช้หลักการเดียวกัน
- [[GitHub Actions Workflow Syntax]] — `actions/cache` สำหรับ dependency ทั่วไป (คนละกลไกกับ Docker layer cache)

## 📖 อ่านต่อ

- [Docker Docs — Cache management with GitHub Actions](https://docs.docker.com/build/ci/github-actions/cache/)
- [Depot — How to use Docker layer caching in GitHub Actions](https://depot.dev/blog/docker-layer-caching-in-github-actions)
