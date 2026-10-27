> Agent: Claude Code (Claude Opus 5.5, `claude-opus-5-5`) — run `a` (zero-shot), context: `none`

# ADR-0002: Publish an "OrderPlaced" event to a message broker for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** Not evidenced in the provided files (ดู Open questions)

## Context and problem

หลังจากลูกค้าสั่งซื้อสำเร็จ Online Boutique ต้องทำงานหลังการสั่งซื้อ (post-order actions) ใหม่ 3 อย่าง (ตาม requirement):

| # | Action | ผู้รับผิดชอบ | หมายเหตุ |
|---|---|---|---|
| 1 | ส่งอีเมลยืนยันคำสั่งซื้อ | MailFast (external email provider) | แทนที่ logic แบบ mock ของ `emailservice` |
| 2 | ให้คะแนนสะสม (loyalty points) | `loyaltyservice` (ยังไม่ได้สร้าง) | service ใหม่ |
| 3 | บันทึก analytics event "order placed" | ทีม marketing | ปลายทางของ event ยังไม่ระบุ |

- Marketing คาดว่า **ปีหน้าจะมี post-order action เพิ่มอีก** (SMS, fraud scoring, การอัปเดต recommendations)
- ปัญหาที่ต้องตัดสินใจ: **service ที่ place order จะส่งต่องานหลังการสั่งซื้อไปยังผู้รับหลายรายอย่างไร** ให้เพิ่มผู้รับรายใหม่ได้ง่าย และไม่ให้ความล้มเหลวของผู้รับรายใดรายหนึ่ง (เช่น MailFast ล่ม) ทำให้การสั่งซื้อล้มเหลว
- **[Assumption]** ผู้ที่ place order และเรียก `emailservice` ในปัจจุบันคือ `checkoutservice` ผ่าน gRPC แบบ synchronous — งานนี้ไม่ได้อ่านไฟล์ใดใน `system/` (context: `none`) จึง Not evidenced in the provided files
- ไม่มีตัวเลขเรื่องปริมาณคำสั่งซื้อ, latency เป้าหมาย, SLA ของ MailFast หรือข้อกำหนดด้านการส่งซ้ำ (ดู Open questions)

## Decision drivers

1. **Modifiability / extensibility** — เพิ่ม post-order action ใหม่ (SMS, fraud scoring, recommendations) ได้โดยไม่ต้องแก้ service ที่ place order ทุกครั้ง (มาจาก requirement โดยตรง)
2. **Availability ของ checkout** — external provider (MailFast) หรือ service ใหม่ (`loyaltyservice`) ล่มหรือช้า ต้องไม่ทำให้การสั่งซื้อล้มเหลว **[Inferred]**
3. **Reliability ของการส่งต่อ** — action ต้องไม่หายไปเงียบ ๆ (อีเมลต้องถูกส่ง, คะแนนต้องถูกให้) และต้องจัดการการส่งซ้ำ (duplicate) ได้ **[Inferred]**
4. **Performance (checkout latency)** — การเพิ่ม action ไม่ควรทำให้ latency ของการสั่งซื้อเพิ่มตามจำนวน action **[Inferred]**
5. **Cost / operational complexity** — โครงสร้างพื้นฐานใหม่ต้องมีคนดูแลได้ (ทักษะของทีมยังไม่ทราบ)

## Considered options

1. **Option A — Synchronous orchestration:** service ที่ place order เรียกผู้รับแต่ละรายโดยตรงทีละตัว (gRPC/HTTP) หลังบันทึกคำสั่งซื้อ
2. **Option B — Publish/subscribe ผ่าน message broker:** service ที่ place order publish event `OrderPlaced` หนึ่งครั้งไปยัง message broker; email (adapter ไป MailFast), `loyaltyservice` และ analytics แต่ละตัว subscribe แยกกัน
3. **Option C — Asynchronous fan-out ภายใน process:** service ที่ place order ยิงคำขอไปยังผู้รับแต่ละรายแบบ async (background goroutine/thread + retry) โดยไม่มี broker

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A — Synchronous orchestration | • ง่าย ไม่มี infrastructure ใหม่<br>• รู้ผลทันทีว่าแต่ละ action สำเร็จหรือไม่<br>• debug/trace ง่าย (call chain เดียว) | • latency ของ checkout = ผลรวมของทุก call และเพิ่มขึ้นทุกครั้งที่เพิ่ม action<br>• MailFast หรือ `loyaltyservice` ล่ม → checkout ล้มหรือต้องเขียน error handling เองทุกจุด<br>• ทุก action ใหม่ต้องแก้ code และ deploy service ที่ place order (coupling สูง) |
| B — Pub/sub ผ่าน message broker | • เพิ่มผู้รับใหม่ได้ด้วยการเพิ่ม subscriber โดยไม่แก้ publisher<br>• checkout ไม่ต้องรอ action ใด ๆ; ผู้รับล่มได้โดย event ค้างใน queue รอ retry<br>• ผู้รับแต่ละรายสเกลและ retry ได้อิสระ<br>• ทีม marketing ใช้ event เดียวกันทำ analytics ได้ | • ต้องติดตั้งและดูแล message broker (cost, ops)<br>• eventual consistency: อีเมล/คะแนนมาถึงช้ากว่าการสั่งซื้อ<br>• ต้องออกแบบ idempotency (at-least-once delivery) และ dead-letter queue<br>• ต้องมี event schema ที่ตกลงร่วมกันและควบคุมเวอร์ชัน<br>• ต้องจัดการปัญหา "บันทึก order สำเร็จแต่ publish ล้มเหลว" (เช่น outbox pattern) |
| C — Async fan-out ใน process | • checkout ไม่ต้องรอผู้รับ<br>• ไม่มี infrastructure ใหม่ | • งานหายเมื่อ pod restart/crash (ไม่มี durable storage)<br>• publisher ยังต้องรู้จักผู้รับทุกรายและแก้ code ทุกครั้งที่เพิ่ม action<br>• retry/backoff ต้องเขียนเองใน service เดียว ทำให้ service ซับซ้อนขึ้น |

## Decision

เลือก **Option B — publish event `OrderPlaced` ไปยัง message broker** และให้แต่ละ post-order action เป็น subscriber อิสระ:

- service ที่ place order (**[Assumption]** `checkoutservice`) publish `OrderPlaced` หนึ่งครั้งหลังบันทึกคำสั่งซื้อสำเร็จ แทนการเรียก `emailservice` โดยตรง
- subscriber ชุดแรก: (1) email consumer ที่เรียก MailFast (แทน logic แบบ mock ของ `emailservice`), (2) `loyaltyservice`, (3) analytics consumer สำหรับทีม marketing
- action ในอนาคต (SMS, fraud scoring, recommendations) เพิ่มเป็น subscriber ใหม่ได้โดยไม่แก้ publisher
- เทคโนโลยี broker (เช่น RabbitMQ, Kafka, หรือ managed service ของ cloud) **ยังไม่เลือก** — ขึ้นกับ Open questions ด้านปริมาณงาน, การ replay event และทักษะของทีม

เหตุผล: Option B ตอบ driver อันดับ 1 (modifiability) และ 2 (availability) ได้ดีที่สุด เพราะ requirement ระบุชัดว่าจะมี action เพิ่มอีก และมีผู้รับภายนอก (MailFast) ที่เราควบคุม availability ไม่ได้ Option A ทำให้ coupling และ latency เพิ่มขึ้นตามจำนวน action; Option C ไม่ทนต่อ crash และยังมี coupling เท่าเดิม ราคาที่ต้องจ่ายคือ cost/ops ของ broker และ eventual consistency

**Fraud scoring:** หาก fraud scoring ในอนาคตต้อง **บล็อก** คำสั่งซื้อก่อนยืนยัน งานนี้ไม่ใช่ post-order action และต้องอยู่ในเส้นทาง synchronous ของ checkout — ควรแยกเป็น ADR ต่างหาก **[Inferred]**

## Consequences

- **Positive:** เพิ่ม post-order action ใหม่ได้ด้วยการเพิ่ม subscriber โดยไม่แก้ หรือ deploy service ที่ place order ใหม่ (quality attribute: modifiability)
- **Positive:** MailFast, `loyaltyservice` หรือ analytics ล่ม ไม่ทำให้ checkout ล้ม; event ค้างใน queue และ retry ภายหลัง (quality attribute: availability, fault isolation)
- **Positive:** latency ของ checkout ไม่เพิ่มตามจำนวน action; มีแค่ต้นทุนของการ publish หนึ่งครั้ง (quality attribute: performance)
- **Positive:** แต่ละ consumer สเกลได้อิสระตามปริมาณงานของตัวเอง (quality attribute: scalability)
- **Negative:** มี component ใหม่ (broker) ที่ต้อง deploy, monitor, backup และเสียค่าใช้จ่าย (quality attribute: cost, operability)
- **Negative:** อีเมลยืนยันและคะแนนสะสมเกิดขึ้นแบบ eventual ไม่ใช่ทันที; ลูกค้าอาจเห็นคะแนนช้ากว่าคำสั่งซื้อ (quality attribute: consistency, usability)
- **Negative:** การ trace คำสั่งซื้อหนึ่งรายการข้ามหลาย consumer ยากขึ้น ต้องส่ง correlation/trace ID ไปใน event (quality attribute: observability / testability)
- **Negative:** broker กลายเป็นจุดที่ทุก post-order action พึ่งพา (quality attribute: availability)
- **Risks and mitigations:**
  - *บันทึก order สำเร็จแต่ publish event ล้มเหลว → อีเมล/คะแนนหาย* → ใช้ transactional outbox หรือ retry การ publish พร้อม alert
  - *at-least-once delivery ทำให้ส่งอีเมลซ้ำ/ให้คะแนนซ้ำ* → consumer ต้อง idempotent โดยใช้ order ID เป็น idempotency key
  - *MailFast ปฏิเสธหรือ rate-limit* → retry แบบ exponential backoff และ dead-letter queue พร้อม alert
  - *event schema เปลี่ยนแล้ว consumer พัง* → กำหนด schema (เช่น protobuf) พร้อมกติกา backward compatibility
  - *broker ล่ม* → ใช้ broker แบบ replicated/managed และกำหนดพฤติกรรมของ publisher เมื่อ broker ไม่พร้อม (buffer ผ่าน outbox)
  - *ข้อมูลส่วนบุคคล (อีเมลลูกค้า) ไหลไปยัง analytics และ MailFast* → ส่งเฉพาะ field ที่จำเป็นให้แต่ละ consumer และตรวจข้อกำหนดด้าน privacy (ดู Open questions)

## Validation

- **Load/latency test:** วัด checkout latency ก่อนและหลังเปลี่ยนมา publish event; ไม่ควรเพิ่มขึ้นตามจำนวน subscriber (เกณฑ์ตัวเลข: ดู Open questions)
- **Fault-injection test:** ปิด MailFast mock และ `loyaltyservice` ระหว่างทดสอบ → checkout ต้องสำเร็จ และ action ต้องทำงานครบเมื่อระบบกลับมา
- **Duplicate-delivery test:** ส่ง `OrderPlaced` เดียวกันซ้ำ → ต้องได้อีเมล 1 ฉบับและคะแนน 1 ครั้ง
- **Metrics:** queue depth/consumer lag, จำนวนข้อความใน dead-letter queue, เวลาจากการสั่งซื้อถึงการส่งอีเมลสำเร็จ
- **Extensibility check:** เพิ่ม subscriber ทดลอง (เช่น SMS stub) โดยไม่แก้ publisher
- **Review date:** Not evidenced in the provided files — ทบทวนก่อนเพิ่ม action ชุดใหม่ของปีหน้า

## Open questions

- ใครคือ deciders ของ ADR นี้?
- service ใดเป็นผู้ place order และเรียก `emailservice` ในปัจจุบัน และใช้ protocol อะไร? (สมมติว่าเป็น `checkoutservice` ผ่าน gRPC แต่ไม่ได้ตรวจกับไฟล์)
- ปริมาณคำสั่งซื้อปัจจุบันและที่คาดไว้ (orders/second ช่วง peak) เท่าไร?
- checkout latency เป้าหมาย และ availability target ของ checkout เท่าไร?
- อีเมลยืนยันและคะแนนสะสมต้องมาถึงภายในกี่วินาที/นาทีหลังการสั่งซื้อ?
- MailFast มี API แบบใด (REST/SMTP), rate limit, SLA, webhook สำหรับสถานะการส่ง และรองรับ idempotency key หรือไม่?
- ปลายทางของ analytics event สำหรับทีม marketing คืออะไร (data warehouse, analytics tool ภายนอก)? ต้องการ replay event ย้อนหลังหรือไม่? (มีผลต่อการเลือก Kafka กับ queue แบบดั้งเดิม)
- ตอนนี้ระบบมี message broker อยู่แล้วหรือไม่? ทีมมีทักษะกับเทคโนโลยี broker ใด? งบประมาณสำหรับ infrastructure ใหม่เท่าไร?
- ระบบรันบน platform ใด (Kubernetes, cloud provider ใด) — ใช้ managed broker ได้หรือไม่?
- ข้อกำหนดด้าน privacy/PII สำหรับการส่งอีเมลลูกค้าไปยัง MailFast และข้อมูลคำสั่งซื้อไปยัง analytics คืออะไร?
- การสั่งซื้อถูกบันทึกลง data store หรือไม่ (จำเป็นสำหรับ transactional outbox)?
- fraud scoring ในอนาคตต้องบล็อกคำสั่งซื้อ (synchronous) หรือแค่ตรวจหลังการสั่งซื้อ?
- คะแนนสะสมต้องถูกหักคืนเมื่อมีการยกเลิก/คืนสินค้าหรือไม่ (ต้องมี event อื่น เช่น `OrderCancelled`)?
