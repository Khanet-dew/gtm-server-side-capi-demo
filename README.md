# GTM Server-Side Tracking + Meta CAPI Demo

> A complete implementation of server-side tracking with Meta Conversions API — built as a learning portfolio and reusable template. **ภาษาไทยอยู่ด้านล่าง / Thai version below** ↓

---

## Case Study

**Problem** Browser-side pixel tracking loses an estimated 20–40% of conversion events (industry benchmark) due to ad blockers, Safari ITP, and iOS privacy restrictions. When Meta receives incomplete data, it optimizes ad campaigns on a partial picture — leading to inefficient ad spend and inaccurate attribution.

**Solution** Implemented GTM Server-Side Tracking with Meta Conversions API on GCP Cloud Run. Events flow browser → first-party server → Meta server-to-server, with `event_id`-based deduplication to prevent double-counting when both browser pixel and server CAPI are used.

**Result (verified in this project)**

- Purchase events confirmed reaching Meta — verified as **Processed** in Meta Events Manager Test Events, source = Server
- End-to-end pipeline working: dataLayer → Web Container → Server Container → Meta CAPI
- Deduplication implemented via `event_id` generated before dataLayer push
- Internal traffic filtering implemented via cookie flag (handles dynamic IPs)

**Expected Business Impact** *(based on industry benchmarks, not measured in this demo)*

- Server-side tracking typically recovers 20–40% of events lost to browser-side blocking
- Improved Event Match Quality (EMQ) from sending hashed first-party data server-side can reduce CPA by 15–25% (per Meta's published figures)
- First-party cookies extend lifetime from ~7 days (Safari ITP) to up to 400 days, improving returning-user attribution

---

## Architecture

```
User's Browser
    │  dataLayer.push({ event: 'purchase', event_id: '...', value: 1490, currency: 'THB', ... })
    ▼
GTM Web Container (browser)
    │  Checks exception trigger → skips if is_internal cookie = 1
    │  HTTP POST → transport_url (your first-party server, not Google/Meta directly)
    ▼
GTM Server Container (GCP Cloud Run)
    │
    ├──► GA4 Client → parses incoming request into event data object
    │
    ├──► GA4 Tag → www.google-analytics.com/mp/collect
    │
    └──► Meta CAPI Tag (stape.io) → graph.facebook.com/events
              │  Enriches with: fbp, fbc, hashed email, IP, user agent
              └──► Meta deduplication via event_id → counts once
```

---

## Before / After

| Aspect | Browser-side (before) | Server-side (after) |
|---|---|---|
| Ad blockers | Block 20–40% of events | Server-to-server, not blocked |
| Safari ITP cookie life | ~7 days | Up to 400 days (first-party) |
| iOS 14+ restrictions | Limited data | Hashed user_data sent server-side |
| JavaScript errors | Pixel may not fire | Server receives if request arrives |
| Internal traffic filtering | Static IP only (breaks with dynamic IP) | Cookie flag — works with any IP |
| EMQ signals | fbp/fbc only (browser-limited) | + hashed email/phone from backend |

*Percentages are industry benchmarks, not measured in this specific project.*

---

## Cost Estimate (GCP Cloud Run)

| Traffic level | Estimated monthly cost |
|---|---|
| Dev / test | ~$0 (free tier: 2M requests/month) |
| Small site (<100k visits/mo) | ~$5 |
| Medium site | ~$10–20 |

> **Note:** Min instances = 0 is fine for dev/test. Set min instances = 1 (~$5–8/month) only when going to production to prevent cold start event loss.

---

## Repo Structure

```
gtm-server-side-capi-demo/
├── README.md
├── gtm-config/
│   ├── GTM-web.json          ← Web container export (placeholders only)
│   └── GTM-server.json       ← Server container export (placeholders only)
├── test-page/
│   └── gtm-test-page.html    ← Local test page with all dataLayer events
├── docs/
│   ├── qa-checklist.md           ← Production QA checklist
│   ├── gtm-troubleshooter.md     ← 7-layer diagnosis guide
│   ├── gtm-server-audit.md       ← Config audit + CAPI payload debug
│   ├── gtm-reliability-monitoring.md  ← Cloud Run config + alert setup
│   └── gtm-data-quality-emq.md   ← Data quality + identity signals
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

### Step 5 — Verify internal traffic filter

Open `your-site.com?internal=true` once → GTM sets `is_internal=1` cookie → all subsequent events from that browser are blocked from firing. Check DevTools → Application → Cookies to verify.

---

## Key Concepts

### Transport URL

Points GTM Web Container to your own server instead of Google/Meta directly. Browser sees a first-party request → ad blockers don't block it.

Set in GA4 Configuration Tag: `transport_url = https://your-server-url.run.app`

### Event Match Quality (EMQ)

Meta's 0–10 score for how well it matches your events to real users. Server-side wins because it can send hashed first-party data (email, phone) from the backend.

| Signals present | EMQ estimate |
|---|---|
| IP + User Agent only | ~3–4 |
| + fbp cookie | ~5–6 |
| + fbc cookie (from ad click) | ~7 |
| + hashed email/phone | ~8–9 |

> **Coverage > Score:** EMQ 6 with 99% event coverage beats EMQ 9 with 92% coverage. Target EMQ 6+ with high coverage, not a perfect score.

### Deduplication

Both browser pixel and server CAPI send the same event. Without dedup, Meta counts twice. The fix: identical `event_id` in both paths.

```javascript
// Generate BEFORE dataLayer.push — use the same value for browser pixel AND server
const eventId = "evt_" + Date.now() + "_" + Math.random().toString(36).substr(2, 6);

window.dataLayer.push({
  event: "purchase",
  event_id: eventId,   // ← same value sent via browser pixel fbq() call
  transaction_id: "ORD-98765",
  value: 1490,         // number, not string
  currency: "THB"
});
```

### Internal Traffic Filtering (Cookie-Based)

Static IP filters break with dynamic IPs (common in Thailand and most home/office connections). This project uses a cookie flag instead:

```javascript
// GTM Custom HTML Tag — fires when URL contains ?internal=true
document.cookie = "is_internal=1; max-age=31536000; path=/; SameSite=Lax";
// Cookie lasts 1 year — click the URL once per browser, done
```

All conversion tags have an exception trigger: do not fire when `cookie - is_internal = 1`.

### Consent Mode v2

Server-side does NOT make consent optional — it makes it more important, since the server sends data reliably. Consent state flows browser → server via the `gcs` parameter. Required for EEA advertisers since March 2024.

---

## Troubleshooting Quick Reference

| Symptom | Layer | First check |
|---|---|---|
| Server Preview empty | Layer 1 | `transport_url` in GA4 Config Tag |
| Client not claiming request | Layer 2 | GA4 Client exists in server container |
| Meta tag not firing | Layer 3 | Trigger event name (case-sensitive) |
| Events Manager shows nothing | Layer 4 | Access token not expired, no Test Event Code |
| Meta counting double | Layer 5 | `event_id` present and identical both sides |
| Revenue = 0 | Layer 6 | `value` is number not string |
| EMQ below 6 | Layer 7 | fbp/fbc forwarded, email normalized before hash |

Full diagnosis guide: [`docs/gtm-troubleshooter.md`](docs/gtm-troubleshooter.md)

---

## Production Checklist (Before Go-Live)

- [ ] Remove Test Event Code from Meta CAPI tag
- [ ] Set `transport_url` to production server
- [ ] Use System User Token (long-lived), not short-lived token
- [ ] Set Cloud Run min instances = 1 (prevents cold start event loss, ~$5–8/month)
- [ ] Set up GCP alert: error rate > 1%
- [ ] Set up GA4 Custom Insight: purchase event anomaly detection
- [ ] Note baseline EMQ score and purchase count before go-live

Full checklist: [`docs/qa-checklist.md`](docs/qa-checklist.md)

---

## Tech Stack

| Layer | Technology |
|---|---|
| Tag Management | Google Tag Manager (Web + Server) |
| Infrastructure | GCP Cloud Run |
| Destinations | Meta Conversions API, GA4 |
| CAPI Template | stape.io Meta CAPI Tag |
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

**ปัญหา** Browser-side pixel เสีย conversion event ประมาณ 20–40% (industry benchmark) เพราะ ad blocker, Safari ITP และข้อจำกัด iOS เมื่อ Meta ได้ข้อมูลไม่ครบ มันจะ optimize campaign บนภาพที่ไม่สมบูรณ์ → budget เสียเปล่า attribution ผิด

**วิธีแก้** Implement GTM Server-Side + Meta CAPI บน GCP Cloud Run ส่ง event แบบ browser → first-party server → Meta server-to-server พร้อม deduplication ด้วย event_id กันนับซ้ำ

**ผลลัพธ์ (ทดสอบจริงในโปรเจคนี้)**

- ยืนยัน Purchase event ถึง Meta — แสดงเป็น Processed ใน Meta Events Manager Test Events, source = Server
- pipeline ทำงานครบ: dataLayer → Web Container → Server Container → Meta CAPI
- implement deduplication ด้วย event_id ที่ generate ก่อน push dataLayer
- filter internal traffic ด้วย cookie flag รองรับ dynamic IP

**Business Impact ที่คาดหวัง** *(อ้างอิง industry benchmark — ยังไม่ได้วัดจริงในโปรเจคนี้)*

- server-side tracking โดยทั่วไปกู้คืน event ที่หายไป 20–40%
- EMQ ที่ดีขึ้นจากการส่ง hashed first-party data ช่วยลด CPA ได้ 15–25% (ตามตัวเลขที่ Meta เผยแพร่)
- first-party cookie อายุยาวขึ้นจาก ~7 วัน เป็นสูงสุด 400 วัน

## Architecture

```
Browser ของ User
    │  dataLayer.push({ event: 'purchase', event_id: '...', value: 1490, ... })
    ▼
GTM Web Container (browser)
    │  ตรวจ exception trigger → ข้ามถ้า cookie is_internal = 1
    │  HTTP POST → transport_url (server ของคุณเอง ไม่ใช่ Google/Meta โดยตรง)
    ▼
GTM Server Container (GCP Cloud Run)
    │
    ├──► GA4 Client → แปลง request เป็น event data object
    ├──► GA4 Tag → ส่งไป Google Analytics
    └──► Meta CAPI Tag (stape.io) → graph.facebook.com/events
              │  เพิ่ม fbp, fbc, hashed email, IP, user agent
              └──► Meta dedup ด้วย event_id → นับครั้งเดียว
```

## Before / After

| ด้าน | Browser-side (เดิม) | Server-side (ใหม่) |
|---|---|---|
| Ad blocker | บล็อก 20–40% | server-to-server ไม่โดนบล็อก |
| Safari ITP cookie | ~7 วัน | สูงสุด 400 วัน (first-party) |
| iOS 14+ | ข้อมูลจำกัด | ส่ง hashed user_data ผ่าน server |
| JavaScript error | pixel อาจไม่ fire | server รับได้ถ้า request ถึง |
| Filter internal traffic | static IP เท่านั้น (พังถ้า dynamic IP) | cookie flag — ใช้ได้ทุก IP |
| EMQ signals | fbp/fbc เท่านั้น | + hashed email/phone จาก backend |

*ตัวเลขเป็น industry benchmark ไม่ได้วัดจากโปรเจคนี้โดยตรง*

## Concept สำคัญ

### Transport URL
ชี้ GTM Web Container ไปที่ server ของคุณแทน Google/Meta โดยตรง browser เห็นเป็น first-party request → ad blocker ไม่บล็อก

### Event Match Quality (EMQ)
คะแนน 0–10 ที่ Meta ให้ว่าจับคู่ event กับ user จริงได้ดีแค่ไหน server-side ได้เปรียบเพราะส่ง hashed first-party data จาก backend ได้ เป้าหมาย EMQ 6+ พร้อม coverage สูง (coverage สำคัญกว่า score)

### Deduplication
browser pixel กับ server CAPI ส่ง event เดียวกัน ถ้าไม่ทำ dedup Meta นับ 2 ครั้ง แก้โดยใส่ event_id เดียวกันทั้ง 2 ทาง generate ก่อน push dataLayer เสมอ

### Cookie-Based Internal Traffic Filter
filter IP แบบ static ไม่ได้ผลกับ dynamic IP (ซึ่งพบบ่อยในไทย) วิธีนี้ใช้ cookie flag แทน คลิก URL พิเศษครั้งเดียว cookie อยู่ 1 ปี

### Consent Mode v2
server-side ไม่ได้ทำให้ consent ไม่จำเป็น — กลับสำคัญกว่าเดิม เพราะ server ส่งข้อมูลแน่นอน consent ไหลจาก browser → server ผ่าน gcs parameter บังคับใช้สำหรับ advertiser EEA ตั้งแต่มีนาคม 2024

## Troubleshooting Quick Reference (ภาษาไทย)

| อาการ | Layer | ตรวจที่ไหนก่อน |
|---|---|---|
| Server Preview ว่าง | Layer 1 | transport_url ใน GA4 Config Tag |
| Client ไม่ claim request | Layer 2 | มี GA4 Client ใน server container ไหม |
| Meta CAPI tag ไม่ fire | Layer 3 | ชื่อ trigger (case-sensitive) |
| Events Manager ไม่เห็น event | Layer 4 | token หมดอายุ, มี Test Event Code อยู่ไหม |
| Meta นับซ้ำ | Layer 5 | event_id มีและเหมือนกันทั้ง 2 ทาง |
| Revenue = 0 | Layer 6 | value เป็น number ไม่ใช่ string |
| EMQ ต่ำกว่า 6 | Layer 7 | ส่ง fbp/fbc ผ่านไหม, normalize email ก่อน hash |

ดูรายละเอียดเต็ม: [`docs/gtm-troubleshooter.md`](docs/gtm-troubleshooter.md)

## References

- Meta CAPI Tag template by [stape.io](https://stape.io)
- GTM Server-Side docs by [Simo Ahava](https://simoahava.com)

---

Built by [Khanet-dew (Devver)](https://github.com/Khanet-dew)
