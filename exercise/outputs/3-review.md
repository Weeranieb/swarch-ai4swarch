**Agent / model:** Claude Code, Claude Opus 5.5 (`claude-opus-5-5`)

# Review: ADR-0002 Publish "OrderPlaced" to Kafka

ADR ที่รีวิว: `docs/adr/0002-publish-order-placed-to-kafka.md` · หลักฐานที่ใช้: ADR นี้และ `docs/scenario.md` เท่านั้น

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QAS-1 Performance | ลูกค้า | กด place order | peak 11.11 ที่ **200 orders/s** และ Kafka ปกติ | `PlaceOrder` publish `OrderPlaced` แล้วตอบทันที | p95 `PlaceOrder` **≤ 500 ms** (ปัจจุบัน ~300 ms) |
| QAS-2 Availability + correctness | MailFast หรือ `loyaltyservice` | ล่ม หรือได้ record ซ้ำ | peak 200 orders/s | order ยังสำเร็จ และ consumer ตามงานทันเมื่อกลับมา | order fail เพราะ post-order action = **0**; loyalty ได้แต้ม **ครั้งเดียว** ภายใน **1 นาที** หลัง order |
| QAS-3 Modifiability | marketing | ขอ action ใหม่ (SMS) ปีหน้า | ระบบ production | เพิ่ม consumer group ใหม่ | แก้และ deploy `checkoutservice` **0 ครั้ง** |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| **Risk** | ถ้า publish ล้มเหลวหลังตัดเงินแล้ว order ยังสำเร็จแต่ event หาย (ADR บรรทัด 64, 110) และไม่มี outbox | loyalty จะ**ไม่ถูกให้เลย** ซึ่งขัด "exactly once" ใน scenario; ADR มีแค่ alert แต่ไม่มีทางกู้ event คืน |
| **Risk** | เหตุผลที่เลือก Kafka คือ "replay log สำหรับ analytics/recommendations" (บรรทัด 71) แต่ข้อนี้**ไม่อยู่ใน decision drivers 1–5** และ scenario ไม่ได้ขอ replay | ตามเกณฑ์ของ ADR เอง driver 1–4 เสมอกัน ส่วน driver 5 (ทีมเคยใช้ RabbitMQ 2 คน) ชี้ไปที่ RabbitMQ จึงเท่ากับตัดสินด้วยเกณฑ์นอกตาราง |
| **Risk** | retry แบบ backoff ใน email consumer (บรรทัด 66) จะ block partition เพราะ Kafka commit offset ตามลำดับ ไม่มี ack รายข้อความ | record อีเมลที่ fail (0.5 %) ทำให้ partition นั้นค้าง ถ้า loyalty ใช้วิธีเดียวกัน poison record ตัวเดียวจะทำให้ lag เกิน 60 s |
| **Risk** | ทีม 4 คนไม่มีประสบการณ์ Kafka (scenario: Team) แต่ต้องดูแล cluster ที่มี state และรับ peak 11.11 ตั้งแต่ครั้งแรก | operability: ถ้า broker ล่มหรือตั้งค่า replication ผิดช่วง peak จะไม่มีใครในทีมเคยแก้มาก่อน |
| **Non-risk** | เอา MailFast (p95 1.5 s, p99 8 s) ออกจาก critical path | ถ้าเรียกแบบ synchronous จะเกินงบ p95 500 ms แน่นอน การทำเป็น async จึงจำเป็น |
| **Non-risk** | แยก consumer group ของ `email` กับ `loyalty` | backlog อีเมลไม่บล็อก loyalty เพราะทำงานที่ 50/s ขณะ order เข้า 200/s ทำให้ loyalty ทัน 1 นาที |
| **Non-risk** | dedupe ด้วย `order_id` (unique constraint) ใน transaction เดียวกับการเพิ่มแต้ม (บรรทัด 111) | at-least-once delivery กลายเป็น effectively exactly-once สำหรับแต้ม |
| **Sensitivity point** | จำนวน partition ของ topic | กำหนดเพดาน parallelism ของ loyalty consumer ที่ต้องรับ 200/s ภายใน 1 นาที และกำหนดว่าจะแบ่งโควตา 50/s ของอีเมลให้แต่ละ instance อย่างไร |
| **Sensitivity point** | producer timeout + retry ภายใน `PlaceOrder` | ถ้ายาวไปจะกินงบ ~200 ms ที่เหลือ ถ้าสั้นไป event หายบ่อยขึ้น |
| **Trade-off point** | `acks=all` + publish แบบ fail-soft | ด้าน **availability** ดีขึ้น (order ไม่ล้ม) แต่ **reliability** ของ loyalty แย่ลง (event หายได้); `acks=all` เพิ่ม durability แต่เพิ่ม latency |
| **Trade-off point** | Kafka แทน RabbitMQ | **modifiability**/replay ดีขึ้น แต่ **operability/cost** แย่ลง และต้องสร้าง retry/DLQ/rate limit เองที่ consumer |

## 3. Missing or weak

- **ทางเลือกที่ไม่ได้พิจารณา:** RabbitMQ Streams (replay ได้บน broker ที่ทีมรู้จัก) และ transactional outbox ที่มี order store ซึ่งถูกเลื่อนไปไว้ใน Open questions แทนที่จะเป็นทางเลือก
- **ไม่ได้ระบุว่า "Kafka ล่ม" นับเป็น "loyalty ล่ม" หรือไม่:** scenario ห้าม order fail เพราะ email/loyalty/analytics ล่ม แต่ไม่ได้พูดถึง broker ADR เลือก fail-soft โดยไม่ได้ให้เหตุผลข้อนี้
- **Consequence ที่ขาด:** อีเมลซ้ำเมื่อ retry (at-least-once ไปยัง MailFast); retention period ของ topic ที่ต้องยาวพอสำหรับ replay; ลำดับ record ต่อ key (ใช้ `order_id` เป็น key หรือไม่)
- **Claim ที่ไม่มีหลักฐาน:** peak ยาว 10 นาที, backlog 90,000 ฉบับ และอีเมลช้า 30 นาที ล้วนเป็น**สมมติฐาน** (scenario ไม่ได้ระบุความยาว peak); "p95 ยังอยู่ใต้ 500 ms" ยังไม่ได้วัด latency ของ `acks=all`
- **Mitigation อ่อน:** "รันมากกว่าหนึ่ง broker" (บรรทัด 113) ไม่ได้ระบุ replication factor หรือ min in-sync replicas ซึ่งเป็นตัวกำหนดว่า `acks=all` ปลอดภัยจริงหรือไม่

## 4. Verdict

**Accept with changes**

1. **ปิดช่อง event หาย:** เลือกระหว่าง outbox, การทำให้ order fail เมื่อ publish ไม่ได้ (ถ้าธุรกิจถือว่า broker ไม่ใช่ "loyalty ล่ม") หรือการเขียนชัดว่ายอมรับ loss rate เท่าไร พร้อม reconciliation
2. **ให้เหตุผลการเลือก broker ใหม่:** เพิ่ม "replay" เป็น decision driver พร้อมหลักฐานว่าต้องใช้จริง ถ้าไม่มีหลักฐาน RabbitMQ ชนะตามเกณฑ์ของ ADR เอง ถ้ายังเลือก Kafka ให้ใส่แผน ops (replication, operator, on-call, การฝึกทีม)
3. **ออกแบบ retry ที่ไม่ block partition:** ใช้ retry topic แยกจาก DLQ สำหรับ `email` และ `loyalty` กำหนดจำนวน partition ให้ loyalty ทัน 60 s ที่ 200/s และตั้ง alert จาก consumer lag
