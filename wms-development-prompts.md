# Prompt พัฒนาโปรแกรม: ระบบสแกนรับ-จ่ายสินค้า + แดชบอร์ดสต็อกเรียลไทม์
### สำหรับ SmartStock Logistics Co., Ltd. | พัฒนาด้วย JavaScript (Node.js)

เอกสารนี้รวบรวม **Prompt สำเร็จรูป** สำหรับใช้สั่งงาน AI (เช่น Claude Code) พัฒนาโปรแกรมทีละเฟส
คัดลอกแต่ละ Prompt ไปใช้ตามลำดับ เฟสละ 1 ครั้ง เพื่อให้ได้โค้ดที่ควบคุมคุณภาพและตรวจสอบได้ในแต่ละขั้น

---

## ภาพรวมเฟสการพัฒนา

| เฟส | ชื่อ | เป้าหมาย |
|---|---|---|
| 0 | Setup โปรเจกต์ | โครงสร้างโปรเจกต์ Node.js พร้อม dependencies |
| 1 | ฐานข้อมูล & Data Model | ออกแบบตาราง items, transactions, suppliers |
| 2 | Backend API - Master Data | CRUD สินค้า/ผู้ส่งมอบ |
| 3 | Backend API - รับสินค้า (Receiving) | บันทึกรับเข้า + ตัดยอด |
| 4 | Backend API - จ่ายสินค้า (Picking/Shipping) | บันทึกจ่ายออก + ตรวจสอบสต็อกไม่ติดลบ |
| 5 | Real-time Dashboard | WebSocket แสดงสต็อกเรียลไทม์ |
| 6 | Frontend - หน้าสแกนบาร์โค้ด | UI สแกนผ่านกล้องมือถือ |
| 7 | Frontend - Dashboard UI | กราฟ/การ์ดแสดง KPI |
| 8 | รายงานและ KPI | Export Excel/PDF, คำนวณ accuracy, turnover |
| 9 | Testing & Deployment | ทดสอบ, เขียน README, เตรียม deploy |

---

## เฟส 0: Setup โครงสร้างโปรเจกต์

```
สร้างโปรเจกต์ Node.js สำหรับระบบ Warehouse Management System (WMS) ขนาดเล็ก
ชื่อโปรเจกต์ "smartstock-wms"

ข้อกำหนด:
- ใช้ Express.js เป็น backend framework
- ใช้ SQLite (ผ่าน better-sqlite3 หรือ Prisma) เป็นฐานข้อมูลเริ่มต้น
- โครงสร้างโฟลเดอร์แบบแยกส่วนชัดเจน: /src/routes, /src/controllers, /src/models, /src/db, /public
- ตั้งค่า nodemon สำหรับ dev mode
- สร้างไฟล์ .env.example สำหรับ config (PORT, DB_PATH)
- สร้าง package.json พร้อม scripts: "dev", "start", "test"
- เขียน README.md อธิบายวิธีติดตั้งและรันโปรเจกต์เบื้องต้น

ให้สร้างเฉพาะโครงสร้างพื้นฐานก่อน ยังไม่ต้องเขียน business logic
```

---

## เฟส 1: ฐานข้อมูลและ Data Model

```
ออกแบบและสร้างฐานข้อมูลสำหรับระบบ WMS โดยใช้ SQLite

ให้สร้างตารางดังนี้:

1. suppliers (ผู้ส่งมอบ)
   - id, name, contact, phone, created_at

2. items (สินค้า)
   - id, sku (unique), name, category, unit, reorder_point, quantity_on_hand, location, created_at, updated_at

3. transactions (ประวัติรับ-จ่าย)
   - id, item_id (FK), type ('IN' หรือ 'OUT'), quantity, supplier_id (FK, nullable),
     reference_no, note, created_at

4. stock_counts (ตรวจนับสต็อก สำหรับคำนวณ accuracy)
   - id, item_id (FK), counted_quantity, system_quantity, counted_at

ข้อกำหนดเพิ่มเติม:
- เขียน migration script หรือ schema.sql สำหรับสร้างตารางทั้งหมด
- เขียน seed script สำหรับใส่ข้อมูลตัวอย่าง (สินค้า 10 รายการ, ผู้ส่งมอบ 3 ราย)
- สร้าง connection/db helper module ที่ reusable ทั่วทั้งโปรเจกต์
- เพิ่ม index ที่ sku และ item_id เพื่อ performance
```

---

## เฟส 2: Backend API - จัดการข้อมูลหลัก (Master Data)

```
สร้าง REST API สำหรับจัดการข้อมูลหลัก (Master Data) ของระบบ WMS

Endpoints ที่ต้องการ:

Items:
- GET /api/items — ดึงรายการสินค้าทั้งหมด (รองรับ query filter/search ด้วยชื่อหรือ sku)
- GET /api/items/:id — ดึงข้อมูลสินค้ารายตัว
- POST /api/items — เพิ่มสินค้าใหม่
- PUT /api/items/:id — แก้ไขข้อมูลสินค้า
- DELETE /api/items/:id — ลบสินค้า (soft delete)

Suppliers:
- GET /api/suppliers
- POST /api/suppliers
- PUT /api/suppliers/:id
- DELETE /api/suppliers/:id

ข้อกำหนด:
- ใช้ Express Router แยกไฟล์ตาม resource
- Validate input ด้วย library เช่น zod หรือ express-validator
- ตอบกลับเป็น JSON format มาตรฐาน { success, data, error }
- จัดการ error ด้วย middleware กลาง (centralized error handler)
- เขียน unit test เบื้องต้นด้วย Jest หรือ Vitest สำหรับ endpoint สำคัญ
```

---

## เฟส 3: Backend API - รับสินค้าเข้า (Receiving)

```
สร้าง API สำหรับกระบวนการรับสินค้าเข้าคลัง (Receiving) ตามหลักการ WMS

Endpoint:
- POST /api/receiving
  รับ payload: { item_id, quantity, supplier_id, reference_no, note }
  การทำงาน:
    1. ตรวจสอบว่า item_id มีอยู่จริง
    2. บันทึกลงตาราง transactions โดย type = 'IN'
    3. อัปเดต quantity_on_hand ในตาราง items (+quantity) แบบ transaction-safe
       (ใช้ database transaction ป้องกันข้อมูลไม่สอดคล้องกัน)
    4. คืนค่าผลลัพธ์พร้อมยอดคงเหลือล่าสุด

- GET /api/receiving/history — ดึงประวัติการรับสินค้า (filter ตามวันที่, item, supplier ได้)

ข้อกำหนดเพิ่มเติม:
- ป้องกัน race condition กรณีมีการรับสินค้าพร้อมกันหลาย request
- เขียน log ทุกการทำรายการ พร้อม timestamp และ reference_no ที่ generate อัตโนมัติถ้าไม่ได้ส่งมา
- เขียน test case: รับสินค้าสำเร็จ, รับสินค้าด้วย item_id ที่ไม่มีอยู่ (ต้อง error)
```

---

## เฟส 4: Backend API - จ่ายสินค้าออก (Picking/Shipping)

```
สร้าง API สำหรับกระบวนการจัดเตรียมและจัดส่งสินค้าออกจากคลัง (Picking/Shipping)

Endpoint:
- POST /api/shipping
  รับ payload: { item_id, quantity, reference_no, note }
  การทำงาน:
    1. ตรวจสอบว่า quantity_on_hand เพียงพอหรือไม่ ถ้าไม่พอให้ error พร้อมข้อความชัดเจน
    2. บันทึกลงตาราง transactions โดย type = 'OUT'
    3. อัปเดต quantity_on_hand (-quantity) แบบ transaction-safe
    4. ถ้ายอดคงเหลือหลังตัด <= reorder_point ให้ flag "low_stock: true" ในผลลัพธ์

- GET /api/shipping/history — ดึงประวัติการจ่ายสินค้าออก

- GET /api/items/low-stock — ดึงรายการสินค้าที่ใกล้หมด (quantity_on_hand <= reorder_point)

ข้อกำหนดเพิ่มเติม:
- ห้ามให้ quantity_on_hand ติดลบเด็ดขาด
- เขียน test case: จ่ายสินค้าสำเร็จ, จ่ายเกินจำนวนที่มี (ต้อง error), กรณี trigger low stock
```

---

## เฟส 5: Real-time Dashboard ด้วย WebSocket

```
เพิ่มความสามารถ Real-time ให้กับระบบ WMS โดยใช้ Socket.IO

ข้อกำหนด:
- ติดตั้งและตั้งค่า Socket.IO บน server เดียวกับ Express
- ทุกครั้งที่มีการรับสินค้า (เฟส 3) หรือจ่ายสินค้า (เฟส 4) สำเร็จ
  ให้ emit event "stock:update" พร้อมข้อมูล { item_id, quantity_on_hand, sku, name }
  ไปยัง client ทุกตัวที่เชื่อมต่ออยู่
- สร้าง event "stock:low" แยกต่างหาก เมื่อสินค้ารายการใดต่ำกว่า reorder_point
- เขียนตัวอย่าง client-side listener (JavaScript) สำหรับทดสอบการรับ event
- อธิบายวิธีทดสอบด้วย Postman/socket.io client ใน README
```

---

## เฟส 6: Frontend - หน้าสแกนบาร์โค้ด

```
สร้างหน้าเว็บสำหรับสแกนบาร์โค้ดรับ-จ่ายสินค้า โดยใช้ HTML/JavaScript ธรรมดา (Vanilla JS)
ทำงานร่วมกับ backend Express ที่มีอยู่แล้ว

ข้อกำหนด:
- ใช้ library "html5-qrcode" หรือ "quaggaJS" อ่านบาร์โค้ดผ่านกล้องมือถือ/เว็บแคม
- หน้าจอมี 2 โหมด: "รับสินค้าเข้า" และ "จ่ายสินค้าออก" (สลับด้วยปุ่ม toggle)
- เมื่อสแกนได้ SKU ให้เรียก GET /api/items?sku=xxx เพื่อดึงชื่อสินค้ามาแสดงยืนยัน
- มีช่องกรอกจำนวน แล้วกดยืนยันเพื่อเรียก POST /api/receiving หรือ /api/shipping ตามโหมดที่เลือก
- แสดงผลลัพธ์ทันที (สำเร็จ/ไม่สำเร็จ) พร้อมยอดคงเหลือล่าสุด
- รองรับกรณีกล้องใช้งานไม่ได้ ให้มีช่องกรอก SKU ด้วยมือ (manual input) เป็นทางเลือกสำรอง
- ออกแบบ UI ให้ใช้งานง่ายบนมือถือ (mobile-first, ปุ่มใหญ่ กดง่าย)
```

---

## เฟส 7: Frontend - Dashboard แสดงผล KPI

```
สร้างหน้า Dashboard แสดงข้อมูลคลังสินค้าแบบเรียลไทม์ โดยใช้ HTML/CSS/JavaScript
เชื่อมต่อกับ Socket.IO ที่สร้างไว้ในเฟส 5

ข้อกำหนด:
- แสดงการ์ดสรุป KPI (อ้างอิงรูปแบบจากเอกสารบริษัท SmartStock):
  1. จำนวนสินค้าทั้งหมด (total items)
  2. รายการสินค้าใกล้หมด (low stock count) — เชื่อม event "stock:low"
  3. ยอดรับเข้าวันนี้ (today's receiving total)
  4. ยอดจ่ายออกวันนี้ (today's shipping total)
  5. ความแม่นยำสต็อก (stock accuracy %) — คำนวณจากตาราง stock_counts
- ตารางแสดงสินค้าทั้งหมดพร้อมยอดคงเหลือ อัปเดตแบบเรียลไทม์เมื่อมี event "stock:update"
- ไฮไลต์แถวสีแดง/เหลืองสำหรับสินค้าที่ต่ำกว่า reorder_point
- ใช้ Chart.js แสดงกราฟแนวโน้มการรับ-จ่ายสินค้าย้อนหลัง 7 วัน
- ไม่ต้องใช้ framework หนัก (ไม่ใช้ React) เพื่อให้ deploy ง่ายและเบา
```

---

## เฟส 8: รายงานและการคำนวณ KPI

```
สร้างโมดูลรายงานสำหรับระบบ WMS ตามตัวชี้วัดในเอกสารบริษัท

Endpoints:
- GET /api/reports/accuracy — คำนวณ % ความแม่นยำสต็อก
  จากสูตร: (จำนวนรายการที่ counted_quantity ตรงกับ system_quantity) / (จำนวนรายการที่นับทั้งหมด) x 100

- GET /api/reports/turnover — คำนวณอัตราการหมุนเวียนสินค้า (stock turnover rate)
  จากยอดรวมการจ่ายออกในช่วงเวลาที่กำหนด หารด้วยยอดคงเหลือเฉลี่ย

- GET /api/reports/space-utilization — ประเมินการใช้พื้นที่คลัง
  (รับค่าพื้นที่ทั้งหมดจาก config แล้วคำนวณสัดส่วนตำแหน่งที่มีสินค้าจัดเก็บ)

- GET /api/reports/export?format=excel|pdf&type=receiving|shipping
  ใช้ library "exceljs" สำหรับ export Excel และ "pdfkit" สำหรับ export PDF

ข้อกำหนดเพิ่มเติม:
- ทุก endpoint รองรับ query parameter ช่วงวันที่ (from, to)
- จัดรูปแบบตัวเลขเปอร์เซ็นต์และหน่วยให้อ่านง่าย
- เขียนเอกสารอธิบายสูตรคำนวณแต่ละตัวใน README
```

---

## เฟส 9: Testing และเตรียม Deploy

```
ทำการทดสอบและเตรียมระบบ WMS สำหรับ deploy

ข้อกำหนด:
1. เขียน integration test ครอบคลุม flow หลัก: รับสินค้า → ตรวจสอบยอด → จ่ายสินค้า → ตรวจสอบยอด
2. เขียน test สำหรับ edge case: จ่ายเกินสต็อก, สินค้าไม่พบ, ข้อมูล input ผิดรูปแบบ
3. เพิ่ม middleware ด้าน security เบื้องต้น: helmet, rate-limiting, input sanitization
4. สร้างไฟล์ Dockerfile และ docker-compose.yml สำหรับ deploy ง่าย
5. เขียน README ฉบับสมบูรณ์ ประกอบด้วย:
   - วิธีติดตั้งและรัน
   - โครงสร้าง API ทั้งหมด (endpoint list)
   - ตัวอย่างการใช้งานหน้าสแกนและ dashboard
   - แผนผังฐานข้อมูล (ER diagram แบบข้อความ)
6. สรุปรายการสิ่งที่ควรทำต่อในเฟสถัดไป (roadmap) เช่น เชื่อม RFID, เชื่อม ERP, ทำ multi-warehouse
```

---

## หมายเหตุการใช้งาน

- ใช้ Prompt ทีละเฟสตามลำดับ อย่าข้ามเฟส เพราะแต่ละเฟสอ้างอิงโครงสร้างจากเฟสก่อนหน้า
- หลังจบแต่ละเฟส ควรทดสอบให้ทำงานถูกต้องก่อนไปเฟสถัดไป
- สามารถปรับแต่ง Prompt เพิ่มเติมได้ตามความต้องการเฉพาะของหน้างานจริง เช่น เพิ่มระบบ login/สิทธิ์ผู้ใช้ในเฟสเสริม
