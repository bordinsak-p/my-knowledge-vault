---
tags:
  - quarkus
  - java
  - cdi
type: reference
created: 2026-09-30
---

# 🫘 Quarkus Bean — CDI managed bean คืออะไร

> **คำว่า "bean" ในโลก Java มีอย่างน้อย 3 ความหมายที่ไม่เหมือนกันเลย** สับสนกันบ่อยมาก — เทียบกันไว้ในตารางด้านล่าง
>
> **ทั้งสามคำนี้ไม่ได้ผูกกันเลย** — class เดียวอาจเป็นได้ทั้งสามอย่างพร้อมกัน หรือเป็นแค่อย่างเดียวก็ได้ โน้ตนี้พูดถึงเฉพาะ **CDI bean** เพราะเป็นความหมายที่ใช้ตลอดในวอลต์นี้ (`@ApplicationScoped`, `@Inject`, ทุกโน้ต Quarkus ที่ผ่านมา)

| ความหมาย | คืออะไร | มาจากไหน |
|---|---|---|
| **JavaBeans** | มาตรฐานเก่าแก่: constructor ว่าง + getter/setter ตามชื่อ + `Serializable` — แค่ convention การเขียน class ไม่เกี่ยวกับ dependency injection เลย | Sun Microsystems, 1996 |
| **Spring bean** | object ที่ Spring IoC container จัดการ lifecycle ให้ | Spring Framework |
| **CDI bean** (โน้ตนี้พูดถึงตัวนี้) | object ที่ CDI container (ArC ใน Quarkus) จัดการ lifecycle/scope/injection ให้ | Jakarta CDI spec |

---

## 1. นิยามจริงจาก CDI spec

> **"A bean is a source of contextual objects which define application state and/or logic"** — object ที่ CDI container จัดการ lifecycle ให้ตามโมเดล scope ที่กำหนด

bean หนึ่งตัวมีคุณสมบัติเหล่านี้ติดตัวเสมอ: **bean type** (type ที่ inject ได้ — class/interface นั้นเอง), **scope** (อยู่นานแค่ไหน, ข้อ 4), **qualifier** (ตัวช่วยเลือกเวลามี bean type เดียวกันหลายตัว), และ **interceptor binding** ที่อาจแปะไว้ ([[Quarkus Interceptor]])

---

## 2. 2 วิธีที่ class จะกลายเป็น bean

### 2.1 Managed bean — แปะ scope annotation ลงบน class ตรงๆ

```java
@ApplicationScoped
public class ProductService {
    @Inject ProductRepository repository;
}
```

วิธีที่ใช้ 90% ของเวลา — class ที่เราเขียนเอง แปะ scope แล้วจบ

### 2.2 Producer — สร้าง bean จาก class ที่ไม่ได้เขียนเอง

```java
@ApplicationScoped
public class JacksonConfig {

    @Produces
    @ApplicationScoped
    public ObjectMapper objectMapper() {
        return new ObjectMapper().registerModule(new JavaTimeModule());
    }
}
```

ใช้เมื่อต้อง inject class จาก library ภายนอกที่แก้ source ไม่ได้ (แปะ `@ApplicationScoped` บน `ObjectMapper` เองไม่ได้) หรือ logic การสร้างซับซ้อนกว่าแค่ `new` เปล่าๆ

---

## 3. ทำไม `@ApplicationScoped` ถึงจำเป็น — กลไกจริงของ Quarkus

Quarkus ใช้ **"simplified bean discovery"** — ไม่ต้องมี `beans.xml` เลย (มีก็ถูกเมิน **ไม่อ่าน content**) กลไกจริงคือ:

**class ที่ไม่มี "bean-defining annotation" ถูกมองข้ามไปเลย ไม่ถูกจัดการโดย container** — ต่อให้ syntax ถูกทุกอย่าง แต่ไม่มี scope annotation = ไม่ใช่ bean = `@Inject` หาไม่เจอ (build fail หรือ error ตอน startup)

**bean-defining annotation คือ:** scope ใดๆ (`@ApplicationScoped`/`@RequestScoped`/`@Dependent`/`@Singleton`) หรือ stereotype

**ข้อยกเว้นเดียว:** producer method/field (ข้อ 2.2) ถูก discover เสมอแม้ class ที่ประกาศจะไม่มี scope annotation เลย — container มองว่า class นั้นเป็น `@Dependent` โดยปริยาย

---

## 4. Scope หลักที่ใช้บ่อย

| Scope                    | instance กี่ตัว           | อยู่นานแค่ไหน                         | ใช้เมื่อ                                                                      |
| ------------------------ | ------------------------- | ------------------------------------- | ----------------------------------------------------------------------------- |
| **`@ApplicationScoped`** | 1 ตัวทั้งแอป (แชร์ทุกที่) | ตลอดอายุแอป                           | ค่า default ที่ใช้บ่อยสุด — service/repository ที่ไม่เก็บ state เฉพาะ request |
| **`@RequestScoped`**     | 1 ตัวต่อ 1 HTTP request   | แค่ 1 request                         | ของที่ผูกกับ request เดียว เช่น current user, request-specific data           |
| **`@Dependent`**         | ใหม่ทุกครั้งที่ inject    | อายุเท่ากับ bean ที่ inject มันเข้าไป | ของเบาๆ ไม่มี state ที่ควรแชร์กัน (เป็นค่า default ถ้าไม่ระบุอะไรเลย)         |
| **`@Singleton`**         | 1 ตัวทั้งแอป              | ตลอดอายุแอป                           | คล้าย `@ApplicationScoped` แต่**ไม่ผ่าน proxy** (ข้อ 5)                       |

`@RequestScoped` ใช้ใน background thread/`@Scheduled` job ไม่ได้ — ไม่มี HTTP request ให้ผูก (รายละเอียดเต็มที่ [[Quarkus Thread Pool]])

---

## 5. `@ApplicationScoped` vs `@Singleton` — ต่างกันตรง proxy

`@ApplicationScoped` เป็น **normal scope** — CDI แทรก **client proxy** ให้เสมอ (object กลางที่ forward ทุก call ไปยัง instance จริงอีกที) และสร้าง instance จริงแบบ **lazy** (ยังไม่สร้างจนกว่าจะถูกเรียกใช้จริงครั้งแรก)

`@Singleton` (`jakarta.inject.Singleton`) เป็น **pseudo-scope** — **ไม่มี proxy เลย** สร้าง instance แบบ **eager** ทันทีที่ถูก inject

**proxy มีไว้ทำไม:** ทำให้ inject ข้าม scope ได้ปลอดภัย (เช่น inject `@ApplicationScoped` bean เข้าไปใน bean ที่ scope สั้นกว่า) และสลับ instance จริงข้างหลัง proxy ได้โดยจุด injection ไม่ต้องรู้ตัว — ถ้าไม่ต้องการเรื่องพวกนี้และอยากได้ overhead ต่ำสุด ใช้ `@Singleton` แทนได้

---

## 6. Bean ต่างจาก "object ธรรมดา" ตรงไหน

```java
ProductService s1 = new ProductService();   
// object ธรรมดา — CDI ไม่รู้จัก ไม่มีใคร inject dependency ให้
// s1.repository เป็น null แน่นอน ต่อให้ ProductService มี @ApplicationScoped ก็ตาม

@Inject ProductService s2;                  
// bean จริง — CDI สร้างให้ inject dependency ให้ครบ คุม scope/lifecycle ให้
```

**`new` เองข้าม CDI ไปเลย** — ได้ instance มาก็จริง แต่ field ที่ควรถูก `@Inject` จะเป็น `null` และ interceptor ที่แปะไว้ก็จะไม่ทำงาน ([[Quarkus Interceptor]] กับดัก) เพราะ instance นั้นไม่เคยผ่านมือ container เลย

---

## 🎯 แนวทางการเริ่มศึกษา

1. **เริ่มต้น** — จำไว้ว่า "bean" ในวอลต์นี้ = CDI bean เท่านั้น (ไม่ใช่ JavaBeans/Spring bean) แล้วเข้าใจกฎเหล็ก: **ไม่มี bean-defining annotation = ไม่ใช่ bean = inject ไม่ได้** (ข้อ 3) ก่อนแตะเรื่องอื่น
2. **ลงมือทำจริง** — ลองสร้าง class ธรรมดาไม่มี annotation แล้ว `@Inject` ดู เห็น error จริงด้วยตาตัวเอง แล้วค่อยเติม `@ApplicationScoped` กลับไปดูว่าหายเลย
3. **ใช้งานได้คล่อง** — เข้าใจ scope ทั้ง 4 แบบ (ข้อ 4) เลือกให้ตรงกับงาน โดยเฉพาะ `@RequestScoped` ที่ใช้ผิดที่บ่อยสุด และรู้ว่า producer (ข้อ 2.2) มีไว้แก้ปัญหาอะไร

---

## กับดัก

- **ลืมใส่ scope annotation แล้วงงว่าทำไม `@Inject` หา bean ไม่เจอ** — class ที่ไม่มี bean-defining annotation ถูก Quarkus มองข้ามตั้งแต่ตอน build (ข้อ 3)
- **ใส่ `beans.xml` เปล่าตามตัวอย่างเก่า** — Quarkus ไม่อ่าน content ของมันเลย ไม่จำเป็นต้องมี
- **สับสนคำว่า "bean" กับ JavaBeans/Spring bean** เวลาอ่านบทความที่ไม่ได้เจาะจง framework — เช็คบริบทก่อนเสมอ (ดูตารางเปิดโน้ต)
- **`new` object เองแทนที่จะ `@Inject`** แล้วงงว่าทำไม dependency ข้างในเป็น `null` หรือ interceptor ไม่ทำงาน (ข้อ 6)
- **ใช้ `@RequestScoped` bean ใน background thread/scheduled job** — ไม่มี request context ให้ผูก พังทันที (ข้อ 4)
- **คิดว่า `@Singleton` กับ `@ApplicationScoped` เหมือนกันเป๊ะ** — ต่างกันเรื่อง proxy/lazy-vs-eager (ข้อ 5) ผลต่างมีผลจริงตอน inject ข้าม scope หรือ serialize

---

## Cheat sheet

```java
@ApplicationScoped              // 1 ตัวทั้งแอป — ค่า default ที่ใช้บ่อยสุด
public class ProductService {
    @Inject ProductRepository repository;   // inject bean อื่นเข้ามา
}

@Produces
@ApplicationScoped
public ObjectMapper objectMapper() { ... }   // สร้าง bean จาก class ที่ไม่ได้เขียนเอง
```

| อาการ | สาเหตุ |
|---|---|
| `@Inject` แล้ว build fail/หา bean ไม่เจอ | ลืมใส่ scope annotation บน class เป้าหมาย |
| dependency เป็น `null` | `new` object เอง แทนที่จะให้ CDI `@Inject` ให้ |
| ใช้ `@RequestScoped` แล้ว error ใน background thread | ไม่มี HTTP request context ให้ผูก |
| งงว่า "bean" ที่บทความพูดถึงคืออะไร | เช็คว่าพูดถึง JavaBeans/Spring/CDI — คนละเรื่องกัน |

---

## 🔗 เกี่ยวข้อง

- [[Java OOP]] — interface/polymorphism พื้นฐานที่ CDI ใช้เลือก concrete bean ตอน inject, ตัวอย่าง `@All` inject bean หลายตัวพร้อมกันจริง (ข้อ 5)
- [[Dependency Injection]] — หลักการ DI ทั่วไปไม่ผูก framework, CDI เป็น implementation หนึ่ง
- [[Quarkus Interceptor]] — bean ต้องถูก CDI จัดการก่อน interceptor ถึงจะทำงาน (proxy, self-invocation)
- [[Quarkus Project Structure]] — ตัวอย่าง `@ApplicationScoped` จริงในโครงสร้างมาตรฐาน resource→service→repository
- [[Quarkus Thread Pool]] — ทำไม `@RequestScoped` ใช้ใน background thread ไม่ได้
- [[Quarkus]] — หน้ารวม

## 📖 อ่านต่อ

- [Quarkus — Introduction to Contexts and Dependency Injection (CDI)](https://quarkus.io/guides/cdi)
- [Quarkus — Contexts and Dependency Injection (CDI reference guide)](https://quarkus.io/guides/cdi-reference/)
- [Jakarta CDI 4.1 Specification](https://jakarta.ee/specifications/cdi/4.1/jakarta-cdi-spec-4.1.html)
- [JavaBeans Specification — Oracle](https://www.oracle.com/java/technologies/javase/javabeans-spec.html)
