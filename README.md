# GTM Server-Side Tracking + Meta CAPI Demo

> A complete implementation of server-side tracking with Meta Conversions API — built as a learning portfolio and reusable template.
> **ภาษาไทยอยู่ด้านล่าง / Thai version below** ↓

---

## Case Study

**Problem**
Browser-side pixel tracking loses an estimated 20–40% of conversion events (industry benchmark) due to ad blockers, Safari ITP, and iOS privacy restrictions. When Meta receives incomplete data, it optimizes ad campaigns on a partial picture — leading to inefficient ad spend and inaccurate attribution.

**Solution**
Implemented GTM Server-Side Tracking with Meta Conversions API on GCP Cloud Run. Events flow browser → first-party server → Meta server-to-server, with `event_id`-based deduplication to prevent double-counting when both browser pixel and server CAPI are used.

**Result (verified in this project)**
- Purchase events confirmed reaching Meta — verified as **Processed** in Meta Events Manager Test Events, source = Server
- End-to-end pipeline working: dataLayer → Web Container → Server Container → Meta CAPI
- Deduplication implemented via `event_id` generated before dataLayer push

**Expected Business Impact** *(based on industry benchmarks, not measured in this demo)*
- Server-side tracking typically recovers 20–40% of events lost to browser-side blocking
- Improved Event Match Quality (EMQ) from sending hashed first-party data server-side can reduce CPA by 15–25% (per Meta's published figures)
- First-party cookies extend lifetime from ~7 days (Safari ITP) to up to 400 days, improving returning-user attribution

---

## Architecture

```
User's Browser
    │  dataLayer.push({ event: 'purchase', event_id: '...', ... })
    ▼
GTM Web Container
    │  HTTP POST → transport_url (your first-party server, not Google/Meta directly)
    ▼
GTM Server Container (GCP Cloud Run)
    │
    ├──► GA4 Client → parses incoming request into event data
    │
    └──► Meta CAPI Tag → POST graph.facebook.com/events (server-to-server)
```

---

## Before / After

| Aspect | Browser-side (before) | Server-side (after) |
|---|---|---|
| Ad blockers | Block 20–40% of events | Server-to-server, not blocked |
| Safari ITP cookie life | ~7 days | Up to 400 days (first-party) |
| iOS 14+ restrictions | Limited data | Hashed user_data sent server-side |
| JavaScript errors | Pixel may not fire | Server receives if request arrives |

*Percentages are industry benchmarks, not measured in this specific project.*

---

## Cost Estimate (GCP Cloud Run)

| Traffic level | Estimated monthly cost |
|---|---|
| Dev / test | ~$0 (free tier: 2M requests/month) |
| Small site (<100k visits/mo) | ~$5 |
| Medium site | ~$10–20 |

---

## Repo Structure

```
gtm-server-side-capi-demo/
├── README.md
├── gtm-config/
│   ├── web-container.json
│   └── server-container.json
├── test-page/
│   └── gtm-test-page.html
├── tracking-spec/
│   └── tracking-spec-template.xlsx
└── docs/
    └── qa-checklist.md
```

---

## Quick Start

### Step 1 — Replace placeholders

| Placeholder | Replace with |
|---|---|
| `YOUR_GTM_ACCOUNT_ID` | Your GTM Account ID |
| `YOUR_GTM_WEB_CONTAINER_ID` | e.g. `GTM-XXXXXXX` |
| `YOUR_GTM_SERVER_CONTAINER_ID` | e.g. `GTM-XXXXXXX` |
| `YOUR_SERVER_URL` | e.g. `https://xxxxx-uc.a.run.app` |
| `YOUR_GA4_MEASUREMENT_ID` | e.g. `G-XXXXXXXXXX` |
| `YOUR_META_PIXEL_ID` | e.g. `1234567890123456` |
| `YOUR_META_ACCESS_TOKEN` | Never commit the real value |

### Step 2 — Deploy Server Container on GCP
1. [tagmanager.google.com](https://tagmanager.google.com) → Create container → Target: **Server**
2. Choose **Automatically provision tagging server** → select GCP project
3. Wait ~3–5 min → get your server URL

### Step 3 — Import GTM containers
GTM → Admin → Import Container → select the JSON files in `gtm-config/`

### Step 4 — Test locally
1. VS Code + Live Server → open `test-page/gtm-test-page.html`
2. Enable Preview on both Web and Server containers
3. Click any button → verify Server Preview shows **Meta CAPI → Fired**
4. Check Meta Events Manager → Test Events → **Purchase → Processed**

---

## Key Concepts

### Transport URL
Points GTM Web Container to your own server instead of Google/Meta directly. Browser sees a first-party request → ad blockers don't block it.

### Event Match Quality (EMQ)
Meta's 0–10 score for how well it matches your events to real users. Higher EMQ = better targeting. Server-side wins here because it can send hashed first-party data (email, phone) from the backend. **Note:** EMQ is a minimum-viable-signal metric — event coverage (% of conversions captured) matters more than chasing a perfect score. Target EMQ 6+ with high coverage.

### Deduplication
Both browser pixel and server CAPI send the same event. Without dedup, Meta counts twice. The fix: identical `event_id` in both. Meta sees matching IDs → counts once. This project generates `event_id` before dataLayer push.

### Consent Mode v2
Server-side does NOT make consent optional — it makes it more important, since the server sends data reliably. Consent state flows browser → server via the `gcs` parameter; server forwards events only when consent is granted. Advanced Mode sends cookieless pings for modeling when consent is denied. Required for EEA advertisers since March 2024 (needs a Google-certified CMP in production).

---

## Tech Stack

| Layer | Technology |
|---|---|
| Tag Management | Google Tag Manager (Web + Server) |
| Infrastructure | GCP Cloud Run |
| Destinations | Meta Conversions API, GA4 |
| Test Environment | HTML + JS, VS Code Live Server |

## Deployment Options

GTM Server Container is a Docker image — runs anywhere via the `CONTAINER_CONFIG` env variable: GCP Cloud Run, AWS App Runner/ECS, Azure Container Apps, or any VPS with Docker.

## References
- Meta CAPI Tag template by [stape.io](https://stape.io)
- GTM Server-Side docs by [Simo Ahava](https://simoahava.com)

---
---

# GTM Server-Side Tracking + Meta CAPI (ภาษาไทย)

โปรเจคนี้คือการ implement server-side tracking ร่วมกับ Meta Conversions API สร้างเป็น learning portfolio และ template ใช้ซ้ำได้

## Case Study

**ปัญหา**
Browser-side pixel เสีย conversion event ประมาณ 20–40% (industry benchmark) เพราะ ad blocker, Safari ITP และข้อจำกัด iOS เมื่อ Meta ได้ข้อมูลไม่ครบ มันจะ optimize campaign บนภาพที่ไม่สมบูรณ์ → budget เสียเปล่า attribution ผิด

**วิธีแก้**
Implement GTM Server-Side + Meta CAPI บน GCP Cloud Run ส่ง event แบบ browser → first-party server → Meta server-to-server พร้อม deduplication ด้วย event_id กันนับซ้ำ

**ผลลัพธ์ (ทดสอบจริงในโปรเจคนี้)**
- ยืนยัน Purchase event ถึง Meta — แสดงเป็น Processed ใน Meta Events Manager Test Events, source = Server
- pipeline ทำงานครบ: dataLayer → Web Container → Server Container → Meta CAPI
- implement deduplication ด้วย event_id ที่ generate ก่อน push dataLayer

**Business Impact ที่คาดหวัง** *(อ้างอิง industry benchmark — ยังไม่ได้วัดจริงในโปรเจคนี้)*
- server-side tracking โดยทั่วไปกู้คืน event ที่หายไป 20–40%
- EMQ ที่ดีขึ้นจากการส่ง hashed first-party data ช่วยลด CPA ได้ 15–25% (ตามตัวเลขที่ Meta เผยแพร่)
- first-party cookie อายุยาวขึ้นจาก ~7 วัน เป็นสูงสุด 400 วัน

## Architecture

```
Browser ของ User
    │  dataLayer.push({ event: 'purchase', event_id: '...', ... })
    ▼
GTM Web Container
    │  HTTP POST → transport_url (server ของคุณเอง ไม่ใช่ Google/Meta โดยตรง)
    ▼
GTM Server Container (GCP Cloud Run)
    │
    ├──► GA4 Client → แปลง request เป็น event data
    │
    └──► Meta CAPI Tag → POST graph.facebook.com/events (server-to-server)
```

## Before / After

| ด้าน | Browser-side (เดิม) | Server-side (ใหม่) |
|---|---|---|
| Ad blocker | บล็อก 20–40% | server-to-server ไม่โดนบล็อก |
| Safari ITP cookie | ~7 วัน | สูงสุด 400 วัน (first-party) |
| iOS 14+ | ข้อมูลจำกัด | ส่ง hashed user_data ผ่าน server |
| JavaScript error | pixel อาจไม่ fire | server รับได้ถ้า request ถึง |

*ตัวเลขเป็น industry benchmark ไม่ได้วัดจากโปรเจคนี้โดยตรง*

## ค่าใช้จ่าย (GCP Cloud Run)

| ระดับ traffic | ค่าใช้จ่าย/เดือน |
|---|---|
| Dev / test | ~$0 (free tier 2M requests) |
| เว็บเล็ก (<100k visits) | ~$5 |
| เว็บกลาง | ~$10–20 |

## Concept สำคัญ

### Transport URL
ชี้ GTM Web Container ไปที่ server ของคุณแทน Google/Meta โดยตรง browser เห็นเป็น first-party request → ad blocker ไม่บล็อก

### Event Match Quality (EMQ)
คะแนน 0–10 ที่ Meta ให้ว่าจับคู่ event กับ user จริงได้ดีแค่ไหน EMQ สูง = targeting ดีขึ้น server-side ได้เปรียบเพราะส่ง hashed first-party data จาก backend ได้ **หมายเหตุ:** EMQ เป็น minimum-viable-signal — event coverage (จับ conversion ได้กี่ %) สำคัญกว่าการไล่คะแนนสูง เป้าหมาย EMQ 6+ พร้อม coverage สูง

### Deduplication
browser pixel กับ server CAPI ส่ง event เดียวกัน ถ้าไม่ทำ dedup Meta นับ 2 ครั้ง แก้โดยใส่ event_id เดียวกันทั้ง 2 ทาง Meta เห็น ID ตรงกัน → นับครั้งเดียว โปรเจคนี้ generate event_id ก่อน push dataLayer

### Consent Mode v2
server-side ไม่ได้ทำให้ consent ไม่จำเป็น — กลับสำคัญกว่าเดิม เพราะ server ส่งข้อมูลแน่นอน consent ไหลจาก browser → server ผ่าน gcs parameter server forward เฉพาะตอน consent granted บังคับใช้สำหรับ advertiser EEA ตั้งแต่มีนาคม 2024 (production ต้องใช้ Google-certified CMP)

## References
- Meta CAPI Tag template by [stape.io](https://stape.io)
- GTM Server-Side docs by [Simo Ahava](https://simoahava.com)

---
Built by [Khanet-dew (Devver)](https://github.com/Khanet-dew)
