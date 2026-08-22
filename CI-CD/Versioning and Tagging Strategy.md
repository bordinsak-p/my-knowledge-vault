---
tags:
  - ci-cd
  - versioning
  - docker
  - git
type: reference
created: 2026-08-18
---

# 🏷️ กลยุทธ์การตั้ง Tag / เลขเวอร์ชัน

> คำถามที่ต้องตอบก่อนตั้ง tag ทุกครั้ง: **"เห็น tag นี้แล้ว ย้อนกลับไปหา commit ต้นทางได้ไหม"** ถ้าตอบไม่ได้ ระบบ tag นั้นยังไม่พร้อมใช้งานจริง

---

## 1. สามระบบหลัก — และใช้ต่างกันคนละจุดประสงค์

```
SemVer          1.4.2              ← สื่อสารกับ "คน" ว่า compatible แค่ไหน
Git SHA          a1b2c3d            ← สื่อสารกับ "เครื่อง" ว่ามาจาก commit ไหนแน่นอน
CalVer           2026.08.18         ← สื่อสารว่า "เก่าแค่ไหน" เห็นปุ๊บรู้ทันที
```

**ไม่ต้องเลือกแค่อันเดียว** — ระบบที่ดีมักใช้ทั้งสามแบบผสมกัน คนละ tag บน image เดียวกัน (ข้อ 5)

---

## 2. SemVer — สื่อสารกับคน

```
MAJOR . MINOR . PATCH
  1   .   4   .   2
  │       │       │
  │       │       └── แก้บั๊ก ไม่มีอะไรพังถ้าอัปเดต
  │       └── เพิ่มฟีเจอร์ใหม่ ยังใช้ของเดิมได้ปกติ (backward compatible)
  └── เปลี่ยนแบบที่ของเดิมอาจพัง (breaking change)
```

**ประโยชน์หลักคือบอก "ความเสี่ยงของการอัปเดต" ให้คนที่ใช้ image ของเรา** โดยไม่ต้องอ่าน changelog เอง — เห็น `1.x → 2.x` รู้ทันทีว่าต้องเช็คก่อนอัปเดต ต่างจาก `1.4.x → 1.5.x` ที่มั่นใจได้ว่าไม่พัง

### multi-tag ตาม SemVer

```
1.4.2      ← ตายตัวเป๊ะ เวอร์ชันเดียว immutable
1.4        ← อัปเดตอัตโนมัติทุก patch (1.4.0, 1.4.1, 1.4.2, ...)
1          ← อัปเดตอัตโนมัติทุก minor/patch (1.x.x ทั้งหมด)
```

ผู้ใช้เลือกได้ว่าจะ pin แน่นแค่ไหน — งาน production มัก pin ที่ `1.4.2` เป๊ะ ส่วนงาน dev อาจใช้ `1` เพื่อได้ของใหม่อัตโนมัติ

**ข้อจำกัดของ SemVer:** เหมาะกับ "library/application ที่มีคนอื่นมาใช้" (npm package, Docker base image) แต่ **ไม่ได้บอกว่ามาจาก commit ไหน** — สอง build จาก branch คนละอันอาจได้เลข SemVer ชนกันโดยไม่ตั้งใจถ้าไม่มีวินัยเรื่อง release ที่ดีพอ

---

## 3. Git SHA — สื่อสารกับเครื่อง ตรวจสอบย้อนกลับได้

```
myapp:a1b2c3d4e5f6...
```

**เอกลักษณ์เฉพาะ (unique) เสมอ ไม่มีวันชนกัน** เพราะผูกกับ commit hash ตรง ๆ — เห็น tag นี้ ไล่กลับไปดู `git show a1b2c3d` ได้ทันทีว่าโค้ดตอนนั้นหน้าตาเป็นยังไง

```yaml
# ตัวอย่างใน GitHub Actions — ใช้ short SHA เป็นส่วนหนึ่งของ tag
- run: echo "sha_short=$(git rev-parse --short HEAD)" >> "$GITHUB_OUTPUT"
  id: vars
- uses: docker/build-push-action@v6
  with:
    tags: myapp:${{ steps.vars.outputs.sha_short }}
```

**ควร tag ด้วย Git SHA เสมอ ไม่ว่าจะมีระบบ tag อื่นด้วยหรือไม่** — เพราะเป็น tag เดียวที่การันตี trace กลับไปหาต้นทางได้แบบไม่มีข้อโต้แย้ง SemVer/CalVer อาจตั้งผิดโดยมนุษย์ได้ แต่ SHA มาจาก git โดยตรง

---

## 4. CalVer — สื่อสารว่า "เก่าแค่ไหน"

```
2026.08.18          หรือ  2026.08
      │   │                    │
    ปี  เดือน.วัน            ปี.เดือน (ละเอียดน้อยกว่า)
```

**เหมาะกับของที่ "ความใหม่" สำคัญกว่า "ความเข้ากันได้"** เช่น base image ที่อัปเดต patch ความปลอดภัยบ่อย ๆ, ข้อมูล snapshot รายวัน, tool ที่ออกเวอร์ชันตามรอบเวลาไม่ใช่ตามฟีเจอร์

**ข้อดีที่ SemVer ให้ไม่ได้:** เห็น `myimage:2023.01.15` ปุ๊บ **รู้ทันทีโดยไม่ต้องเช็คอะไรเพิ่มว่ามันเก่าและอาจขาด security patch** ต่างจาก `myimage:1.2.0` ที่ตัวเลขไม่ได้บอกอะไรเรื่องเวลาเลย

---

## 5. ใช้ร่วมกัน — pattern ที่ใช้จริงในงาน

**หนึ่ง build หนึ่งครั้ง ได้หลาย tag ชี้ไป image เดียวกัน**

```yaml
- uses: docker/metadata-action@v5
  id: meta
  with:
    images: myrepo/myapp
    tags: |
      type=semver,pattern={{version}}
      type=sha,prefix=
      type=raw,value=latest,enable={{is_default_branch}}

- uses: docker/build-push-action@v6
  with:
    push: true
    tags: ${{ steps.meta.outputs.tags }}
```

```
push tag v1.4.2 บน git ──► build ครั้งเดียว ──► ได้ image เดียวกัน แปะหลาย tag:
                                                   myrepo/myapp:1.4.2
                                                   myrepo/myapp:a1b2c3d
                                                   myrepo/myapp:latest   (เฉพาะ default branch)
```

**`docker/metadata-action` เป็นตัวช่วยมาตรฐานสำหรับ GitHub Actions** — คำนวณ tag ที่ควรใช้จาก git ref อัตโนมัติ (branch, git tag, PR number) ไม่ต้องเขียน logic เองทุกโปรเจกต์

---

## 6. `latest` — ใช้ได้ แต่ต้องรู้ขอบเขต

```
✅ เหมาะกับ: local dev, demo เร็ว ๆ, ตัวอย่างในเอกสาร
❌ ไม่เหมาะกับ: production deploy, การอ้างอิงใน compose.yaml ของระบบจริง
```

**ปัญหาของ `latest`:** ไม่ immutable — วันนี้ `latest` อาจเป็นคนละ image กับเมื่อวาน โดยไม่มีร่องรอยว่าเปลี่ยนตอนไหน ถ้า production อ้างอิง `latest` แล้วมีคน push image ใหม่ที่พัง **production พังตามทันทีโดยไม่มีใครตั้งใจ deploy**

**production ควรอ้างอิง tag ที่ตายตัวเสมอ** — SemVer เป๊ะ (`1.4.2`) หรือ Git SHA — ไม่ใช่ `latest` หรือ SemVer แบบลอย (`1` / `1.4`)

---

## 7. Digest — ระดับที่ immutable แน่นอนที่สุด

```
myrepo/myapp@sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

**tag เปลี่ยนชี้ไป image อื่นได้เสมอ (ผูกกับชื่อ ไม่ใช่เนื้อหา)** แต่ **digest ผูกกับเนื้อหาไฟล์จริง ๆ** — เปลี่ยนเนื้อหาแม้แต่ byte เดียว digest เปลี่ยนทันที

```
tag        1.4.2  ────► อาจถูกสั่ง push ทับด้วยเนื้อหาใหม่ได้ (ถ้า registry อนุญาต)
digest     sha256:e3b0...  ────► ไม่มีทางถูกสวมรอยได้เลย
```

**ใช้ digest เมื่อความถูกต้อง 100% สำคัญที่สุด** เช่น production deployment ที่ต้องมั่นใจว่าได้ image ตัวที่ผ่านการทดสอบจริง ๆ ไม่ใช่ตัวที่ชื่อ tag เดียวกันแต่เนื้อหาถูกแก้ไปแล้ว — หลาย organization บังคับ pin ด้วย digest ในระบบที่ security สำคัญมาก

---

## 8. กับดัก

- **ใช้ `latest` ใน production** — deploy โดยไม่รู้ว่าได้เวอร์ชันไหน ย้อนกลับไม่ได้ว่า deploy ตัวไหนไปเมื่อไหร่
- **ตั้งเลข SemVer มือ ไม่ได้ผูกกับ git tag จริง** — คนลืมอัปเดตเลข หรือสองคน bump เลขชนกัน
- **ไม่มี Git SHA tag เลยสักตัว** — เวลามีปัญหา production ไม่มีทางรู้แน่ชัดว่า image นั้นมาจาก commit ไหน ต้องเดา
- **CalVer ใช้กับของที่ compatibility สำคัญกว่าความใหม่** — ผู้ใช้ไม่รู้ว่าอัปเดตจาก `2026.01` ไป `2026.08` จะพังอะไรบ้าง เพราะ CalVer ไม่สื่อเรื่อง breaking change เหมือน SemVer
- **สับสน tag กับ digest** — คิดว่า pin tag แล้วปลอดภัย 100% ทั้งที่ tag เปลี่ยนเนื้อหาได้ (ถ้า registry ไม่ได้ตั้ง immutable tag ไว้)

---

## 9. Cheat sheet

```
production          → SemVer เป๊ะ (1.4.2) หรือ Git SHA — ไม่ใช้ latest
dev/demo             → latest ก็ได้ ยอมรับความเสี่ยง
base image ที่อัปเดตบ่อย → CalVer (2026.08.18)
security สูงสุด        → digest (sha256:...)
เก็บไว้ทุกกรณี          → Git SHA เสมอ เพราะ trace กลับ commit ได้แน่นอนที่สุด
```

| อาการ | สาเหตุ |
|---|---|
| production พังทันทีโดยไม่มีใคร deploy | อ้างอิง `latest` แล้วมีคน push image ใหม่ทับ |
| ไม่รู้ว่า image ที่ deploy อยู่มาจาก commit ไหน | ไม่มี Git SHA tag เลย |
| SemVer เลขชนกันระหว่างสองคน | ตั้งเลขมือแทนที่จะผูกกับ git tag/release process |
| pin tag แล้วยัง "โดนเปลี่ยนเนื้อหา" ได้ | เข้าใจผิดว่า tag = immutable ต้องใช้ digest ถึงจะการันตีจริง |

---

## 🔗 เกี่ยวข้อง

- [[CI-CD]] — หน้ารวม
- [[GitHub Actions Workflow Syntax]] — ใช้ output จาก step มาคำนวณ tag
- [[Docker Layer Caching in CI]] — cache image มักผูกกับ tag แยกต่างหาก (เช่น `:cache`)
- [[CI-CD — Jenkins vs GitHub Actions]] — checklist เรื่อง "กฎการตั้ง tag" ที่เคยทิ้งไว้ในโปรเจกต์นั้น

## 📖 อ่านต่อ

- [Semantic Versioning 2.0.0](https://semver.org/)
- [Docker Docs — Image tagging](https://docs.docker.com/build/building/best-practices/#tag-images)
