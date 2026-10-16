---
tags:
  - research
  - research/seed
  - security
  - auth
  - quarkus
  - angular
type: project
status: seed
area: fullstack
created: 2026-10-15
updated: 2026-10-16
---

# 🔐 Auth Flows Lab — เรียน auth ทุกแบบด้วยการสร้างจริง

ที่มา: ผู้ใช้ขอเมื่อ 15 ต.ค. 2026 — อยากเรียนรู้ว่า flow การทำ auth มีกี่แบบ เอาทุกแบบ อธิบายละเอียดว่ามีอะไร ต้องทำอะไร ใช้อะไร รวมถึงมาตรฐานที่คนใช้กัน (OAuth) แล้วทำเป็นสเปค — รายละเอียดเชิงสเปคทั้งหมดอยู่ที่ [[Auth Flow Spec]] (หน้ารวม) แตกเป็น 4 ส่วนตามตระกูล โน้ตนี้เก็บเฉพาะมุมตัดสินใจ: ปัญหา / สมมติฐาน / prior art / การทดลอง / ความเสี่ยง

## 🎯 ปัญหา

ไม่รู้ภาพรวมว่า "ทำ auth" มีกี่แบบ แต่ละแบบเหมาะกับงานแบบไหน ต้องสร้างอะไรฝั่ง server / client / IdP และใช้เครื่องมืออะไร — อยากได้ภาพครบ **ทุกแบบ** ไม่ใช่แค่ OAuth/Keycloak ที่มีโน้ตแล้ว พร้อมสเปคที่ลงมือสร้างตามได้จริง

สิ่งที่ vault มีแล้ว: OIDC แบบ Authorization Code + PKCE กับ Keycloak/Quarkus/Angular, grant type และ provider, custom permission, workload identity ของ CI (ดู 🔍 ด้านล่าง) สิ่งที่ยังไม่มี: session/cookie, API key, Basic, JWT ที่ออกเอง, วงจรชีวิตบัญชี (สมัคร/ยืนยัน/รีเซ็ต), MFA/TOTP, passkey, SAML, mTLS, HMAC signature, LDAP/Kerberos, gateway auth และที่สำคัญคือ **การเอามาวางเทียบกันแบบลงมือจริง**

ตอบให้ชัดก่อนขยับจาก `seed`: เรียนทั้ง 38 รายการในสเปคด้วยการสร้างจริงทั้งหมด หรือสร้างเฉพาะแกนหลัก (M0–M6 ใน [[Auth Flow Spec]] ข้อ 6) แล้วอ่านที่เหลือเป็นสเปคเฉยๆ — ดูคำถามเปิดข้อ 9 ของสเปค

## 💭 สมมติฐาน

(ร่างเพิ่มตอนบันทึก — ยังไม่ได้ยืนยันกับผู้ใช้)

- เชื่อว่า endpoint `whoami` ตัวเดียวที่เปิดให้ผ่านได้ทีละ mechanism แล้วดู HTTP จริง (header / cookie / token / 401 challenge) ทำให้เห็นความต่างได้เร็วกว่าอ่านทีละหัวข้อ
- เชื่อว่า auth ทั้งหมดจัดเป็นตระกูลได้ด้วยคำถามไม่กี่ข้อ (ใครตรวจ credential / พิสูจน์ด้วยอะไร / state อยู่ที่ไหน / ใครเชื่อใคร) — ถ้าแผนที่ใน [[Auth Flow Spec]] ข้อ 2–3 ใช้ตัดสินใจเลือก flow ตอนลองจริงไม่ได้ แปลว่าจัดหมวดผิด
- เชื่อว่าเขียนเองแบบหยาบก่อน (session, JWT, TOTP) แล้วค่อยใช้ IdP จะเข้าใจว่า Keycloak ทำอะไรแทนเราบ้าง

**ถ้าสมมติฐานผิด จะรู้จาก:** ทำ M0–M2 แล้วยังตอบไม่ได้ว่า session กับ JWT ต่างกันที่ revoke/XSS/CSRF ยังไง หรือเปิดสเปคแล้วยังเลือก flow ให้งานจริงไม่ถูก

## ❓ ทำไมต้องตอนนี้

(ยังไม่ได้ตอบ — ผู้ใช้ไม่ได้ระบุเหตุผลเชิงเวลา) เบาะแสที่เห็น:

- [[LINE Slip Bot]] ต้องล็อกอินหน้าแอดมินด้วย LINE Login และยังไม่ได้เลือกวิธีเก็บ session ของหน้าแอดมิน → มีโปรเจกต์จริงรอใช้ความรู้นี้
- มาตรฐานเพิ่งขยับ: RFC 10017 (BCP 212, OAuth 2.0 for Browser-Based Applications) เผยแพร่ ส.ค. 2026, W3C WebAuthn Level 3 เป็น Recommendation 25 ส.ค. 2026, ส่วน OAuth 2.1 ยังเป็นแค่ draft (v16, ก.ย. 2026)

ถ้าตอบไม่ได้ ไม่ผิด — แปลว่าไม่รีบ

## 🔍 Prior art

| แหล่ง | ครอบคลุม | จุดที่หยุด |
|---|---|---|
| [[OAuth 2.0]] | grant type, provider (Keycloak/Auth0/Okta ฯลฯ) | ไม่ลงระดับ HTTP จริง ไม่บอกว่าต้องสร้างอะไร ไม่มี auth นอก OAuth |
| [[Keycloak]] / [[Quarkus Keycloak]] | OIDC code + PKCE แบบ Angular + Quarkus, client credentials | flow เดียวเป็นหลัก ไม่เทียบกับทางเลือกอื่น |
| [[Quarkus Security]] | augmentor, custom permission, multi-tenant | ฝั่ง authorization มากกว่า authentication |
| [[CI-CD Secrets]] | workload identity (OIDC จาก GitHub Actions) | เฉพาะ CI |
| เอกสาร Quarkus Security | ครบทีละ mechanism | แยกหลายคู่มือ ไม่มีที่ไหนเทียบกัน |
| OWASP ASVS 5.0 (บท V6–V10) | checklist ที่ทดสอบได้ | ไม่ใช่ tutorial |
| RFC 9700, RFC 10017 | best practice ทางการ | อ่านยาก ไม่มีโค้ด |

## ✅ รู้ได้ยังไงว่าสำเร็จ

(ร่างจากเป้าหมาย "เรียนรู้" — ยังไม่ใช่ข้อตกลง)

- ทุก flow ที่เลือกสร้าง มี endpoint `whoami` ที่ผ่านได้ด้วย credential แบบนั้น และมี transcript (curl / Bruno) ว่าส่งอะไรไปได้อะไรกลับ เก็บใน `notes/` ของ repo
- อธิบายได้โดยไม่ดูโน้ตว่า session ต่างจาก JWT, SPA ตรงๆ ต่างจาก BFF, API key ต่างจาก client credentials ยังไง
- ตอบได้ต่อทุก flow ว่า "ถ้า credential/token หลุด เสียหายแค่ไหน และกันยังไง"
- ผ่าน checklist ชุดย่อยที่เลือกจาก ASVS ([[Auth Flow Spec]] ข้อ 7)

## 🧪 การทดลองที่เล็กที่สุด

**ข้อเสนอ (ยังไม่ยืนยัน) — วันเสาร์เดียว: Quarkus 1 ตัว มี `GET /lab/<mechanism>/whoami` 4 ทางเข้า**

1. Basic (`quarkus.http.auth.basic=true`)
2. Form login + cookie
3. JWT bearer (ออก token เองด้วย SmallRye JWT)
4. API key (เขียน `HttpAuthenticationMechanism` เอง)

ยิงด้วย curl ทุกทาง บันทึกว่า: ไม่ส่ง credential ได้อะไรกลับ (status + `WWW-Authenticate`), ส่งผิดได้อะไร, credential ของทางหนึ่งเอาไปใช้อีกทางได้ไหม แล้วเขียนตารางเทียบ 1 หน้า

- ไปต่อถ้า: อธิบายตารางนั้นให้คนอื่นฟังได้โดยไม่ดูโน้ต และเริ่มสงสัย "แล้ว OIDC อยู่ตรงไหนของตารางนี้" (= พร้อมไป M3–M4)
- พอถ้า: ทำแล้วรู้สึกว่าไม่ได้เห็นอะไรที่อ่านเอาไม่ได้ — ให้ถอยไปอ่านสเปคอย่างเดียว

**ผู้บันทึกรันสเก็ตช์ตัวนี้ไปแล้ว 1 รอบเพื่อตรวจสเปค** (Quarkus 3.33.4 LTS + JDK 21, โฟลเดอร์ชั่วคราวที่รันถูกลบทิ้งแล้ว ไม่ได้เก็บใน repo ของผู้ใช้ — โค้ดสำคัญอยู่ในสเปคส่วนที่ 1 ข้อ A4/A6 และหน้ารวมข้อ 5.1) ผลจริงอยู่ใน [[Auth Flow Spec 1 — Own Accounts]] — นับเป็นการยืนยันว่าสเปคทำตามได้ ไม่นับเป็นการทดลองของผู้ใช้เอง

## ⚠️ ความเสี่ยง / สิ่งที่อาจทำให้พัง

- **ขอบเขตใหญ่มาก** — 38 รายการใน 4 ตระกูล ถ้าตั้งใจสร้างหมดจะไม่จบ → ใช้ milestone เลือกเฉพาะแกนหลัก (M0–M6) แล้วอ่านที่เหลือ
- **โค้ดที่เขียนเองเพื่อเรียนไม่ใช่ของ production** — session/JWT/TOTP ที่ทำเองมีไว้เข้าใจ ห้ามยกไปใช้จริง (ใช้ IdP หรือไลบรารี)
- **มาตรฐานกำลังขยับ** — OAuth 2.1 ยังเป็น draft, Quarkus 4.0.0.Beta1 ออก 1 ต.ค. 2026 (Maven Central), Keycloak ออก 26.x ถี่ → เช็กเวอร์ชันก่อนทำทุก milestone
- **ส่วนที่พึ่ง Docker/Keycloak ยังไม่เคยรันจริง** — ตอนตรวจสเปค Docker daemon ในเครื่องไม่ได้เปิด ทุกอย่างของ OIDC/Keycloak ในสเปคจึงมาจากการอ่านเอกสาร (ป้าย 🔎) ไม่ใช่การรัน (ป้าย ▶️)
- **บาง flow ต้องมี infra เพิ่ม** — WebAuthn ต้อง HTTPS หรือ localhost, Apple/Facebook/LINE ต้อง callback เป็น HTTPS สาธารณะ (ใช้ tunnel ตาม [[Cloudflare]]), SAML ต้องมี SAML IdP, mTLS ต้องมี CA ทดลอง
- **Windows + JDK 21 อาจ start Quarkus ไม่ขึ้น** — เจอจริงตอนตรวจสเปค วิธีแก้อยู่ใน [[Auth Flow Spec]] ข้อ 8

## 📚 ต้องไปเรียนรู้เพิ่ม

- Keycloak 26.x: ตั้งค่า standard token exchange (26.2+), DPoP และ passkeys (26.4+) ใช้ยังไงในหน้า admin — ยังไม่เคยเปิดดูจริง
- Quarkus: ใช้ DPoP ได้ไหม (ยังไม่พบในเอกสารที่ตรวจ), BFF แบบเต็มที่ proxy API ด้วย `quarkus-oidc` web-app + token propagation
- SAML: อ่าน XML จริงของ AuthnRequest/Response ด้วย browser extension, XML signature wrapping
- WebAuthn: conditional UI (autofill), Related Origin Requests, recovery เมื่อเครื่องหาย
- OIDC logout 3 แบบ (RP-initiated / front-channel / back-channel) ทำงานต่างกันยังไงเมื่อมีหลายแอป
- OWASP ASVS 5.0 บท V6–V10: เลือกข้อที่เอามาเป็น checklist ของ lab

## 🔗 เกี่ยวข้องกับ

- [[Auth Flow Spec]] — หน้ารวมสเปค แผนที่ทั้ง 38 รายการ + ลำดับสร้าง
  - [[Auth Flow Spec 1 — Own Accounts]] · [[Auth Flow Spec 2 — OAuth OIDC]] · [[Auth Flow Spec 3 — Federation Enterprise]] · [[Auth Flow Spec 4 — MFA Passkeys Machine]]
- [[OAuth 2.0]] — grant type และ provider (ไม่พูดซ้ำในสเปค)
- [[Keycloak]], [[Quarkus Keycloak]] — IdP และตัวอย่าง Angular + Quarkus ที่ lab ใช้ต่อยอด
- [[Quarkus Security]] — authorization (permission, multi-tenant)
- [[CI-CD Secrets]] — workload identity (GitHub Actions OIDC)
- [[LINE Slip Bot]] — โปรเจกต์ที่ต้องใช้ LINE Login ฝั่งแอดมิน
- [[Cloudflare]] — tunnel สำหรับ callback ที่ต้อง HTTPS สาธารณะ
- [[Greenhouse]]

## 📝 Log

### 2026-10-15

- สร้างโน้ตจากคำขอของผู้ใช้ ("เพิ่มโปรเจ็ค flow การทำ auth ... ทุกแบบ ... ทำเป็น spec") แยกสเปคเป็นหน้ารวม [[Auth Flow Spec]] + 4 ส่วนลูก เพราะรวมแล้วยาวกว่าโน้ตที่ใหญ่สุดใน vault หลายเท่า
- **สถานะตั้งต้น `seed`** — ยังไม่ได้เลือกขอบเขต (สร้างทั้งหมดหรือเฉพาะแกนหลัก) ← ต้องตอบก่อนขยับเป็น `sprouting`
- ค้นข้อมูลจากต้นทาง (RFC, เอกสาร Quarkus, W3C, NIST, OWASP, LINE, Cloudflare, MCP spec) แล้วทำเครื่องหมายสถานะการตรวจไว้ในสเปคทุกส่วน
- รันสเก็ตช์ Quarkus 3.33.4 เพื่อตรวจสเปค (Basic / Form / JWT / API key / HMAC webhook) ผลจริงอยู่ในสเปคส่วนที่ 1 และ 4 — เจอ 3 เรื่องที่เอกสารไม่ได้เน้น: Basic ไม่ทำงานเงียบๆ ถ้าไม่ตั้ง `quarkus.http.auth.basic=true` เมื่อเปิด mechanism อื่นอยู่, cookie ของ form login ไม่มี `HttpOnly` โดย default, SmallRye JWT ไม่ตรวจ `aud` ถ้าไม่ตั้ง `mp.jwt.verify.audiences`
- ตรวจรูปแบบโน้ต: wikilink ทุกอันในโน้ตใหม่และโน้ตที่แก้ resolve ได้, แผนภาพ Mermaid ทั้ง 17 อัน parse + render ผ่านด้วย Mermaid 11.4.1, ไม่มีตารางใน blockquote
- เจอ Quarkus start ไม่ขึ้นบน Windows + JDK 21 ในเครื่องที่ใช้ตรวจ (`Selector.open()` ล้มเหลว) แก้ด้วย `-Djdk.net.unixdomain.tmpdir` — รายละเอียดอยู่ใน [[Auth Flow Spec]] ข้อ 8
- ยังไม่ได้ตอบ: ขอบเขต, ขยับ status หรือยัง, ใช้ Keycloak เป็น IdP หลักไหม, SPA default เป็น BFF หรือ SPA ตรงๆ

### 2026-10-16

- ผู้ใช้บอกว่ายังไม่เคยเขียน reactive จึงปรับโค้ด A4 (API key) ในสเปคส่วนที่ 1: เพิ่มคู่มือ `Uni` 1 นาที, ย้ายงานที่ block ได้เข้า `runBlocking`, แยก `ApiKeyStore` เป็น Java ธรรมดา แล้วคอมไพล์ + รันซ้ำบน Quarkus 3.33.4 (ผลเดิมเหมือนเดิมทุกข้อ) และลองกับดัก 2 ข้อ — `.await()` ใน mechanism ได้ 500, block ตรง ๆ ไม่พังแต่ log เตือน event loop ค้าง
- ตามนี้ตัวอย่างโค้ดส่วนอื่นของสเปคควรเลือกแบบ blocking/Java ธรรมดาก่อน และอธิบาย `Uni` เฉพาะจุดที่ Quarkus บังคับ

## 🪦 ถ้าเลิกทำ

(ร่าง ยังไม่ใช่ข้อตกลง) เลิกถ้าอ่านสเปค + ทำ M0–M2 แล้วรู้สึกว่าได้ภาพครบโดยไม่ต้องสร้างต่อ หรือถ้ามีโปรเจกต์จริงที่ต้องใช้ auth (เช่น [[LINE Slip Bot]]) ก็ย้ายไปเรียนในโปรเจกต์นั้นแทน
จะกลับมาคิดใหม่เมื่อ: (ยังไม่ได้ตอบ)

---

## 📖 แหล่งอ้างอิง

รายการเต็มอยู่ท้ายสเปคแต่ละส่วน ที่ใช้หลักๆ:

- [Quarkus — Security overview](https://quarkus.io/guides/security-overview) และ [Authentication mechanisms](https://quarkus.io/guides/security-authentication-mechanisms)
- [RFC 10017 — OAuth 2.0 for Browser-Based Applications](https://www.rfc-editor.org/info/rfc10017)
- [RFC 9700 — Best Current Practice for OAuth 2.0 Security](https://www.rfc-editor.org/rfc/rfc9700.html)
- [W3C — Web Authentication Level 3](https://www.w3.org/TR/webauthn-3/)
- [NIST SP 800-63B-4](https://pages.nist.gov/800-63-4/sp800-63b.html)
- [OWASP ASVS 5.0](https://github.com/OWASP/ASVS/tree/master/5.0/en) และ [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
