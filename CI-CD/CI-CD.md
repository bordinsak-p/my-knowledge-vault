---
tags:
  - ci-cd
  - devops
  - ci-cd/index
type: moc
created: 2026-08-18
---

# 🔁 CI/CD

หน้ารวมโน้ตชุด CI/CD — ต่อยอดจากโปรเจกต์ [[CI-CD — Jenkins vs GitHub Actions]] ใน [[Greenhouse]]
6 หัวข้อนี้คือรายการ "ต้องไปเรียนรู้เพิ่ม" ที่ทิ้งไว้ในโน้ตนั้น ตอนนี้เขียนเป็นเนื้อหาเต็มแล้ว

---

## โน้ตในนี้

| โน้ต | ว่าด้วย |
|---|---|
| [[GitHub Actions Workflow Syntax]] | job / step / matrix / cache — โครงสร้าง YAML ทั้งหมด |
| [[CI-CD Secrets]] | GitHub Secrets vs Jenkins Credentials, OIDC, การ mask log |
| [[Self-Hosted Runners]] | ตั้งยังไง เสี่ยงตรงไหน โดยเฉพาะกับ repo public |
| [[Docker Layer Caching in CI]] | ทำไม cache หายทุกรอบใน CI ทั้งที่ local เร็ว, ผูกกับหลักการเดียวกับ fast-jar |
| [[Versioning and Tagging Strategy]] | SemVer vs Git SHA vs CalVer เลือกยังไง ใช้ร่วมกันยังไง |
| [[Quarkus Testing]] | `@QuarkusTest` + REST Assured — เขียนเทสให้ stage test ใน pipeline มีความหมายจริง |

---

## กฎรวมที่จำไว้ได้เลย

1. **CI runner เป็นของชั่วคราวเสมอ** — ไม่มีอะไรอยู่ข้ามรอบ build เว้นแต่จะสั่ง cache ไว้ชัดเจน ต่างจากเครื่อง dev ที่ของสะสมอยู่ตลอด
2. **secret ที่ไม่ต้องเก็บ ดีกว่า secret ที่เก็บอย่างดี** — OIDC ที่แลก token ชั่วคราวปลอดภัยกว่าการเก็บ long-lived key ไว้เสมอ
3. **self-hosted runner กับ public repo ไปด้วยกันไม่ได้** ถ้าไม่ตั้งค่าป้องกันดีพอ — ใครก็ fork แล้วรันโค้ดบนเครื่องเราได้
4. **tag ที่ไม่ชี้ตายตัว (`latest`) ใช้ได้แค่ตอน dev** งานจริงต้อง trace กลับไปหา commit ได้เสมอ

---

## 🔗 ที่อื่นใน vault

- [[Greenhouse]] — โปรเจกต์ CI/CD อยู่ในนี้
- [[Docker]] — พื้นฐาน Docker ที่ต้องมีก่อนอ่านเรื่อง layer caching ใน CI
- [[Quarkus Build]] — fast-jar และ native build ที่ผูกกับเรื่อง cache ใน CI โดยตรง
