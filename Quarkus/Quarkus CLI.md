---
tags:
  - quarkus
  - cli
  - tooling
type: reference
created: 2026-09-21
---

# ⌨️ Quarkus CLI

> **Quarkus CLI (`quarkus`) เป็นคำสั่งทางการที่ครอบ Maven/Gradle plugin ของ Quarkus ไว้อีกชั้น** — เขียนคำสั่งเดียวกันได้ไม่ว่าโปรเจกต์จะใช้ Maven หรือ Gradle เพราะ CLI ตรวจจับ build tool เองแล้วแปลไปเรียกของจริงข้างใต้ให้ (เช่น `quarkus dev` → `./mvnw quarkus:dev` หรือ `./gradlew quarkusDev` แล้วแต่โปรเจกต์) ไม่ต้องจำ syntax ของแต่ละ build tool แยกกัน

---

## 1. ติดตั้ง

| วิธี | คำสั่ง |
|---|---|
| SDKMAN! (Linux/macOS) | `sdk install quarkus` |
| Homebrew (Linux/macOS) | `brew install quarkusio/tap/quarkus` |
| Chocolatey (Windows) | `choco install quarkus` |
| Scoop (Windows) | `scoop install quarkus-cli` |
| JBang (cross-platform, ไม่ต้องมี Java มาก่อนก็ได้) | `curl -Ls https://sh.jbang.dev \| bash -s - app install --fresh --force quarkus@quarkusio` |

เช็คว่าลงสำเร็จ: `quarkus --version`

---

## 2. สร้างโปรเจกต์ใหม่ — แทนการเข้าเว็บ code.quarkus.io

```bash
quarkus create app com.acme:my-app
quarkus create app com.acme:my-app --dry-run   # ดูค่าที่จะใช้ก่อนสร้างจริง ยังไม่สร้างไฟล์
```

รูปแบบ coordinate คือ `groupId:artifactId:version` — ส่วนไหนไม่ใส่ใช้ค่า default ให้ (เช่น `quarkus create app my-app` ใช้ default groupId)

**กำหนดเวอร์ชัน Quarkus ที่จะใช้ได้ 2 แบบ:**

```bash
quarkus create app com.acme:my-app -P io.quarkus.platform:quarkus-bom:3.33.0   # ระบุ BOM ตรงๆ
quarkus create app com.acme:my-app -S quarkus:3.33                              # ระบุผ่าน platform stream
```

---

## 3. จัดการ extension ผ่าน CLI — ไม่ต้องแก้ `pom.xml` เอง

```bash
quarkus ext list                    # extension ที่ลงอยู่ในโปรเจกต์นี้แล้ว
quarkus ext list --installable      # extension ทั้งหมดที่เพิ่มได้ (ใช้ --search กรองชื่อ)
quarkus ext add hibernate-validator
quarkus ext add smallrye-*          # glob pattern — เพิ่มได้ทีเดียวหลายตัวที่ชื่อขึ้นต้นเหมือนกัน
quarkus ext remove hibernate-validator
```

**ผลลัพธ์เหมือนแก้ `pom.xml`/`build.gradle` เอง** แค่ CLI แก้ไฟล์ให้แทน — เปิดไฟล์ดูหลังรันคำสั่งได้ตามปกติ

---

## 4. Dev mode / build / test — คำสั่งเดียวไม่ว่า build tool ไหน

```bash
quarkus dev              # แทน ./mvnw quarkus:dev หรือ ./gradlew quarkusDev
quarkus build            # แทน ./mvnw package หรือ ./gradlew build
quarkus build --clean    # build ใหม่ทั้งหมด ไม่ใช้ cache เดิม
quarkus test              # continuous testing แยกจาก dev mode (ดู [[Quarkus Testing]])
```

**CLI เองไม่ได้ compile/run โค้ดตรง ๆ** — แค่ตรวจว่าโปรเจกต์เป็น Maven/Gradle/JBang แล้วสั่ง build tool ตัวจริงให้ทำงานแทน ถ้า build พังต้อง debug ที่ Maven/Gradle ปกติ ไม่ใช่ที่ตัว CLI

---

## 5. Container image — ไม่ต้องเขียน Dockerfile เอง

```bash
quarkus image build docker                                    # ใช้ Docker
quarkus image build podman                                    # ใช้ Podman
quarkus image build buildpack --builder-image <builder-image> # Cloud Native Buildpacks
quarkus image push --registry=quay.io/myorg/my-app
```

**ใช้ได้เลยโดยไม่ต้องเพิ่ม image extension ลงโปรเจกต์ถาวรก่อน** — CLI จัดการ classpath ชั่วคราวให้ตอนรันคำสั่งนี้เท่านั้น (ถ้าย้ายไป build วิธีอื่นที่ไม่ผ่าน CLI เช่นบาง CI setup ต้องเพิ่ม extension เองต่างหาก)

---

## 6. `info` / `update` — เช็คสุขภาพโปรเจกต์

```bash
quarkus info      # ดู extension + เวอร์ชันที่ใช้อยู่จริงในโปรเจกต์ พร้อมเช็คว่าตรง platform ล่าสุดไหม
quarkus update     # แนะนำ/ทำการอัปเดต dependency ให้ตรงกับ platform เวอร์ชันล่าสุด
```

---

## 7. Plugin — เพิ่มคำสั่งเองได้

```bash
quarkus plugin list
quarkus plugin add <name-or-location>   # จาก catalog, URL, หรือ Maven coordinate ก็ได้
quarkus plugin remove <name>
```

plugin เก็บที่ `~/.quarkus/cli/plugins/` (ระดับ user) หรือ `.quarkus/cli/plugins/` (ระดับโปรเจกต์ — override ของ user ได้)

---

## กับดัก

- **ติดตั้งผ่าน JBang แต่ JBang เวอร์ชันเก่ากว่า 0.56.0** — จัดการ app (`quarkus create app`) ทำงานไม่ถูกต้อง อัปเดต JBang ก่อนเสมอถ้าเจอปัญหาแปลก ๆ ตอนสร้างโปรเจกต์
- **คิดว่า `quarkus dev`/`quarkus build` มี logic รันเองแยกต่างหาก** — จริง ๆ เป็นแค่ wrapper แปลไปเรียก Maven/Gradle plugin ข้างใต้เท่านั้น (ข้อ 4) debug build error ต้องดูที่ Maven/Gradle output จริง ไม่ใช่ที่ตัว CLI
- **ลืมว่า CLI ต้องติดตั้งแยกในทุกเครื่อง/CI runner** — ต่างจาก `mvnw`/`gradlew` wrapper ที่ commit ติดไปกับ repo แล้วรันได้ทันทีทุกเครื่อง เครื่องใหม่ที่ไม่ได้ลง `quarkus` CLI ไว้ล่วงหน้าจะรันคำสั่งที่ขึ้นต้นด้วย `quarkus` ไม่ได้เลย — ถ้าต้องการ build script ที่รันได้แน่นอนทุกที่ ใช้ `./mvnw`/`./gradlew` ตรง ๆ แทนใน CI
- **เวอร์ชันของ CLI เองกับเวอร์ชัน Quarkus ที่โปรเจกต์ใช้อยู่คนละตัวกัน** — อัปเดต CLI (เช่นผ่าน SDKMAN) ไม่ได้แปลว่าโปรเจกต์อัปเดตตามอัตโนมัติ ต้องสั่ง `quarkus update` แยกต่างหาก (ข้อ 6)
- **`quarkus image build` โดยไม่มี Docker/Podman daemon รันอยู่** — เลือก builder (`docker`/`podman`/`buildpack`) ให้ตรงกับสิ่งที่มีอยู่จริงในเครื่อง/CI ไม่งั้น build จะ fail ตั้งแต่ขั้นตอนแรก

---

## Cheat sheet

```bash
quarkus create app com.acme:my-app
quarkus ext add hibernate-validator rest-client-jackson
quarkus dev
quarkus build
quarkus image build docker
quarkus info
quarkus update
```

| ต้องการ | คำสั่ง |
|---|---|
| สร้างโปรเจกต์ใหม่ | `quarkus create app <groupId:artifactId>` |
| ดู/เพิ่ม/ลบ extension | `quarkus ext list` / `quarkus ext add` / `quarkus ext remove` |
| dev mode | `quarkus dev` |
| build jar/native | `quarkus build` |
| build container image | `quarkus image build docker` |
| ดูข้อมูลโปรเจกต์ | `quarkus info` |
| อัปเดต dependency ตาม platform | `quarkus update` |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus Build]] — ขั้นตอน build เต็ม (JVM vs native, fast-jar) ที่ `quarkus build`/`quarkus image` เรียกใช้อยู่ข้างใต้
- [[Quarkus Project Structure]] — โครงสร้างไฟล์ที่ `quarkus create app` สร้างให้
- [[Quarkus Testing]] — `quarkus test` เทียบกับ continuous testing ที่เห็นใน dev mode
- [[Quarkus]] — หน้ารวม, ส่วน Agent Skill/MCP (`quarkus-agent-mcp` ใช้กลไกเดียวกับ CLI นี้คุม lifecycle ให้ AI agent)

## 📖 อ่านต่อ

- [Quarkus — Building Quarkus apps with the Quarkus CLI](https://quarkus.io/guides/cli-tooling/)
- [SDKMAN! — Quarkus CLI](https://sdkman.io/sdks/quarkus/)
