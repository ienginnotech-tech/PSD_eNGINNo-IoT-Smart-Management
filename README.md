# PSD_eNGINNo IoT SMART Building — v0.1.3

ระบบบริหารอาคารอัจฉริยะสำหรับอาคาร **BTE / BTB / UBCII**
Stack: `index.html` (GitHub Pages) + `Code.gs` (Google Apps Script) + Google Sheets

## ลำดับการพัฒนา (Roadmap)

| Phase | โมดูล | สถานะ v0.1 |
|---|---|---|
| 1 | Dashboard ติดตามเซนเซอร์ (อุณหภูมิ ความชื้น CO₂ กำลังไฟ) | ✅ |
| 2 | Alarm / แจ้งเตือน (กฎในชีต Rules, รับทราบ ACK, Email + LINE) | ✅ |
| 3 | รายงานพลังงาน & ค่าไฟ (รายวัน, เดือนนี้, คาดการณ์, CSV) | ✅ |
| 4 | ควบคุมแอร์/ไฟ (ON/OFF, Setpoint, AUTO/MANUAL, PIN) | ✅ (ทำงานกับตัวจำลอง) |
| 5 | ต่ออุปกรณ์จริง (ESP32/Gateway → `action=ingest`) | 🔜 API พร้อมแล้ว |

> เหตุผลที่ "ควบคุม" อยู่ท้ายสุด: เป็นส่วนเดียวที่มีผลต่ออุปกรณ์จริง จึงควรใช้งานส่วนดูข้อมูลและ Alarm ให้นิ่งก่อน

## ติดตั้ง

1. สร้าง Google Sheet ใหม่ ชื่อ `PSD_eNGINNo IoT SMART DB`
2. **Extensions → Apps Script** → ลบโค้ดเดิม → วาง `Code.gs` → Save
3. เลือกฟังก์ชัน `setup` → **Run** → อนุญาตสิทธิ์
   ระบบจะสร้างชีต 7 แผ่น, อุปกรณ์ตัวอย่าง 63 ตัว, กฎ Alarm 7 ข้อ และ Trigger ทุก 15 นาที
4. **Deploy → New deployment → Web app**
   - Execute as: **Me** · Who has access: **Anyone**
   - คัดลอก URL ที่ลงท้าย `/exec`
5. อัปโหลด `index.html` ขึ้น GitHub Pages → เปิดเว็บ → เมนู **อุปกรณ์** → วาง URL → **บันทึก & เชื่อมต่อ**
   (หรือใส่ URL ในค่า `DEFAULT_API_URL` ใน `index.html` เพื่อให้ทุกเครื่องใช้ข้อมูลจริงทันที)
6. **เปลี่ยน `CONTROL_PIN`** ในชีต Settings (ค่าเริ่มต้น 1234)

ถ้าไม่ใส่ URL หน้าเว็บจะทำงานใน **โหมด DEMO** (จำลองข้อมูลในเบราว์เซอร์ PIN = 1234)

## ชีตในระบบ

| ชีต | ใช้ทำอะไร |
|---|---|
| Devices | ทะเบียนอุปกรณ์ + ค่าล่าสุด — เพิ่ม/ลบชั้น/โซนได้ที่นี่ (`enabled`=FALSE เพื่อปิด) |
| Readings | ข้อมูลดิบทุกรอบ (ลบอัตโนมัติเมื่อเกิน `RETENTION_DAYS`) |
| Alarms | ประวัติ Alarm: ACTIVE → ACK → CLEARED |
| Rules | กฎ Alarm: type (ENV/AC/LIGHT/METER/ALL), metric, operator, threshold, severity |
| Commands | บันทึกคำสั่งควบคุม (ผู้สั่ง, เวลา, สถานะ) |
| EnergyDaily | สรุป kWh / ค่าไฟ รายวัน ต่ออาคาร ต่อประเภท |
| Settings | ค่าตั้งระบบ (PIN, ค่าไฟ/หน่วย, รอบเวลา, Email, LINE ฯลฯ) |

ประเภทอุปกรณ์: `ENV` เซนเซอร์ห้อง · `AC` แอร์ · `LIGHT` วงจรไฟ · `METER` มิเตอร์หลักอาคาร

## แจ้งเตือน
- **Email**: ใส่ `ALERT_EMAIL`
- **LINE**: ใช้ LINE Messaging API (LINE Notify ปิดบริการแล้ว) ใส่ `LINE_CHANNEL_TOKEN` และ `LINE_TO` (userId/groupId)
- **ทดสอบ LINE**: ใน Apps Script เลือกฟังก์ชัน `testLine` → Run → ดูผลใน Execution log
- `NOTIFY_MIN_SEVERITY` = CRITICAL (ค่าเริ่มต้น) หรือ WARNING
- **เวลาทำการ**: `COMFORT_WORKHOURS_ONLY` = TRUE → กฎอุณหภูมิ/ความชื้น/CO₂ ของเซนเซอร์ห้องเตือนเฉพาะ `WORK_START_H`–`WORK_END_H` (ค่าเริ่มต้น 07:30–18:00, จันทร์–เสาร์ ตั้ง `WORK_SATURDAY`) นอกเวลา Alarm เหล่านี้จะปิดอัตโนมัติ ส่วนกฎอุปกรณ์หลุด/ไฟฟ้ายังเตือนตลอด 24 ชม.
- ระหว่างใช้ข้อมูลจำลอง แนะนำให้เว้นว่างไว้ หรือปิด `SIM_FAULTS` เพื่อไม่ให้ได้รับแจ้งเตือนจำลอง

## ส่งแจ้งเตือนเข้ากลุ่ม LINE ทีม
1. LINE Official Account Manager → **Settings → Account settings** → เปิด **อนุญาตให้บัญชีเข้าร่วมกลุ่ม (Join groups and multi-person chats)**
2. **Settings → Response settings** → เปิด **Webhooks**
3. LINE Developers Console → แท็บ **Messaging API** → **Webhook URL** = URL Web App เดียวกัน (…/exec) → Update → เปิด **Use webhook**
   - ปุ่ม Verify อาจขึ้น error 302 เพราะ Apps Script ตอบแบบ redirect — ไม่เป็นไร ระบบยังรับข้อมูลได้
4. เชิญ **PSD_eNGINNo Alert** เข้ากลุ่ม → บอทตอบ **Group ID** (C…) กลับในกลุ่ม และบันทึกในชีต **LineTargets**
   (ถ้าไม่ตอบ พิมพ์ `id` ในกลุ่ม)
5. นำ ID ไปเพิ่มใน `LINE_TO` คั่นด้วย , เช่น `Uxxxx…, Cxxxx…` → รัน `testLine`

ระบบไม่เปลี่ยน `LINE_TO` ให้เองเพื่อความปลอดภัย · ข้อความที่บอทตอบกลับ (reply) ไม่นับโควตา

## API

GET `…/exec?action=`
- `dashboard` — อุปกรณ์ทั้งหมด + Alarm ที่ยังไม่ปิด + พลังงานวันนี้
- `history&deviceId=A,B&hours=24`
- `energy&days=30` · `alarms&days=7` · `commands` · `ping`

POST (body JSON, `Content-Type: text/plain`)
- `{action:"control", deviceId, field:"state|setpoint|mode", value, pin, user}`
- `{action:"ack", alarmId, user}`
- `{action:"ingest", apiKey, deviceId, data:{temp,hum,co2,kw,kwh,state,setpoint}}`
  หรือ `{action:"ingest", apiKey, items:[{deviceId,data},…]}` สำหรับ Gateway
  → ตอบกลับ `commands` ที่ค้างของอุปกรณ์นั้น (สถานะ PENDING → SENT)

## เมื่อต่ออุปกรณ์จริง
1. ตั้ง `SIM_ENABLED` = FALSE
2. ให้อุปกรณ์ POST `ingest` ด้วย `API_KEY` จากชีต Settings ทุก 1–15 นาที
3. อุปกรณ์ที่ไม่ส่งข้อมูลเกิน 3 รอบ จะถูกตั้งเป็นออฟไลน์และเกิด Alarm R07

## ข้อจำกัด v0.1
- PIN เดียวสำหรับทุกคน (ยังไม่มีระบบผู้ใช้/สิทธิ์)
- Google Sheets เหมาะกับข้อมูลรอบ ≥ 5 นาที; ถ้าต้องการเรียลไทม์ระดับวินาที ควรย้ายไป Firebase/MQTT ในอนาคต

## อัปเดตเวอร์ชัน (ไม่ต้องเปลี่ยน URL)
1. วาง `Code.gs` ใหม่ทับใน Apps Script → Save
2. **Deploy → Manage deployments → ✏️ Edit → Version: New version → Deploy**
3. อัปโหลด `index.html` ใหม่ขึ้น GitHub Pages
ค่าตั้งใหม่จะถูกเพิ่มลงชีต Settings ให้อัตโนมัติในรอบถัดไป

## ประวัติเวอร์ชัน
- **0.1.3** — LINE Webhook: บันทึก Group/User ID อัตโนมัติ (ชีต LineTargets) · `LINE_TO` ใส่ได้หลายปลายทาง · `testLine` ทดสอบทุกปลายทาง
- **0.1.2** — เพิ่ม `testLine()` ทดสอบส่ง LINE พร้อมคำอธิบายข้อผิดพลาด · บันทึก log เมื่อส่ง LINE ไม่สำเร็จ
- **0.1.1** — Alarm ความสบายของห้องเตือนเฉพาะเวลาทำการ · ตัวเลข Alarm ที่เมนูนับตามอาคารที่เลือก · กราฟกำลังไฟขยายเต็มการ์ด
- **0.1.0** — เวอร์ชันแรก: Dashboard, Alarm, พลังงาน, ควบคุม, โหมด DEMO
