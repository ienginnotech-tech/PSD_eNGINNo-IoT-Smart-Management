# PSD_eNGINNo IoT SMART Building — v0.1.0

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
- `NOTIFY_MIN_SEVERITY` = CRITICAL (ค่าเริ่มต้น) หรือ WARNING
- ระหว่างใช้ข้อมูลจำลอง แนะนำให้เว้นว่างไว้ หรือปิด `SIM_FAULTS` เพื่อไม่ให้ได้รับแจ้งเตือนจำลอง

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
- กฎ Alarm ประเมินตลอด 24 ชม. (นอกเวลาทำการที่แอร์ปิด อาจเห็น WARNING อุณหภูมิ)
