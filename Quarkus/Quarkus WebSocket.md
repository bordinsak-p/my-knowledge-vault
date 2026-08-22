---
tags:
  - java
  - quarkus
  - websocket
  - realtime
type: reference
created: 2026-08-18
---

# 🔌 Quarkus WebSocket — เริ่มจากศูนย์

> HTTP ปกติคือ client ถามทีเดียว server ตอบทีเดียว จบ ([[Quarkus HTTP Methods]])
> **WebSocket คือการเปิดสายที่ค้างไว้** — เปิดครั้งเดียว แล้วทั้งสองฝั่งส่งข้อความหากันได้ตลอดเวลา โดยไม่ต้องขอใหม่ทุกครั้ง
> ใช้ทำ chat, แจ้งเตือนสด, dashboard ที่อัปเดตเองโดยไม่ต้อง refresh, progress ของงานที่รันนาน

---

## 1. ก่อนอื่น — ทำไมไม่ใช้ HTTP ธรรมดา

```
HTTP (request/response)              WebSocket (persistent connection)

client ──request──► server           client ══════ handshake ══════► server
client ◄─response── server                    (เปิดสายค้างไว้)
        (สายปิด)                      client ◄──message── server
                                       client ──message──► server
client ──request──► server            client ◄──message── server
client ◄─response── server                    (ส่งได้ทั้งสองทาง ตลอดเวลา)
        (สายปิด)                              (จนกว่าจะปิดเอง)
```

**ถ้าไม่มี WebSocket แล้วอยากได้ real-time** ทางเลือกคือ **polling** — ให้ client ถามซ้ำ ๆ ทุกไม่กี่วินาที ("มีอะไรใหม่ไหม") ซึ่งเปลืองทั้ง network และทำให้ข้อมูลช้ากว่าที่ควรเสมอ (ต้องรอถึงรอบถัดไป) WebSocket แก้ตรงนี้โดยให้ server **ส่งเองได้ทันทีที่มีเหตุการณ์เกิดขึ้น** ไม่ต้องรอถูกถาม

**ใช้เมื่อไหร่ ไม่ใช้เมื่อไหร่**

| ใช้ WebSocket | ใช้ HTTP ปกติพอ |
|---|---|
| chat, collaborative editing | CRUD ทั่วไป |
| แจ้งเตือนแบบ push ทันที | ข้อมูลที่ไม่ต้องเรียลไทม์ |
| dashboard ที่ต้องอัปเดตสด (ราคาหุ้น, สถานะเครื่องจักร) | โหลดหน้าเว็บครั้งเดียวจบ |
| progress ของงานที่รันนาน (ทางเลือกหนึ่ง) | ก็ยังใช้ polling ง่ายกว่าถ้าไม่ต้องเป๊ะเรื่องเวลา |

---

## 2. Quarkus มีสอง extension — เลือกให้ถูกกับเวอร์ชัน

| | `quarkus-websockets` (classic) | `quarkus-websockets-next` (ตัวใหม่) |
|---|---|---|
| อิงมาตรฐาน | Jakarta WebSocket (JSR-356) | API ของ Quarkus เอง ไม่ตาม JSR-356 |
| annotation | `@ServerEndpoint`, `@OnOpen`, `@OnMessage` | `@WebSocket`, `@OnOpen`, `@OnTextMessage` |
| จัดการ session เอง | ต้องทำ `Map<String, Session>` เอง | มี broadcast ในตัว ไม่ต้องเขียนเอง |
| JSON serialize | ต้องเขียน encoder/decoder เอง | อัตโนมัติ |
| ต้องการ Quarkus | ใช้ได้ทุกเวอร์ชันของ Quarkus 3.x | **ต้อง Quarkus ~3.11 ขึ้นไป** |
| สถานะ | ยังใช้ได้ ได้รับการซัพพอร์ตต่อ แต่ไม่มีฟีเจอร์ใหม่แล้ว | ทิศทางที่ Quarkus กำลังผลักดัน |

**⚠️ เช็คเวอร์ชัน Quarkus ก่อนเริ่ม** — `quarkus-websockets-next` ต้องการ Quarkus ประมาณ 3.11 ขึ้นไป ถ้าโปรเจกต์อยู่บน Quarkus รุ่นเก่ากว่านั้น (เช่น 3.3.x) **extension ตัวใหม่ใช้ไม่ได้** ต้องใช้ตัว classic แทน หรืออัปเกรด Quarkus ก่อน

```bash
# เช็คเวอร์ชัน Quarkus ของโปรเจกต์
grep quarkus.platform.version pom.xml
```

โน้ตนี้สอนทั้งสองแบบ — **ถ้าอยากลองเล่นเร็ว ๆ แนะนำสร้างโปรเจกต์ใหม่แยกต่างหากด้วย Quarkus ล่าสุด** แล้วใช้ `websockets-next` เพราะเขียนสั้นกว่าและ error น้อยกว่ามาก (ดู [[Greenhouse]] — เหมาะเป็นโปรเจกต์ทดลองแบบเดียวกับ [[URL Shortener (Quarkus)]])

---

## 3. แบบ Classic (`quarkus-websockets`) — ใช้ได้ทุกเวอร์ชัน

### ติดตั้ง

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-websockets</artifactId>
</dependency>
```

### ตัวอย่างเต็ม — ห้องแชท

```java
package org.acme.ws;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.websocket.*;
import jakarta.websocket.server.PathParam;
import jakarta.websocket.server.ServerEndpoint;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@ServerEndpoint("/chat/{username}")
@ApplicationScoped
public class ChatSocket {

    Map<String, Session> sessions = new ConcurrentHashMap<>();

    @OnOpen
    public void onOpen(Session session, @PathParam("username") String username) {
        sessions.put(username, session);
        broadcast("🟢 " + username + " เข้าห้องแชท");
    }

    @OnMessage
    public void onMessage(String message, @PathParam("username") String username) {
        broadcast(username + ": " + message);
    }

    @OnClose
    public void onClose(Session session, @PathParam("username") String username) {
        sessions.remove(username);
        broadcast("🔴 " + username + " ออกจากห้อง");
    }

    @OnError
    public void onError(Session session, @PathParam("username") String username, Throwable t) {
        sessions.remove(username);
        broadcast("⚠️ " + username + " หลุดการเชื่อมต่อ: " + t.getMessage());
    }

    private void broadcast(String message) {
        sessions.values().forEach(s ->
            s.getAsyncRemote().sendObject(message, result -> {
                if (result.getException() != null) {
                    System.out.println("ส่งไม่สำเร็จ: " + result.getException());
                }
            }));
    }
}
```

**สี่ annotation ที่ต้องมี — ครอบ lifecycle ทั้งหมดของการเชื่อมต่อหนึ่งเส้น**

```
เปิดสาย ──► @OnOpen ──► @OnMessage (ซ้ำได้หลายครั้ง) ──► @OnClose
                                    │
                                    └──► @OnError (ถ้าพังระหว่างทาง)
```

**`sessions` map คือหัวใจของแบบ classic** — ไม่มี broadcast ให้ในตัว ต้องเก็บ `Session` ของทุกคนที่เชื่อมต่ออยู่เอง แล้ววนส่งเองตอนอยากกระจายข้อความ **ต้องเป็น `ConcurrentHashMap`** เพราะหลาย connection เข้า-ออกพร้อมกันได้ตลอดเวลา ใช้ `HashMap` ธรรมดาจะพังแบบสุ่ม ๆ ตอนมีคนใช้เยอะ

---

## 4. แบบ WebSockets Next (`quarkus-websockets-next`) — สั้นกว่ามาก

### ติดตั้ง

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-websockets-next</artifactId>
</dependency>
```

### ตัวอย่างเต็ม — ห้องแชทเดียวกัน สั้นกว่าครึ่งหนึ่ง

```java
package org.acme.ws;

import io.quarkus.websockets.next.*;
import jakarta.inject.Inject;

@WebSocket(path = "/chat/{username}")
public class ChatWebSocket {

    public enum Type { JOINED, LEFT, MESSAGE }
    public record ChatMessage(Type type, String from, String text) {}

    @Inject WebSocketConnection connection;

    @OnOpen(broadcast = true)
    public ChatMessage onOpen() {
        return new ChatMessage(Type.JOINED, connection.pathParam("username"), null);
    }

    @OnTextMessage(broadcast = true)
    public ChatMessage onMessage(ChatMessage incoming) {
        return incoming;      // ส่งกลับไปให้ทุกคนที่เชื่อมต่ออยู่
    }

    @OnClose
    public void onClose() {
        var msg = new ChatMessage(Type.LEFT, connection.pathParam("username"), null);
        connection.broadcast().sendTextAndAwait(msg);
    }
}
```

**สามข้อที่ต่างจาก classic อย่างชัดเจน**

1. **ไม่ต้องเก็บ `Map<String, Session>` เอง** — `WebSocketConnection` inject เข้ามาแทน `Session`, และ `broadcast = true` บน annotation จัดการกระจายข้อความให้อัตโนมัติ
2. **รับ/ส่งเป็น object ได้ตรง ๆ** — ประกาศ parameter เป็น `ChatMessage` (record) แล้ว Quarkus serialize/deserialize เป็น JSON ให้เอง ไม่ต้องแปลง `String` เอง
3. **ไม่ต้อง `@ApplicationScoped` เอง** — connection แต่ละเส้นมี instance ของตัวเองอัตโนมัติ

---

## 5. ทดสอบด้วย HTML ธรรมดา — ไม่ต้องมี framework ฝั่ง client

```html
<!DOCTYPE html>
<html>
<body>
  <input id="msg"><button onclick="send()">ส่ง</button>
  <ul id="log"></ul>
  <script>
    const ws = new WebSocket("ws://localhost:8080/chat/Somchai");

    ws.onopen    = () => log("เชื่อมต่อแล้ว");
    ws.onmessage = (e) => log("รับ: " + e.data);
    ws.onclose   = () => log("ปิดการเชื่อมต่อ");
    ws.onerror   = (e) => log("error: " + e);

    function send() {
      ws.send(document.getElementById("msg").value);
    }
    function log(text) {
      document.getElementById("log").innerHTML += "<li>" + text + "</li>";
    }
  </script>
</body>
</html>
```

**เปิดสอง browser tab พร้อมกัน คนละ username** (`ws://localhost:8080/chat/Somchai` กับ `.../chat/Malee`) — พิมพ์ข้อความจาก tab หนึ่ง จะเห็นขึ้นในอีก tab ทันที **นี่คือวิธีเห็นภาพ broadcast ได้เร็วที่สุด**

> `ws://` คือ WebSocket แบบไม่เข้ารหัส (คู่กับ `http://`) ส่วน `wss://` คือแบบเข้ารหัส (คู่กับ `https://`) — **production ต้องใช้ `wss://` เสมอ**

---

## 6. Path parameter, query parameter, header

```java
// classic
@ServerEndpoint("/rooms/{roomId}/chat/{username}")
public void onOpen(Session s, @PathParam("roomId") String roomId,
                              @PathParam("username") String username) { ... }
```

```java
// websockets-next
@WebSocket(path = "/rooms/{roomId}/chat/{username}")
public class RoomSocket {
    @Inject WebSocketConnection connection;

    @OnOpen
    public void onOpen() {
        String roomId = connection.pathParam("roomId");
        String username = connection.pathParam("username");
    }
}
```

**ห้อง (room) แยกกันยังไง:** ทั้งสองแบบไม่มี "ห้อง" ให้อัตโนมัติ — ต้องแยกเองโดยเก็บ session/connection เป็นกลุ่มตาม `roomId` (classic ใช้ `Map<String, Map<String, Session>>`, next ใช้ `WebSocketConnection.getOpenConnections()` แล้ว filter เอาเฉพาะที่ path param ตรงกับห้องที่ต้องการ)

---

## 7. ส่งข้อความจากที่อื่นในระบบ ไม่ใช่แค่ตอบกลับ

**เคสที่เจอบ่อยกว่าที่คิด:** อยาก push แจ้งเตือนจาก service อื่นที่ไม่เกี่ยวกับ WebSocket เลย เช่น มี order ใหม่เข้ามาทาง REST endpoint ปกติ แล้วอยากแจ้งเตือนไปที่หน้าจอ dashboard ที่เปิด WebSocket ค้างอยู่

```java
// websockets-next — inject ตัว broadcaster เข้าไปในคลาสอื่นได้เลย
@ApplicationScoped
public class OrderNotifier {

    @Inject WebSocketConnection.BroadcastSender broadcaster;   // ตัวอย่างแนวคิด — ดูชื่อ API จริงจาก reference guide

    public void notifyNewOrder(Order o) {
        // ส่งแจ้งเตือนไปทุก client ที่เชื่อมต่อ endpoint นี้อยู่ โดยไม่ต้องรอ client ถาม
    }
}
```

**นี่คือจุดที่ WebSocket ต่างจาก HTTP จริง ๆ** — HTTP endpoint ตอบได้เฉพาะตอนมีคนถามเท่านั้น แต่ WebSocket connection ที่เปิดค้างไว้ ทำให้ **server เป็นฝ่ายเริ่มส่งได้เอง** จาก thread ไหนก็ได้ในระบบ ไม่ต้องรอ client ร้องขอ

---

## 8. กับดัก

- **ไม่จับกรณี connection หลุดกลางทาง** — เน็ตล่ม, มือถือสลับ wifi/4G, ปิดแท็บกะทันหัน ต้องมี `@OnClose`/`@OnError` เก็บกวาด session ที่ค้างเสมอ ไม่งั้น memory leak
- **`HashMap` ธรรมดาเก็บ session** — ต้องเป็น concurrent collection เพราะ event เข้า-ออกพร้อมกันได้
- **ลืมว่า WebSocket connection กินทรัพยากรค้างไว้ตลอด** — ต่างจาก HTTP request ที่จบแล้วปล่อยทันที การเปิดค้างเป็นพัน connection มีผลกับ memory/thread ของ server ต้องคิดเรื่อง scale ตั้งแต่ต้น (โดยเฉพาะถ้า deploy หลาย instance — connection หนึ่งอยู่ที่ instance หนึ่ง ต้องมีกลไกกระจายข้อความข้าม instance เช่นผ่าน Redis Pub/Sub ดู [[Quarkus Redis]])
- **ไม่มี authentication บน WebSocket endpoint** — URL แบบ `/chat/{username}` ให้ client กำหนด username เองเฉย ๆ ไม่ปลอดภัยเลยสำหรับงานจริง ต้องผูกกับ token/session ที่ตรวจสอบได้จริง
- **ส่งข้อมูลก้อนใหญ่ผ่าน WebSocket** — เหมาะกับข้อความเล็ก ๆ ถี่ ๆ ไม่ใช่ไฟล์ใหญ่ ไฟล์ใหญ่ใช้ HTTP endpoint ปกติแบบ [[Quarkus MIME Types]] ที่คุยกันไปแทน
- **สับสนระหว่างสอง extension** — โค้ดตัวอย่างบนเว็บส่วนใหญ่ (รวมถึงจาก AI) มักเป็น `websockets-next` เพราะสั้นและใหม่กว่า ถ้าโปรเจกต์เป็น Quarkus รุ่นเก่า ก๊อปมาแล้ว build ไม่ผ่านเพราะ annotation คนละชุดกับ classic

---

## 9. เริ่มลองยังไง — ขั้นตอนที่แนะนำ

```
1. สร้างโปรเจกต์ทดลองแยกต่างหาก (code.quarkus.io เลือก extension "WebSockets Next")
2. ก๊อปตัวอย่างห้องแชทในข้อ 4 ไปวาง
3. รัน mvn quarkus:dev
4. เปิดไฟล์ HTML ในข้อ 5 สองแท็บ คนละ username
5. พิมพ์ข้อความ ดู broadcast เกิดขึ้นจริง
6. ลองปิดแท็บหนึ่งกลางคัน ดูว่า @OnClose ทำงานไหม
7. ลองเพิ่ม roomId เข้าไป (ข้อ 6) แยกห้องแชท
8. ค่อยคิดเรื่อง authentication และ scale ทีหลัง เมื่อของพื้นฐานเข้าใจแล้ว
```

**ห้ามเริ่มกับโปรเจกต์งานจริง** — ใช้หลักการเดียวกับที่แนะนำไว้ตอนคุยเรื่อง [[Quarkus HTTP Methods]] และ [[Greenhouse]]: ทดลองในที่ที่พังได้อย่างปลอดภัยก่อน โดยเฉพาะเรื่อง WebSocket ที่มีเรื่อง connection lifecycle ให้งงได้ง่ายกว่า REST ธรรมดามาก

---

## 10. Cheat sheet

```java
// classic
@ServerEndpoint("/path/{param}")
@OnOpen  public void onOpen(Session s, @PathParam("param") String p) { }
@OnMessage public void onMessage(String msg, @PathParam("param") String p) { }
@OnClose public void onClose(Session s) { }
@OnError public void onError(Session s, Throwable t) { }
session.getAsyncRemote().sendObject(data, callback);

// websockets-next
@WebSocket(path = "/path/{param}")
@Inject WebSocketConnection connection;
@OnOpen(broadcast = true)      public ReturnType onOpen() { }
@OnTextMessage(broadcast = true) public ReturnType onMessage(InType msg) { }
@OnClose public void onClose() { }
@OnError public void onError(Throwable t) { }
connection.pathParam("name");
connection.broadcast().sendTextAndAwait(obj);
```

| อาการ | สาเหตุ |
|---|---|
| ก๊อปโค้ดจากเว็บมาแล้ว build ไม่ผ่าน | ใช้ annotation ของ `websockets-next` แต่โปรเจกต์มีแค่ `websockets` classic (หรือกลับกัน) |
| `websockets-next` เพิ่ม dependency แล้วยัง error | เวอร์ชัน Quarkus เก่าเกินไป (ต้องการ ~3.11+) |
| memory โตขึ้นเรื่อย ๆ เมื่อมีคนต่อ-หลุดบ่อย | ไม่ได้ลบ session ออกจาก map ตอน `@OnClose`/`@OnError` |
| `ConcurrentModificationException` ตอน broadcast | ใช้ `HashMap` ธรรมดาเก็บ session แทน concurrent collection |
| deploy หลาย instance แล้ว broadcast ไปไม่ถึงทุกคน | connection กระจายอยู่คนละ instance ต้องมีกลไกกระจายข้อความข้าม instance |
| ใครก็เข้าห้องแชทของใครก็ได้ | ไม่มี authentication ผูกกับ path parameter ตรง ๆ |

---

## 🔗 เกี่ยวข้อง

- [[Quarkus HTTP Methods]] — HTTP request/response เดี่ยว ๆ ต่างจาก WebSocket ที่เปิดสายค้าง
- [[Quarkus Redis]] — Pub/Sub ใช้กระจายข้อความข้าม instance เมื่อ deploy หลายตัว
- [[Quarkus Build]] — เช็คเวอร์ชัน Quarkus ก่อนเลือก extension
- [[Greenhouse]] — ที่ทดลองของก่อนเอาไปใช้จริง

## 📖 อ่านต่อ

- [Quarkus — Getting started with WebSockets Next](https://quarkus.io/guides/websockets-next-tutorial)
- [Quarkus — WebSockets Next reference guide](https://quarkus.io/guides/websockets-next-reference)
- [Quarkus — WebSockets (classic)](https://quarkus.io/guides/websockets)
