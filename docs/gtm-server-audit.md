# GTM Server Audit + CAPI Payload Debug

## Overview

ใช้ skill นี้เมื่อต้องการตรวจสอบสิ่งที่มีอยู่แล้ว — ไม่ว่าจะเป็น GTM container config, event payload ที่ส่งออกไป, หรือตรวจความพร้อมก่อนขึ้น production เป้าหมายคือหาว่ามีอะไรผิด ขาด หรือควรแก้ไข

## Hand-off — ส่งงานต่อหลัง audit เสร็จ

Skill นี้ตรวจว่า "อะไรผิด/ขาด" แต่บางเรื่องต้องลงมือตั้งค่าหรือเจาะลึกต่อ เมื่อ audit เสร็จแล้ว → hand off ไป skill เฉพาะทาง:

- **ไปที่ `gtm-reliability-monitoring`** — หลังทำ Mode C (Production QA) เสร็จ → ตั้ง monitoring/alert ก่อน go-live และตรวจ Cloud Run config (min instances, timeout) ที่ audit นี้ไม่ครอบ
- **ไปที่ `gtm-data-quality-emq`** — เมื่อ Mode B (CAPI Payload Debug) พบปัญหา value/currency/items/hash ที่ต้องแก้เชิงลึก หรือ EMQ readiness ต่ำ → เจาะลึก normalize/hash + before/after code
- **ไปที่ `gtm-troubleshooter`** — ถ้า audit พบว่า event ไม่ fire เลย (ไม่ใช่แค่ config ไม่ครบ) → diagnose ก่อน


## Trigger Conditions

ใช้ skill นี้เมื่อ user:
- Upload หรือ paste GTM container JSON export
- บอกว่า "ตรวจ GTM", "audit config", "setup ถูกไหม", "check container"
- Paste dataLayer object หรือ CAPI event payload แล้วถามว่าถูกไหม
- บอกว่า "debug CAPI", "ตรวจ payload", "event ถูกไหม", "dedup ถูกไหม"
- ถาม Server Preview แล้วเห็น tag ไม่ fire หรือ event ไม่ถึง Meta
- บอกว่า "พร้อม launch ไหม", "QA ก่อน deploy", "production ready ยัง"

---

## Mode A — GTM Config Audit

### Input ที่รับได้
- GTM container export JSON (web หรือ server)
- คำอธิบาย setup ปัจจุบัน
- Screenshot จาก GTM Preview

### Checklist — Web Container

**GA4 Configuration Tag:**
- [ ] มี tag ประเภท Google Tag หรือ GA4 Configuration
- [ ] `transport_url` ชี้ไปที่ server URL ของตัวเอง ไม่ใช่ `www.google-analytics.com`
- [ ] `server_container_url` ชี้ไปที่ server URL เดียวกับ `transport_url`
- [ ] Trigger เป็น `Initialization - All Pages`

**GA4 Event Tags:**
- [ ] มี Event Tag แยกสำหรับแต่ละ conversion event
- [ ] แต่ละ Event Tag มี Trigger เป็น Custom Event ที่ตรงกับชื่อ event ใน dataLayer
- [ ] Measurement ID ถูกต้อง ไม่ใช่ placeholder

**Consent (ถ้า setup มี Consent Mode):**
- [ ] มี Consent Initialization trigger ที่ตั้ง default = denied ก่อน tag อื่นโหลด
- [ ] CMP fire บน Consent Initialization ไม่ใช่ All Pages
- [ ] consent state ถูกส่งไป server (ผ่าน gcs parameter)

**Security:**
- [ ] ไม่มี access token hardcode ใน tag
- [ ] ไม่มี pixel ID ใน plain text field ที่ไม่ควรอยู่

### Checklist — Server Container

**GA4 Client:**
- [ ] มี GA4 Client อยู่
- [ ] Status เป็น "Claimed" เมื่อมี request เข้ามา

**Meta CAPI Tag:**
- [ ] มี Facebook Conversion API tag (แนะนำ stape.io template)
- [ ] Pixel ID field ไม่ว่าง
- [ ] API Access Token field ไม่ว่าง
- [ ] Trigger ถูกต้อง
- [ ] Test ID ลบออกแล้วก่อน deploy production
- [ ] Tag Execution Consent Settings ตั้งถูกต้องตามนโยบาย privacy

**EMQ readiness:**
- [ ] มีการส่ง user_data (email/phone hashed) ถ้ามีข้อมูล
- [ ] Automap User Data เปิดอยู่ (stape.io hash ให้อัตโนมัติ)

### Output format

```
## GTM Config Audit — [ชื่อ Container]

### ✅ ผ่าน
- [รายการ]: [เหตุผล]

### ⚠️ ควรแก้
- [รายการ]: [ปัญหา] → [วิธีแก้]

### ❌ Critical — ต้องแก้ก่อน production
- [รายการ]: [ปัญหา] → [วิธีแก้]

### คำแนะนำเพิ่มเติม
- [สิ่งที่ทำได้เพื่อปรับปรุง]
```

---

## Mode B — CAPI Payload Debug

### Input ที่รับได้
- dataLayer.push() object
- CAPI event payload (JSON)
- GTM Preview screenshot แสดง Event data tab

### Checklist — Basic Validation

- [ ] `event` เป็น lowercase underscore (เช่น `purchase` ไม่ใช่ `Purchase`)
- [ ] `value` เป็น number ไม่ใช่ string
- [ ] `currency` เป็น ISO 4217
- [ ] `items` เป็น array ไม่ใช่ object
- [ ] `transaction_id` มีและ unique (purchase)
- [ ] `value` ตรงกับ sum ของ price × quantity ใน items[]

### Checklist — Deduplication

- [ ] `event_id` มีอยู่ใน dataLayer push
- [ ] `event_id` เป็น unique string ต่อ event instance (ไม่ hardcode)
- [ ] `event_id` เดียวกันถูกส่งทั้งจาก browser pixel และ server CAPI
- [ ] `event_name` ตรงกันทั้ง 2 ทาง (case-sensitive)

### Checklist — items[]

- [ ] `item_id` มี
- [ ] `item_name` มี
- [ ] `price` เป็นตัวเลข
- [ ] `quantity` เป็น integer

### Checklist — User Data (EMQ)

- [ ] ไม่มี PII plain text
- [ ] stape.io template hash ให้อัตโนมัติ
- [ ] ส่ง user_data เพียงพอ (email/phone แรงสุด)

### Common Errors

| ปัญหา | สาเหตุ | วิธีแก้ |
|---|---|---|
| Meta นับ 2 ครั้ง | event_id ไม่มีหรือต่างกัน | generate event_id ก่อน push |
| value = 0 | ส่งเป็น string | ลบ quotes |
| items[] ไม่ map | ใช้ `id` แทน `item_id` | แก้ key ให้ถูก |
| CAPI ไม่ fire | Trigger ไม่ตรง | ตรวจชื่อ event |
| EMQ ต่ำ | ไม่มี user_data | เพิ่ม email/phone hashed |
| เข้า Test Events ไม่เข้า production | Test ID ติดอยู่ | ลบ Test ID |

### Output format

```
## CAPI Payload Debug — [event_name]

### ✅ ถูกต้อง
- [field]: [value] — [เหตุผล]

### ⚠️ ควรแก้
- [field]: [ปัญหา] → [วิธีแก้]

### ❌ Error
- [field]: [ปัญหา] → [วิธีแก้]

### Deduplication Status
- event_id: [มี / ไม่มี]
- transaction_id: [มี / ไม่มี]
- ความเสี่ยง dedup: [ปลอดภัย / เสี่ยงนับซ้ำ]
```

---

## Mode C — Production QA Checklist

### Trigger
User ถามว่าพร้อม launch ยัง, ขอ QA ก่อน deploy, หรือต้องการตรวจความพร้อม production

### ① Before Deploy
- [ ] ลบ Test ID / test_event_code ออกจาก Meta CAPI tag
- [ ] เปลี่ยน Measurement ID จาก placeholder เป็นค่าจริง
- [ ] transport_url ชี้ไป production server ไม่ใช่ test
- [ ] Access Token เป็น System User Token (long-lived) ไม่ใช่ short-lived
- [ ] ทดสอบทุก event ใน Preview Mode ครบ
- [ ] Web + Server Container published แล้ว

### ② Data Quality
- [ ] value เป็น number ทุก event
- [ ] currency เป็น ISO 4217
- [ ] event_id มีและ unique ทุก purchase
- [ ] transaction_id ไม่ซ้ำ มาจาก database จริง
- [ ] items[] เป็น array เสมอ
- [ ] ไม่มี PII plain text

### ③ Deduplication
- [ ] browser pixel กับ server ใช้ event_id เดียวกัน
- [ ] event_name ตรงกันทั้ง 2 ทาง
- [ ] ตรวจ Meta Events Manager → Deduplication tab

### ④ Monitoring หลัง go-live
- [ ] ตั้ง alert เมื่อ event volume ตกผิดปกติ
- [ ] เช็ค Meta Events Manager ทุกสัปดาห์ (EMQ, coverage, errors)
- [ ] ดู GCP Cloud Run logs หา error spike (4xx/5xx)
- [ ] เตือนล่วงหน้าก่อน access token หมดอายุ
- [ ] ตรวจ Meta Diagnostics tab หา warning

> **→ Hand off:** ข้อ ④ ทั้งหมดเป็นการตั้ง monitoring เชิงระบบ → ส่งต่อ `gtm-reliability-monitoring` เพื่อตั้ง alert ทีละ step (P0/P1/P2), Cloud Run config, weekly routine

### Error Handling Note

Option **Use Optimistic Scenario** ใน Meta CAPI tag:
- เปิด = รายงานสำเร็จทันทีไม่รอ Meta (เร็ว แต่ไม่รู้ถ้า fail)
- ปิด (แนะนำ production) = รอ response, fail เรียก gtmOnFailure() → monitoring จับได้

### Response Code Reference

| Code | ความหมาย | จัดการ |
|---|---|---|
| 200 | สำเร็จ | ปกติ (delay 5–20 นาที) |
| 4xx | credentials/payload ผิด | ตรวจ token, pixel ID, payload |
| 5xx | ปัญหาฝั่ง Meta | retry / รอ |

### Output format

```
## Production QA — [โปรเจค]

### ✅ พร้อม launch
- [item ที่ผ่าน]

### ❌ ต้องแก้ก่อน launch
- [item]: [ปัญหา] → [วิธีแก้]

### ⚠️ แนะนำตั้งก่อน/หลัง go-live
- [monitoring item]
```
