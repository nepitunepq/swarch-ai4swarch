> Agent: Claude Code (Anthropic) · Model: Claude Opus 5.5 (`claude-opus-5-5`)

# ATAM review: ADR-0002 — `OrderPlaced` events via RabbitMQ

อ่านเฉพาะ: `docs/adr/0002-orderplaced-events-via-rabbitmq.md` และ `docs/scenario.md` (การอ้างอิง `main.go` ใน ADR ตรวจสอบไม่ได้จากไฟล์ที่อนุญาต)

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| S1 Performance | ลูกค้าช่วง 11.11 | 200 `PlaceOrder`/s | peak load; MailFast p95 1.5 s, p99 8 s | order สำเร็จ, publish `OrderPlaced` แทนการรอ MailFast | p95 `PlaceOrder` ≤ 500 ms (ปัจจุบันประมาณ 300 ms → budget สำหรับ publish ประมาณ 200 ms) |
| S2 Reliability | RabbitMQ node / `loyaltyservice` pod | crash ระหว่าง publish หรือก่อน ack | peak 200 orders/s | order ไม่ล้ม; event ไม่หาย; redelivery ถูก dedupe ด้วย `order_id` | order fail จากสาเหตุนี้ 0 %; loyalty points = 1 ครั้งต่อ `order_id`; ภายใน 60 s |
| S3 Throughput (external) | `email-worker` | backlog 150 อีเมล/s (200 เข้า − 50 ออก) | MailFast rate limit 50/s, fail 0.5 % | buffer ใน queue, retry + DLQ, ไม่เกิน rate limit | อีเมลออก ≤ 50/s รวมทุก replica; อีเมลหาย 0 ฉบับ; เวลารอสูงสุด: **ไม่ระบุใน scenario** |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| **Risk** | ถ้า publish ล้มเหลว ADR ให้ "log + เก็บไว้ retry" ใน `checkoutservice` ซึ่งไม่มี database → buffer อยู่ใน memory ของ pod (ปัญหาเดียวกับ Option 2 ที่ ADR เองปฏิเสธ) | pod restart = event หาย → loyalty ได้ **0 ครั้ง** ไม่ใช่ exactly once; S2 ไม่ผ่าน ฝั่ง producer เป็น at-most-once |
| **Risk** | โควตาอีเมลแบบคงที่ต่อ replica (เช่น 2 × 25/s) | ถ้า replica ตาย throughput เหลือ 25/s; ถ้า scale เพิ่มโดยไม่ปรับโควตาจะเกิน 50/s และโดน throttle → S3 อ่อนไหวต่อการ deploy |
| **Risk** | อีเมลเป็น at-least-once และยังไม่รู้ว่า MailFast รองรับ idempotency key | redelivery หลัง MailFast timeout (p99 8 s) → ลูกค้าได้อีเมลซ้ำ |
| **Risk** | RabbitMQ cluster 3 nodes + quorum queues สำหรับทีม 4 คนที่ "เคยใช้" 2 คน | broker กลายเป็น single point ของ post-order ทั้งหมด; ไม่มีแผน on-call / upgrade / capacity |
| **Sensitivity point** | publish timeout + publisher confirms | ค่า timeout กำหนดทั้ง p95 ของ S1 และโอกาสที่ event จะตกไปอยู่ใน buffer ที่ไม่ durable (S2) |
| **Sensitivity point** | ความยาวของ peak 11.11 | backlog โต 150/s → peak 10 นาทีได้ประมาณ 90,000 ฉบับ, drain ประมาณ 30 นาที; ADR ยอมรับว่าไม่ทราบ |
| **Trade-off point** | "order ต้องไม่ล้ม" vs "loyalty exactly once" | ถ้า publish ไม่สำเร็จ ต้องเลือกระหว่าง fail order (เสีย availability) หรือ commit order โดยไม่มี event (เสีย correctness) — ADR ยังไม่ตัดสิน |
| **Trade-off point** | RabbitMQ เทียบกับ sync call | ได้ performance/availability/modifiability แต่เสีย operability และ consistency (eventual, trace ผ่าน message) |
| **Non-risk** | queue แยกต่อ action | backlog อีเมลไม่ถ่วง loyalty → ช่วยให้ทัน 60 s |
| **Non-risk** | dedupe ด้วย `order_id` (unique constraint) + manual ack หลัง commit | รูปแบบมาตรฐานที่ทำให้ exactly-once effect ฝั่ง consumer ถูกต้อง |
| **Non-risk** | ปฏิเสธ Kafka | 200 msg/s ไม่ต้องการ Kafka และทีมไม่มีประสบการณ์ — เหตุผลสอดคล้องกับ scenario |

## 3. Missing or weak

- **Options ที่ไม่ได้พิจารณา:** transactional outbox (เพิ่ม store ให้ checkout) เป็น option หลักไม่ใช่แค่ risk; ให้ `emailservice` เดิมเป็น adapter ของ MailFast + queue ภายใน; ขอเพิ่ม rate limit กับ MailFast
- **Claim ที่ไม่มีหลักฐาน / อาจผิด:**
  - "เป็นทางเดียวที่ผ่าน driver 1–3" — driver 3 (exactly once, อีเมลไม่หาย) ยังไม่ผ่านเพราะปัญหาฝั่ง producer ข้างบน
  - "publish เร็ว" — ไม่มีตัวเลข; ADR เองยอมรับใน Open questions
  - "message รออยู่ใน queue จน consumer กลับมา" — จริงเฉพาะเมื่อ queue durable **และ** message persistent; ADR ระบุ durable queue แต่ไม่ระบุ persistent delivery
- **Consequences ที่ไม่ได้ระบุ:** อีเมลลูกค้า (PII) ถูกเก็บใน broker (security/privacy); schema versioning ของ `OrderPlaced` เมื่อเพิ่ม consumer ปีหน้า (modifiability); กลยุทธ์ retry/DLQ ของ loyalty (ถ้าติด DLQ จะเกิน 1 นาทีและไม่มีใครเห็น)
- **Validation:** ไม่มี chaos test สำหรับ **broker ล่มขณะ publish** ซึ่งเป็นกรณีเสี่ยงที่สุด (S2)

## 4. Verdict: **Accept with changes**

1. **กำหนดความ durable ฝั่ง producer:** เลือก outbox (เพิ่ม store ให้ `checkoutservice`) หรือระบุชัดว่ายอมให้ event หายได้ต่อ consumer ใด — และแก้ข้อความ "exactly once" ให้ตรงกับที่ออกแบบได้จริง
2. **ระบุ budget ของ publish:** ค่า timeout, confirm mode (sync/async), persistent delivery และวัด p95 ที่ 200 orders/s ก่อน approve
3. **ใช้ global rate limiter สำหรับ MailFast** แทนโควตาคงที่ต่อ replica พร้อมกำหนด SLA ความล่าช้าของอีเมลและ alert บนความยาว queue/DLQ
