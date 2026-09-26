---
tags:
  - design-patterns
  - oop
  - interview
type: reference
created: 2026-09-20
---

# 👁️ Observer Pattern

> **Observer คือ pattern ที่ให้ object หนึ่งตัว (subject) แจ้งเตือน object อื่นๆ ที่ "สมัครติดตาม" ไว้ (observer) โดยอัตโนมัติทุกครั้งที่ตัวเองมีการเปลี่ยนแปลง** โดย subject ไม่จำเป็นต้องรู้จัก observer แต่ละตัวในรายละเอียดเลย — **นี่คือรากฐานของ [[Observable]] ที่เขียนไว้แล้วทั้งหมดในวอลต์นี้** เรียนตัวนี้แล้วจะเห็นภาพ RxJS ชัดขึ้นอีกชั้น

---

## 1. โครงสร้างพื้นฐาน

```java
interface Observer {
    void update(String event);
}

interface Subject {
    void attach(Observer o);
    void detach(Observer o);
    void notifyObservers(String event);
}

class NewsPublisher implements Subject {
    private final List<Observer> observers = new ArrayList<>();

    public void attach(Observer o) { observers.add(o); }
    public void detach(Observer o) { observers.remove(o); }

    public void notifyObservers(String event) {
        for (Observer o : observers) {
            o.update(event);   // แจ้งทุกตัวที่ subscribe ไว้ ไม่สนใจว่าใครเป็นใคร
        }
    }

    public void publish(String news) {
        notifyObservers(news);
    }
}
```

**`NewsPublisher` ไม่รู้เลยว่า observer แต่ละตัวเป็นใคร/ทำอะไรกับข่าวที่ส่งไป** — อาจเป็นการส่งอีเมล, แสดงบนหน้าจอ, หรือบันทึก log ก็ได้ subject สนใจแค่ว่ามี `update()` ให้เรียกก็พอ (loose coupling แบบเดียวกับที่ [[Dependency Injection]] ต้องการ)

---

## 2. เชื่อมกับ [[Observable]] ที่เขียนไว้แล้ว — pattern เดียวกันเป๊ะ

| Observer pattern (ดั้งเดิม) | RxJS Observable |
|---|---|
| `Subject` | `Observable` |
| `Observer.update()` | `next()`/`error()`/`complete()` callback |
| `attach()` | `subscribe()` |
| `detach()` | `unsubscribe()` |
| `notifyObservers()` | Observable ปล่อยค่าใหม่ออกมา (`next(value)`) |

RxJS **ไม่ได้คิดค้น concept ใหม่** — เอา Observer pattern ที่มีมาตั้งแต่ GoF (1994) มาต่อยอดด้วย operator สำหรับแปลง/กรอง/รวม stream (ดู [[RxJS]]) แนวคิดพื้นฐานเรื่อง push-based/subscribe/unsubscribe ที่เขียนไว้ใน [[Observable]] ทั้งหมดสืบสาวไปถึง pattern นี้โดยตรง

---

## 3. ตัวอย่างที่ใช้ในภาษา/framework จริง

- **Java `PropertyChangeListener`** — GUI framework เก่าใช้ pattern นี้ตรงๆ
- **DOM `addEventListener`** — `element.addEventListener('click', handler)` คือ `attach()` แบบหนึ่ง
- **`java.util.Observable`/`Observer`** — มากับ JDK เองแต่ **deprecated ตั้งแต่ Java 9** (design เก่าไม่ยืดหยุ่นพอ เช่นบังคับต้อง extend คลาส ไม่ใช่ implement interface) ปัจจุบันแนะนำให้ใช้ `PropertyChangeSupport` หรือ Reactive Streams (RxJS/Reactor) แทน

---

## กับดัก

- **ลืม `detach()`/unsubscribe observer ที่ไม่ใช้แล้ว** — subject ยังถือ reference ค้างไว้ กัน observer ไม่ให้ถูก garbage collect ได้ (memory leak) — เหตุผลเดียวกับที่ [[takeUntil]] ต้องมีอยู่ในโลกของ RxJS
- **observer ตัวหนึ่ง throw exception ตอน `update()` แล้วขัดขวางไม่ให้ observer ตัวอื่นได้รับแจ้งเตือน** — ถ้า loop เรียก `update()` แบบไม่มี try-catch ต่อตัว exception ของ observer ตัวแรกจะทำให้ตัวที่เหลือไม่ได้รับ event เลย ควร isolate exception ต่อ observer
- **notify observer ระหว่างที่กำลัง iterate list ของ observer เอง** (เช่น observer หนึ่ง `detach()` ตัวเองตอน `update()`) — เสี่ยง `ConcurrentModificationException` ถ้าไม่ copy list ก่อน iterate หรือใช้ collection ที่ thread-safe/iteration-safe
- **ลำดับการแจ้งเตือนสำคัญแต่ไม่ได้การันตีลำดับ** — ถ้า business logic ต้องพึ่งว่า observer ไหนทำงานก่อนหลัง ต้องออกแบบ priority เอง pattern พื้นฐานไม่รับประกันลำดับให้

---

## Cheat sheet

```java
interface Observer { void update(String event); }
interface Subject {
    void attach(Observer o);
    void detach(Observer o);
    void notifyObservers(String event);
}
```

| เจตนา | เทียบกับ |
|---|---|
| แจ้งเตือนแบบ one-to-many, ไม่รู้จัก observer ในรายละเอียด | Observer pattern (คลาสสิก) |
| แบบเดียวกัน + มี operator แปลง/กรอง/รวม stream ในตัว | [[Observable]] / RxJS |

## 🔗 เกี่ยวข้อง

- [[Observable]] — implementation จริงของ pattern นี้ที่ใช้ในโปรเจกต์ (push/pull, cold/hot)
- [[RxJS]] — operator ที่ต่อยอดจาก pattern นี้ไปอีกขั้น
- [[takeUntil]] — วิธีป้องกัน memory leak จากการลืม unsubscribe ในโลกของ RxJS

## 📖 อ่านต่อ

- [Refactoring Guru — Observer](https://refactoring.guru/design-patterns/observer)
