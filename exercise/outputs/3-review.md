ผู้รีวิว: AI assistant using Copilot SDK in VS Code | โมเดล: ไม่ทราบ

# ผลรีวิว ADR-0002: แยกงานหลังสั่งซื้อด้วย RabbitMQ

## Quality-attribute scenarios

| คุณภาพ | Source · stimulus · environment | Response · response measure |
|---|---|---|
| Performance / Availability | ลูกค้าส่งคำสั่งซื้อในช่วง 11.11 ที่ 200 รายการ/วินาที ขณะที่ MailFast ช้า (p95 1.5 วินาที, p99 8 วินาที) หรือล่ม | checkout ตอบรับโดยไม่รอ consumer; `PlaceOrder` มี p95 ≤ 500 ms และคำสั่งซื้อไม่ล้มเหลวเพราะ MailFast |
| Reliability / Correctness | RabbitMQ หรือ consumer หยุดชั่วคราว แล้ว broker ส่ง `OrderPlaced` ซ้ำระหว่าง retry | เหตุการณ์ไม่สูญหายหลังระบบกู้คืน และแต้มถูกเพิ่มครั้งเดียวต่อ `order_id` ภายใน 60 วินาทีภายใต้โหลด 200 รายการ/วินาที |
| Performance / Scalability | ในช่วงพีกมีคำสั่งซื้อ 200 รายการ/วินาที แต่ MailFast รับได้ไม่เกิน 50 อีเมล/วินาที | consumer จำกัดการส่งไม่เกิน 50/วินาทีและเก็บส่วนเกินใน backlog; ต้องมีเกณฑ์เวลาส่ง/อายุคิวที่ธุรกิจยอมรับ ซึ่ง ADR ยังไม่กำหนด |

## Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | เส้นทางส่งเหตุการณ์ไม่ชัด: ระบุทั้งว่า checkout ส่งผ่าน exchange และว่า outbox relay ส่งซ้ำ ขณะที่ผลทดสอบต้องให้คำสั่งซื้อสำเร็จเมื่อ RabbitMQ ล่ม | หาก checkout ต้องรอ publisher confirmation การล่ม/ช้าของ broker อาจทำให้คำสั่งซื้อล้มเหลวหรือใช้เวลาเกินงบ p95 500 ms; ควรระบุว่า request บันทึก order+outbox แบบ atomic แล้ว relay ส่งแบบ asynchronous |
| Risk | Fallback log เมื่อ outbox ใช้งานไม่ได้ไม่มีตำแหน่งจัดเก็บ ความทนทาน หรือวิธี reconcile ที่กำหนด | local log บนเครื่องอาจหายเมื่อ pod ถูกเปลี่ยน/รีสตาร์ต และถ้าเขียน outbox ไม่ได้ก็ยังไม่มีหลักประกันว่าจะกู้ event กลับมาได้ จึงขัดกับข้ออ้างว่าไม่สูญหาย |
| Sensitivity point | อัตราประมวลผลของ loyalty และระยะเวลาที่ broker/relay หยุด เป็นตัวกำหนดว่าจะทัน SLA 60 วินาทีหรือไม่ | ที่ 200 orders/s ต้องรับมือเหตุการณ์ loyalty อย่างน้อย 200 รายการ/วินาที; ADR ไม่มี capacity target หรือขอบเขต outage ที่ SLA ครอบคลุม จึงยังพิสูจน์ผลตามเป้าไม่ได้ |
| Trade-off point | การแยกงานด้วยคิวรักษา latency/availability ของ checkout แต่เปลี่ยนการตอบกลับทันทีเป็น eventual consistency และยอมให้อีเมลล่าช้า | MailFast รับได้น้อยกว่า peak 4 เท่า ทำให้ backlog เพิ่มได้ถึง 150 อีเมล/วินาทีหาก peak คงอยู่; การจำกัดอัตราช่วยรักษาโควตาแต่ไม่ได้กำหนดเวลาระบายคิว |
| Non-risk | ใช้ `order_id` เป็น idempotency key ร่วมกับ durable processing และ outbox เป็นทิศทางที่เหมาะกับการส่งซ้ำ | ช่วยแยกการรับประกัน “ผลลัพธ์แต้มไม่ซ้ำ” ออกจากการรับประกันส่งข้อความครั้งเดียว ซึ่ง broker/retry ไม่ควรถูกสมมติว่าให้ exactly-once เอง |

## Missing or weak

- ระบุขอบเขต transaction ให้ชัดว่า order และ outbox commit พร้อมกันอย่างไร รวมถึงความสัมพันธ์กับการเรียก payment/shipping ที่มีอยู่; หากทำ side effect เหล่านั้นไปแล้วแต่บันทึก order/outbox ไม่สำเร็จ จะกู้คืนหรือชดเชยอย่างไร
- แก้ความขัดแย้งระหว่างการ publish จาก checkout กับการ relay จาก outbox และกำหนดพฤติกรรมเมื่อฐานข้อมูล, broker หรือ relay ล่ม โดยไม่อ้าง fallback ที่ความทนทานยังไม่ระบุ
- กำหนด throughput/ขนาด backlog และเวลาระบายคิวสำหรับ loyalty ให้ตรวจสอบ SLA 60 วินาทีได้ รวมถึงขอบเขตของ SLA ระหว่าง outage
- ระบุ SLA อีเมลที่ธุรกิจยอมรับและนโยบาย backlog เมื่อ peak ต่อเนื่อง: 200/s เทียบกับเพดาน 50/s ทำให้คิวโต ไม่ใช่เพียงคิวชั่วคราวที่ token bucket แก้ได้
- การเลือก RabbitMQ มีเหตุผลตามทีมและรูปแบบโหลด แต่การเปรียบเทียบยังไม่แสดงเกณฑ์ด้าน HA/recovery และต้นทุนปฏิบัติการที่ใช้ตัดสินเทียบ Kafka; ข้อสรุปจึงยังขึ้นกับ open questions เหล่านี้

## Verdict

**Accept with changes.** ก่อนอนุมัติใช้งานจริง ให้:
1. กำหนด flow เป็น transactional outbox ที่ atomic กับการบันทึก order และให้ relay เป็นผู้ publish แบบ asynchronous; ระบุการจัดการความล้มเหลวและเลิกพึ่ง fallback log ที่ไม่ระบุ durability
2. แสดงหลักฐาน capacity/โหลดว่า `PlaceOrder` p95 ≤ 500 ms และ loyalty เสร็จภายใน 60 วินาทีที่ 200 orders/s รวมกรณี retry, backlog และ broker/consumer outage ที่กำหนดขอบเขตไว้
3. ให้ธุรกิจอนุมัติ email delay/backlog SLA พร้อมเกณฑ์เตือนและการกู้คิว เมื่อรับได้ 50/s แต่เกิดงานได้ 200/s
