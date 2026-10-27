**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** RAVIKAN CHAYAKUL / 6630283021
**agent ที่ใช้:** Claude Code (desktop app, แท็บ Code)
**โมเดล:** Claude Opus 5.5 (`claude-opus-5-5`)
**แพ็กเกจที่ใช้:** Claude Pro

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code / Opus 5.5 | `prompts/1-recover-architecture.md` run A, evidence: `protos/demo.proto` | `prompts/1-recover-architecture.md`, `protos/demo.proto` | อยู่ในขอบเขต | 3 | |
| 2 | 1-B | Claude Code / Opus 5.5 | `prompts/1-recover-architecture.md` run B, evidence: `protos/demo.proto`, `kubernetes-manifests/` | `prompts/1-recover-architecture.md`, `protos/demo.proto`, `kubernetes-manifests/` (README.md, kustomization.yaml และไฟล์ yaml ของ service 12 ไฟล์) | อยู่ในขอบเขต | 3 | |
| 3 | 2-a | Claude Code / Opus 5.5 | `prompts/2-write-adr.md` run a, context: none, requirement: R-12, language: Thai | `prompts/2-write-adr.md`, `docs/adr/template.md` | อยู่ในขอบเขต (output ระบุว่าไม่ได้อ่านไฟล์ใดใน `system/`) | 3 | |
| 4 | 2-b | Claude Code / Opus 5.5 | `prompts/2-write-adr.md` run b, context: `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `src/checkoutservice/main.go`, language: Thai | `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `src/checkoutservice/main.go` | อยู่ในขอบเขต | 3 | |
| 5 | 3 | Claude Code / Opus 5.5 | `prompts/3-review-adr.md`, adr: `docs/adr/0002-my-adr.md` (ฉบับก่อนแก้ ยังเหมือนร่าง 2b), language: Thai | `prompts/3-review-adr.md`, `docs/adr/0002-my-adr.md`, `docs/scenario.md` | อยู่ในขอบเขต | 3 | |
| 6 | (ขอคำแนะนำ) | Claude Code / Opus 5.5 | ถามว่าต้องกรอกอะไรในใบงาน และให้ดึงเนื้อหาจาก output ของ AI | AGENTS.md, ใบงานใน exercise/, outputs 1A/1B/2a/2b/3, docs/adr/0002-my-adr.md, prompts/2-write-adr.md, prompts/3-review-adr.md, Downloads/04-review-board.md | ไม่ใช่งานตาม prompts จึงไม่มีขอบเขตหลักฐาน | 3 | agent ไม่ได้แก้ใบงาน |
| 7 | 3 (รันใหม่) | Claude Code / Opus 5.5 | `prompts/3-review-adr.md`, adr: `docs/adr/0002-orderplaced-events-via-rabbitmq.md` (ฉบับที่แก้แล้ว), language: Thai | `prompts/3-review-adr.md`, `docs/adr/0002-orderplaced-events-via-rabbitmq.md`, `docs/scenario.md` | อยู่ในขอบเขต | 3 | |
| 8 | bonus | Claude Code / Opus 5.5 | `prompts/bonus-recover-decision.md`, decision: sync gRPC + shared demo.proto, evidence: `protos/demo.proto`, `src/checkoutservice/main.go`, `src/frontend/main.go`, language: Thai | | | 3 | |