---
name: gtm-reliability-monitoring
description: >
  ทำให้ GTM server-side tracking เชื่อถือได้ระดับ production และตั้งระบบเฝ้าระวัง ใช้ skill นี้เมื่อ: ต้องตั้งค่า Cloud Run ให้ไม่มี cold start / event หายตอน traffic เบา, ต้องตั้ง alert หรือ monitoring, ต้องเตรียม production readiness ด้าน infrastructure, หรือเมื่อ user ถามเรื่อง Cloud Run config (min/max instances, scale to zero, cold start), reliability, uptime, event loss แบบไม่มี error, หรือ monitoring/alert/health check. รับ hand-off จาก gtm-troubleshooter (เมื่อ root cause เป็นปัญหาเชิงระบบเช่น cold start) และจาก gtm-server-audit (เมื่อตรวจ production readiness แล้วต้องตั้ง monitoring ต่อ). Trigger แม้ user พิมพ์แค่ "event หายตอนกลางคืน", "ตั้ง alert ยังไง", "cold start", "min instances", "tracking พังแบบเงียบ", "เฝ้าระวัง tracking", "พร้อม production ไหมด้าน infra".
---

# GTM Reliability + Monitoring

## Overview

Skill นี้ครอบ 2 เสาของ production-grade tracking:

- **Reliability** — event มาถึง server ครบทุกครั้งไหม (Cloud Run config)
- **Monitoring** — ถ้าอะไรพัง จะรู้ก่อนที่ data จะเสียนานไหม (alert + routine)

ต่างจาก `gtm-troubleshooter` ที่ "แก้ตอนพัง" — skill นี้ "ป้องกันไม่ให้พังเงียบ + ตั้งระบบจับก่อนสาย"

## เมื่อไหร่ใช้ skill นี้

ใช้เมื่อปัญหาหรือเป้าหมายเป็นเชิง **ระบบ/โครงสร้าง** ไม่ใช่ payload เฉพาะ event:

| สถานการณ์ | ใช้ skill นี้ |
|---|---|
| event หายเป็นช่วงๆ โดยไม่มี error | ✓ Reliability |
| event ชุดแรกของเช้าหาย | ✓ Cold start |
| ต้องตั้ง alert / monitoring | ✓ Monitoring |
| เตรียม production readiness ด้าน infra | ✓ ทั้งคู่ |
| payload ของ event ใด event หนึ่งผิด | ✗ ใช้ gtm-troubleshooter |
| EMQ ต่ำ / hash ผิด | ✗ ใช้ gtm-data-quality-emq |

## Hand-off — เข้า

Skill นี้รับงานต่อมาจาก:

- **จาก `gtm-troubleshooter`** — เมื่อ diagnose แล้วพบว่า root cause เป็น cold start / scale-to-zero / capacity overflow → hand off มาที่นี่เพื่อ fix config ถาวร
- **จาก `gtm-server-audit`** — เมื่อ audit production readiness เสร็จ → hand off มาที่นี่เพื่อตั้ง monitoring/alert ก่อน go-live

เมื่อรับ hand-off: ไม่ต้องถามอาการซ้ำ ใช้ context ที่ skill หลักส่งมาต่อได้เลย

## Hand-off — ออก

Skill นี้ส่งงานต่อไปยัง:

- **ไปที่ `gtm-data-quality-emq`** — ถ้าระหว่างตั้ง monitoring พบว่า EMQ ต่ำ / revenue เพี้ยน / dedup พัง (เป็นปัญหา data ไม่ใช่ infra)
- **ไปที่ `gtm-troubleshooter`** — ถ้า user รายงานอาการเฉพาะหน้าที่ต้อง diagnose ก่อน

## Flow การทำงาน

```
1. ระบุว่าเป็นปัญหา Reliability หรือ Monitoring (หรือทั้งคู่)
   │
   ├─ Reliability → ดู references/cloud-run-config.md
   │     ตรวจ: min instances, max instances, timeout, concurrency
   │     fix: ตั้งค่าให้ถูก + gcloud command
   │
   └─ Monitoring → ดู references/monitoring-setup.md
         ตั้ง: 3 ชั้น (Infra → Volume → Data Quality)
         alert: P0 ก่อน go-live, P1/P2 ตามมา
         routine: weekly 15 นาที
   │
2. ให้ checklist ที่ actionable พร้อม path จริง (GCP Console / GA4 / Meta)
3. ถ้าเจอ data issue ระหว่างทาง → hand off ไป gtm-data-quality-emq
```

## หลักการสำคัญ (อย่าลืม)

1. **Silent degradation คือศัตรูตัวจริง** — tracking ที่ดูทำงานปกติแต่ data เพี้ยน อันตรายกว่าพังเห็นชัด เพราะคนเชื่อตัวเลขผิดด้วยความมั่นใจ
2. **Cold start เกิดทุกเช้า** ถ้า min instances = 0 — เป็น default ของ GTM auto-provision ต้องแก้เป็น 1 เสมอก่อน production
3. **Preview ผ่าน ≠ production reliable** — Preview เปิด container ค้างไว้ จึงไม่เจอ cold start ที่ production เจอ
4. **Monitoring = เครื่องตรวจควัน, Troubleshooter = ถังดับเพลิง** — ต้องมีทั้งคู่ แต่ตรวจควันสำคัญกว่าเพราะจับก่อนไฟลาม
5. **ชั้นล่างพัง ชั้นบนไร้ความหมาย** — Infra พัง → Volume/Quality ไม่มีข้อมูล ต้องดู 3 ชั้นพร้อมกัน

## Output Format

```
## 🏗 Reliability / Monitoring Review — [หัวข้อ]

### สถานะปัจจุบัน
[อะไร config ไว้แล้ว / ยังขาดอะไร]

### ❌ ความเสี่ยง
- [ความเสี่ยง]: [ผลถ้าไม่แก้]

### 🛠 สิ่งที่ต้องทำ (เรียงตาม priority)
1. [P0] [action] → [path จริง]
2. [P1] [action] → [path จริง]

### ✅ วิธีตรวจว่าสำเร็จ
- [วิธีตรวจ] → ควรเห็น [ผลที่ถูกต้อง]

### 📋 Monitoring ที่แนะนำตั้งต่อ
- [alert/routine ที่เกี่ยวข้อง]
```

## Reference Files

- `references/cloud-run-config.md` — ทุก setting ของ Cloud Run พร้อม gcloud command, cold start mechanics, cost estimate
- `references/monitoring-setup.md` — 3 ชั้น monitoring, 6 alerts (P0–P2), วิธีตั้ง GCP alert ทีละ step, Meta Events Manager review, weekly/monthly routine

อ่าน reference เมื่อต้องการรายละเอียดเชิงลึก — SKILL.md นี้พอสำหรับ route และตัดสินใจว่าจะทำอะไร

---

# 📎 Reference 1: Cloud Run Config (เนื้อหาเต็ม)


## Mental Model

GTM Server Container = โปรแกรมที่รอรับ HTTP request ตลอดเวลา
Cloud Run = "ห้อง" ที่โปรแกรมนั้นอยู่

ปัญหา: Cloud Run ออกแบบมาเพื่อประหยัดเงิน → ถ้าไม่มีคนใช้ มันปิดห้องทิ้ง พอมีคนมาต้องเปิดใหม่ (ใช้เวลา)
ปัญหาของ tracking: event ไม่รอ — browser ส่ง request แบบ fire-and-forget ถ้า server ยังไม่พร้อม event หายเลย ไม่มี retry

---

## Cold Start คืออะไร

### 3 แนวคิดหลัก

| แนวคิด | ความหมาย | เปรียบเทียบ |
|---|---|---|
| Scale to Zero | ไม่มี traffic 15 นาที → ปิด container ทิ้ง | ร้านปิดครัวเมื่อไม่มีลูกค้า |
| Instance | "ห้อง" 1 ห้อง รับได้หลาย request พร้อมกัน | พนักงาน 1 คน รับได้หลายโต๊ะ |
| Warm Instance | instance ที่เปิดอยู่แล้ว รับทันที 0ms | พนักงานยืนรออยู่เสมอ |

### Timeline เปรียบเทียบ

**Min instances = 0 (default):**
```
Request ถึง Cloud Run    →  0ms
Cold start (เปิด container) →  ~1,500ms  ← ดีเลย์ตรงนี้
Process + ส่ง Meta        →  ~300ms
รวม                       →  ~1,800ms+ → browser อาจ navigate ไปแล้ว → event หาย
```

**Min instances = 1 (แนะนำ):**
```
Request ถึง Cloud Run    →  0ms
Cold start                →  0ms (instance พร้อมอยู่แล้ว)
Process + ส่ง Meta        →  ~300ms
รวม                       →  ~300ms → ปลอดภัย
```

### ทำไม browser timeout
GTM Web ส่ง request แบบ fire-and-forget — ไม่รอ response ถ้า user navigate ไปหน้าอื่นก่อน request เสร็จ → event หายทันที นี่คือเหตุผลหลักที่ cold start อันตราย

### Cold Start เกิดบ่อยที่สุดเมื่อ
- ช่วงดึก-เช้า (traffic เบานาน 15+ นาที) → event ชุดแรกของวันหาย
- หลัง deploy version ใหม่
- Traffic spike ฉับพลัน (sale, launch) เกิน capacity ปัจจุบัน
- ช่วงแรกของ campaign เปิด

---

## Config ทุกค่า

### Min Instances — สำคัญที่สุด
- **แนะนำ: 1**
- Default (0) = scale to zero = cold start ทุกเช้า
- 1 instance พร้อมตลอด 24 ชม. ค่าใช้จ่ายเพิ่ม ~$5–8/เดือน แต่กัน event loss ได้หมด
- ตั้งที่: GCP Console → Cloud Run → เลือก service → Edit & Deploy New Revision → Capacity → Minimum number of instances → 1 → Deploy

### Max Instances — ป้องกัน spike
- **แนะนำ: 4–10** (ขึ้นกับ traffic)
- Default (1000) สูงเกิน — เสี่ยง bill พุ่งถ้ามี traffic ผิดปกติ/attack
- site ทั่วไป 4–5 พอ / high traffic 10–20
- ตั้งต่ำเกินก็ไม่ดี — traffic เกิน max → request reject → event หาย

### Concurrency — request ต่อ instance
- **แนะนำ: 80** (default ของ Cloud Run)
- GTM Server จัดการ concurrent ได้ดี ใช้ default ได้เลย ไม่ต้องแก้

### CPU
- **แนะนำ: 1 vCPU** (default)
- GTM auto-provision ตั้งให้แล้ว ถ้า latency สูงค่อยเพิ่มเป็น 2

### Memory
- **แนะนำ: 512Mi** (default)
- พอสำหรับ GTM Server ทั่วไป ถ้าเจอ OOM error ค่อยเพิ่มเป็น 1Gi

### Request Timeout
- **แนะนำ: 30s** (default)
- อย่าตั้งต่ำกว่า 10s — Meta CAPI อาจใช้เวลา 3–8 วินาที ถ้า timeout สั้น tag จะ fail

---

## gcloud Command — ตั้งทุกค่าครั้งเดียว

```bash
gcloud run services update [SERVICE_NAME] \
  --min-instances=1 \
  --max-instances=5 \
  --concurrency=80 \
  --timeout=30 \
  --region=us-central1
```

แทน `[SERVICE_NAME]` ด้วยชื่อจริง เช่น `server-side-tagging-szry2lggmq`

---

## Reliability Mental Model

```
Reliability = P(event เกิด) × P(request ถึง server) × P(server พร้อมรับ) × P(forward สำเร็จ)
```

ทุกตัวเป็นความน่าจะเป็น < 100% — server-side เพิ่มตัวแปร **"server พร้อมรับ"** ที่ browser-side ไม่เคยมี

นี่คือ paradox: บางคนย้ายมา server-side เพื่อความ reliable แต่ตั้ง Cloud Run ผิด (min instances = 0) แล้วได้ reliability **แย่กว่าเดิม** เพราะเพิ่มจุดพังใหม่โดยไม่ harden มัน

---

## Cost Estimate (หลัง config)

| รายการ | ค่าใช้จ่าย |
|---|---|
| Min 1 instance (always warm) | ~$5–8/เดือน |
| Traffic ปกติรวม | ~$10–20/เดือน |
| Free tier | request แรก 2 ล้าน req/เดือน ฟรี |

dev/test แทบไม่มีค่าใช้จ่าย เพราะอยู่ใน free tier

---

## Checklist — Reliability

- [ ] Min instances = 1 (กัน cold start ทุกเช้า) — **สำคัญสุด**
- [ ] Max instances = 4–10 (กัน spike)
- [ ] Request timeout ≥ 30s (ถ้าต่ำกว่า 10s แก้ทันที)
- [ ] เปิด Cloud Run Metrics ดู request count / error rate / instance count
- [ ] ตั้ง alert error rate > 1% (ดู monitoring-setup.md)
- [ ] ทดสอบ warm-up ตอนเช้าวันแรกหลัง config — request ควรเข้าทันทีไม่มี delay

---

# 📎 Reference 2: Monitoring Setup (เนื้อหาเต็ม)


## Mental Model

Reliability + Data Quality = สิ่งที่ "พังได้"
Monitoring = ระบบที่ทำให้ "รู้ว่ามันพัง" ก่อนคนอื่นรู้

ความต่างระหว่างคนทำ tracking กับ consultant: consultant มีระบบแจ้งเตือนก่อน client โทรมาบอกว่าตัวเลขผิด

---

## 3 ชั้น Monitoring

แต่ละชั้นจับปัญหาคนละแบบ — ต้องดูพร้อมกัน

| ชั้น | ตรวจอะไร | เครื่องมือ | พังแล้วเป็นยังไง |
|---|---|---|---|
| 1. Infrastructure | Cloud Run ทำงานไหม, error, latency | GCP Cloud Monitoring | event ไม่ถึง server เลย |
| 2. Event Volume | จำนวน event ผิดปกติไหม | GA4 Anomaly Detection | tracking ถูกลบตอน deploy |
| 3. Data Quality | EMQ, dedup, revenue เปลี่ยนไหม | Meta Events Manager (manual) | data มาแต่เชื่อไม่ได้ |

### หลักการ: ชั้นล่างพัง ชั้นบนไร้ความหมาย
- Infra พัง → Volume/Quality ไม่มีข้อมูลให้ดู
- Volume ผิดปกติ → Quality อาจดูดีเพราะ sample เล็กลง
- ต้องดู 3 ชั้นพร้อมกัน ไม่ดูชั้นเดียว

### เปรียบกับ vital signs
- Infrastructure = ชีพจร (ไม่มี = ตาย ไม่ต้องดูอย่างอื่น)
- Event Volume = อุณหภูมิ (baseline ปกติ vs ผิดปกติ)
- Data Quality = ค่าเลือด (ชีพจร/อุณหภูมิดี แต่เลือดผิด = ป่วยเงียบ)

---

## Alerts ที่ต้องตั้ง (เรียงตาม priority)

### P0 — ตั้งก่อน go-live

**1. Cloud Run Error Rate สูง**
- Metric: `error_rate > 1%` (4xx/5xx เกิน 1% ใน 5 นาที)
- ทำไม: request กำลังหาย — token หมดอายุ, Meta ล่ม, config ผิด
- ตั้งที่: GCP Cloud Monitoring → Alerting

**2. Purchase Event หายไป**
- Metric: `purchase_count = 0` เกิน 2 ชั่วโมงใน business hours
- ทำไม: tracking พังหรือ payment มีปัญหา ต้องรู้ใน 2 ชม. ไม่ใช่เช้าวันรุ่งขึ้น
- ตั้งที่: GA4 Custom Insight หรือ Looker Studio Alert

### P1 — ตั้งภายใน 1 สัปดาห์

**3. Event Volume ตกผิดปกติ**
- Metric: `event_count < 50%` ของ 7-day average
- ทำไม: deploy ใหม่อาจลบ tracking หรือหน้าสำคัญ error
- ตั้งที่: GA4 Custom Insight → Anomaly Detection

**4. Cloud Run Instance = 0**
- Metric: `instance_count = 0` เกิน 5 นาทีใน business hours
- ทำไม: ถ้าตั้ง min=1 แล้วยังเกิด = ผิดปกติมาก (billing หยุด, service disabled)
- ตั้งที่: GCP Cloud Monitoring → Cloud Run Metrics

### P2 — ตั้งภายใน 1 เดือน

**5. EMQ Score ตกฉับพลัน**
- Metric: `emq_score < 6.0` (ตรวจ manual ทุกสัปดาห์ — Meta ไม่มี alert API)
- ทำไม: signal หยุดไหล (fbp/fbc ไม่ส่งผ่าน หรือ email double hash หลัง deploy)
- ดูที่: Meta Events Manager → Event Detail → Match Quality
- → ถ้าเจอ hand off ไป gtm-data-quality-emq

**6. Revenue Per Order ผิดปกติ**
- Metric: `avg_order_value` เปลี่ยน > 30%
- ทำไม: value ส่งเป็น string (= 0) หรือ currency mismatch
- ตั้งที่: GA4 Custom Insight หรือ Looker Studio
- → ถ้าเจอ hand off ไป gtm-data-quality-emq

---

## วิธีตั้ง GCP Alert — Error Rate (ทีละ step)

1. **เปิด Cloud Monitoring** — GCP Console → ค้นหา "Monitoring" → Alerting → Create Policy
2. **เลือก Metric** — Select a metric → Cloud Run Revision → `request_count` → filter `response_code_class = 5xx`
3. **ตั้ง Threshold** — Condition: `is above 0.01` (1%) for 5 minutes (กัน false alarm)
4. **Notification Channel** — Add Notification Channel → Email → ใส่ email
5. **ตั้งชื่อ** — เช่น `[ProjectName] GTM Server — High Error Rate` → Save

---

## Log Explorer — ดู Error จริง

เมื่อ alert แจ้ง ไปดู log ว่าเกิดอะไร:

1. GCP Console → Logging → Logs Explorer
2. Filter:
```
resource.type="cloud_run_revision"
AND resource.labels.service_name="[ชื่อ service]"
AND severity>=ERROR
```
3. คลิก log entry → ดู `textPayload` / `jsonPayload` — มักบอกชัดว่า Meta reject เพราะอะไร (token expired, invalid pixel ID)

---

## Meta Events Manager — ดูอะไรทุกสัปดาห์

| ข้อมูล | path | ปกติ | อันตราย |
|---|---|---|---|
| Event volume + source | Overview tab → Server vs Browser | สม่ำเสมอ | Server events หายฉับพลัน |
| EMQ Score | Purchase → Match Quality | ≥ 7.0 | ตกต่ำกว่า 6 ฉับพลัน |
| Deduplication | Event → Received vs Deduplicated | Received > Deduplicated | Received = Deduplicated |
| CAPI Errors | Diagnostics tab → Issues | ไม่มี | มี error ต้องอ่าน |

### 5 สัญญาณอันตรายที่ต้องจำ

| สัญญาณ | แปลว่า | ตรวจที่ |
|---|---|---|
| Server events = 0 ฉับพลัน | CAPI tag ไม่ fire / Cloud Run ล่ม | Layer 1 → 3 (troubleshooter) |
| EMQ ตก 8→4 ฉับพลัน | signal สำคัญหาย | Match Quality → signal breakdown |
| Received = Deduplicated | dedup ไม่ทำงาน นับซ้ำ | event_id ใน dataLayer |
| Events มาแต่ไม่มี Revenue | value เป็น string / = 0 | GTM Preview → Event data |
| Purchase สูงกว่า backend 50%+ | dedup พัง / test ปน | reconcile GA4/Meta/Backend |

---

## Weekly Routine — 15 นาที (ทุกวันจันทร์เช้า)

ทำก่อนงานอื่น เพื่อจับปัญหาจากสุดสัปดาห์ (deploy วันศุกร์มักพัง tracking)

| เวลา | ทำอะไร | ดูที่ | ถ้าผิดปกติ |
|---|---|---|---|
| 2 นาที | Cloud Run error rate | GCP → Cloud Run → Metrics | spike → Logs Explorer |
| 3 นาที | purchase count เทียบสัปดาห์ก่อน | GA4 → Reports → Events | ตก >15% → ตรวจ Layer 1-3 |
| 3 นาที | EMQ score + signal breakdown | Meta → Purchase → Match Quality | <6 → ตรวจ signal หาย |
| 3 นาที | dedup ratio | Meta → Received vs Deduplicated | Received=Dedup → ตรวจ event_id |
| 4 นาที | reconcile GA4 vs Backend | GA4 vs order DB | ต่าง >15% → ตรวจต่อ |

---

## Monthly Check — ลึกกว่า weekly

**ตรวจ:**
- Access token ยังไม่หมดอายุ
- GCP billing ปกติ ไม่มี cost spike
- EMQ trend ตลอดเดือน (ขึ้น/ลง)
- Revenue per order ผิดปกติไหม
- Test events ปนมาไหม (filter IP)

**Document ไว้เสมอ:**
- วันที่ deploy อะไร → เปรียบ tracking before/after
- วันที่ Meta token ต่ออายุ
- วันที่ baseline เปลี่ยน (เริ่ม campaign ใหม่)
- EMQ score ณ วันนั้น

---

## มูลค่าสำหรับ consulting

ถ้าส่ง weekly report ให้ client ทุกจันทร์ว่า "tracking ปกติ, EMQ 8.2, purchase ตรงกับ backend 96%" — client จะไม่ต้องโทรมาถามว่า "tracking ยังทำงานไหม?" นั่นคือมูลค่าที่ client จ่ายเงินซื้อ: **ความมั่นใจ ไม่ใช่แค่คนมาแก้ตอนพัง**

---

## Checklist — Monitoring (ก่อน go-live)

**Infrastructure (GCP):**
- [ ] Alert: Cloud Run Error Rate > 1%
- [ ] Alert: Instance Count = 0 นาน > 5 นาที
- [ ] ทดสอบ Logs Explorer ดู log ได้

**Event Volume (GA4):**
- [ ] Custom Insight: Purchase event ตกผิดปกติ
- [ ] Note baseline purchase count/day ก่อน go-live

**Data Quality (Meta):**
- [ ] Note EMQ score เริ่มต้น
- [ ] ตรวจ dedup ratio ครั้งแรก
- [ ] ตั้ง calendar reminder weekly check ทุกจันทร์
