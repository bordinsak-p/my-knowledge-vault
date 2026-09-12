---
tags:
  - docker
  - devops
  - beginner
type: tutorial
created: 2026-08-18
---

# 🐳 Docker ภาค 1 — พื้นฐานที่ต้องเข้าใจก่อน

> เขียนสำหรับคนที่**ไม่เคยใช้ Docker มาก่อนเลย** ทุกคำสั่งในนี้ลองพิมพ์ตามได้จริง
> เป้าหมายของภาคนี้: เข้าใจว่า image/container คืออะไร และคล่องกับคำสั่งพื้นฐานพอจะไปเขียน Dockerfile เองในภาค 2

---

## 1. ก่อนอื่น — ปัญหาที่ Docker แก้

ลองนึกภาพนี้: เขียนโปรแกรมเสร็จบนเครื่องตัวเอง รันได้ปกติ ส่งให้เพื่อนหรือเอาขึ้น server แล้ว **พังไม่รู้สาเหตุ**

```
เครื่องเรา                          เครื่อง server
─────────────                       ─────────────
Windows 11                          Linux Ubuntu
Java 17                             Java 11        ← เวอร์ชันไม่ตรง!
มี library X ติดตั้งไว้แล้ว           ไม่มี library X   ← ขาดของ!
```

**Docker แก้ปัญหานี้โดยห่อทั้งโปรแกรมและทุกอย่างที่มันต้องการไว้ในกล่องเดียว** แล้วเอากล่องนั้นไปรันที่ไหนก็ได้ ได้ผลลัพธ์เดียวกันเป๊ะ — ไม่ว่าเครื่องปลายทางจะเป็น Windows, Mac, Linux, หรือ cloud server

### ต่างจากการรันตรง ๆ ยังไง

```
┌─────────────────────────────┐     ┌─────────────────────────────┐
│   รันแบบธรรมดา (ไม่มี Docker)   │     │        รันด้วย Docker         │
├─────────────────────────────┤     ├─────────────────────────────┤
│  โปรแกรม A ─┐                │     │  ┌──────────┐  ┌──────────┐ │
│  โปรแกรม B ─┼─ ใช้ OS ร่วมกัน    │     │  │โปรแกรม A │  │โปรแกรม B  │ │
│             │  แชร์ library   │     │  │+ ของที่ A  │  │+ ของที่ B │ │
│             │  ชนกันได้        │     │  │ ต้องการ   │  │ ต้องการ   │ │
│             ┘                │     │  └──────────┘  └──────────┘│
│                              │     │       แยกกันเด็ดขาด          │
└─────────────────────────────┘     └─────────────────────────────┘
```

โปรแกรม A กับ B อาจต้องการ Node.js คนละเวอร์ชัน ถ้าลงตรง ๆ บนเครื่องเดียวจะชนกัน Docker แยกแต่ละโปรแกรมออกจากกันโดยสมบูรณ์ แม้จะรันอยู่บนเครื่องเดียวกัน

---

## 2. สองคำที่ต้องแยกให้ออกก่อนอย่างอื่น — Image กับ Container

นี่คือจุดที่มือใหม่สับสนที่สุด เปรียบเทียบง่าย ๆ:

```
Image                              Container
────────────────────               ────────────────────
= พิมพ์เขียว / สูตรอาหาร              = ตัวจริงที่ทำงานอยู่
= คลาส (class) ในการเขียนโค้ด        = instance ของคลาสนั้น
= ไฟล์ .exe ที่ยังไม่ได้เปิด          = โปรแกรมที่กำลังรันอยู่บนจอ

image เดียว ────► สร้าง container ได้หลายตัวพร้อมกัน
                  ┌─────────────┐
   nginx:latest ──┼─► container 1 (กำลังรันเว็บ A)
                  ├─► container 2 (กำลังรันเว็บ B)
                  └─► container 3 (กำลังรันเว็บ C)
```

**Image** = ไฟล์นิ่ง ๆ เก็บไว้บนเครื่อง ไม่ได้ "รัน" อะไร
**Container** = image ที่ถูกสั่งให้เริ่มทำงาน กลายเป็น process จริงที่กินทรัพยากรเครื่อง

พูดสั้นที่สุด: **`docker run` คือคำสั่งที่เอา image มาสร้างเป็น container**

---

## 3. ติดตั้ง — Docker Desktop บน Windows

### ขั้นตอน

```powershell
# 1. เปิด PowerShell แบบ Admin แล้วรัน (เปิดใช้งาน WSL2 ซึ่ง Docker ต้องใช้)
wsl --install
```

รีสตาร์ตเครื่องหลังจากนั้น

```
2. ดาวน์โหลด Docker Desktop จาก https://docs.docker.com/desktop/setup/install/windows-install/
3. ติดตั้งตามปกติ (ตัวเลือก WSL2 backend เป็นค่าเริ่มต้นอยู่แล้ว ไม่ต้องแก้อะไร)
4. เปิดเครื่องใหม่ Docker Desktop จะรันขึ้นมาอัตโนมัติ
```

### เช็คว่าติดตั้งสำเร็จ

```powershell
docker --version
docker run hello-world
```

**ผลลัพธ์ที่ควรเห็น:**

```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

ถ้าเห็นข้อความนี้ = ติดตั้งสำเร็จ พร้อมไปต่อ

> **สิ่งที่เพิ่งเกิดขึ้นเบื้องหลัง:** คำสั่งนี้ไปดาวน์โหลด image ชื่อ `hello-world` จากอินเทอร์เน็ต (จากที่เก็บกลางชื่อ **Docker Hub**) มาเก็บไว้ที่เครื่อง แล้วสร้าง container จาก image นั้นขึ้นมารันหนึ่งครั้ง พอรันเสร็จ container ก็หยุดทำงาน — นี่คือวงจรพื้นฐานที่สุดของ Docker

---

## 4. รัน container จริงตัวแรก — เว็บเซิร์ฟเวอร์

```powershell
docker run -d -p 8080:80 --name my-web nginx
```

เปิดเบราว์เซอร์ไปที่ `http://localhost:8080` — **ควรเห็นหน้าเว็บต้อนรับของ nginx**

### แปลความหมายทีละส่วนของคำสั่ง

```
docker run  -d          -p 8080:80         --name my-web    nginx
   │        │             │                     │              │
   │        │             │                     │              └── ใช้ image ชื่อ nginx
   │        │             │                     └── ตั้งชื่อ container ว่า my-web
   │        │             └── เชื่อมพอร์ต 8080 ของเครื่องเรา ──► พอร์ต 80 ข้างใน container
   │        └── รันแบบพื้นหลัง (detached) ไม่ค้าง terminal
   └── สั่งสร้างและรัน container
```

**`-p 8080:80` คือส่วนที่มือใหม่งงบ่อยที่สุด** — ตัวเลขซ้าย (`8080`) คือพอร์ตบนเครื่องเรา (โฮสต์) ตัวเลขขวา (`80`) คือพอร์ตข้างใน container เปรียบเหมือนสร้างท่อเชื่อมจากข้างนอกเข้าไปข้างใน — เข้าเว็บที่ `localhost:8080` แต่ nginx ข้างใน container ยังคิดว่าตัวเองรันที่พอร์ต 80 ตามปกติ

```
เบราว์เซอร์ ──► localhost:8080 ══(ท่อเชื่อม)══► container:80 ──► nginx
```

### ดูว่ามี container อะไรรันอยู่บ้าง

```powershell
docker ps
```

```
CONTAINER ID   IMAGE   COMMAND                  STATUS         PORTS                  NAMES
a1b2c3d4e5f6   nginx   "/docker-entrypoint.…"   Up 2 minutes   0.0.0.0:8080->80/tcp   my-web
```

### หยุดและลบ

```powershell
docker stop my-web      # หยุด container (ยังไม่ลบ)
docker start my-web     # เริ่มใหม่ตัวเดิม
docker rm my-web        # ลบ container ทิ้ง (ต้อง stop ก่อน)
```

**ลอง `docker ps` อีกครั้งหลัง `stop`** — จะไม่เห็น `my-web` แล้ว เพราะ `docker ps` เปล่า ๆ โชว์แค่ที่ **กำลังรัน** อยู่

```powershell
docker ps -a             # ← ใส่ -a เห็นทั้งที่หยุดและกำลังรัน
```

---

## 5. เข้าไปดูข้างใน container — เหมือนเปิด terminal เข้าไปในกล่อง

```powershell
docker exec -it my-web bash
```

ถ้า `my-web` ถูกลบไปแล้วจากข้อ 4 ให้รันใหม่ก่อน:

```powershell
docker run -d -p 8080:80 --name my-web nginx
docker exec -it my-web bash
```

**ตอนนี้ prompt จะเปลี่ยนเป็นเหมือนอยู่ข้างในเครื่อง Linux อีกเครื่องหนึ่ง** ลองสั่ง:

```bash
ls /usr/share/nginx/html      # ดูไฟล์เว็บที่ nginx เสิร์ฟอยู่
cat /etc/os-release             # ดูว่าข้างในเป็น Linux อะไร (Debian ตาม image nginx)
exit                             # ออกจาก container กลับมาที่เครื่องเรา
```

**ข้อสังเกตสำคัญ:** container ที่รันบน **Windows** ข้างในเป็น **Linux** — Docker บน Windows ใช้ WSL2 มารันทุก container เป็น Linux ทั้งหมด (เว้นแต่จะตั้งใจสลับไปโหมด Windows container ซึ่งใช้น้อยมาก) นี่คือเหตุผลที่คำสั่งข้างในเป็น `bash`, `ls`, `cat` แบบ Linux ไม่ใช่ PowerShell

---

## 6. Docker Hub — ที่เก็บ image กลาง

```
Docker Hub ≈ GitHub แต่เก็บ "image" แทน "โค้ด"
```

ทุกครั้งที่สั่ง `docker run <ชื่อ>` โดยที่เครื่องยังไม่มี image นั้น Docker จะไปดึงจาก Docker Hub มาให้อัตโนมัติ

```powershell
docker pull python:3.12        # ดึง image มาเก็บไว้เฉย ๆ ยังไม่รัน
docker images                  # ดู image ทั้งหมดที่มีอยู่บนเครื่อง
```

```
REPOSITORY   TAG      IMAGE ID       SIZE
nginx        latest   b8c1a3e...     190MB
python       3.12     f2a9d1c...     1.02GB
hello-world  latest   d2c94e...      13.3kB
```

**ตัวเลข tag (เช่น `3.12`, `latest`) คือเวอร์ชัน** — `python:3.12` กับ `python:3.13` เป็นคนละ image กัน ถ้าไม่ระบุ tag เลย Docker จะใช้ `latest` โดยอัตโนมัติ (แต่ **ไม่แนะนำใช้ `latest` ในงานจริง** เพราะไม่รู้ว่าจะได้เวอร์ชันไหนเป๊ะ ๆ ระบุเวอร์ชันชัดเจนดีกว่าเสมอ)

---

## 7. Volume เบื้องต้น — ทำไมข้อมูลใน container หายง่าย

```powershell
docker run -d --name test-db -e POSTGRES_PASSWORD=pass postgres:16
# ... ใส่ข้อมูลอะไรบางอย่างเข้าไป ...
docker rm -f test-db
# ข้อมูลหายหมด! container ถูกลบ = ทุกอย่างข้างในหายไปด้วย
```

**Container คือของชั่วคราวโดยธรรมชาติ** — ลบ container = ลบทุกอย่างที่เขียนไว้ข้างในด้วย ถ้าอยากเก็บข้อมูลข้ามการลบ/สร้างใหม่ ต้องใช้ **volume** ผูกพื้นที่บนเครื่องจริงเข้ากับข้างใน container

```powershell
docker run -d --name my-db -v pgdata:/var/lib/postgresql/data -e POSTGRES_PASSWORD=pass postgres:16
```

```
-v pgdata:/var/lib/postgresql/data
     │              │
     │              └── path ข้างใน container ที่ postgres เก็บข้อมูลจริง
     └── ชื่อ volume บนเครื่องเรา (Docker จัดการที่เก็บให้เอง)
```

ตอนนี้ต่อให้ลบ container `my-db` ทิ้งแล้วสร้างใหม่โดยผูก volume ชื่อเดิม (`pgdata`) ข้อมูลจะยังอยู่ครบ — รายละเอียดเรื่อง volume แบบเต็มอยู่ในภาค 3

---

## 8. คำสั่งที่ใช้บ่อยที่สุด — สรุปไว้ก่อนไปภาค 2

```powershell
# วงจรชีวิตพื้นฐาน
docker run <image>              # สร้าง + รัน container ใหม่
docker start <ชื่อ/id>          # เริ่ม container ที่หยุดอยู่
docker stop <ชื่อ/id>           # หยุด (แต่ยังไม่ลบ)
docker restart <ชื่อ/id>        # รีสตาร์ต

# ดูสถานะ
docker ps                       # container ที่กำลังรัน
docker ps -a                    # ทั้งหมด รวมที่หยุดแล้ว
docker images                   # image ที่มีอยู่บนเครื่อง
docker logs <ชื่อ/id>           # ดู log ของ container
docker logs -f <ชื่อ/id>        # ดู log แบบ real-time (follow)

# เข้าไปข้างใน
docker exec -it <ชื่อ/id> bash  # เปิด terminal เข้าไปข้างใน

# ล้างทำความสะอาด
docker rm <ชื่อ/id>             # ลบ container (ต้อง stop ก่อน หรือใส่ -f บังคับ)
docker rmi <image>              # ลบ image
docker system prune             # ล้างของที่ไม่ได้ใช้ทั้งหมดทีเดียว (ระวัง — ดูข้อ 9)
```

### flag ที่เจอบ่อย

| flag | ความหมาย |
|---|---|
| `-d` | รันแบบพื้นหลัง (detached) |
| `-p host:container` | เชื่อมพอร์ต |
| `-v name:/path` | ผูก volume |
| `-e KEY=value` | ตั้ง environment variable |
| `--name` | ตั้งชื่อ container |
| `-it` | โหมด interactive (ใช้กับ `exec`/`run bash`) |
| `--rm` | ลบ container อัตโนมัติทันทีที่หยุด (สะดวกตอนทดลองสั้น ๆ) |

---

## 9. กับดักของมือใหม่

- **สับสน image กับ container** — สั่ง `docker run` ซ้ำ ๆ ด้วยชื่อเดิมแล้วงงว่าทำไม error ว่าชื่อซ้ำ (เพราะสร้าง container ใหม่ทุกครั้งที่ `run` ทั้งที่ตัวเก่ายังอยู่ ควรใช้ `docker start` กับตัวเดิมแทน)
- **ลืมว่า container หายข้อมูลเมื่อถูกลบ** — เข้าใจผิดว่า Docker "เซฟงานให้อัตโนมัติ" ทั้งที่ต้องผูก volume เองถึงจะข้อมูลอยู่ถาวร
- **`-p` สลับด้าน** — เขียน `-p 80:8080` ทั้งที่ตั้งใจ `-p 8080:80` แล้วงงว่าทำไมเข้า `localhost:8080` ไม่ได้ (จำว่า **ซ้าย = เครื่องเรา, ขวา = ข้างใน container** เสมอ)
- **`docker system prune` ทำไปโดยไม่รู้ผล** — ลบ image/container/network ที่ไม่ได้ใช้งานอยู่**ทั้งหมด** รวมของที่อาจยังอยากเก็บไว้ ก่อนรันควรรู้ตัวว่ากำลังลบอะไร (`docker system prune -a` ยิ่งลบเยอะกว่าเดิมอีก รวม image ที่ยัง cache ไว้ด้วย)
- **พอร์ตชนกัน** — รัน container สองตัวแล้วตั้ง `-p 8080:...` เหมือนกันทั้งคู่ ตัวที่สองจะ error ว่าพอร์ตถูกใช้แล้ว ต้องเปลี่ยนตัวเลขฝั่งซ้าย (host) ให้ไม่ซ้ำ
- **ลืมว่า container ข้างในเป็น Linux** — พยายามสั่งคำสั่ง PowerShell (`dir`, `Get-Content`) ข้างใน container ที่รันบน Linux แล้วไม่มีคำสั่งนั้น

---

## 10. เช็คตัวเองก่อนไปภาค 2

ถ้าตอบคำถามพวกนี้ได้ พร้อมไปเขียน Dockerfile เองแล้ว

- [ ] อธิบายความต่างระหว่าง image กับ container ได้โดยไม่ต้องเปิดโน้ตดู
- [ ] รัน, หยุด, ลบ container เองได้โดยไม่ต้องก๊อปคำสั่ง
- [ ] อธิบายได้ว่า `-p 8080:80` ตัวเลขไหนคือเครื่องเรา ตัวเลขไหนคือข้างใน container
- [ ] เข้าไปดูข้างใน container ที่กำลังรันอยู่ได้ด้วย `docker exec`
- [ ] อธิบายได้ว่าทำไมข้อมูลใน container หายตอนลบ container

---

## 🔗 เกี่ยวข้อง

- [[Docker]] — หน้ารวม
- [[Docker Dockerfile]] — ภาค 2: เขียน image ของตัวเอง

## 📖 อ่านต่อ

- [Docker Docs — What is a container?](https://www.docker.com/resources/what-container/)
- [Docker Docs — Get Started](https://docs.docker.com/get-started/)
