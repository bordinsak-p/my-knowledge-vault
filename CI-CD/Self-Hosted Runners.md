---
tags:
  - ci-cd
  - github-actions
  - security
  - runner
type: reference
created: 2026-08-18
---

# 🏃 Self-Hosted Runner — ตั้งยังไง เสี่ยงตรงไหน

> runner ของ GitHub/cloud คือเครื่องที่ **ยืมเขาใช้ชั่วคราว** สร้างใหม่ทุกรอบ build แล้วทิ้ง
> **self-hosted runner คือเครื่องของเราเอง** ที่ตั้งไว้ให้ CI มารันงานได้ — ข้อดีคือคุมสเปกได้ เข้าถึงเครือข่ายภายในได้ แต่แลกมาด้วยว่า**ต้องดูแลความปลอดภัยเอง** ซึ่ง cloud runner ไม่ต้องคิดเรื่องนี้เลย

---

## 1. ทำไมถึงต้องมี self-hosted runner

| ต้องการ | cloud runner | self-hosted |
|---|---|---|
| เข้าถึงเครือข่ายภายในองค์กร (DB, internal API) | ❌ ทำไม่ได้ | ✅ |
| สเปกเฉพาะ (RAM/CPU เยอะ, GPU) | จำกัดตามที่ผู้ให้บริการมี | ✅ กำหนดเอง |
| build ที่ใช้เวลานาน/ใช้ทรัพยากรเยอะ (เช่น native image) | อาจไม่พอ (ดู [[Quarkus Build]]) | ✅ |
| ไม่ต้องดูแลอะไรเลย | ✅ | ❌ ต้องดูแลเอง |
| ปลอดภัยโดย default | ✅ เครื่องใหม่ทุกครั้ง | ❌ ต้องตั้งเอง |

**เกณฑ์ตัดสินใจง่าย ๆ:** ถ้า cloud runner ทำงานได้ครบ ใช้ cloud ก่อนเสมอ ใช้ self-hosted เฉพาะตอนมีเหตุผลจากตารางด้านบนจริง ๆ

---

## 2. ตั้ง self-hosted runner (GitHub Actions)

```
Repo/Org → Settings → Actions → Runners → New self-hosted runner
```

GitHub จะให้คำสั่งสำเร็จรูปมา หน้าตาประมาณนี้ (รันบนเครื่องที่จะเป็น runner):

```bash
mkdir actions-runner && cd actions-runner
curl -o actions-runner.tar.gz -L https://github.com/actions/runner/releases/download/.../actions-runner-linux-x64-....tar.gz
tar xzf actions-runner.tar.gz
./config.sh --url https://github.com/<org>/<repo> --token <TOKEN>
./run.sh
```

```yaml
# ใน workflow — สั่งให้ job นี้รันบน self-hosted runner แทน cloud
jobs:
  build:
    runs-on: self-hosted        # หรือ runs-on: [self-hosted, linux, high-mem] ถ้าตั้ง label ไว้
```

**label ใช้แยกว่างานไหนควรไปเครื่องไหน** ถ้ามี runner หลายเครื่องสเปกไม่เหมือนกัน ตั้ง label ตอน `config.sh` แล้วเลือกใน `runs-on` เป็น array ได้

---

## 3. ⚠️ ความเสี่ยงที่แท้จริง — ทำไม self-hosted อันตรายกว่าที่คิด

**Self-hosted runner คือเครื่องที่รันโค้ดใครก็ได้ที่มีสิทธิ์เขียน workflow** — ถ้าไม่ระวัง กลายเป็นการเปิดประตูให้คนนอกรันโค้ดบนเครื่องในองค์กรได้ตรง ๆ

### สถานการณ์ที่อันตรายที่สุด: repo public + self-hosted runner

```
1. คนแปลกหน้า fork repo public ของเรา
2. แก้ workflow ในฟอร์กของตัวเอง ใส่โค้ดอะไรก็ได้
3. เปิด Pull Request กลับมาที่ repo เรา
4. ถ้า workflow ตั้งค่าให้รันบน self-hosted runner ของเรา
   → โค้ดที่เขาเขียน (ใน PR) รันบนเครื่องเราจริง ๆ
5. เครื่องนั้นอาจเข้าถึง secret, เครือข่ายภายใน, credential ที่ cache ไว้
```

```
❌ กฎเหล็ก: repo public + self-hosted runner = อันตรายเสมอ
   เว้นแต่จะตั้งค่าป้องกันอย่างระมัดระวังมาก (ข้อ 4)
```

**เหตุผลที่ runner กลายเป็นเป้าหมายที่น่าสนใจสำหรับผู้โจมตี:** มันมักเข้าถึงเครือข่ายภายใน มี credential cache ไว้ และเป็นจุดที่ **ถูกออกแบบมาให้รันโค้ดที่ไม่รู้จักได้อยู่แล้ว** (นั่นคือหน้าที่ของ CI) ต่างจากเซิร์ฟเวอร์อื่นในองค์กรที่ปกติไม่มีใครคาดหวังว่าจะรันโค้ดแปลกหน้า

---

## 4. ถ้าจำเป็นต้องใช้กับ repo public — ตั้งค่าป้องกันเหล่านี้

```
Settings → Actions → General → Fork pull request workflows
```

| ตัวเลือก | ควรตั้งเป็น |
|---|---|
| Run workflows from fork pull requests | **ปิด** เว้นแต่เข้าใจความเสี่ยงจริง ๆ |
| Require approval for all outside collaborators | **เปิด** — ต้องมีคนกด approve ก่อน workflow จาก fork จะรันทุกครั้ง |

**"Require approval" คือด่านป้องกันสำคัญที่สุด** — แม้จะไม่ได้ปิด fork PR ไปเลย อย่างน้อยมีคนตรวจโค้ดก่อนปล่อยให้รันบนเครื่องจริง

### ใช้ ephemeral runner — เครื่องใช้แล้วทิ้ง

```
แทนที่จะมี runner เครื่องเดียวรันตลอดไป (persistent)
ตั้งให้สร้าง runner ใหม่ทุกครั้งที่มีงาน แล้วทำลายทิ้งหลังงานเสร็จ (ephemeral)
```

```
persistent runner                    ephemeral runner
──────────────────                  ──────────────────
รันงาน A → รันงาน B → รันงาน C        รันงาน A → ทำลายทิ้ง
   (สภาพเครื่องสะสมไปเรื่อย ๆ)          สร้างใหม่ → รันงาน B → ทำลายทิ้ง
   ถ้างาน A ทิ้งของอันตรายไว้            สร้างใหม่ → รันงาน C → ทำลายทิ้ง
   งาน B เห็นของนั้นได้ (persistence)    งานแต่ละงานไม่เห็นกันเลย
```

**ป้องกันการฝังตัว (persistence)** — ถ้าโค้ดร้ายรันได้สักครั้ง มันจะพยายามฝังตัวเองไว้รอโจมตีต่อ (เช่นใส่ backdoor, scheduled task) ephemeral runner ทำลายทุกอย่างทิ้งทันทีที่งานจบ ไม่มีอะไรให้ฝังตัวอยู่ได้

### อย่า checkout และรันโค้ดจาก PR head โดยไม่ระวัง

```yaml
# ⚠️ อันตรายถ้าเป็น self-hosted + fork PR
on: pull_request_target      # trigger นี้เห็น secret ของ repo หลักได้
jobs:
  build:
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}   # ← checkout โค้ดจาก fork มารัน
```

**`pull_request_target` ผสมกับการ checkout โค้ดจาก fork คือสูตรอันตรายที่สุด** — trigger นี้ให้สิทธิ์เห็น secret ของ repo หลัก (ต่างจาก `pull_request` ธรรมดา) แต่ถ้าเอาโค้ดจาก fork (ที่ใครก็แก้ได้) มารันด้วย เท่ากับให้ secret กับโค้ดที่ไม่น่าเชื่อถือมาเจอกันตรง ๆ

---

## 5. Self-hosted runner กับ repo private/internal — ปลอดภัยกว่ามาก

ถ้า repo เป็น private และจำกัดคนที่ push/เปิด PR ได้อยู่แล้ว ความเสี่ยงจาก fork แทบไม่มี (คนนอกเปิด PR ไม่ได้ตั้งแต่แรก) — **นี่คือเหตุผลที่ self-hosted runner ส่วนใหญ่ในองค์กรใช้กับ private repo** ความเสี่ยงที่เหลือจะเป็นเรื่องความปลอดภัยของตัวเครื่อง runner เอง ไม่ใช่เรื่อง fork

---

## 6. ดูแลเครื่อง runner เอง — สิ่งที่ cloud runner ทำให้ฟรี ตอนนี้ต้องทำเอง

| เรื่อง | ต้องคิดเอง |
|---|---|
| อัปเดต OS/patch security | ไม่มีใครทำให้ ต้องตั้ง schedule เอง |
| พื้นที่ดิสก์เต็ม | image/cache สะสมเรื่อย ๆ ถ้าไม่ล้าง (`docker system prune` ดู [[Docker Basics]]) |
| เครื่องล่ม | ไม่มี auto-healing เหมือน cloud ต้องมี monitoring/alert เอง |
| scale ตาม load | เครื่องเดียวรับงานได้จำกัด ต้องคิดเรื่อง runner pool เอง |
| แยกงานคนละ trust level | เครื่องเดียวรันทั้งงาน trusted และ (อาจ) untrusted ปนกันถ้าไม่ระวัง |

---

## 7. กับดัก

- **เปิด self-hosted runner ให้ repo public โดยไม่ตั้งค่าป้องกัน** — ความเสี่ยงข้อ 3 เกิดขึ้นจริงมาแล้วหลายกรณีในโลกจริง ไม่ใช่แค่ทฤษฎี
- **persistent runner ที่ไม่เคยล้างเลย** — ของสะสมข้ามงานเรื่อย ๆ ทั้ง cache แปลก ๆ และความเสี่ยงเรื่อง persistence
- **`pull_request_target` + checkout โค้ดจาก fork** — เห็น secret ได้พร้อมรันโค้ดที่ไม่น่าเชื่อถือ (ข้อ 4)
- **runner เครื่องเดียวใช้รันทั้งงาน production deploy และงานทดลองทั่วไป** — งานที่ไม่สำคัญพังแล้วดึงเครื่องที่ควรสงวนไว้สำหรับงานสำคัญไปด้วย
- **ลืมอัปเดต runner agent เอง** — runner software เก่าอาจมีช่องโหว่หรือใช้กับฟีเจอร์ใหม่ของ GitHub ไม่ได้

---

## 8. Cheat sheet

```yaml
jobs:
  build:
    runs-on: self-hosted              # หรือ [self-hosted, label-อื่น]
```

**Checklist ก่อนเปิด self-hosted runner ให้ repo public:**

- [ ] "Require approval for all outside collaborators" เปิดอยู่
- [ ] ไม่มี workflow ไหนใช้ `pull_request_target` ร่วมกับการ checkout โค้ดจาก fork
- [ ] ใช้ ephemeral runner ไม่ใช่ persistent
- [ ] แยก runner สำหรับงานที่แตะ secret สำคัญ ออกจาก runner ทั่วไป
- [ ] มี monitoring ว่า runner ล่มหรือดิสก์เต็มไหม

| อาการ | สาเหตุ |
|---|---|
| โค้ดแปลกหน้ารันบนเครื่องในองค์กรได้ | self-hosted runner เปิดให้ repo public โดยไม่ป้องกัน fork PR |
| runner เครื่องเดิมมีพฤติกรรมแปลก ๆ ข้ามงาน | ใช้ persistent runner แทน ephemeral |
| ดิสก์เต็มเรื่อย ๆ | ไม่ได้ล้าง Docker image/cache เป็นประจำ |
| secret รั่วผ่าน PR จาก fork | ใช้ `pull_request_target` ผิดวิธี |

---

## 🔗 เกี่ยวข้อง

- [[CI-CD]] — หน้ารวม
- [[CI-CD Secrets]] — secret ที่ runner เข้าถึงได้คือสิ่งที่มีค่าที่สุดที่ต้องป้องกัน
- [[Docker Basics]] — ดูแลพื้นที่ดิสก์ของ runner ที่ใช้ Docker
- [[CI-CD — Jenkins vs GitHub Actions]] — บริบทของโปรเจกต์ทดลองที่เคยพิจารณาเรื่องนี้

## 📖 อ่านต่อ

- [GitHub Docs — Security hardening for self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/security-hardening-for-github-actions)
- [Wiz — Hardening GitHub Actions: Lessons from Recent Attacks](https://www.wiz.io/blog/github-actions-security-guide)
