**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์


- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ 🟩 / ❌ / ? | สิ่งที่นิสิตพบ |
| :--- | :--- | :---: | :--- |
| End user → Frontend | `kubernetes-manifests/frontend.yaml` | 🟩 | พบ Service `frontend-external` แบบ LoadBalancer บนพอร์ต 80 ตามที่ระบุ |
| Frontend → Product Catalog Service | `kubernetes-manifests/frontend.yaml`, `protos/demo.proto` | 🟩 | มี `PRODUCT_CATALOG_SERVICE_ADDR` ใน manifest และโค้ดเรียกจริงใน `frontend/rpc.go` (`getProducts`, `getProduct`) |
| Frontend → Currency Service | `kubernetes-manifests/frontend.yaml` | 🟩 | มี `CURRENCY_SERVICE_ADDR` และโค้ดเรียกใน `frontend/rpc.go` (`getCurrencies`, `convertCurrency`) |
| Frontend → Cart Service | `kubernetes-manifests/frontend.yaml` | 🟩 | มี `CART_SERVICE_ADDR` และโค้ดเรียกใน `frontend/rpc.go` (`getCart`, `insertCart`, `emptyCart`) |
| Frontend → Shopping Assistant endpoint | `kubernetes-manifests/frontend.yaml` | ? | dependency มีจริงในโค้ด: `frontend/handlers.go` (`chatBotHandler`) ส่ง HTTP POST ไปยัง `SHOPPING_ASSISTANT_SERVICE_ADDR` แต่ไม่มี manifest และไม่อยู่ใน `kustomization.yaml` จึงไม่ถูก deploy; AI วาดเป็น System_Ext ซึ่งผิด เพราะซอร์สอยู่ใน `src/shoppingassistantservice/` และ protocol คือ HTTP ไม่ใช่ "Not specified" |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B:
  - Shopping Assistant ถูกวาดเป็นระบบภายนอก (System_Ext) ทั้งที่เป็น service ของระบบเอง (`src/shoppingassistantservice/`) และ protocol จริงคือ HTTP POST
  - ขาด dependency ของ Shopping Assistant ไปยัง AlloyDB และ Gemini (`shoppingassistantservice.py`) เพราะ manifests ไม่มีข้อมูลนี้
  - Email Service ถูกบรรยายว่า "Sends order confirmations" แต่โค้ดรันแบบ dummy mode ซึ่งแค่ log ไม่ได้ส่งอีเมลจริง (`emailservice/email_server.py`, `start(dummy_mode = True)`)
  - ไม่ระบุภาษาของแต่ละ service (จริง ๆ คือ Go, C#, Python, Node.js, Java) เป็นข้อจำกัดของหลักฐาน ไม่ใช่ความผิดของ AI
  - Redis Cart ใช้ volume แบบ `emptyDir` (`cartservice.yaml`) ข้อมูลตะกร้าจะหายเมื่อ pod restart แต่แผนภาพไม่ได้บอก

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. **เปรียบเทียบ A กับ B: agent ไม่สามารถรู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด**
   ไฟล์ `.proto` บอกเพียง interface ของแต่ละ service แต่ไม่บอกว่าใครเรียกใคร service ใด deploy จริง และมี data store อะไรบ้าง Run A จึงไม่มี dependency เลย ไม่มี frontend ที่เป็นทางเข้าของผู้ใช้ และไม่มี Redis ส่วน Run B เห็นสิ่งเหล่านี้จากตัวแปร `*_SERVICE_ADDR` และ `REDIS_ADDR` ใน Kubernetes manifests

2. **ข้อผิดพลาดข้อใดข้อหนึ่งที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด**
   Email Service ในแผนภาพเขียนว่า "Sends order confirmations" แต่โค้ดจริงรันแบบ dummy ที่แค่ log ไม่ได้ส่งอีเมล สมาชิกใหม่อาจเข้าใจว่าระบบส่งอีเมลจริงอยู่แล้ว ซึ่งสำคัญมากกับ R-12 ที่ต้องเปลี่ยนไปใช้ MailFast
