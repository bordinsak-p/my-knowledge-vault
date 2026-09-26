---
tags:
  - design-patterns
  - oop
  - design-patterns/index
type: moc
created: 2026-09-20
---

# 🧩 Design Patterns

หน้ารวมโน้ต design pattern / หลักการ OOP ทั่วไป — เน้นเรื่องที่มักถูกถามในสัมภาษณ์งาน เขียนแบบทั่วไป ไม่ผูกกับภาษา/framework ใดโดยเฉพาะ (มีตัวอย่างเป็น Java เพราะอ่านง่ายสุด แต่แนวคิดใช้ได้ทุกภาษา)

---

## โน้ตในนี้

| โน้ต | ว่าด้วย | สถานะ |
|---|---|---|
| [[Singleton Pattern]] | มี instance เดียว, thread-safety, ข้อเสีย | ✅ |
| [[Dependency Injection]] | inject dependency จากภายนอก, constructor/setter/field injection, DI container | ✅ |
| [[Strategy Pattern]] | สลับ algorithm/พฤติกรรมได้โดยไม่แก้โค้ดเดิม, ต่างจาก State ยังไง | ✅ |
| [[Observer Pattern]] | แจ้งเตือนแบบ one-to-many, รากฐานของ [[Observable]]/RxJS ทั้งระบบ | ✅ |
| [[Idempotency]] | ไม่ใช่ GoF pattern แต่ถูกถามคู่กันบ่อย — idempotency key, exactly-once เป็นไปไม่ได้จริง, claim pattern | ✅ |

**เริ่มอ่านคู่กัน** — คำถามสัมภาษณ์คลาสสิกที่สุดคือ "Singleton pattern กับ DI container's singleton scope ต่างกันยังไง" อยู่ใน [[Dependency Injection]] ข้อ 5 และ "Strategy vs State ต่างกันยังไง" อยู่ใน [[Strategy Pattern]] ข้อ 5

### ยังไม่ได้เขียน (แนะนำไว้ให้ตอนคุยกัน)

- **Tier 2:** Adapter, Template Method, Chain of Responsibility, Facade
- **Tier 3:** Composite, Command, State, Iterator, Mediator, Visitor
- **SOLID principles** — หลักการเบื้องหลังว่าทำไม pattern พวกนี้ถูกออกแบบมาแบบนี้

---

## 🔗 ที่อื่นใน vault

- [[Angular Services and DI]] — ตัวอย่าง DI จริงฝั่ง frontend
- [[Quarkus Project Structure]] — ตัวอย่าง DI จริงฝั่ง backend (CDI)

## 📖 อ้างอิงหลัก

- [Refactoring Guru — Design Patterns](https://refactoring.guru/design-patterns)
- [Martin Fowler — Inversion of Control Containers and the Dependency Injection pattern](https://martinfowler.com/articles/injection.html)
