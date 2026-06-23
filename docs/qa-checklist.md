# GTM Server-Side + Meta CAPI — Production QA Checklist

ใช้ checklist นี้ก่อนและหลัง deploy server-side tracking ขึ้น production
Use this checklist before and after deploying server-side tracking to production.

---

## ① Before Deploy — ตรวจก่อนขึ้น production

- [ ] ลบ Test ID / `test_event_code` ออกจาก Meta CAPI tag
      *ถ้าลืม event จะเข้า Test Events ไม่เข้า production data*
- [ ] เปลี่ยน Measurement ID จาก placeholder เป็นค่าจริง (`G-XXXXXXXXXX`)
- [ ] ตรวจว่า `transport_url` ชี้ไป production server ไม่ใช่ test environment
- [ ] Access Token เป็น System User Token (long-lived) ไม่ใช่ short-lived token
      *short-lived token หมดอายุใน 1–2 ชั่วโมง*
- [ ] ทดสอบทุก event ใน GTM Preview Mode ครบทุกตัว (purchase, add_to_cart, ฯลฯ)
- [ ] Web Container และ Server Container published แล้ว ไม่ใช่แค่ preview

---

## ② Data Quality — ตรวจคุณภาพข้อมูล

- [ ] `value` เป็น number ไม่ใช่ string ทุก event
      *`"1490"` (string) จะทำให้ Meta อ่าน value = 0*
- [ ] `currency` เป็น ISO 4217 ถูกต้อง (`THB`, `USD`, `EUR`)
- [ ] `event_id` มีและ unique ทุก purchase event
- [ ] `transaction_id` ไม่ซ้ำ มาจาก database จริง ไม่ใช่ generate มั่ว
- [ ] `items[]` เป็น array เสมอ แม้มีสินค้าชิ้นเดียว
- [ ] ไม่มี PII (email, phone, name) ส่งแบบ plain text — ต้อง hashed

---

## ③ Deduplication — ตรวจว่าไม่นับซ้ำ (ถ้าใช้ทั้ง browser pixel + server CAPI)

- [ ] browser pixel กับ server CAPI ใช้ `event_id` เดียวกัน (ค่าตรงเป๊ะ)
- [ ] `event_name` ตรงกันทั้ง 2 ทาง (`purchase` = `purchase`, case-sensitive)
- [ ] ตรวจใน Meta Events Manager → Event Deduplication tab ว่า dedup ทำงานจริง

---

## ④ Monitoring — เฝ้าระวังหลัง go-live

*tracking ที่พังมักพังแบบเงียบ ไม่มี error เตือน — ต้องมีระบบ monitoring*

- [ ] ตั้ง alert เมื่อ event volume ตกผิดปกติ (ตกฮวบ = อาจพัง)
- [ ] เช็ค Meta Events Manager ทุกสัปดาห์ (EMQ, event coverage, errors)
- [ ] ดู GCP Cloud Run logs หา error spike (4xx/5xx response จาก Meta)
- [ ] เตือนล่วงหน้าก่อน Access Token หมดอายุ
- [ ] ตรวจ Meta Diagnostics tab หา warning เป็นระยะ

---

## Error Handling Note — Meta CAPI Tag (stape.io template)

Option **"Use Optimistic Scenario"** ใน Meta CAPI tag:

- **เปิด** — tag รายงานสำเร็จทันทีโดยไม่รอ response จาก Meta → เร็วขึ้น แต่ถ้า Meta ปฏิเสธจริงจะไม่รู้
- **ปิด** (default, แนะนำสำหรับ production) — tag รอ response จาก Meta ถ้า fail จะเรียก `gtmOnFailure()` → รู้เมื่อมีปัญหา

แนะนำ **ปิด** ใน production เพื่อให้ระบบ monitoring จับ error ได้

---

## Response Code Reference (Meta CAPI)

| Status Code | ความหมาย | วิธีจัดการ |
|---|---|---|
| 200 | ส่งสำเร็จ | ปกติ (อาจ delay 5–20 นาทีก่อนขึ้น Events Manager) |
| 4xx | credentials ผิด / payload ผิด | ตรวจ access token, pixel ID, payload format |
| 5xx | ปัญหาฝั่ง Meta | retry / รอ Meta แก้ |
