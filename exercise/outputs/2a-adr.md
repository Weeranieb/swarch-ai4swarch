> Agent: Claude Code — Model: Claude Opus 5.5 (`claude-opus-5-5`) — run: a (zero-shot, context: none)

# ADR-0002: Publish an "order placed" event and handle post-order actions with asynchronous subscribers

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** Not evidenced in the provided files (ดู Open questions)

## Context and problem

หลังจากลูกค้าสั่งซื้อสำเร็จ Online Boutique ต้องทำงานต่อจากการสั่งซื้อ (post-order actions) เพิ่มขึ้น ตาม requirement:

| งานหลังสั่งซื้อ | รายละเอียดจาก requirement |
|---|---|
| ส่งอีเมลยืนยันคำสั่งซื้อ | ส่งผ่าน MailFast ซึ่งเป็น external email provider รายใหม่ แทนที่ logic จำลอง (mock) ใน `emailservice` |
| ให้คะแนนสะสม (loyalty points) | ผ่าน `loyaltyservice` ใหม่ที่ต้องสร้างขึ้น |
| บันทึก analytics event "order placed" | สำหรับทีม marketing |
| งานในอนาคต (ปีหน้า) | SMS, fraud scoring, การอัปเดต recommendations |

ปัญหาที่ต้องตัดสินใจ: **service ที่ทำการสั่งซื้อควรเชื่อมต่อกับงานหลังสั่งซื้อเหล่านี้อย่างไร** — เรียกแต่ละงานโดยตรงแบบ synchronous หรือประกาศเหตุการณ์ (event) ให้ผู้สนใจไปรับเอง

ข้อเท็จจริงที่ทราบและไม่ทราบ:

- ทำ ADR นี้แบบ zero-shot (`context: none`) จึงไม่ได้อ่านโค้ดของระบบ ชื่อ service ที่ทำการสั่งซื้อ (น่าจะเป็น `checkoutservice`) และวิธีที่เรียก `emailservice` อยู่ในปัจจุบัน ถือเป็น **[Inferred]** ไม่ใช่ Evidenced
- requirement ไม่ได้ให้ตัวเลขใดๆ (ปริมาณคำสั่งซื้อ, latency target ของ checkout, SLA ของ MailFast, ความถี่ที่ข้อมูล analytics ต้องสดใหม่) — ดู Open questions
- requirement ระบุชัดว่าจะมีงานหลังสั่งซื้อเพิ่มอีกอย่างน้อย 3 รายการในปีหน้า ดังนั้นความง่ายในการเพิ่มงานใหม่ (modifiability) เป็นแรงผลักดันหลัก

## Decision drivers

1. **Availability ของ checkout** — ความล้มเหลวหรือความช้าของ MailFast (external), `loyaltyservice` (ของใหม่ ยังไม่ผ่านการใช้งานจริง) หรือ analytics ไม่ควรทำให้การสั่งซื้อล้มเหลว **[Inferred]** จากธรรมชาติของงานเหล่านี้ที่เป็นงานรองหลังการชำระเงิน
2. **Modifiability / extensibility** — เพิ่มงานหลังสั่งซื้อใหม่ (SMS, fraud scoring, recommendations) ได้โดยแก้ service ที่ทำการสั่งซื้อให้น้อยที่สุด
3. **Performance (latency ของ checkout)** — ไม่ให้เวลาตอบสนองของการสั่งซื้อเพิ่มขึ้นตามจำนวนงานหลังสั่งซื้อ
4. **Reliability ของงานหลังสั่งซื้อ** — อีเมลยืนยันและคะแนนสะสมต้องไม่สูญหาย และต้องไม่ถูกทำซ้ำจนลูกค้าได้คะแนน/อีเมลซ้ำ
5. **Operability และ cost** — จำนวน component ใหม่ที่ต้องดูแล, การ monitor และ debug

## Considered options

1. **Option A — Synchronous orchestration:** service ที่ทำการสั่งซื้อเรียก `emailservice` (ซึ่งเปลี่ยนไปเรียก MailFast), `loyaltyservice` และ analytics โดยตรงทีละตัวผ่าน gRPC/HTTP ภายในขั้นตอนการสั่งซื้อ
2. **Option B — Event-driven publish/subscribe ผ่าน message broker:** service ที่ทำการสั่งซื้อ publish event `OrderPlaced` เพียงครั้งเดียวไปยัง message broker (เทคโนโลยียังไม่กำหนด — ดู Open questions) แล้ว `emailservice` (adapter ไป MailFast), `loyaltyservice` และ analytics consumer subscribe แยกกัน แต่ละตัวมี queue/consumer group ของตัวเอง
3. **Option C — Dedicated post-order orchestrator service:** service ที่ทำการสั่งซื้อเรียก service ใหม่ตัวเดียว (เช่น `postorderservice`) แบบ fire-and-forget แล้ว orchestrator นั้นเรียกงานหลังสั่งซื้อแต่ละงานต่อ (พร้อม retry) แบบรวมศูนย์

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A — Synchronous orchestration | • ง่ายที่สุด ไม่มี infrastructure ใหม่<br>• ลำดับการทำงานชัดเจน debug ง่าย (ดู call chain เดียว)<br>• รู้ผลทันทีว่าแต่ละงานสำเร็จหรือไม่ | • Availability ของ checkout = ผลคูณของ availability ทุก dependency รวม MailFast ที่เป็น external<br>• Latency ของ checkout เพิ่มตามจำนวนงาน (ถ้าเรียกเรียงกัน)<br>• ทุกงานใหม่ (SMS, fraud, recommendations) ต้องแก้และ redeploy service ที่ทำการสั่งซื้อ → coupling สูง<br>• ต้องตัดสินใจเองว่าถ้างานหนึ่งล้ม จะ fail ทั้ง order หรือข้าม (และจะ retry อย่างไร) |
| B — Event-driven pub/sub | • Service ที่ทำการสั่งซื้อไม่ต้องรู้จักผู้รับ → เพิ่ม subscriber ใหม่โดยไม่แก้ producer<br>• ความล้มเหลว/ความช้าของ MailFast หรือ `loyaltyservice` ไม่กระทบ checkout (broker เป็น buffer)<br>• Latency ของ checkout คงที่ (publish ครั้งเดียว) ไม่ขึ้นกับจำนวนงาน<br>• แต่ละ consumer scale และ retry ได้อิสระ; analytics อ่าน event เดียวกันได้ | • ต้องเพิ่มและดูแล message broker (cost, operability) — เป็น single point of failure ใหม่ถ้าไม่ทำ HA<br>• Eventual consistency: อีเมล/คะแนนมาถึงช้ากว่าการสั่งซื้อเล็กน้อย<br>• At-least-once delivery → consumer ต้อง idempotent (กันอีเมล/คะแนนซ้ำ)<br>• ความเสี่ยง dual-write: order สำเร็จแต่ publish ไม่สำเร็จ (ต้องมี outbox หรือ retry)<br>• Debug ข้าม service ยากกว่า ต้องมี tracing/correlation ID<br>• ต้องออกแบบและ version schema ของ event |
| C — Post-order orchestrator | • Service ที่ทำการสั่งซื้อพึ่งพา dependency ใหม่แค่ตัวเดียว<br>• Logic ของงานหลังสั่งซื้อ (ลำดับ, retry, เงื่อนไข เช่น fraud ก่อน loyalty) อยู่ที่เดียว มองเห็นง่าย<br>• ไม่ต้องมี message broker | • Orchestrator เป็นจุดรวมที่ต้องแก้ทุกครั้งที่เพิ่มงานใหม่ → ย้าย coupling ไปอีกที่ ไม่ได้หายไป<br>• ถ้าเรียกแบบ fire-and-forget โดยไม่มี durable storage งานอาจหายเมื่อ orchestrator ล่ม → ต้องสร้างกลไก persistence/retry เอง (ซึ่งคือการสร้าง broker แบบย่อเอง)<br>• Orchestrator เองเป็น single point of failure ของงานหลังสั่งซื้อทั้งหมด |

## Decision

เลือก **Option B — Event-driven publish/subscribe ผ่าน message broker**

- Service ที่ทำการสั่งซื้อ publish event `OrderPlaced` (มี order ID, ข้อมูลลูกค้าที่จำเป็น, รายการสินค้า, ยอดรวม, timestamp, event ID) หลังการสั่งซื้อสำเร็จ
- `emailservice` เปลี่ยนจาก mock เป็น adapter ที่ subscribe event แล้วเรียก MailFast; `loyaltyservice` และ analytics consumer subscribe แยกกัน
- ใช้ transactional outbox หรือกลไก retry ฝั่ง producer เพื่อลดความเสี่ยง order สำเร็จแต่ event หาย (รายละเอียดขึ้นกับ data store ของ order ซึ่ง Not evidenced in the provided files)

เหตุผลเทียบกับ decision drivers:

- **Availability (driver 1):** checkout ไม่พึ่งพา MailFast หรือ `loyaltyservice` โดยตรงอีกต่อไป — พึ่งพาแค่ broker ตัวเดียว
- **Modifiability (driver 2):** SMS, fraud scoring และ recommendations ในปีหน้าเพิ่มเป็น subscriber ใหม่ได้โดยไม่แก้ producer — เป็นข้อได้เปรียบชัดเจนเหนือ A และ C
- **Performance (driver 3):** latency ของ checkout ไม่เพิ่มตามจำนวนงานหลังสั่งซื้อ
- Option A ถูกปฏิเสธเพราะขัดกับ driver 1–3 โดยตรง Option C ถูกปฏิเสธเพราะยังต้องแก้ orchestrator ทุกครั้งที่เพิ่มงาน และต้องสร้างกลไก durability เองซึ่ง broker มีให้อยู่แล้ว

ข้อสังเกต: **fraud scoring** ในอนาคตอาจต้องทำ *ก่อน* ยืนยันการสั่งซื้อ (ต้องการผลแบบ synchronous) ซึ่งไม่เหมาะกับ event หลังสั่งซื้อ — ต้องมี ADR แยกเมื่อถึงเวลา **[Inferred]**

## Consequences

- **Positive:** ความล้มเหลวหรือความช้าของ MailFast, `loyaltyservice` และ analytics ไม่ทำให้การสั่งซื้อล้มเหลว (quality attribute: availability)
- **Positive:** เพิ่มงานหลังสั่งซื้อใหม่ได้โดย deploy subscriber ใหม่ ไม่ต้องแก้หรือ redeploy service ที่ทำการสั่งซื้อ (quality attribute: modifiability)
- **Positive:** latency ของ checkout คงที่ไม่ว่าจะมีกี่ subscriber (quality attribute: performance)
- **Positive:** แต่ละ consumer scale ได้อิสระตามโหลดของตัวเอง (quality attribute: scalability)
- **Negative:** ต้องติดตั้ง ดูแล และ monitor message broker เพิ่ม (quality attribute: cost, operability)
- **Negative:** อีเมลยืนยันและคะแนนสะสมมาถึงแบบ eventual — ลูกค้าอาจเห็นคะแนนช้ากว่าคำสั่งซื้อ (quality attribute: consistency / usability)
- **Negative:** การติดตามว่า order หนึ่งผ่านงานใดบ้างยากขึ้น ต้องมี distributed tracing และ correlation ID (quality attribute: testability, observability)
- **Negative:** ข้อมูลลูกค้า (เช่น อีเมล) อยู่ใน event และส่งต่อไปยัง MailFast ซึ่งเป็น external (quality attribute: security / privacy)
- **Risks and mitigations:**

| Risk | Mitigation |
|---|---|
| Order สำเร็จแต่ publish event ไม่สำเร็จ (dual-write) | Transactional outbox หรือ retry ฝั่ง producer พร้อม alert |
| Event ถูกส่งซ้ำ (at-least-once) → อีเมลซ้ำ / คะแนนซ้ำ | Consumer idempotent โดยใช้ event ID หรือ order ID เป็น dedup key |
| MailFast ล่มหรือ rate-limit | Retry แบบ exponential backoff + dead-letter queue + alert |
| Broker ล่ม | Deploy broker แบบ HA; producer buffer/retry ผ่าน outbox |
| Schema ของ event เปลี่ยนแล้ว subscriber พัง | กำหนด schema ชัดเจน (เช่น protobuf) และเปลี่ยนแบบ backward-compatible เท่านั้น |
| ข้อมูลส่วนบุคคลรั่วไปยัง subscriber ที่ไม่จำเป็น | ใส่เฉพาะข้อมูลที่จำเป็นใน event; ควบคุมสิทธิ์การ subscribe |

## Validation

- **Fault-injection test:** ปิด MailFast (หรือ mock ให้ timeout) และปิด `loyaltyservice` แล้วยืนยันว่าการสั่งซื้อยังสำเร็จ และงานค้างถูกประมวลผลครบเมื่อ service กลับมา
- **Load test:** เปรียบเทียบ latency ของ checkout ก่อน/หลังเพิ่มการ publish event — เกณฑ์ที่ยอมรับได้ยังไม่ได้กำหนด (ดู Open questions)
- **Metrics ใน production:** consumer lag ต่อ subscriber, จำนวนข้อความใน dead-letter queue, อัตราอีเมลซ้ำ/คะแนนซ้ำ (เป้าหมาย 0), เวลาจากสั่งซื้อจนอีเมลถูกส่ง
- **Extensibility check:** เมื่อเพิ่ม subscriber SMS ในปีหน้า ยืนยันว่าไม่ต้องแก้ service ที่ทำการสั่งซื้อเลย
- **Review date:** ทบทวน ADR นี้ก่อนเริ่มงาน fraud scoring (วันที่ยังไม่กำหนด)

## Open questions

- ใครคือ **Deciders** ของ ADR นี้?
- Service ใดเป็นผู้ทำการสั่งซื้อจริง และปัจจุบันเรียก `emailservice` แบบใด (synchronous หรือไม่, ถ้าเรียกไม่สำเร็จ order ล้มหรือไม่)? — ไม่ได้อ่านโค้ดใน run นี้
- ปริมาณคำสั่งซื้อ (ปัจจุบัน/สูงสุด) และ latency target ของ checkout คือเท่าใด?
- MailFast มี SLA, rate limit, รูปแบบ API (REST/SMTP/webhook) และค่าใช้จ่ายต่อข้อความอย่างไร?
- อีเมลยืนยันต้องถึงลูกค้าภายในกี่วินาที/นาที? คะแนนสะสมต้องแสดงทันทีหรือยอมรับ eventual consistency ได้?
- ทีม marketing ต้องการ analytics event ไปที่ใด (data warehouse, analytics tool ภายนอก?) และต้องสดใหม่แค่ไหน?
- จะใช้ message broker ตัวใด (เช่น managed cloud pub/sub, Kafka, RabbitMQ)? ทีมมีความชำนาญและงบประมาณสำหรับตัวใด?
- Order ถูกเก็บไว้ใน data store ใดหรือไม่ (จำเป็นต่อการออกแบบ transactional outbox)?
- Fraud scoring ต้องบล็อกการสั่งซื้อ (synchronous ก่อนยืนยัน) หรือเป็นการตรวจหลังสั่งซื้อ?
- มีข้อกำหนดด้าน privacy/PDPA สำหรับการส่งข้อมูลลูกค้าไปยัง MailFast และ analytics หรือไม่?
