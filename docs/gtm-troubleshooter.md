---
name: gtm-troubleshooter
description: >
  Diagnose and fix GTM server-side tracking problems across the full browser → GTM Web → GTM Server → Meta CAPI pipeline. Use this skill whenever user reports something broken, missing, or wrong — including symptoms like: event หาย, ไม่เข้า, CAPI ไม่ fire, deduplication พัง, นับซ้ำ, revenue เพี้ยน, EMQ ต่ำ, purchase ไม่ขึ้น, ค่าผิด, Server Preview ว่าง, tag ไม่ fire, request ไม่เข้า server, cookie หาย, pixel ไม่ทำงาน. Use even if user พิมพ์แค่ "ทำไม event ไม่เข้า", "พัง", "ตรวจให้หน่อย", หรือ paste error มาให้โดยไม่อธิบาย.
---

# GTM Server-Side Troubleshooter

## Overview

Skill นี้ใช้ diagnose ปัญหาใน GTM server-side pipeline ทั้งหมด — ตั้งแต่ browser ไม่ส่ง event จนถึง Meta ไม่รับ conversion ขั้นตอนคือ **ถามอาการ → ระบุ layer ที่พัง → walk-through checklist → ให้ fix**

---

## Hand-off — ส่งงานต่อเมื่อ root cause เป็นปัญหาเชิงลึก

Skill นี้เก่งเรื่อง diagnose ว่า "พังที่ layer ไหน" แต่บาง root cause ต้องเจาะลึกกว่านั้น เมื่อ diagnose เสร็จแล้วพบว่าเป็นปัญหาเชิงระบบหรือ data → hand off ไป skill เฉพาะทาง:

- **ไปที่ `gtm-reliability-monitoring`** — เมื่อพบว่า root cause เป็น cold start / scale-to-zero / capacity overflow / event หายเป็นช่วงโดยไม่มี error (เชิง infrastructure) หรือเมื่อ user ต้องตั้ง alert/monitoring เพื่อจับปัญหาในอนาคต
- **ไปที่ `gtm-data-quality-emq`** — เมื่อพบว่าเป็นปัญหา Layer 6 (payload: value/currency/items ผิด type) หรือ Layer 7 (EMQ: signal หาย, hash ผิด, double hash) ที่ต้องเจาะลึก + ให้ before/after code

เมื่อ hand off: สรุป context ที่ diagnose ได้ส่งต่อไป ไม่ต้องให้ skill ปลายทางถามซ้ำ

---

## Step 1 — ถามอาการก่อนเสมอ

ถ้า user ยังไม่ได้บอกอาการชัดเจน ถามข้อใดข้อหนึ่งที่ยังไม่รู้:

```
1. อาการที่เห็นคืออะไร? (event ไม่เข้า / ค่าผิด / นับซ้ำ / EMQ ต่ำ / อื่นๆ)
2. เห็นปัญหาจากที่ไหน? (GTM Preview / Meta Events Manager / GA4 DebugView / อื่นๆ)
3. เพิ่งเกิดขึ้น หรือไม่เคยทำงานเลยตั้งแต่ต้น?
```

จากอาการที่ได้ → map ไปยัง Layer ที่เหมาะสมด้านล่าง

---

## Step 2 — ระบุ Layer ที่พัง

ใช้ตารางนี้เพื่อ shortlist layer ที่น่าจะมีปัญหา:

| อาการที่เห็น | Layer ที่น่าตรวจก่อน |
|---|---|
| GTM Server Preview ว่าง ไม่มี request เข้ามาเลย | Layer 1 — Browser → Server |
| Server Preview มี request แต่ Client ไม่ claim | Layer 2 — GA4 Client |
| Client claim แต่ Meta CAPI Tag ไม่ fire | Layer 3 — Server Tags & Triggers |
| Meta tag fire แต่ Events Manager ไม่เห็น event | Layer 4 — Meta CAPI Connection |
| Events Manager เห็น event แต่นับ 2 ครั้ง | Layer 5 — Deduplication |
| Conversion เข้าแต่ revenue เพี้ยน / ค่าผิด | Layer 6 — Payload Data Quality |
| EMQ score ต่ำ (< 6) | Layer 7 — User Data |
| Purchase ไม่เข้าเลย | ตรวจทั้ง Layer 3, 4, 6 |

---

## Layer 1 — Browser ไม่ส่งถึง Server

**อาการ:** GTM Server Preview ว่าง ไม่มี request เข้ามาเลย

### Checklist

**GA4 Configuration Tag (Web Container):**
- [ ] `transport_url` ชี้ไปที่ server URL ของตัวเอง — ไม่ใช่ `www.google-analytics.com`
  - ถูก: `https://server-side-tagging-xxxxx-uc.a.run.app`
  - ผิด: `https://www.google-analytics.com`
- [ ] `server_container_url` ตั้งค่าเป็น URL เดียวกับ `transport_url`
- [ ] GA4 Configuration Tag มี trigger เป็น `Initialization - All Pages`
- [ ] Web Container **published** แล้ว — ไม่ใช่แค่ preview

**dataLayer Push:**
- [ ] `window.dataLayer.push()` อยู่ในหน้าที่ถูกต้อง
- [ ] Event name ตรงกับ Custom Event trigger ใน GTM
- [ ] ไม่มี JavaScript error ขัดก่อน push (ดูใน Browser Console)

**ทดสอบ:**
```javascript
// เปิด Browser Console แล้วรัน:
console.log(window.dataLayer);
// ควรเห็น array ที่มี event objects อยู่
```

**Root causes ที่พบบ่อย:**
- ลืม publish web container หลังแก้ไข
- `transport_url` ยังเป็นค่า default ของ Google
- Push dataLayer ก่อน GTM snippet โหลด

> **→ Hand off:** ถ้า request เข้าบ้างไม่เข้าบ้าง โดยเฉพาะช่วง traffic เบา/ตอนเช้า (ไม่ใช่ config ผิด) → น่าจะเป็น cold start → ส่งต่อ `gtm-reliability-monitoring`

---

## Layer 2 — GA4 Client ไม่ Claim Request

**อาการ:** Server Preview เห็น incoming request แต่ไม่มี Client claim / Tags ไม่ fire

### Checklist

- [ ] มี GA4 Client ใน Server Container (Admin → Clients)
- [ ] GA4 Client config ไม่ได้ restrict path ที่ผิด
- [ ] Request format เป็น GA4 Measurement Protocol (มี `en=` parameter)
- [ ] ดู Request tab ใน Server Preview — column "Claimed by" ว่างหรือเปล่า

**วิธีตรวจ Request Tab:**
ใน Server Preview → คลิก request → ดู tab "Request" → ถ้า "Client name" ว่าง = ไม่มี Client claim

**Root causes ที่พบบ่อย:**
- ลบ GA4 Client โดยไม่ตั้งใจ
- ใช้ Custom Client อื่นที่ intercept request ก่อน
- Server Container ยังเป็น version เก่าก่อนเพิ่ม GA4 Client

**Fix:**
Admin → Clients → New Client → เลือก "GA4" → Save → Preview → Publish

---

## Layer 3 — Meta CAPI Tag ไม่ Fire

**อาการ:** Client claim แล้ว แต่ Meta CAPI Tag ไม่ปรากฏใน Tags tab

### Checklist

**Trigger:**
- [ ] Meta CAPI Tag มี trigger ที่ถูกต้อง (ไม่ใช่ Unpublished หรือ None)
- [ ] ถ้าใช้ Custom Event trigger — ชื่อ event ตรงกับที่มาจาก dataLayer เป๊ะ
  - ผิด: trigger ชื่อ `Purchase` แต่ dataLayer push `purchase` (case sensitive)
- [ ] ถ้า fire ทุก event — ใช้ trigger "All Events"

**Tag Config:**
- [ ] Pixel ID field ไม่ว่าง — ไม่ใช่ placeholder เช่น `YOUR_PIXEL_ID`
- [ ] Access Token field ไม่ว่าง
- [ ] ไม่มี Test Event Code ติดอยู่ใน production (ทำให้ event ไปแสดงใน Test tab เท่านั้น)

**วิธีตรวจ:**
Server Preview → คลิก event → ดู "Tags Fired" vs "Tags Not Fired" → ถ้า Meta tag อยู่ใน "Not Fired" → ดู reason

**Root causes ที่พบบ่อย:**
- Event name case mismatch (`purchase` vs `Purchase`)
- Test Event Code ยังอยู่ใน tag → event ส่งไปแต่ไม่ขึ้นใน Events Manager หลัก
- Tag ยังอยู่ใน Draft ไม่ได้ publish

---

## Layer 4 — Meta CAPI Connection พัง

**อาการ:** Tag fire ใน Preview แต่ Meta Events Manager ไม่เห็น event

### Checklist

**Access Token:**
- [ ] Token ยังไม่หมดอายุ — ตรวจที่ Meta Events Manager → Settings → Conversions API
- [ ] Token ถูก generate จาก Pixel ที่ถูกต้อง
- [ ] Token มี permission `ads_management` หรือ `ads_read`

**Pixel ID:**
- [ ] Pixel ID ตรงกับ Pixel ใน Events Manager
- [ ] Pixel ยังไม่ถูก deactivate

**Network:**
- [ ] Server Container เข้าถึง `graph.facebook.com` ได้ (ตรวจ Firewall/GCP rules)
- [ ] ดู Server Preview → Tags tab → คลิก Meta tag → ดู Response code
  - `200` = ส่งถึงแล้ว แต่อาจยังไม่ขึ้น Events Manager ทันที (delay 5–20 นาที)
  - `4xx` = credentials ผิด
  - `5xx` = ปัญหาจาก Meta

**Root causes ที่พบบ่อย:**
- Token หมดอายุหลังผ่านไปหลายเดือน (Meta ให้ expiry token)
- Copy pixel ID ผิด (มีหลาย pixel ใน Business Manager)
- Test Event Code ยังอยู่ → event เข้าแค่ Test Events tab ไม่ใช่ Live

---

## Layer 5 — Deduplication พัง (Meta นับซ้ำ)

**อาการ:** Events Manager เห็น conversion 2 ครั้งต่อ 1 action จริง

### วิธีตรวจ

ใน Meta Events Manager → เลือก event → ดู column "Deduplicated" vs "Total"
- ถ้า Total > Deduplicated → dedup ทำงาน (OK)
- ถ้า Total = Deduplicated → dedup ไม่ทำงาน → ตรวจ `event_id`

### Checklist

**event_id:**
- [ ] dataLayer push มี `event_id` field
- [ ] `event_id` ไม่ hardcode (ต้อง generate ทุกครั้ง)
- [ ] `event_id` ส่งผ่าน GTM Web → Server → Meta CAPI ด้วยค่าเดียวกัน
- [ ] Browser Pixel (ถ้ามี) ส่ง `event_id` เดียวกัน

**ตรวจ event_id ใน Server Preview:**
Server Preview → คลิก event → tab "Event data" → หา field `event_id`
- ถ้ามี = ผ่านมาจาก web container แล้ว
- ถ้าไม่มี = web container ไม่ได้ส่ง `event_id` มา

**Template code ที่ถูกต้อง:**
```javascript
// สร้าง event_id ก่อน push
const eventId = "evt_" + Date.now() + "_" + Math.random().toString(36).substr(2,6);

window.dataLayer.push({
  event: "purchase",
  event_id: eventId,       // ← ค่านี้ต้องเหมือนกันทั้ง browser pixel และ server
  transaction_id: "TXN-001",
  value: 1490,
  currency: "THB"
});
```

**Root causes ที่พบบ่อย:**
- ไม่มี `event_id` ใน dataLayer เลย
- `event_id` hardcode เป็น string คงที่ เช่น `"purchase_event"`
- Browser pixel ใช้ random `event_id` ของตัวเอง แทนที่จะอ่านจาก dataLayer

---

## Layer 6 — Payload Data Quality (ค่าผิด / Revenue เพี้ยน)

**อาการ:** Event เข้าแต่ value ผิด / revenue รวมไม่ตรง / items แสดงผิด

### Checklist — Value & Currency

- [ ] `value` เป็น **number** ไม่ใช่ string
  - ผิด: `value: "1490"` หรือ `value: "1,490"`
  - ถูก: `value: 1490`
- [ ] `value` ไม่รวม shipping ถ้า Meta expect revenue สุทธิ (ขึ้นกับ business)
- [ ] `currency` เป็น ISO 4217 uppercase: `THB`, `USD`, `EUR`
- [ ] `value` ตรงกับ sum ของ `price × quantity` ใน items[]

**ตรวจ Math:**
```
value = Σ (item.price × item.quantity)
ตัวอย่าง: [{price: 500, quantity: 2}, {price: 490, quantity: 1}]
= (500×2) + (490×1) = 1490 ✓
```

### Checklist — items[] Array

- [ ] `items` เป็น **array** เสมอ — แม้จะมีสินค้าชิ้นเดียว
  - ผิด: `items: { item_id: "SKU-001" }`
  - ถูก: `items: [{ item_id: "SKU-001" }]`
- [ ] แต่ละ item มี `item_id`, `item_name`, `price` (number), `quantity` (integer)
- [ ] ไม่ใช้ `id` แทน `item_id` (GA4 spec ใช้ `item_id`)

### Checklist — Purchase Timing

- [ ] `purchase` event fire **หลัง** payment gateway confirm เท่านั้น
- [ ] ไม่ fire จาก client-side ก่อนรับ response จาก server

**Root causes ที่พบบ่อย:**
- Value ส่งเป็น string ทำให้ Meta/GA4 รับค่าเป็น 0 หรือ NaN
- items[] เป็น object แทน array ทำให้ mapping พัง
- Purchase fire ก่อน payment confirm → นับ false conversion

> **→ Hand off:** ถ้าต้องเจาะลึก data quality (normalize, reconcile GA4/Meta/backend, แก้ before/after) → ส่งต่อ `gtm-data-quality-emq`

---

## Layer 7 — Event Match Quality ต่ำ

**อาการ:** EMQ score < 6 ใน Meta Events Manager

### คะแนน EMQ มาจากอะไร

Meta ให้คะแนน 0–10 ตาม user signals ที่ส่งมาด้วย:

| Signal | น้ำหนัก | วิธีส่ง |
|---|---|---|
| Email (hashed) | สูงมาก | `em` field ใน user_data |
| Phone (hashed) | สูงมาก | `ph` field ใน user_data |
| fbp cookie | สูง | อ่านอัตโนมัติจาก cookie ถ้าใช้ stape.io |
| fbc cookie | สูง | อ่านอัตโนมัติจาก cookie ถ้าใช้ stape.io |
| IP address | ปานกลาง | stape.io ส่งอัตโนมัติ |
| User agent | ปานกลาง | stape.io ส่งอัตโนมัติ |
| external_id | ปานกลาง | user_id จาก database |

### Checklist

- [ ] stape.io template ตั้งค่า user data enrichment ถูกต้อง
- [ ] ถ้ามี email/phone จาก user (login, checkout) → ส่งผ่าน dataLayer เป็น hashed value
- [ ] ไม่ส่ง PII แบบ plain text — ต้อง hash ด้วย SHA-256 ก่อน
- [ ] fbp/fbc cookies ถูกส่งผ่านจาก web container ไปยัง server

**วิธีเพิ่ม email hashed ใน dataLayer:**
```javascript
// Hash ก่อนส่ง (ถ้า hash เองฝั่ง backend)
window.dataLayer.push({
  event: "purchase",
  user_data: {
    email_hashed: "a665a45920422f9d417e4867efdc4fb8...", // SHA-256 ของ email
    phone_hashed: "..."  // SHA-256 ของ phone (ไม่มีเครื่องหมาย +66)
  }
  // ... event parameters
});

// ถ้าใช้ stape.io template — มัน hash email/phone ให้อัตโนมัติ
// แค่ส่ง raw email มาก็พอ (stape จะจัดการ)
```

**Root causes ที่พบบ่อย:**
- ไม่ได้ส่ง user_data เลย → EMQ อยู่ที่ 4–5
- ส่งแค่ IP/UA (อัตโนมัติ) แต่ไม่มี email/phone
- fbp cookie ไม่ถูกส่งผ่านเพราะ Cookie variable ใน GTM ไม่ได้ตั้งค่า

> **→ Hand off:** EMQ เป็นเรื่องเชิงลึก (fbp/fbc/email/external_id, normalize, double hash, coverage > score) → ส่งต่อ `gtm-data-quality-emq`

---

## Output Format

ทุกครั้งที่ diagnose เสร็จ ให้ใช้ format นี้:

```
## 🔍 Diagnosis — [อาการที่รายงาน]

### Layer ที่พบปัญหา
[Layer N — ชื่อ Layer]

### ❌ Root Cause
[สาเหตุที่น่าจะเป็น พร้อมอธิบายว่าทำไมถึงเกิดอาการนี้]

### 🛠 Fix Steps
1. [ขั้นตอนที่ 1]
2. [ขั้นตอนที่ 2]
...

### ✅ วิธีตรวจว่าแก้สำเร็จ
- [วิธีตรวจ] → ควรเห็น [ผลที่ถูกต้อง]

### ⚠️ ระวัง
[ข้อควรระวังพิเศษถ้ามี — เช่น อย่า publish ก่อนทดสอบใน Preview]
```

---

## Quick Reference — อาการ → Layer

```
Server Preview ว่าง              → Layer 1
Client ไม่ claim                 → Layer 2
Tag ไม่ fire                     → Layer 3
Events Manager ไม่เห็น event     → Layer 4
Meta นับซ้ำ                      → Layer 5
Revenue เพี้ยน / ค่าผิด          → Layer 6
EMQ ต่ำ                          → Layer 7
Purchase ไม่เข้า (ไม่รู้สาเหตุ)  → ตรวจ Layer 1 → 2 → 3 → 4 ตามลำดับ
```
