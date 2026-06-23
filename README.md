# GTM Server-Side Tracking + Meta CAPI Demo

> **ภาษาไทย / Thai version below** ↓

A complete implementation of **GTM Server-Side Tracking** with **Meta Conversions API (CAPI)** — built as a learning portfolio and reusable template for real client work.

## What This Project Does

Replaces browser-side pixel tracking with a server-side architecture that:

- **Bypasses ad blockers** — events are sent server-to-server, not browser-to-vendor
- **Survives Safari ITP** — first-party cookies set by your own server, not restricted by browser
- **Improves Meta CAPI data quality** — purchase events reach Meta even when browser pixel is blocked
- **Reduces data loss** — typically recovers 20–40% of events lost to browser-side blocking

## Architecture

```
User's Browser
    │
    │  dataLayer.push({ event: 'purchase', ... })
    ▼
GTM Web Container
    │
    │  HTTP POST → transport_url (your server, not Google/Meta directly)
    ▼
GTM Server Container (GCP Cloud Run)
    │
    ├──► GA4 Client → parses incoming request
    │
    └──► Meta CAPI Tag → POST to graph.facebook.com/events
                         server-to-server, no browser involved
```

## Repo Structure

```
gtm-server-side-capi-demo/
├── README.md
├── gtm-config/
│   ├── web-container.json          # GTM Web Container export
│   └── server-container.json      # GTM Server Container export
├── test-page/
│   └── gtm-test-page.html         # Local test page with simulated e-commerce events
├── tracking-spec/
│   └── tracking-spec-template.xlsx
└── docs/
    └── setup-guide.md
```

## Quick Start

### Step 1 — Replace all placeholders

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

1. Go to [tagmanager.google.com](https://tagmanager.google.com)
2. Create new container → Target platform: **Server**
3. Choose **Automatically provision tagging server**
4. Select or create a GCP project
5. Wait ~3–5 minutes → get your server URL

### Step 3 — Import GTM containers

1. GTM → Admin → Import Container
2. Select `gtm-config/web-container.json` → Import to Web Container
3. Select `gtm-config/server-container.json` → Import to Server Container

### Step 4 — Test locally

1. Install [VS Code](https://code.visualstudio.com) + Live Server extension
2. Open `test-page/gtm-test-page.html` → right-click → **Open with Live Server**
3. Enable GTM Preview Mode on both Web and Server containers
4. Click any button on the test page
5. Verify in Server Preview: **Meta CAPI - Purchase → Fired**
6. Verify in Meta Events Manager → Test Events: **Purchase → Processed**

---

---

# GTM Server-Side Tracking + Meta CAPI (ภาษาไทย)

โปรเจคนี้คือการ implement **GTM Server-Side Tracking** ร่วมกับ **Meta Conversions API (CAPI)** สร้างขึ้นเพื่อเป็น learning portfolio และ template สำหรับนำไปใช้กับงาน client จริง

## โปรเจคนี้ทำอะไร

เปลี่ยนจากการติด pixel บน browser มาเป็น server-side architecture ที่:

- **ไม่โดน ad blocker บล็อก** — event ส่งแบบ server-to-server ไม่ผ่าน browser
- **รอดจาก Safari ITP** — cookies ถูก set โดย server ของคุณเอง อายุยาวกว่า
- **ข้อมูล Meta CAPI ครบขึ้น** — purchase event ถึง Meta แม้ browser pixel โดนบล็อก
- **ลด data loss** — โดยทั่วไปกู้คืน event ที่หายไป 20–40%

## ทำไม server-side tracking ถึงสำคัญ

ปัญหาของ browser-side tracking แบบเดิม:

- **Ad blockers** — ผู้ใช้ที่ใช้ Brave, uBlock Origin หรือ extension บล็อก ads จะทำให้ pixel ไม่ทำงาน
- **Safari ITP (Intelligent Tracking Prevention)** — Safari จำกัดอายุ cookie จาก third-party script เหลือแค่ 7 วัน ทำให้ returning user ถูกนับเป็น new user
- **iOS 14+** — Apple จำกัดการเข้าถึง IDFA ทำให้ data ที่ Meta ได้ไม่ครบ

Server-side tracking แก้ปัญหาเหล่านี้โดยให้ browser ส่ง event มาที่ server ของคุณก่อน แล้ว server ค่อย forward ไปให้ Meta, GA4 ฯลฯ — browser เห็นว่าคุยกับโดเมนของคุณเอง ad blocker ไม่แตะ

## Architecture

```
Browser ของ User
    │
    │  dataLayer.push({ event: 'purchase', ... })
    ▼
GTM Web Container (ทำงานใน browser)
    │
    │  HTTP POST → transport_url (server ของคุณ ไม่ใช่ Google/Meta โดยตรง)
    ▼
GTM Server Container (รันบน GCP Cloud Run)
    │
    ├──► GA4 Client → รับ request เข้ามา แปลงเป็น event data
    │
    └──► Meta CAPI Tag → ส่งไป graph.facebook.com/events
                         server-to-server ล้วนๆ ไม่ผ่าน browser
```

## โครงสร้างไฟล์

```
gtm-server-side-capi-demo/
├── README.md                           # ไฟล์นี้
├── gtm-config/
│   ├── web-container.json              # export จาก GTM Web Container
│   └── server-container.json          # export จาก GTM Server Container
├── test-page/
│   └── gtm-test-page.html             # หน้าทดสอบจำลอง e-commerce
├── tracking-spec/
│   └── tracking-spec-template.xlsx    # template spec สำหรับส่ง developer
└── docs/
    └── setup-guide.md
```

## วิธีใช้ template นี้

### Step 1 — เปลี่ยน placeholder ทั้งหมด

ค้นหาและ replace ค่าเหล่านี้ในไฟล์ทั้งหมดก่อนใช้งาน:

| Placeholder | เปลี่ยนเป็น |
|---|---|
| `YOUR_GTM_ACCOUNT_ID` | GTM Account ID ของคุณ (ดูจาก URL ใน GTM) |
| `YOUR_GTM_WEB_CONTAINER_ID` | Web Container ID เช่น `GTM-XXXXXXX` |
| `YOUR_GTM_SERVER_CONTAINER_ID` | Server Container ID เช่น `GTM-XXXXXXX` |
| `YOUR_SERVER_URL` | GCP Cloud Run URL เช่น `https://xxxxx-uc.a.run.app` |
| `YOUR_GA4_MEASUREMENT_ID` | GA4 Measurement ID เช่น `G-XXXXXXXXXX` |
| `YOUR_META_PIXEL_ID` | Meta Pixel ID เช่น `1234567890123456` |
| `YOUR_META_ACCESS_TOKEN` | Meta CAPI Access Token — **ห้าม commit ค่าจริงขึ้น GitHub** |

### Step 2 — Deploy Server Container บน GCP

1. ไปที่ [tagmanager.google.com](https://tagmanager.google.com)
2. สร้าง container ใหม่ → Target platform: **Server**
3. เลือก **Automatically provision tagging server**
4. เลือก GCP project
5. รอ ~3–5 นาที → ได้ server URL มา

### Step 3 — Import GTM containers

1. GTM → Admin → Import Container
2. เลือก `gtm-config/web-container.json` → Import เข้า Web Container
3. เลือก `gtm-config/server-container.json` → Import เข้า Server Container

### Step 4 — ทดสอบ local

1. ติดตั้ง [VS Code](https://code.visualstudio.com) + Live Server extension
2. เปิด `test-page/gtm-test-page.html` → คลิกขวา → **Open with Live Server**
3. เปิด GTM Preview Mode ทั้ง Web และ Server container
4. กดปุ่มใดก็ได้บน test page
5. ตรวจสอบใน Server Preview: **Meta CAPI - Purchase → Fired** ✅
6. ตรวจสอบใน Meta Events Manager → Test Events: **Purchase → Processed** ✅

## Concept สำคัญที่ต้องเข้าใจ

### Transport URL คืออะไร

ปกติ GTM Web Container ส่ง event ตรงไปหา Google/Meta — ad blocker รู้จัก vendor domain พวกนั้นแล้วบล็อก

พอ set `transport_url` ให้ชี้มาที่ server ของคุณเอง browser เห็นว่าคุยกับโดเมนของคุณ → ไม่โดนบล็อก

### Deduplication ใน Meta CAPI คืออะไร

ระบบนี้ส่ง event ทั้งจาก browser pixel และ server CAPI — ถ้าไม่ทำ deduplication Meta จะนับ 2 ครั้ง

แก้โดยใส่ `event_id` เดียวกันใน browser pixel และ CAPI event — Meta จะรู้ว่านี่คือ event เดียวกัน นับแค่ครั้งเดียว

### Client vs Tag ใน Server Container

| ส่วนประกอบ | หน้าที่ |
|---|---|
| **Client** (GA4 Client) | รับ HTTP request เข้ามา แปลงเป็น event data object กลาง |
| **Tag** (Meta CAPI Tag) | อ่าน event data แล้ว forward ไปหา Meta ผ่าน server API |

## ไฟล์ในโปรเจค

### `gtm-config/web-container.json`
มี:
- **GA4 Configuration Tag** — set `transport_url` ให้ชี้มาที่ server
- **GA4 Event Purchase Tag** — ส่ง purchase event ไปหา server
- **Custom Event Purchase Trigger** — fire เมื่อ `dataLayer.push({ event: 'purchase' })`

### `gtm-config/server-container.json`
มี:
- **GA4 Client** — รับและ parse request จาก Web Container
- **Meta CAPI Tag** — forward purchase event ไปหา Meta

### `test-page/gtm-test-page.html`
ร้านค้าจำลองสำหรับทดสอบ มี:
- สินค้า 4 รายการ กดปุ่มแล้ว fire event จริง
- Real-time event log แสดง dataLayer state
- Events ครบ funnel: `view_item`, `add_to_cart`, `purchase`, `begin_checkout`, `sign_up` และอื่นๆ
- `event_id` generate อัตโนมัติทุกครั้งที่กด purchase สำหรับ Meta deduplication

### `tracking-spec/tracking-spec-template.xlsx`
Tracking spec template พร้อมใช้งานจริง มี 5 events:
- `purchase` (P0) — ครบทุก parameter รวม Meta deduplication fields
- `add_to_cart` (P0)
- `view_item` (P1)
- `begin_checkout` (P1)
- `sign_up` (P1)

แต่ละ event มี: trigger conditions, dataLayer parameters, code example, และ acceptance criteria สำหรับส่ง developer

## ตัวเลือก Deploy Server Container

GTM Server Container รันเป็น Docker image — deploy ได้ทุกที่ที่รัน Docker ได้:

| Platform | ความยาก | หมายเหตุ |
|---|---|---|
| GCP Cloud Run (auto) | ง่ายที่สุด | คลิกเดียวจาก GTM UI |
| GCP Cloud Run (manual) | ง่าย | ควบคุม config ได้มากขึ้น |
| AWS App Runner | ปานกลาง | เหมาะถ้า client ใช้ AWS อยู่แล้ว |
| Docker บน VPS | ปานกลาง | self-hosted เต็มตัว |

ทุก option ใช้ `CONTAINER_CONFIG` environment variable เดียวกันจาก GTM Server Container settings

## Tech Stack

| Layer | Technology |
|---|---|
| Tag Management | Google Tag Manager (Web + Server Container) |
| Server Infrastructure | GCP Cloud Run |
| Tracking Destination | Meta Conversions API, GA4 Measurement Protocol |
| Test Environment | HTML + JavaScript, VS Code Live Server |

## References

- Meta CAPI Tag template by [stape.io](https://stape.io)
- GTM Server-Side documentation by [Simo Ahava](https://simoahava.com)

---

Built by [Khanet-dew (Devver)](https://github.com/Khanet-dew)
