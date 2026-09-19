---
tags:
  - docker
  - devops
  - cli
type: reference
created: 2026-09-16
---

# 🐳 Docker Command Reference — คำสั่งที่ใช้บ่อยแยกตามหมวด

> โน้ตนี้คือ **cheat sheet รวมคำสั่ง `docker`** ไม่อธิบายพื้นฐาน image/container ซ้ำ (อ่านที่ [[Docker Basics]] ถ้ายังไม่เข้าใจ) เน้นแค่ "มีคำสั่งอะไรบ้าง ใช้ทำอะไร" ให้เปิดมาเจอเร็ว ๆ

---

## 1. Container lifecycle

| คำสั่ง | ทำอะไร |
|---|---|
| `docker run <image>` | สร้าง + รัน container ใหม่จาก image |
| `docker start <ชื่อ/id>` | เริ่ม container ที่หยุดอยู่ (ใช้ container เดิม ไม่สร้างใหม่) |
| `docker stop <ชื่อ/id>` | หยุดแบบสุภาพ (ส่ง SIGTERM รอ grace period แล้วค่อย kill) |
| `docker kill <ชื่อ/id>` | หยุดทันที (ส่ง SIGKILL เลย ไม่รอ) |
| `docker restart <ชื่อ/id>` | หยุดแล้วเริ่มใหม่ |
| `docker pause` / `docker unpause` | หยุด process ข้างใน container ชั่วคราว (freeze) โดยไม่หยุด container |
| `docker rm <ชื่อ/id>` | ลบ container (ต้อง stop ก่อน หรือใส่ `-f`) |
| `docker rename <เก่า> <ใหม่>` | เปลี่ยนชื่อ container |

**`stop` ≠ `kill`:** `stop` ให้เวลาโปรแกรมข้างในปิดตัวเองอย่างถูกต้องก่อน (เช่น ปิด database connection, flush log) ส่วน `kill` ตัดจบทันทีไม่สนใจอะไรทั้งนั้น — ใช้ `kill` เฉพาะตอน `stop` ค้างไม่ยอมหยุดจริง ๆ

---

## 2. ดูสถานะ / debug

| คำสั่ง | ทำอะไร |
|---|---|
| `docker ps` | container ที่กำลังรัน |
| `docker ps -a` | ทั้งหมด รวมที่หยุดแล้ว |
| `docker logs <ชื่อ/id>` | ดู log |
| `docker logs -f <ชื่อ/id>` | ดู log แบบ real-time (follow) |
| `docker logs --tail 100 <ชื่อ/id>` | ดู log 100 บรรทัดล่าสุด |
| `docker exec -it <ชื่อ/id> bash` | เปิด terminal เข้าไปข้างใน |
| `docker attach <ชื่อ/id>` | ต่อเข้ากับ process หลักของ container ตรง ๆ (ต่างจาก `exec` ที่เปิด process ใหม่แยกกัน) |
| `docker inspect <ชื่อ/id>` | ดู config/สถานะทั้งหมดแบบ JSON เต็ม |
| `docker inspect -f '{{.NetworkSettings.IPAddress}}' <ชื่อ>` | ดึงค่าเฉพาะจุดด้วย Go template |
| `docker stats` | ดู CPU/memory/network แบบ real-time ของทุก container |
| `docker top <ชื่อ/id>` | ดู process ที่รันอยู่ข้างใน container |
| `docker port <ชื่อ/id>` | ดูว่า container นี้ mapping พอร์ตอะไรไว้บ้าง |
| `docker diff <ชื่อ/id>` | ดูว่าไฟล์ไหนถูกแก้/เพิ่ม/ลบ เทียบกับ image ต้นทาง |
| `docker cp <ชื่อ>:/path ./local` | copy ไฟล์ออกจาก container มาเครื่องเรา |
| `docker cp ./local <ชื่อ>:/path` | copy ไฟล์จากเครื่องเราเข้า container |

**`exec` vs `attach`:** `exec` เปิด process ใหม่แยกต่างหาก (ปิดไปก็ไม่กระทบ process หลัก) — `attach` ไปเกาะกับ process หลัก (PID 1) ของ container ตรง ๆ กด `Ctrl+C` อาจทำให้ container หยุดไปด้วย ส่วนใหญ่ใช้ `exec` ปลอดภัยกว่าเสมอ

---

## 3. Image management

| คำสั่ง | ทำอะไร |
|---|---|
| `docker build -t <name> .` | build image จาก Dockerfile (ดู [[Docker Dockerfile]]) |
| `docker images` | ดู image ทั้งหมดบนเครื่อง |
| `docker pull <image>` | ดึง image จาก registry มาเก็บไว้เฉย ๆ |
| `docker push <user>/<name>:<tag>` | ส่ง image ขึ้น registry (ดู [[Docker Compose and Deploy]]) |
| `docker tag <local> <user>/<name>:<tag>` | ตั้งชื่อ/เวอร์ชันใหม่ให้ image เดิม (ไม่ copy ข้อมูลซ้ำ) |
| `docker rmi <image>` | ลบ image |
| `docker history <image>` | ดู layer ทั้งหมดที่ประกอบเป็น image นี้ |
| `docker save -o out.tar <image>` | เซฟ image เป็นไฟล์ .tar (มี layer/history ครบ) |
| `docker load -i out.tar` | โหลด image กลับจากไฟล์ .tar |
| `docker export <container> -o out.tar` | เซฟ **filesystem ของ container** ที่รันอยู่เป็น .tar (แบน ไม่มี layer/history) |
| `docker import out.tar <name>` | สร้าง image ใหม่จากไฟล์ .tar ของ `export` |
| `docker commit <container> <name>` | สร้าง image จากสถานะปัจจุบันของ container (เร็วแต่ไม่แนะนำงานจริง — ดูกับดัก) |

**`save`/`load` ≠ `export`/`import`:** `save` เก็บ **image** พร้อม layer/history/metadata ครบ ใช้ย้าย image ข้ามเครื่องโดยไม่ผ่าน registry — `export` เก็บแค่ **filesystem ของ container ณ ขณะนั้น** แบนเป็นชั้นเดียว ไม่มี history/metadata ใด ๆ เอาไปทำอะไรต่อ (เช่น multi-stage build) ได้จำกัดกว่ามาก

---

## 4. Volume — เก็บข้อมูลข้ามการลบ container

> พื้นฐาน `-v` ตอน `run` ดูที่ [[Docker Basics]] ข้อ 7 — นี่คือคำสั่งจัดการ volume แยกเป็นของตัวเอง

| คำสั่ง | ทำอะไร |
|---|---|
| `docker volume create <name>` | สร้าง volume เปล่า ๆ ไว้ก่อน |
| `docker volume ls` | ดู volume ทั้งหมดบนเครื่อง |
| `docker volume inspect <name>` | ดู path จริงบนเครื่องที่ volume นี้เก็บข้อมูลอยู่ |
| `docker volume rm <name>` | ลบ volume (ลบไม่ได้ถ้ายังมี container ใช้อยู่) |
| `docker volume prune` | ลบ volume ที่ไม่มี container ไหนใช้อยู่เลยทั้งหมด |

---

## 5. Network — ให้ container คุยกัน

| คำสั่ง                                         | ทำอะไร                                           |
| ---------------------------------------------- | ------------------------------------------------ |
| `docker network create <name>`                 | สร้าง network ใหม่ (default driver: bridge)      |
| `docker network ls`                            | ดู network ทั้งหมด                               |
| `docker network inspect <name>`                | ดูว่า container ไหนต่ออยู่ใน network นี้บ้าง     |
| `docker run --network <name> ...`              | รัน container เข้า network ที่สร้างไว้ตั้งแต่ต้น |
| `docker network connect <name> <container>`    | เอา container ที่รันอยู่แล้วเข้า network เพิ่ม   |
| `docker network disconnect <name> <container>` | ถอด container ออกจาก network                     |
| `docker network rm <name>`                     | ลบ network                                       |

**กับดักสำคัญ: container คุยกันด้วยชื่อได้เฉพาะใน network ที่สร้างเอง** — default bridge network (ที่ `docker run` ธรรมดาเข้าไปอยู่โดยไม่ระบุอะไร) **ไม่รองรับ** ให้ container คุยกันด้วยชื่อ (DNS) ต้อง `docker network create` แล้วเอา container เข้า network นั้นก่อน ถึงจะสั่ง เช่น `ping my-db` จากอีก container แล้ว resolve ชื่อได้ (Docker รัน embedded DNS server ให้อัตโนมัติเฉพาะบน user-defined network เท่านั้น) — `docker compose` (ดู [[Docker Compose and Deploy]]) สร้าง network แบบนี้ให้อัตโนมัติอยู่แล้ว เป็นเหตุผลหนึ่งที่ service ใน compose คุยกันด้วยชื่อ service ได้เลยโดยไม่ต้องตั้งอะไรเพิ่ม

---

## 6. ล้างทำความสะอาด — เจาะจงดีกว่าเหวี่ยงแห

| คำสั่ง | ลบอะไร |
|---|---|
| `docker container prune` | container ที่หยุดอยู่ทั้งหมด |
| `docker image prune` | image ที่ "dangling" (ไม่มี tag, ไม่มีใครอ้างอิง) เท่านั้น |
| `docker image prune -a` | image ที่ไม่มี container ไหนใช้อยู่**ทั้งหมด** (กว้างกว่าด้านบนมาก) |
| `docker volume prune` | volume ที่ไม่มี container ไหนใช้อยู่ |
| `docker network prune` | network ที่ไม่มี container ไหนต่ออยู่ |
| `docker builder prune` | build cache |
| `docker system prune` | container ที่หยุด + image dangling + network ที่ไม่ได้ใช้ + build cache รวดเดียว |
| `docker system prune -a --volumes` | เหวี่ยงแหสุด — รวม image ที่ไม่ได้ใช้ทั้งหมด + volume ที่ไม่ได้ใช้ด้วย |

**แนะนำ:** ใช้คำสั่งเจาะจง (`container prune`, `image prune` เฉย ๆ) มากกว่า `system prune -a --volumes` เสมอ ยิ่งกว้างยิ่งเสี่ยงลบของที่ยังอยากเก็บไว้โดยไม่ตั้งใจ โดยเฉพาะ `--volumes` ที่ลบข้อมูลจริงถาวร กู้คืนไม่ได้

---

## 7. กับดัก

- **`docker rm` กับ `docker rmi` สลับกัน** — `rm` ลบ container, `rmi` ลบ image (มี "i" = image) พิมพ์ผิดตัวเดียวลบผิดของ
- **`export`/`import` ไม่เก็บ history/metadata** — เอา image ที่ได้จาก `import` ไป build ต่อแบบ multi-stage อ้างอิง layer เดิมไม่ได้ ถ้าต้องการย้าย image แบบครบ ๆ ใช้ `save`/`load` แทน
- **`docker commit` ทำ image ที่ build ซ้ำไม่ได้** — ไม่มี Dockerfile บันทึกไว้ว่า image นี้มาจากไหน ใช้ได้แค่ debug ชั่วคราว งานจริงควรเขียน Dockerfile เสมอ (ดู [[Docker Dockerfile]])
- **คาดหวังว่า container คุยกันด้วยชื่อได้ทั้งที่ยังอยู่ default bridge network** — ต้องสร้าง user-defined network ก่อน (ข้อ 5)
- **`docker image prune` เฉย ๆ ไม่ได้ลบ image ที่ไม่ได้ใช้ทั้งหมด** — ลบแค่ dangling image ถ้าอยากลบที่ไม่ได้ใช้จริงทั้งหมดต้องใส่ `-a`
- **`stop` ค้างนานเกินไปแล้วไม่รู้ว่าทำไม** — โปรแกรมข้างในไม่ handle SIGTERM เลยรอจน timeout (default 10 วินาที) ถึงจะถูก kill บังคับ ปกติ ถ้าอยากเร็วกว่านั้นใช้ `kill` แทนตรง ๆ (แต่เสี่ยงข้อมูลไม่ flush)

---

## 8. Cheat sheet รวม

```bash
# lifecycle
docker run <image> / start / stop / kill / restart <ชื่อ>
docker rm <ชื่อ>                       # ลบ container
docker rename <เก่า> <ใหม่>

# ดูสถานะ/debug
docker ps [-a]
docker logs [-f] [--tail N] <ชื่อ>
docker exec -it <ชื่อ> bash
docker inspect [-f '{{...}}'] <ชื่อ>
docker stats
docker cp <ชื่อ>:/path ./local

# image
docker build -t <name> .
docker pull / push <image>
docker tag <local> <user>/<name>:<tag>
docker rmi <image>
docker save -o out.tar <image> / docker load -i out.tar

# volume
docker volume create/ls/inspect/rm <name>

# network
docker network create/ls/inspect/rm <name>
docker run --network <name> ...

# ล้าง (เจาะจงก่อนเสมอ)
docker container prune
docker image prune [-a]
docker volume prune
docker network prune
docker system prune [-a --volumes]   # เหวี่ยงแหสุด ระวัง
```

## 🔗 เกี่ยวข้อง

- [[Docker Basics]] — พื้นฐาน image/container, ตัวอย่างใช้งานจริงทีละขั้น
- [[Docker Dockerfile]] — `docker build` แบบละเอียด, layer/cache/multi-stage
- [[Docker Compose and Deploy]] — `docker push`/registry/deploy ขึ้น server จริง
- [[Docker]] — หน้ารวม

## 📖 อ่านต่อ

- [Docker CLI reference](https://docs.docker.com/reference/cli/docker/)
- [Docker — Prune unused objects](https://docs.docker.com/config/pruning/)
- [Docker — Bridge network driver](https://docs.docker.com/engine/network/drivers/bridge/)
