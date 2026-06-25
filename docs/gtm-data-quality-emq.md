---
name: gtm-data-quality-emq
description: >
  ทำให้ data ที่ส่งเข้า GA4/Meta CAPI ถูกต้อง สม่ำเสมอ เชื่อถือได้ และเพิ่ม Event Match Quality (EMQ) ใช้ skill นี้เมื่อ: ต้องตรวจหรือแก้คุณภาพข้อมูล (value/currency/items/transaction_id ผิด type หรือ format), EMQ score ต่ำ, ต้องเข้าใจหรือแก้ identity signals (fbp, fbc, email hash, external_id), hash ผิด/double hash, test data ปน production, หรือ reconcile ตัวเลขระหว่าง GA4/Meta/backend. รับ hand-off จาก gtm-troubleshooter (Layer 6 payload / Layer 7 EMQ) และจาก gtm-reliability-monitoring (เมื่อ monitoring เจอ EMQ ตก/revenue เพี้ยน). Trigger แม้ user พิมพ์แค่ "EMQ ต่ำ", "fbc คืออะไร", "hash email ยังไง", "revenue เพี้ยน", "Meta match ไม่ได้", "value เป็น 0", "ตัวเลขไม่ตรงกัน", "test ปน production".
---

# GTM Data Quality + EMQ

## Overview

Skill นี้ครอบ 2 เสาของ trustworthy tracking:

- **Data Quality** — สิ่งที่ส่งเข้าระบบ ถูกต้อง/ครบ/สม่ำเสมอ/สะอาด ไหม (4 มิติ)
- **Event Match Quality** — Meta จับคู่ event กับ user จริงได้แม่นแค่ไหน (identity signals)

ทั้งคู่เกี่ยวกับ **"data เชื่อถือได้ไหม"** ไม่ใช่ "data มาถึงไหม" (นั่นคือ reliability)

## เมื่อไหร่ใช้ skill นี้

| สถานการณ์ | ใช้ skill นี้ |
|---|---|
| value เป็น 0 / string / revenue เพี้ยน | ✓ Data Quality |
| items[] ผิด format / field ผิดชื่อ | ✓ Data Quality |
| EMQ score ต่ำ | ✓ EMQ |
| ไม่เข้าใจ fbp/fbc/email hash/external_id | ✓ EMQ |
| hash ผิด / double hash | ✓ ทั้งคู่ |
| ตัวเลข GA4/Meta/backend ไม่ตรง | ✓ Reconciliation |
| event ไม่ fire / tag ไม่ทำงาน | ✗ ใช้ gtm-troubleshooter |
| cold start / ตั้ง alert | ✗ ใช้ gtm-reliability-monitoring |

## Hand-off — เข้า

Skill นี้รับงานต่อจาก:

- **จาก `gtm-troubleshooter`** — Layer 6 (payload data quality) หรือ Layer 7 (EMQ user data) เมื่อ diagnose แล้วพบว่าเป็นปัญหา data → hand off มาเจาะลึก + fix
- **จาก `gtm-reliability-monitoring`** — เมื่อ monitoring พบ EMQ ตก หรือ revenue per order ผิดปกติ → hand off มาที่นี่

เมื่อรับ hand-off: ใช้ context ที่ส่งมาต่อ ไม่ต้องถามซ้ำ

## Hand-off — ออก

- **ไปที่ `gtm-troubleshooter`** — ถ้าพบว่าจริงๆ แล้ว event ไม่ fire เลย (ไม่ใช่ data ผิด แต่ไม่มี data)
- **ไปที่ `gtm-reliability-monitoring`** — ถ้าต้องตั้ง monitoring เพื่อจับ data issue ในอนาคต

## Flow การทำงาน

```
1. ระบุว่าเป็นปัญหา Data Quality หรือ EMQ
   │
   ├─ Data Quality → ดู references/data-quality.md
   │     ตรวจ 4 มิติ: Correctness, Completeness, Consistency, Cleanliness
   │     fix: type, format, dedup, test contamination, reconcile
   │
   └─ EMQ → ดู references/emq-signals.md
         เข้าใจ 4 signals: fbp, fbc, email hash, external_id
         fix: normalize + hash ถูก, forward cookie, coverage > score
   │
2. ให้ before/after code จริง + วิธีตรวจ
3. ถ้าต้องตั้งระบบเฝ้าระวัง → hand off ไป gtm-reliability-monitoring
```

## หลักการสำคัญ (อย่าลืม)

1. **Coverage > Score** — EMQ 9 แต่จับ conversion ได้ 92% แย่กว่า EMQ 6 ที่จับได้ 99% เป้าหมายคือ EMQ 6+ พร้อม coverage สูง ไม่ใช่ไล่ score ให้สุด
2. **_fbc คือ signal แข็งที่สุด** — เป็น deterministic link กับ ad click จริง (จาก fbclid) ส่วน fbp/email/external_id เป็น probabilistic
3. **Normalize ก่อน hash เสมอ** — email: trim → lowercase → SHA-256 / phone: ตัวเลขล้วน → E.164 → SHA-256 ถ้า normalize ผิด hash ไม่ตรง = signal เสียเปล่า
4. **Double hash เป็น bug คลาสสิกของ stape.io** — ถ้าใช้ stape ส่ง raw ไป มัน hash ให้เอง ห้าม hash ซ้ำ
5. **value เป็น number เสมอ** — string ทำให้ Meta/GA4 อ่านเป็น 0 → revenue หาย → ROAS เพี้ยน → algorithm ลดงบโฆษณาที่เวิร์ก
6. **ตัวเลข 3 แหล่งไม่เท่ากันเป็นเรื่องปกติ** — GA4 ต่ำกว่า backend 5–15% ปกติ / Meta สูงกว่า backend = ผิดปกติเสมอ ต้องเข้าใจช่องว่างที่อธิบายได้

## Output Format

```
## 🔬 Data Quality / EMQ Review — [หัวข้อ]

### สถานะปัจจุบัน
[ค่าที่ส่งอยู่ / signal ที่มี / EMQ ปัจจุบัน]

### ❌ ปัญหา
- [field/signal]: [ผิดยังไง] → [ผลกระทบ]

### 🛠 Fix (before → after)
[code before ผิด → code after ถูก]

### ✅ วิธีตรวจว่าสำเร็จ
- [วิธีตรวจ] → ควรเห็น [ผลถูกต้อง]

### 📊 EMQ Coverage Note (ถ้าเกี่ยวกับ EMQ)
- signals ที่มี: [list] / ที่ขาด: [list]
- เป้าหมาย: EMQ 6+ พร้อม coverage สูง
```

## Reference Files

- `references/data-quality.md` — 4 มิติ, data types (value/currency/items/transaction_id), normalize+hash ทีละ step, double hash, test contamination 4 วิธีกัน, purchase timing, reconciliation benchmarks
- `references/emq-signals.md` — fbp/fbc/email hash/external_id เชิงลึก, ตารางเปรียบเทียบ, scenario Add to Cart → Purchase, EMQ score breakdown ต่อ signal

อ่าน reference เมื่อต้องการรายละเอียด — SKILL.md นี้พอสำหรับ route และตัดสินใจ

---

# 📎 Reference 1: Data Quality (เนื้อหาเต็ม)


## Mental Model

Reliability = event มาถึงไหม
Data Quality = สิ่งที่มาถึง เชื่อถือได้ไหม

ปัญหาของ Data Quality: ไม่มี error — ระบบทำงานปกติ event เข้าครบ แต่ตัวเลขใช้ตัดสินใจไม่ได้

---

## 4 มิติ

| มิติ | ตรวจอะไร | พังแบบ | ตัวอย่าง |
|---|---|---|---|
| Correctness | ค่าถูก type/format ไหม | เงียบ | `value: "1490"` → Meta อ่านเป็น 0 |
| Completeness | field จำเป็นครบไหม | เงียบ | ไม่มี event_id → นับซ้ำ |
| Consistency | event เดียวกันเหมือนกันทุกหน้าไหม | เงียบ | หน้า A `purchase`, หน้า B `Purchase` |
| Cleanliness | มีขยะปนไหม | มีร่องรอย | QA test purchase ปน production |

หลักการ: ระบบ reliable 100% แต่ data quality ต่ำ = เก็บขยะไว้ครบ เหมือนตาชั่งแม่นแต่ชั่งผิดหน่วย

---

## Data Types

### value — number เสมอ
```javascript
// ✗ ผิด
value: "1490"        // string
value: "1,490"       // มีจุลภาค
value: "฿1490"       // มีสัญลักษณ์

// ✓ ถูก
value: 1490          // number
value: parseFloat(orderTotal)  // ถ้ามาจาก string
value: Number(orderTotal)
```
ผลของ string: GA4 → 0/NaN (revenue ว่าง) | Meta → 0 (ROAS ต่ำ → algorithm ลดงบโฆษณาที่เวิร์ก)

### currency — ISO 4217 uppercase
```
✗ "thb" "Thb" "baht" "฿" undefined
✓ "THB" "USD" "EUR" "SGD" "MYR"
```
**Currency mismatch อันตราย:** Meta account = USD แต่ส่ง THB → 1490 ถูกตีเป็น $1,490 แทน ~$41 → ROAS เว่อ 35x → algorithm เพิ่มงบจนบาน

### items[] — array เสมอ + field names ถูก
```javascript
// ✗ ผิด
items: {            // object
  id: "SKU-001",    // ต้องเป็น item_id
  name: "Course",   // ต้องเป็น item_name
  price: "1490",    // string
  qty: 1            // ต้องเป็น quantity
}

// ✓ ถูก
items: [{           // array
  item_id: "SKU-001",
  item_name: "GTM Course",
  price: 1490,      // number
  quantity: 1       // integer
}]
```

| Field ถูก | Type | Required | ที่มักผิด |
|---|---|---|---|
| item_id | string | required | id, sku, product_id |
| item_name | string | required | name, title |
| price | number | required | string, มีจุลภาค |
| quantity | integer | required | qty, count, amount |
| item_category | string | optional | - |
| item_brand | string | optional | - |

### transaction_id — unique ต่อ order, มาจาก backend
```javascript
// ✗ patterns ที่ทำให้ซ้ำ
transaction_id: "order_1"   // hardcode
transaction_id: Date.now()  // generate client → refresh = ID ใหม่
transaction_id: undefined   // ไม่ส่ง

// ✓ ถูก
transaction_id: "ORD-2024-98765"  // prefix + order_id จาก DB
```
กรณีพบบ่อย: confirmation page reload → purchase fire ซ้ำ → ถ้า server-side dedup ไม่ทำงาน → นับ 2 ครั้ง แก้โดย flag ฝั่ง server ว่า transaction_id นี้ fire แล้ว

---

## Hashing

### ทำไมต้อง hash
Meta กำหนดให้ PII ผ่าน SHA-256 ก่อนส่ง เพื่อไม่ให้ข้อมูลส่วนตัวออกไป plain text

### Email — normalize 3 step
```
1. Trim whitespace:  "  user@example.com  " → "user@example.com"
2. Lowercase:        "User@Example.COM"     → "user@example.com"
3. SHA-256:          → "a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a27ae3"
```

### Phone — normalize 4 step
```
1. ลบทุกอย่างที่ไม่ใช่ตัวเลข:  "081-234-5678" → "0812345678"
2. ตัด leading zero:           "0812345678"   → "812345678"
3. เติม country code (ไม่มี +): TH → "66812345678"   (E.164)
4. SHA-256:                     → hash
```

### Double Hash — bug คลาสสิกของ stape.io
```javascript
// ✗ Double hash — hash เองแล้วส่งให้ stape hash อีก = hash(hash(email))
const emailHash = sha256(email);
dataLayer.push({ user_data: { email: emailHash } });  // stape hash ซ้ำ!

// ✓ ถ้าใช้ stape — ส่ง raw ไป มัน normalize + hash เอง
dataLayer.push({ user_data: { email: "user@example.com" } });
```

### Hash field reference
| Field | ต้อง hash | normalize |
|---|---|---|
| email (em) | ✓ | trim + lowercase |
| phone (ph) | ✓ | E.164 (ตัวเลขล้วน + country code) |
| first name (fn) | ✓ | lowercase |
| last name (ln) | ✓ | lowercase |
| fbp, fbc | ✗ | ส่ง raw |
| IP, user agent | ✗ | ส่ง raw |

---

## Test Data Contamination

### ตัวอย่างที่ปน
QA test purchase ทุกวัน, dev test จาก localhost, traffic IP office, bot/crawler, GTM Preview ปน production

### ผลกระทบ
Conversion rate สูงเกินจริง, revenue เพิ่มทุกครั้ง QA test, ROAS ดูดีเกินจริง, algorithm optimize ผิดทาง

### 4 วิธีกัน

**1. Filter IP office (GA4)** — GA4 Admin → Data Streams → Define Internal Traffic → เพิ่ม IP → Data Filters → สร้าง filter
- ⚠️ ใช้ไม่ได้ถ้า dynamic IP

**2. Cookie/flag b่งชี้ test traffic** — เหมาะกับ dynamic IP
```javascript
// GTM Custom HTML — fire เมื่อ URL มี ?internal=true
document.cookie = "is_internal=1; max-age=31536000; path=/";
// trigger: fire CAPI เฉพาะเมื่อ cookie is_internal != 1
```

**3. แยก staging environment** — GTM ตรวจ hostname → ถ้าไม่ใช่ production domain → ไม่ส่ง event บังคับ QA test บน staging

**4. Test Event Code (ชั่วคราว)** — Meta CAPI Tag → ใส่ Test Event Code ขณะ test → event เข้า "Test Events" tab เท่านั้น
- ⚠️ ต้องลบก่อน production เสมอ ถ้าลืม event ไม่ขึ้น Events Manager หลัก

### Dynamic IP — แนะนำ
วิธี 2 (cookie flag) + วิธี 3 (staging) ใช้คู่กัน: staging จัดการ QA, cookie จัดการกรณีดู production จริง (คลิก URL พิเศษ 1 ครั้ง, cookie อยู่ 1 ปี)

---

## Purchase Timing

```javascript
// ✗ fire ก่อน payment confirm — false conversion
payBtn.addEventListener('click', () => {
  dataLayer.push({ event: 'purchase' });  // ยังไม่รู้ว่าจ่ายได้!
});

// ✓ fire หลัง payment confirm — บนหน้า success เท่านั้น
// URL: /order/success?txn_id=ORD-98765
dataLayer.push({ event: 'purchase', transaction_id: txnIdFromURL });
```

---

## Reconciliation

### ทำไมตัวเลขต่างกัน (อธิบายได้)
| เปรียบเทียบ | ปกติไหม | เหตุผล |
|---|---|---|
| Backend 100 vs GA4 95 | ปกติ | browser limitations, ad blocker, JS error, ออกก่อน load เสร็จ |
| Meta สูงกว่า backend | ผิดปกติ | dedup ไม่ทำงาน, test ปน, attribution cross-device |

### Rule of thumb
- GA4 ต่ำกว่า backend 5–10% = ปกติ
- GA4 ต่ำกว่า backend > 15% = น่าตรวจ (tracking อาจพัง)
- Meta สูงกว่า backend = ผิดปกติเสมอ → ตรวจ dedup
- Meta สูงกว่า GA4 20–30% = ปกติ (Meta รวม view-through attribution)

### ตารางตรวจ (weekly)
| ดู | ที่ไหน | ปกติ | อันตราย |
|---|---|---|---|
| Purchase count | GA4 vs Backend | GA4 ต่ำกว่า 5–15% | ต่ำกว่า > 20% |
| Purchase count | Meta vs Backend | Meta สูงกว่า ≤30% | Meta สูงกว่า > 50% |
| Revenue | GA4 vs Backend | ต่าง <10% | ต่าง >20% หรือ = 0 |
| Dedup ratio | Meta Events Manager | Dedup < Total | Dedup = Total |
| EMQ | Meta Events Manager | ≥ 7.0 | ตกต่ำกว่า 6 ฉับพลัน |

---

## Checklist

**Correctness:** value=number / currency=ISO uppercase / items=array + field ถูก
**Completeness:** transaction_id จาก backend / event_id ทุก conversion
**Hashing:** email normalize ก่อน hash / phone E.164 / ไม่ double hash (stape)
**Cleanliness:** Test Event Code ลบก่อน prod / filter test traffic / purchase fire หลัง payment confirm
**Reconciliation:** ดู GA4 vs Backend weekly / ดู dedup ratio weekly

---

# 📎 Reference 2: EMQ Signals (เนื้อหาเต็ม)


## Mental Model

EMQ ไม่เกี่ยวกับว่า event มาถึงไหม (นั่นคือ reliability)
EMQ = เมื่อ event มาถึงแล้ว Meta ผูก event กับ user/ad click ที่เจาะจงได้มั่นใจแค่ไหน

**ทำไมสำคัญ:** เป็นเรื่อง attribution ถ้า match ไม่ได้ Meta ให้เครดิต conversion กับโฆษณาที่ขายได้จริงไม่ได้ → ROAS ดูแย่กว่าจริง → algorithm ลดงบโฆษณาที่กำลังเวิร์ก = เงินหายแบบมองไม่เห็น

EMQ = คะแนน 0–10 (Poor / OK / Good / Great)
- 6+ = ใช้ได้ (Meta benchmark ภายใน ~6)
- 8+ = ดีมาก
- 10 = หายากมาก

---

## ปัญหาที่ Meta ต้องแก้

สมมติเป็น Meta: มีคนคลิกโฆษณาแล้วไปซื้อของในเว็บอื่น "การซื้อนี้มาจากโฆษณาเราไหม?"
Meta ต้องการหลักฐานว่า "คนที่ซื้อ" = "คนที่คลิกโฆษณา"
4 signals คือหลักฐานคนละประเภท แข็งแรงต่างกัน

---

## 4 Signals

### _fbp — Facebook Browser ID
- **คำถามที่ตอบ:** ใครเยี่ยมชมเว็บ?
- Cookie ที่ Meta Pixel สร้างครั้งแรกที่เปิดเว็บ ระบุว่า "browser นี้เคยมา" ไม่เกี่ยวกับคลิกโฆษณา
- รูปแบบ: `fb.1.1685000000000.1234567890` (prefix.subdomain.timestamp.random)
- อายุ 90 วัน, ส่ง raw ไม่ต้อง hash
- EMQ weight: ปานกลาง-สูง

### _fbc — Facebook Click ID (แข็งที่สุด)
- **คำถามที่ตอบ:** คนนี้คลิกโฆษณาเราไหม?
- Cookie ที่เก็บ `fbclid` จาก URL ตอนคลิกโฆษณา = หลักฐาน **deterministic** ว่ามาจากโฆษณาชิ้นนั้น
- URL: `example.com?fbclid=AbCd...` → `_fbc = fb.1.1685000000000.AbCd...`
- fbclid หมดอายุใน URL ทันที แต่ _fbc cookie อยู่ 90 วัน, ส่ง raw
- EMQ weight: **สูงมาก** (แข็งที่สุดเพราะ deterministic)

### Email Hash — SHA-256
- **คำถามที่ตอบ:** คนนี้คือ Meta user คนไหน?
- Meta เอา email hash เทียบ database ตัวเองหา Facebook/Instagram account
- normalize: trim → lowercase → SHA-256 (ดู data-quality.md)
- EMQ weight: สูงมาก

### external_id — Your User ID
- **คำถามที่ตอบ:** ระบุตัวตนข้าม device/session
- User ID จาก database ของคุณ เชื่อม event จากหลาย session ของ user เดียวกัน โดยเฉพาะหลัง login
- ไม่ต้อง hash แต่ hash ได้ถ้าอยากปลอดภัยขึ้น
- EMQ weight: ปานกลาง

---

## ตารางเปรียบเทียบ

| Signal | สร้างโดย | เก็บเมื่อ | อายุ | EMQ weight | hash? |
|---|---|---|---|---|---|
| _fbp | Meta Pixel (auto) | เปิดเว็บครั้งแรก | 90 วัน | ปานกลาง | ไม่ |
| _fbc | Meta Pixel (auto) | คลิกโฆษณาเท่านั้น | 90 วัน | สูงมาก | ไม่ |
| Email | คุณ (checkout/login) | ลูกค้ากรอก email | ไม่หมดอายุ | สูงมาก | ✓ SHA-256 |
| Phone | คุณ (checkout/login) | ลูกค้ากรอกเบอร์ | ไม่หมดอายุ | สูงมาก | ✓ SHA-256 E.164 |
| external_id | คุณ (DB) | user login | ไม่หมดอายุ | ปานกลาง | optional |
| IP | server (auto) | ทุก request | - | ต่ำ | ไม่ |
| User agent | browser (auto) | ทุก request | - | ต่ำ | ไม่ |

ถ้าใช้ stape.io template — _fbp/_fbc อ่านและส่งให้อัตโนมัติ

---

## Scenario: Add to Cart → Purchase

**วันจันทร์ — คลิกโฆษณา**
URL: `example.com/product?fbclid=IwAR3xyz...`
Pixel สร้าง _fbp, เก็บ fbclid ใน _fbc
- signals: _fbp ✓ (ใหม่), _fbc ✓, email ✗

**วันจันทร์ — Add to Cart**
`add_to_cart` ส่งผ่าน CAPI พร้อม _fbp + _fbc → Meta รู้แล้วว่ามาจากโฆษณาชิ้นไหน
- signals: _fbp ✓, _fbc ✓ (แข็งสุด), email ✗

**วันพุธ — กลับมา Purchase (ไม่คลิกโฆษณาใหม่)**
กลับมาเองผ่าน bookmark — URL ไม่มี fbclid แต่ _fbp/_fbc ยังอยู่ใน cookie (90 วัน) + checkout มี email
- signals: _fbp ✓, _fbc ✓ (ยังอยู่), email ✓, external_id ✓ (หลัง login)

**ผลลัพธ์:** Meta รวม signals → ระบุได้ว่า purchase มาจากโฆษณาวันจันทร์ → attribution ถูก

นี่คือเหตุผลที่ forward _fbc ผ่านไป server สำคัญมาก — แม้ purchase เกิดวันที่ user ไม่ได้คลิกโฆษณา cookie ที่เก็บไว้ยังผูก attribution ได้

---

## EMQ Score ตาม signals

| signals ที่มี | EMQ ประมาณ |
|---|---|
| ไม่มี signals | ~1–2 |
| IP + UA เท่านั้น | ~3–4 |
| + _fbp | ~5–6 |
| + _fbc | ~7 |
| + email/phone | ~8–9 |
| ครบทุก signal | ~9–10 |

---

## ข้อคิดสำคัญ — Coverage > Score

**EMQ สูงอาจหลอกตา:** EMQ 9/10 แต่จับ conversion ได้ 92% มีปัญหาใหญ่กว่า EMQ 6 ที่จับได้ 99%

- EMQ = minimum-viable-signal metric ไม่ใช่ performance lever ที่ต้องหมกมุ่น
- event coverage (จับ conversion กี่%) สำคัญกว่าไล่ EMQ ให้สูงสุด
- เป้าหมายถูก: EMQ 6+ พร้อม coverage สูง
- Server-side ได้เปรียบ EMQ เพราะส่ง email/phone hashed จาก backend ได้ ซึ่ง browser pixel ทำไม่ได้

Business impact: Meta ระบุว่า advertiser ที่ปรับ EMQ เห็น CPA ดีขึ้น 15–25%

---

## EMQ ต่ำ — เช็คอะไร

1. **ไม่ forward _fbc/_fbp ไป server** — web container ไม่ส่ง cookie ผ่าน หรือ server tag ไม่อ่าน → ทิ้ง signal ดีสุดฟรี (พบบ่อยมาก)
2. **hash ผิด** — email ไม่ lowercase/trim, phone ไม่ใช่ E.164 → hash ไม่ตรง Meta = signal เสียเปล่า
3. **double hashing** — ส่ง hashed value เข้า stape ที่ hash ซ้ำ
4. **ส่ง user_data ไม่สม่ำเสมอ** — บาง event มี email บาง event ไม่มี → identity แตก
5. **ส่งแค่ IP/UA** (อัตโนมัติ) ไม่มี email/phone → EMQ ค้างที่ 4–5

### วิธีเพิ่ม email ใน dataLayer
```javascript
// ถ้า hash เองฝั่ง backend
dataLayer.push({
  event: "purchase",
  user_data: {
    email_hashed: "a665a459...",  // SHA-256
    phone_hashed: "..."           // SHA-256 E.164
  }
});

// ถ้าใช้ stape — ส่ง raw มัน hash ให้
dataLayer.push({
  event: "purchase",
  user_data: { email: "user@example.com" }  // raw
});
```

---

## วัด EMQ ที่ไหน

Meta Events Manager → เลือก Pixel → เลือก event → Match Quality → ดูคะแนนรวม + signal breakdown ว่าเก็บ parameter อะไรบ้าง (อาจอยู่ Overview หรือ Diagnostics tab ขึ้นกับ version UI)

EMQ เสื่อมแบบเงียบสนิท — ไม่มี error, event มาครบ, ค่าถูก แต่ attribution แอบแย่ลง รู้ได้ต่อเมื่อเปิดดูคะแนนเอง → ควร monitor (ดู gtm-reliability-monitoring)
