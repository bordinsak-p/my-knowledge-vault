#angular-outline/cli 
# Angular CLI Cheat Sheet 🛠️

Angular CLI เป็นเครื่องมือ command line สำหรับช่วยจัดการโปรเจกต์ Angular อย่างมีประสิทธิภาพ ใช้คำสั่งง่ายๆ ในการสร้าง คอมไพล์ และจัดการโค้ดต่างๆ
###### แหล่งอ้างอิง [Angular - CLI Overview and Command Reference](https://v17.angular.io/cli)

---

## 🔰 คำสั่งทั่วไป

#cli/normal

| คำสั่ง                  | คำอธิบาย                                                  |
| ----------------------- | --------------------------------------------------------- |
| `ng new [project-name]` | สร้างโปรเจกต์ Angular ใหม่                                |
| `ng serve`              | รัน dev server ที่ `http://localhost:4200`                |
| `ng build`              | คอมไพล์โปรเจกต์ (ใช้ `--prod` เพื่อ build แบบ production) |
| `ng test`               | รัน unit tests                                            |
| `ng lint`               | ตรวจ style ของโค้ด                                        |
| `ng e2e`                | รัน end-to-end test (deprecated ในบางเวอร์ชัน)            |

---

## 🧱 คำสั่งสำหรับสร้างโค้ด (Generate)

#cli/generate

| คำสั่ง | คำอธิบาย |
|--------|----------|
| `ng generate component [name]`<br>`ng g c [name]` | สร้าง component |
| `ng generate module [name]`<br>`ng g m [name]` | สร้าง module |
| `ng generate service [name]`<br>`ng g s [name]` | สร้าง service |
| `ng generate directive [name]` | สร้าง directive |
| `ng generate pipe [name]` | สร้าง pipe |
| `ng generate guard [name]` | สร้าง route guard |
| `ng generate interface [name]` | สร้าง TypeScript interface |

---

## ⚙️ คำสั่ง Configuration

#cli/config

| คำสั่ง | คำอธิบาย |
|--------|----------|
| `ng config [key] [value]` | เปลี่ยนค่าการตั้งค่าใน `angular.json` |
| `ng analytics` | เปิด/ปิดการส่งข้อมูลการใช้งานไปยัง Angular team |

---

## 📦 การจัดการ Dependency

#cli/dependency

| คำสั่ง | คำอธิบาย |
|--------|----------|
| `ng add [package-name]` | ติดตั้งและ config package ที่รองรับ CLI |
| `ng update` | อัปเดต Angular และ package อื่นๆ |
| `ng doc [keyword]` | เปิดคู่มือจาก Angular documentation |

---

## 💡 ตัวอย่างการใช้งานที่พบบ่อย

#cli/example

```bash
ng new my-app
ng serve
ng build --prod
ng g c components/user-profile
ng g s services/user
ng g m pages/dashboard --route dashboard --module app.module
```

---