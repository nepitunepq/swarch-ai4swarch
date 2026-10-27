> Agent: Claude Code (Claude desktop app) — Model: Claude Opus 5.5 (`claude-opus-5-5`)

# ADR-NNNN: ใช้ gRPC แบบ synchronous ระหว่าง service ทั้งหมด โดยมี contract กลางเพียงไฟล์เดียวคือ `demo.proto`

- **Status:** Reconstructed
- **Date:** 2026-10-27 (วันที่ reconstruct; วันที่ตัดสินใจจริง Not evidenced in the provided files — มีเพียง copyright 2018 ใน `main.go` และ 2020 ใน `demo.proto`)
- **Deciders:** Not evidenced in the provided files (มีเพียง header `Copyright ... Google LLC` ในทั้งสามไฟล์)

> หมายเลข ADR ยังไม่กำหนด เพราะไม่ได้รับอนุญาตให้อ่าน ADR อื่นใน `docs/adr/`

## Context and problem

ข้อเท็จจริงจาก evidence:

- `demo.proto` ไฟล์เดียว (`package hipstershop`) นิยาม **9 services**: `CartService`, `RecommendationService`, `ProductCatalogService`, `ShippingService`, `CurrencyService`, `PaymentService`, `EmailService`, `CheckoutService`, `AdService`
- ทุก RPC เป็นแบบ **unary** (request → response) ไม่มี `stream` เลยในไฟล์
- message หลายตัวถูกใช้ข้าม service: `Money` (Currency, Payment, Shipping, Product, Checkout), `Address` (Shipping, Checkout, Email), `CartItem` (Cart, Shipping, Email), `Empty` (Cart, ProductCatalog, Currency, Email)
- `frontend` รับ HTTP จากผู้ใช้ (`gorilla/mux`, `http.ListenAndServe`) แล้วเปิด gRPC client ไปยัง 7 services (`mustConnGRPC` × 7 ใน `frontend/main.go`)
- `checkoutservice` เป็น gRPC server (`pb.RegisterCheckoutServiceServer`, port `5050`) และเป็น client ของอีก 6 services (`mustConnGRPC` × 6)
- `PlaceOrder` เรียก downstream **ทีละตัวตามลำดับและรอผล**: `GetCart` → `GetProduct` + `Convert` ต่อสินค้าแต่ละชิ้น → `GetQuote` → `Convert` → `Charge` → `ShipOrder` → `EmptyCart` → `SendOrderConfirmation`
- ไม่มี message broker, queue หรือ event ใดๆ ใน evidence

ปัญหาที่ต้องตัดสินใจ: service หลายตัวต้องคุยกันอย่างไร (รูปแบบการสื่อสาร) และจะนิยาม/แชร์ interface contract ระหว่างกันอย่างไร

## Decision drivers

(ลำดับความสำคัญเป็น **[Inferred]** — ไม่มีเอกสารระบุ)

- **Interoperability / modifiability:** contract ต้องเป็นกลางทางภาษา เพราะ service อาจเขียนต่างภาษา [Inferred]
- **Simplicity / understandability:** ระบบเป็น demo ต้องอ่านง่าย flow ตามได้ทีละขั้น [Inferred]
- **Observability:** ต้อง trace request ข้าม service ได้ (`otelgrpc` ทั้งสองฝั่ง) [Evidenced]
- **Performance:** binary protocol, latency ต่ำ [Assumption]
- **Consistency ของ data model:** ใช้ `Money`/`Address` ชุดเดียวกันทุก service [Evidenced ว่าแชร์จริง; ว่าเป็น driver คือ Inferred]

## Considered options

1. **Synchronous gRPC + `demo.proto` กลางไฟล์เดียว** (สิ่งที่เห็นในโค้ด)
2. Synchronous gRPC + proto แยกต่อ service (แต่ละทีมเป็นเจ้าของ contract ของตัวเอง)
3. Synchronous REST/JSON over HTTP (เช่น OpenAPI ต่อ service)
4. Asynchronous messaging ผ่าน message broker สำหรับขั้นที่ไม่ต้องรอผล (เช่น email, empty cart)

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. gRPC + proto กลาง | contract ที่เดียว อ่านภาพรวมทั้งระบบได้ในไฟล์เดียว; type ร่วม (`Money`, `Address`) สอดคล้องกันทุกที่; generate stub ได้หลายภาษา; health check (`grpc_health_v1`) และ tracing (`otelgrpc`) ใช้แบบเดียวกัน | ทุก service ผูกกับไฟล์เดียว — แก้ message ร่วมกระทบทุกคน; ไม่มี ownership ชัด; ไม่มี versioning (package `hipstershop` ไม่มี `v1`); ต้อง regenerate stub ในทุก service (`checkoutservice/genproto`) |
| 2. gRPC + proto แยก | ownership ชัด, versioning ต่อ service ได้อิสระ | type ร่วมซ้ำซ้อนหรือต้องมี "common.proto" อีกชั้น; ภาพรวมกระจายหลายไฟล์ ไม่เหมาะกับ demo |
| 3. REST/JSON | debug ด้วย curl/browser ง่าย; คนรู้จักแพร่หลาย | ไม่มี schema บังคับโดย default; payload ใหญ่กว่า; ต้องเขียน client เองหรือ generate จาก OpenAPI |
| 4. Async broker | decouple เวลา: email/cart ล่มไม่ทำให้ checkout ช้า; รองรับ burst ได้ | เพิ่ม infrastructure (broker) และ eventual consistency; flow ยากต่อการอ่านและ trace; `PlaceOrder` ต้องคืน `OrderResult` ทันที จึงยังต้องมี sync บางส่วน |

## Decision

เลือก **Option 1**: ทุก service สื่อสารด้วย gRPC แบบ unary/synchronous โดยใช้ contract กลาง `demo.proto`

เหตุผลที่ reconstruct ได้ (ส่วนใหญ่เป็น **[Inferred]**):

- ระบบเป็น reference/demo — flow แบบ request/response ทีละขั้นใน `PlaceOrder` อ่านเข้าใจง่ายที่สุด [Inferred]
- proto ไฟล์เดียวทำให้ type ร่วม (`Money` ที่มี `units`/`nanos`) ถูกนิยามครั้งเดียวและตรงกันทุก service [Evidenced ว่ามีการใช้ร่วม; เหตุผล Inferred]
- gRPC ให้ health check และ trace propagation แบบมาตรฐานเดียวกันทุก service (`healthpb.RegisterHealthServer`, `otelgrpc.NewServerHandler/NewClientHandler`) [Evidenced]
- ความต้องการ contract ข้ามภาษา [Inferred — evidence แสดงเฉพาะ service ภาษา Go สองตัว ภาษาของ service อื่น Not evidenced in the provided files]

## Consequences

- **Positive:** contract อยู่ที่เดียว เพิ่ม client ใหม่ได้แค่ generate stub (quality attribute: modifiability, interoperability)
- **Positive:** trace context ส่งต่ออัตโนมัติผ่าน `otelgrpc` + `propagation.TraceContext{}` (quality attribute: observability)
- **Positive:** flow checkout เป็นลำดับตรงไปตรงมา ทดสอบ/debug ง่าย (quality attribute: understandability, testability)
- **Negative:** latency ของ `PlaceOrder` เป็นผลรวมของทุก call ตามลำดับ และโตตามจำนวนสินค้า (`GetProduct` + `Convert` ต่อชิ้นใน `prepOrderItems`) (quality attribute: performance)
- **Negative:** availability ของ checkout ขึ้นกับ downstream ทั้ง 6 ตัว — `Cart`, `ProductCatalog`, `Currency`, `Shipping`, `Payment` ล่มตัวใดตัวหนึ่ง order ล้มทันที (quality attribute: availability)
- **Negative:** แก้ message ร่วมใน `demo.proto` ต้อง regenerate และ deploy หลาย service พร้อมกัน; ไม่มี version ใน package (quality attribute: modifiability, deployability)
- **Negative:** ใช้ `insecure.NewCredentials()` ทุก connection ขณะที่ `CreditCardInfo` (`credit_card_number`, `credit_card_cvv`) วิ่งผ่าน gRPC (quality attribute: security)
- **Risks and mitigations:**
  - ไม่มี deadline/timeout ต่อ call: `context.WithTimeout` ใน `mustConnGRPC` ไม่ถูกนำไปใช้ (`_, cancel := ...`) และ `convertCurrency` ใช้ `context.TODO()` แทน `ctx` → ช้าที่ downstream ลามมาถึง frontend. Mitigation: ส่ง `ctx` ที่มี deadline ทุก call [Evidenced ว่าขาด; mitigation เป็น Assumption]
  - ไม่มี retry/circuit breaker ใน evidence. Mitigation: gRPC retry policy หรือ service mesh [Assumption]
  - ตัด card แล้ว (`Charge`) แต่ `ShipOrder` ล้ม → คืน `codes.Unavailable` โดยไม่มี compensation/refund [Evidenced ว่าไม่มี compensation ในโค้ด]
  - email และ empty cart เป็น soft dependency อยู่แล้ว (`_ = cs.emptyUserCart(...)`, `log.Warnf` เมื่อ email ล้ม) — เป็นจุดที่ย้ายไป async (Option 4) ได้ง่ายที่สุด [Inferred]

## Validation

Not evidenced in the provided files (ไม่มี load test หรือ SLO ใน evidence) แนวทางที่เสนอ [Inferred]:

- วัด p95/p99 latency ของ `PlaceOrder` เทียบกับจำนวนสินค้าในตะกร้า ผ่าน trace จาก `otelgrpc`
- chaos test: ปิด `currencyservice` หรือ `emailservice` แล้วดูผลต่ออัตรา checkout สำเร็จ
- ตรวจ breaking change ของ `demo.proto` ใน CI (เช่น `buf breaking`) [Assumption]

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | `demo.proto` ไฟล์เดียวนิยาม 9 services ใน `package hipstershop` | Evidenced | `protos/demo.proto` (`package hipstershop`; `service CartService` … `service AdService`) |
| 2 | ทุก RPC เป็น unary ไม่มี streaming | Evidenced | `protos/demo.proto` (ไม่มี keyword `stream` ในทุก `rpc`) |
| 3 | message `Money`, `Address`, `CartItem`, `Empty` ถูกใช้ข้ามหลาย service | Evidenced | `protos/demo.proto` (`Product.price_usd`, `GetQuoteResponse.cost_usd`, `ChargeRequest.amount`, `PlaceOrderRequest.address`, `OrderResult`) |
| 4 | `frontend` รับ HTTP จากผู้ใช้และเป็น gRPC client ของ 7 services | Evidenced | `src/frontend/main.go` (`mux.NewRouter`, `http.ListenAndServe`, `mustConnGRPC` × 7) |
| 5 | `checkoutservice` เป็น gRPC server และ client ของ 6 services | Evidenced | `src/checkoutservice/main.go` (`pb.RegisterCheckoutServiceServer`, `mustConnGRPC` × 6) |
| 6 | `PlaceOrder` เรียก downstream ตามลำดับแบบ synchronous | Evidenced | `src/checkoutservice/main.go` (`PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`) |
| 7 | ไม่มี message broker/async messaging | Evidenced (เฉพาะใน 3 ไฟล์นี้) | ไม่มี import หรือ client ของ broker ใน `checkoutservice/main.go`, `frontend/main.go` |
| 8 | ที่อยู่ service มาจาก env var และ panic ถ้าไม่ตั้ง | Evidenced | `mustMapEnv` ทั้งสองไฟล์ (`CART_SERVICE_ADDR` ฯลฯ) |
| 9 | gRPC health check ถูก register | Evidenced | `src/checkoutservice/main.go` (`healthpb.RegisterHealthServer`) |
| 10 | trace ข้าม service ด้วย `otelgrpc` และ W3C TraceContext | Evidenced | ทั้งสองไฟล์ (`otelgrpc.NewClientHandler`, `otelgrpc.NewServerHandler`, `propagation.TraceContext{}`) |
| 11 | ทุก connection ไม่มี TLS | Evidenced | `mustConnGRPC` (`insecure.NewCredentials()`) |
| 12 | ข้อมูลบัตรเครดิตวิ่งผ่าน gRPC | Evidenced | `protos/demo.proto` (`CreditCardInfo`, `ChargeRequest`, `PlaceOrderRequest.credit_card`) |
| 13 | ไม่มี per-call deadline; `convertCurrency` ใช้ `context.TODO()` | Evidenced | `src/checkoutservice/main.go` (`mustConnGRPC`: `_, cancel :=`; `convertCurrency`) |
| 14 | latency ของ checkout โตตามจำนวนสินค้า | Evidenced | `prepOrderItems` (loop `GetProduct` + `convertCurrency` ต่อ item) |
| 15 | downstream ตัวใดล้ม (ยกเว้น email/empty cart) ทำให้ order ล้ม | Evidenced | `PlaceOrder` (คืน `status.Errorf` ทุก error ยกเว้น `_ = cs.emptyUserCart`, `log.Warnf` ของ email) |
| 16 | ไม่มี compensation เมื่อ charge แล้วแต่ ship ล้ม | Evidenced | `PlaceOrder` (`chargeCard` แล้ว `shipOrder` คืน `codes.Unavailable` ไม่มี refund) |
| 17 | stub ถูก generate แยกต่อ service | Evidenced (เฉพาะ checkout) | `src/checkoutservice/main.go` (import `.../src/checkoutservice/genproto`) |
| 18 | Go service ไม่แชร์ package โค้ดกัน | Evidenced | comment `TODO(ahmetb) ... since they are not sharing packages` ใน `initProfiling` |
| 19 | แก้ `demo.proto` ต้อง regenerate/deploy หลาย service | Inferred | ตาม claim 3 + 17; ขั้นตอน build/deploy Not evidenced |
| 20 | เลือก gRPC เพราะ service เขียนหลายภาษา | Inferred | evidence แสดงแค่ Go; ภาษาอื่น Not evidenced |
| 21 | เลือก sync ทีละขั้นเพราะเป็น demo ที่ต้องอ่านง่าย | Inferred | ไม่มีเอกสาร rationale |
| 22 | proto ไฟล์เดียวเพื่อให้ type ร่วมสอดคล้องกัน | Inferred | การใช้ร่วมเป็น Evidenced (claim 3) แต่เจตนาไม่มีบันทึก |
| 23 | gRPC (HTTP/2 + protobuf) มี latency/payload ต่ำกว่า REST/JSON | Assumption | ความรู้ทั่วไป ไม่ได้วัดในระบบนี้ |
| 24 | ข้อดี/ข้อเสียของ Option 2–4 (ownership, debug ง่าย, decouple เวลา, eventual consistency) | Assumption | ความรู้ทั่วไปด้านสถาปัตยกรรม |
| 25 | email และ empty cart ย้ายไป async ได้ง่ายที่สุด | Inferred | อิงจากการที่โค้ดไม่สนใจ error ของสองขั้นนี้ (claim 15) |
| 26 | mitigation: deadline, retry policy, service mesh, `buf breaking` | Assumption | แนวปฏิบัติทั่วไป ไม่มีใน evidence |
| 27 | วันที่ตัดสินใจและผู้ตัดสินใจ | Not evidenced in the provided files | มีเพียง copyright `Google LLC` 2018/2020 |
| 28 | `shoppingAssistantSvcAddr` ถูกอ่านแต่ไม่มี gRPC connection (protocol ไม่ทราบ) | Evidenced (ว่าไม่มี conn) | `src/frontend/main.go` (`mustMapEnv(&svc.shoppingAssistantSvcAddr, ...)` ไม่มี `mustConnGRPC` คู่กัน) |
