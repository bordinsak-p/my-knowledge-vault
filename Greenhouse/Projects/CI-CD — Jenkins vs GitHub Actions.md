---
tags:
  - research
  - research/seed
  - devops
  - ci-cd
  - jenkins
  - github-actions
type: project
status: seed
area: devops
created: 2026-08-18
updated: 2026-08-18
---

# 🔁 CI/CD — Jenkins vs GitHub Actions

## 🎯 ปัญหา

ทุกวันนี้ build กับ deploy ทำมือทั้งหมด — `mvn package` → `docker build` → `docker tag` → `docker push` → `crictl pull` ที่ server
ปัญหาที่ตามมาคือ:

- **ขั้นตอนอยู่ในหัวคน** ถ้าคนที่ทำเป็นไม่อยู่ คนอื่นทำไม่ได้
- **ลืมขั้นตอนได้** เช่นลืม `-Dquarkus.profile=prod` แล้วขึ้น prod ไปหา template query ไม่เจอ
- **ไม่มีร่องรอย** ว่า image ตัวนี้มาจาก commit ไหน ใครสั่ง build เมื่อไหร่
- **ไม่มีใครรัน test** เพราะมันต้องสั่งเอง

## 💭 สมมติฐาน

การมี pipeline อัตโนมัติจะได้ประโยชน์หลัก **ไม่ใช่ความเร็ว** (build native 10-30 นาทีเท่าเดิม) แต่คือ**ความสม่ำเสมอและร่องรอย** — ทุก build ทำเหมือนกันทุกครั้ง และย้อนดูได้ว่าอะไรมาจากไหน

## ❓ ทำไมต้องตอนนี้

- ขั้นตอน build ที่ทำมือทุกวันนี้เขียนเป็นเอกสารไว้แล้ว = **มี spec ให้แปลงเป็น script อยู่แล้ว** ไม่ต้องเริ่มจากศูนย์
- อยากเรียนรู้ CI/CD จริงจัง มีของจริงให้ลองดีกว่าทำตาม tutorial ลอย ๆ

---

## 🔍 Prior art

### สองตัวเลือกต่างกันตรงไหน

| | **GitHub Actions** | **Jenkins** |
|---|---|---|
| ติดตั้ง | ไม่ต้อง — เป็น managed service | ต้องตั้ง server เอง + ดูแลเอง |
| เขียนด้วย | YAML | Groovy (Jenkinsfile) |
| ความชัน | ค่อย ๆ ชัน เริ่มง่าย | ชันกว่า แต่ยืดหยุ่นกว่ามาก |
| plugin | marketplace เยอะ | เยอะมาก แต่คุณภาพปนกัน ต้องดูแลเวอร์ชัน |
| เข้าถึงเครือข่ายภายใน | ❌ ไม่ได้ (ถ้าใช้ cloud runner) | ✅ ได้ ถ้าตั้งไว้ในวง |
| ค่าใช้จ่าย | ฟรีระดับหนึ่ง แล้วคิดตามการใช้ | ฟรี แต่จ่ายค่าเครื่องและเวลาดูแล |

### ⚠️ ข้อจำกัดที่ตัดสินเรื่องนี้เกือบทั้งหมด

**1. runner ของ GitHub แรมไม่พอสำหรับ native build**

| runner | สเปก |
|---|---|
| private repo (มาตรฐาน) | **2 vCPU / 8 GB** |
| public repo (มาตรฐาน) | 4 vCPU / 16 GB |
| larger runner | สูงสุด 64 core / 256 GB — **เสียเงิน** |

native build ต้องการ **8-16 GB** → บน private repo มาตรฐานคือคาบเส้นหรือไม่พอ ([[Quarkus Build]])

**2. เครือข่ายภายในเข้าไม่ถึงจากคลาวด์**

ถ้า pipeline ต้องเข้าถึงของที่อยู่หลัง firewall องค์กร — Maven repository ภายใน, Docker registry ภายใน, คลัสเตอร์ Kubernetes ที่ไม่ได้เปิดสู่สาธารณะ — **runner ที่อยู่บนอินเทอร์เน็ตมองไม่เห็นของพวกนี้**

ทางแก้มีสองแบบ: เอา runner เข้ามาอยู่ในวง (self-hosted) หรือเปิดช่องทางเข้าจากภายนอก ซึ่งแบบหลังมักติดนโยบายความปลอดภัย

### สรุปทางเลือกที่เหลือจริง ๆ

| ทาง | ได้ | เสีย |
|---|---|---|
| **Jenkins ในวง** | เข้าถึงทุกอย่างได้ คุมสเปกเครื่องเอง | ต้องดูแล server เอง |
| **GitHub Actions + self-hosted runner** | เขียน YAML ง่ายกว่า UI ดีกว่า | ยังต้องมีเครื่องในวงอยู่ดี |
| GitHub Actions + cloud runner | ไม่ต้องดูแลอะไร | **ทำได้แค่ build + test ไม่ถึง deploy** |

> **ข้อสังเกต:** ทางที่สามไม่ได้ไร้ค่า — แค่ "รัน test ทุก push" อย่างเดียวก็มีประโยชน์มากแล้ว และเริ่มได้วันนี้โดยไม่ต้องขออนุมัติอะไร

---

## 🧰 ต้องมีอะไรบ้าง

### ถ้าเริ่มด้วย GitHub Actions (แนะนำสำหรับการเรียนรู้)

- [ ] repo อยู่บน GitHub
- [ ] ไฟล์ `.github/workflows/ci.yml`
- [ ] **แค่นี้** — ไม่ต้องติดตั้งอะไรเลย

```yaml
name: CI
on:
  push:
    branches: [ main, bde ]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven

      - name: Build
        run: mvn -B clean package -DskipTests

      - name: Test
        run: mvn -B test
```

### ถ้าเป็น Jenkins

- [ ] เครื่อง (VM หรือ container) แรมอย่างน้อย 4 GB สำหรับตัว Jenkins เอง
- [ ] Java 17 บนเครื่องนั้น
- [ ] Docker บน agent (ถ้าจะ build image)
- [ ] plugin: Pipeline, Git, Docker Pipeline, Credentials
- [ ] credential: Maven repository, Docker registry, kubeconfig
- [ ] ไฟล์ `Jenkinsfile` ที่ root ของ repo

```groovy
pipeline {
    agent any
    tools { maven 'M3'; jdk 'jdk17' }

    stages {
        stage('Build') {
            steps { sh 'mvn -B clean package -DskipTests' }
        }
        stage('Test') {
            steps { sh 'mvn -B test' }
            post { always { junit 'target/surefire-reports/*.xml' } }
        }
        stage('Native') {
            when { branch 'main' }
            steps {
                sh '''mvn clean package -Pnative \
                      -Dquarkus.native.container-runtime=docker \
                      -Dquarkus.profile=prod -DskipTests=true'''
            }
        }
        stage('Image') {
            when { branch 'main' }
            steps {
                script {
                    def tag = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
                    sh "docker build -f src/main/docker/Dockerfile.native -t app:${tag} ."
                }
            }
        }
    }
    post {
        failure { echo 'build failed' }
    }
}
```

### สิ่งที่ต้องมีไม่ว่าจะเลือกทางไหน

- [ ] **ที่เก็บ secret** — รหัส Maven repository / Docker registry **ห้ามอยู่ในไฟล์ pipeline**
- [ ] **กฎการตั้ง tag** — `latest` อย่างเดียวไม่พอ ต้องผูกกับ commit ให้ย้อนได้
- [ ] **ข้อตกลงเรื่อง branch** — build อะไรตอนไหน, อะไรถึงจะ deploy ได้

---

## 🚶 เริ่มต้นยังไง — ทีละขั้น

**ห้ามทำ CD ตั้งแต่วันแรก** อันนี้คือคำแนะนำที่สำคัญที่สุดในโน้ตนี้
คนส่วนใหญ่พังตรงที่พยายามทำ build + test + deploy อัตโนมัติพร้อมกัน แล้วเจอปัญหาสามชั้นพร้อมกันจนแยกไม่ออกว่าอะไรพัง

| ขั้น | ทำอะไร | ได้อะไร |
|---|---|---|
| **1** | build + test ทุก push | รู้ทันทีว่าโค้ดพัง — คุ้มที่สุดต่อแรงที่ลง |
| **2** | build native | เจอปัญหา native ตั้งแต่ CI ไม่ใช่ตอนจะ deploy |
| **3** | build image + push registry | ได้ artifact ที่ย้อนรอยได้ |
| **4** | deploy dev อัตโนมัติ | เห็นของจริงเร็วขึ้น |
| **5** | deploy prod แบบกดยืนยัน | **ต้องมีคนกด ไม่ใช่อัตโนมัติ** |

**อยู่ที่ขั้น 1 ให้นานพอจนมันน่าเบื่อ** ค่อยขยับ

---

## ✅ รู้ได้ยังไงว่าสำเร็จ

- push แล้วรู้ผล build ภายใน **5 นาที** โดยไม่ต้องทำอะไรเอง
- ชี้ image บน registry ได้ว่ามาจาก commit ไหน
- คนที่ไม่เคย build โปรเจกต์นี้ deploy ได้โดยไม่ต้องถามใคร
- ขั้นตอนที่เคยเขียนไว้เป็นเอกสาร กลายเป็นโค้ดหมดแล้ว ไม่มีอะไรเหลือให้ทำมือ

---

## 🧪 การทดลองที่เล็กที่สุด

**อย่าเริ่มกับโปรเจกต์ที่มีคนใช้จริง** — เริ่มกับ [[URL Shortener (Quarkus)]] ใน Greenhouse

เหตุผล:
- เป็น repo ใหม่ ตั้ง public ได้ → **runner ฟรี 4 vCPU / 16 GB**
- ไม่มี dependency ภายในบริษัท → ไม่ติดเรื่องเครือข่าย
- พังได้ ไม่กระทบใคร
- feedback loop เร็วกว่ามาก

**เป้าหมาย:** ไฟล์ `ci.yml` ที่ build + test ทุก push แล้วขึ้นเครื่องหมายถูกสีเขียวบน GitHub — **ประมาณ 1 ชั่วโมง**
พอเข้าใจกลไกแล้วค่อยเอามาปรับใช้กับของจริงที่มีข้อจำกัดเยอะกว่า

---

## ⚠️ ความเสี่ยง

- **secret รั่ว** — เผลอ commit รหัสลงไฟล์ pipeline หรือ echo ออก log อันนี้เป็นความเสี่ยงอันดับหนึ่งของ CI/CD
- **build native กิน CPU เครื่อง Jenkins ยาว 30 นาที** — ถ้ามีคนใช้ร่วมกันจะแย่ง resource กัน ต้องแยก agent
- **ทำ CD เร็วเกินไป** แล้วมีของขึ้น prod โดยไม่ตั้งใจ
- **pipeline กลายเป็นอีกระบบที่ต้องดูแล** — Jenkins ที่ไม่มีใครอัปเดต plugin คือช่องโหว่ที่รอเวลา

---

## 📚 ต้องไปเรียนรู้เพิ่ม

เขียนเป็นโน้ตเต็มแล้วทั้ง 6 หัวข้อ ดูที่ [[CI-CD]]

- [[GitHub Actions Workflow Syntax]] — job / step / matrix / cache
- [[CI-CD Secrets]] — GitHub Secrets vs Jenkins Credentials, OIDC
- [[Self-Hosted Runners]] — ตั้งยังไง ปลอดภัยแค่ไหน โดยเฉพาะกับ repo public
- [[Docker Layer Caching in CI]] — เกี่ยวกับ fast-jar ใน [[Quarkus Build]]
- [[Versioning and Tagging Strategy]] — SemVer / Git SHA / CalVer เลือกยังไง
- [[Quarkus Testing]] — เอา test ที่ comment ไว้กลับมาให้รันได้จริง (ไม่งั้น stage test ก็ไร้ความหมาย)

---

## 🔗 เกี่ยวข้อง

- [[Quarkus Build]] — ขั้นตอน build ที่จะเอามาแปลงเป็น pipeline และข้อจำกัดเรื่องแรม
- [[URL Shortener (Quarkus)]] — สนามทดลองที่เหมาะที่สุด
- [[CI-CD]] — โน้ตความรู้เต็มทั้ง 6 หัวข้อที่ทิ้งไว้ด้านบน
- [[Greenhouse]]

---

## 📝 Log

### 2026-08-18
- สร้างโน้ต รวบรวมข้อมูล
- **เจอข้อจำกัดสำคัญ:** GitHub cloud runner (private repo = 2 vCPU / 8 GB) แรมไม่พอสำหรับ native build และเข้าถึงของที่อยู่หลัง firewall ไม่ได้
- ตัดสินใจเบื้องต้น: **เรียนรู้ด้วย GitHub Actions บน repo public เล็ก ๆ ก่อน** แล้วค่อยตัดสินใจเรื่องของจริง
- ยังไม่ได้ตอบ: มีเครื่องในวงให้ตั้ง Jenkins / self-hosted runner ไหม ← ต้องรู้ก่อนถึงจะขยับเป็น `sprouting`

---

## 🪦 ถ้าเลิกทำ

เลิกถ้า: ไม่มีเครื่องในวงให้ใช้ **และ** ทีมไม่ได้เห็นปัญหาว่าการ build มือเป็นเรื่องน่ากังวล
จะกลับมาคิดใหม่เมื่อ: มีคนที่สองต้อง deploy เป็น หรือมีเหตุ deploy พลาดจนเสียหายจริง
