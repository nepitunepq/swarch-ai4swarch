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

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| `frontend` → `cartservice` | `kubernetes-manifests/frontend.yaml` (`CART_SERVICE_ADDR=cartservice:7070`) |✅  | พบ CART_SERVICE_ADDR ใน frontend.yaml บรรทัด 73 ค่าตรงกับที่ AI อ้าง|
| `checkoutservice` → `paymentservice` | `kubernetes-manifests/checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR=paymentservice:50051`) | ✅ | พบ PAYMENT_SERVICE_ADDR ใน checkoutservice.yaml บรรทัด 61 ค่าตรงกับที่ AI อ้าง|
| `checkoutservice` → `emailservice` | `kubernetes-manifests/checkoutservice.yaml` (`EMAIL_SERVICE_ADDR=emailservice:5000`) | ✅ | EMAIL_SERVICE_ADDR ใน checkoutservice.yaml บรรทัด 63 ค่าตรงกับที่ AI อ้าง|
| `recommendationservice` → `productcatalogservice` | `kubernetes-manifests/recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550`) |✅  | พบ PRODUCT_CATALOG_SERVICE_ADDR ใน recommendationservice.yaml บรรทัด 60 ค่าตรงกับที่ AI อ้าง|
| `cartservice` → `redis-cart` | `kubernetes-manifests/cartservice.yaml` (`REDIS_ADDR=redis-cart:6379`) | ✅ | REDIS_ADDR ใน cartservice.yaml บรรทัด 50 ค่าตรงกับที่ AI อ้าง|

- **สิ่งที่ผิดหรือขาดหายไป** run b ขาด dependency shoppingassistantservice: frontend.yaml มี SHOPPING_ASSISTANT_SERVICE_ADDR แต่ไม่มี manifest ของมัน AI เลยตัดออกจากแผนภาพ

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

   Run A ไม่รู้ว่า service ไหนเรียก service ไหน และไม่มี `frontend`, `redis-cart` หรือข้อมูล port เลย เพราะไฟล์ `.proto` บอกแค่ว่าแต่ละ service ให้บริการอะไรได้บ้าง แต่ไม่บอกว่าใครเป็นผู้เรียก ข้อมูลว่าใครเรียกใครอยู่ในไฟล์ deploy ซึ่ง Run B ได้อ่าน จึงกู้คืน dependency ได้ครบกว่า

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

   `shoppingassistantservice` ไม่อยู่ในแผนภาพ ทั้งที่ `frontend.yaml` มีตัวแปรชี้ไปหามัน สมาชิกใหม่จะเข้าใจว่า `frontend` ไม่ได้คุยกับ service อื่นนอกเหนือจากที่วาดไว้ ถ้าเกิดปัญหาขึ้น เราจะตามหาไม่เจอว่าพังตรงไหน
