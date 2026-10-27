**ภาษา:** ภาษาไทย | [English](02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ประมาณ 20 นาที)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ *ครบถ้วนน้อยกว่ามนุษย์* ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
> **คำถาม:** ADR ที่ AI เขียนดีขึ้นแค่ไหน เมื่อให้ตัวอย่างและบริบทจริงแก่ AI

อ่าน [สถานการณ์ (ภาษาไทย)](../docs/scenario.th.md) ก่อน: ความต้องการ **R-12** ว่าด้วยสิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อสินค้า

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่าง โดยแทนที่ `<paste R-12 here>` ด้วยข้อความ R-12 ภาษาอังกฤษที่อยู่ในกรอบอ้างอิงของ scenario

**Run a: ไม่มีตัวอย่าง (zero-shot)** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์ | Run a | Run b |
|---|---|---|
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้ ระบุว่าไม่มีตัวเลข และย้ายไปไว้ใน Open questions | ใช้ เช่น p95 `PlaceOrder` ≤ 500 ms, peak 200 orders/s, MailFast p95 1.5 s / p99 8 s / fail 0.5% / rate limit 50 emails/s |
| ทางเลือกที่พิจารณา (ระบุ) | A) Synchronous orchestration B) Pub/sub ผ่าน message broker (ยังไม่เลือกเทคโนโลยี) C) Async fan-out ใน process → เลือก B | 1) Sync gRPC fan-out 2) In-process async (goroutine) 3) RabbitMQ events 4) Kafka events → เลือก RabbitMQ |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | แต้มสะสม: consumer ต้อง idempotent ใช้ order ID เป็น key / rate limit: retry + backoff + DLQ แต่ไม่รู้ตัวเลข limit | แต้มสะสม: dedupe ด้วย `order_id` (unique key) + manual ack หลัง commit / rate limit: `email-worker` จำกัด ≤ 50/s (prefetch + token bucket) และใช้ queue เป็น buffer |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | ไม่มีตัวเลขที่แต่งขึ้น เพราะย้ายสิ่งที่ไม่รู้ไปไว้ใน Open questions และติดป้าย [Assumption] / [Inferred] กำกับ | ภาษาของ `paymentservice` (Node.js) และ `frontend` (Go) ในแผนภาพไม่มีใน context; backlog "90,000 ฉบับ" สมมติเองว่า peak ยาว 10 นาที (scenario ระบุแค่ 200 คำสั่งซื้อ/วินาที ไม่ระบุความยาวของ peak); ตัวเลขอื่นในตาราง (500 ms, 200/s, MailFast 1.5 s / 8 s / 0.5% / 50/s, ทีม 2 คนเคยใช้ RabbitMQ) ตรงกับ scenario |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | แผนภาพ: เปลี่ยนเทคโนโลยีของ `frontend` จาก "Go" เป็น "HTTP server" และ `paymentservice` จาก "Node.js" เป็น "gRPC" และลบหมายเหตุ [Inferred] ใต้แผนภาพที่ไม่จำเป็นแล้ว | ภาษาของสอง service นี้ไม่มีในไฟล์ context ที่ AI ได้รับ AI ติดป้าย [Inferred] ไว้ใต้แผนภาพ แต่ในแผนภาพยังแสดงเหมือนเป็นข้อเท็จจริง คนอ่านอาจเข้าใจผิด |
| 2 | Consequences: เขียนกำกับตัวเลข backlog "90,000 ฉบับ" ว่าเป็น "ตัวอย่างสมมติ" ถ้า peak ยาว 10 นาที | scenario ระบุแค่ 200 คำสั่งซื้อ/วินาที ไม่ระบุว่าช่วงพีกยาวเท่าไร ตัวเลข 10 นาทีจึงเป็นการสมมติของ AI ไม่ใช่ข้อเท็จจริง |
| 3 | Decision: rate limit 50 emails/s ของ `email-worker` เปลี่ยนเป็นคุม "รวมทุก replica" โดยแบ่งโควตาให้แต่ละ replica (เช่น 2 replica ละ 25/s) | ร่างเดิมไม่ได้บอกว่าถ้ามีหลาย replica จะคุมรวมอย่างไร ถ้าแต่ละตัวส่ง 50/s อัตรารวมจะเกิน limit ของ MailFast เลือกแบ่งโควตาแทนการใช้ replica เดียว เพื่อให้ replica หนึ่งล่มแล้วอีเมลยังส่งต่อได้ |
