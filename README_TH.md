# ระบบสรุปข่าวหลายผู้ใช้บน Deno Deploy

โปรเจกต์นี้ทำให้ผู้ใช้เปิดเว็บแล้วสรุปข่าวได้ทันที โดยไม่ต้องกรอก API Key

ลำดับ AI:
1. Gemini 3.5 Flash
2. Gemini 3.1 Flash-Lite
3. Groq (หลายโมเดลสำรอง)
4. OpenRouter Free

มีระบบ:
- ซ่อน API Key ฝั่ง Deno Deploy
- Fallback AI อัตโนมัติ
- แยกผลสรุป 2 ส่วน + ปุ่มคัดลอก
- แสดง Input / Thinking / Output / Total tokens
- จำกัดจำนวนครั้งต่อ Browser และต่อ IP
- Cache ข่าวซ้ำ เพื่อลดการเรียก AI
- Deno KV สำหรับ rate limit และ cache แบบถาวร
- Rate limit ใช้เพียง 2 records หลักต่อคำขอ (Client + IP) เพื่อลด KV usage

## ไฟล์
- `server.js` = Backend + API + fallback AI + rate limit/cache
- `index.html` = หน้าเว็บสำหรับผู้ใช้
- `deno.json` = ตั้งค่า Deno Deploy

## Environment Variables / Secrets

อย่างน้อยใส่ API Key 1 ตัว

Secrets:
- `GEMINI_API_KEY`
- `GROQ_API_KEY` (ไม่บังคับ)
- `OPENROUTER_API_KEY` (ไม่บังคับ)

ค่าปรับแต่ง (Plain text):
- `MAX_INPUT_CHARS=30000`
- `CLIENT_PER_MINUTE=2`
- `CLIENT_PER_DAY=10`
- `IP_PER_MINUTE=15`
- `IP_PER_DAY=100`
- `CACHE_TTL_MINUTES=360`

หากไม่ต้องการจำกัดค่าบางตัว ให้ตั้งเป็น `0`

## Deploy

ใช้ Deno Deploy รุ่นใหม่ที่:
https://console.deno.com

อย่าใช้ Deno Deploy Classic (`dash.deno.com`) เพราะ Classic ปิดให้บริการแล้ว

1. สร้าง GitHub repository ใหม่
2. อัปโหลด 3 ไฟล์ในโปรเจกต์นี้
3. เข้า `console.deno.com`
4. สร้าง Organization หากยังไม่มี
5. กด `+ New App`
6. เชื่อม GitHub และเลือก repository
7. ระบบอ่าน `deno.json` และใช้ `server.js` เป็น Dynamic Entrypoint
8. เพิ่ม Environment Variables / Secrets
9. สร้าง Deno KV database แล้ว Assign ให้ App
10. Deploy
11. เปิด URL ของ App เช่น `https://ชื่อแอป.deno.net` / URL ที่ Deno แสดงให้
12. ผู้ใช้เปิด URL นี้ได้ทันที ไม่ต้องมี API Key

## Deno KV

ใน Deno console:
1. ไปที่ `Databases`
2. กด `Provision Database`
3. เลือก `Deno KV`
4. ตั้งชื่อ เช่น `news-summary-kv`
5. กดสร้าง
6. กด `Assign` แล้วเลือก App ระบบสรุปข่าว

ถ้าไม่ทำ KV ระบบยังทำงานได้ แต่ rate limit/cache จะเป็นแบบ memory และอาจรีเซ็ตเมื่อ instance restart

## ทดสอบ

เปิด:
- `/health` ต้องขึ้น `OK`
- `/api/status` จะแสดงว่า Gemini/Groq/OpenRouter ตัวไหนตั้งค่าแล้ว โดยไม่เปิดเผย API Key
- `/` คือหน้าเว็บระบบสรุปข่าว

## หมายเหตุเรื่องโควต้า

Deno Deploy และ AI Provider เป็นคนละโควต้า:
- Deno = จำนวน request / bandwidth / CPU / KV
- Gemini/Groq/OpenRouter = โควต้าการเรียก AI

Cache ช่วยลดจำนวนครั้งที่เรียก AI เมื่อมีหลายคนวางข่าวเดียวกัน

## UI เวอร์ชันปรับปรุง

หน้าเว็บสำหรับผู้ใช้ถูกปรับให้:
- ไม่แสดงข้อความเรื่อง API Key
- ไม่แสดงรายชื่อ Provider ที่ไม่จำเป็นต่อผู้ใช้
- มีสถานะระบบมุมขวาบน
- มีสถานะระหว่างประมวลผล 3 ขั้นตอน
- มีตัวจับเวลาระหว่างรอ AI
- แสดงรายละเอียด Token แบบพับเก็บได้
- ยังคงปุ่มคัดลอกแยก 2 ส่วน

## พาดหัวข่าว 3 แบบ

ระบบสร้างพาดหัวข่าวพร้อมกับการสรุปใน API request เดียว:
- 3 แบบ
- แบบละ 3 บรรทัด
- ปุ่มคัดลอกแยกแต่ละแบบ
- ไม่เรียก AI เพิ่มเพื่อสร้างพาดหัว จึงไม่เพิ่มจำนวน request ต่อข่าว

## แก้พาดหัวไม่แสดง (Structured Output)

เวอร์ชันนี้เปลี่ยน Gemini ให้คืนผลแบบ Structured JSON โดยตรง:
- headlines: 3 แบบ × 3 บรรทัด
- essay
- points

หน้าเว็บจะอ่าน JSON ก่อน และยังมี parser แบบยืดหยุ่นเป็น fallback สำหรับ Groq/OpenRouter
Cache namespace ถูกเปลี่ยนใหม่เพื่อไม่ใช้ผลเก่าที่ไม่มีพาดหัว

## ปรับคุณภาพพาดหัวแบบกระชับ

เวอร์ชันนี้ปรับกติกาพาดหัวให้:
- อ่านเร็ว เข้าใจทันที
- 3 บรรทัดแบบมีหน้าที่ชัดเจน
- ไม่ตัดวลีกลางความหมาย
- ลดภาษาราชการและคำฟุ่มเฟือย
- 3 แบบมีมุมการนำเสนอแตกต่างกัน
- ไม่ใช้ clickbait
- เปลี่ยน cache namespace เพื่อไม่ใช้พาดหัวเก่าที่สร้างไว้ก่อนหน้า

## Gemini 3.1 Flash-Lite fallback

ปรับลำดับ Gemini สำรองเป็น:
- gemini-3.5-flash (LOW thinking)
- gemini-3.1-flash-lite (MINIMAL thinking)

จากนั้นจึง fallback ไป Groq และ OpenRouter Free

## พาดหัวเน้นแกนข่าวหลัก

ปรับกติกาพาดหัวให้:
- หาใจความหลักที่สุดเพียง 1 ประเด็นก่อน
- พาดหัวทั้ง 3 แบบต้องอยู่บนแกนข่าวเดียวกัน
- สถานะปัจจุบันมาก่อนความเสี่ยงในอนาคต
- ถ้ามีคำแนะนำ ให้ตามหลังสถานะหลัก
- ไม่ดึงข้อมูลรอง เช่น กฎหมาย/สิทธิ/พื้นหลัง ขึ้นเป็นพาดหัวโดยไม่จำเป็น
- ใช้สไตล์สั้นและตรง เช่น:
  CAAT แจ้งเที่ยวบินน่าน
  ยังให้บริการตามปกติ
  แนะผู้โดยสารเผื่อเวลาเดินทาง

## พาดหัว 3 สไตล์

- แบบที่ 1: ข่าวตรง กระชับ
- แบบที่ 2: เน้นสิ่งที่คนอ่านต้องรู้/ควรทำ
- แบบที่ 3: ดึงดูดความสนใจมากขึ้น แต่ห้าม clickbait และห้ามเกินข้อเท็จจริง

ทั้ง 3 แบบยังต้องยืนอยู่บนแกนข่าวหลักเดียวกัน

## ตรวจคำผิดภาษาไทยในเนื้อหาต้นฉบับ

- ทำงาน Local ใน Browser
- ไม่เรียก AI เพิ่ม
- แนะนำคำที่อาจสะกดผิด
- กด "แก้ไข" เพื่อแทนคำใน textarea
- กด "ข้าม" สำหรับชื่อเฉพาะ/คำที่ตั้งใจใช้
- มีปุ่ม "คัดลอกเนื้อหาที่แก้แล้ว"
- เนื้อหาหลังแก้จะถูกใช้เป็นต้นฉบับในการสรุปข่าว

หมายเหตุ: ตัวตรวจแบบ Local นี้เน้นคำผิดที่พบบ่อยและรูปแบบตัวอักษรซ้ำ ไม่ใช่พจนานุกรมภาษาไทยเต็มรูปแบบ
