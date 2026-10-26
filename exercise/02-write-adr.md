**ภาษา:** ภาษาไทย | [English](02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ประมาณ 20 นาที)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ _ครบถ้วนน้อยกว่ามนุษย์_ ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
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

| เกณฑ์                                                                              | Run a                                                                                                                                                                                                                                                     | Run b                                                                                                                                                                                                                                                                                                                                                        |
| ---------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast)             | ไม่ใช้ ระบุว่า requirement ไม่มีตัวเลข แล้วโยน latency, ปริมาณออเดอร์ และ SLA/rate limit ของ MailFast ไปที่ Open questions ไม่มี 500 ms, ~300 ms, 200 orders/s, MailFast p95 1.5 s / p99 8 s / fail ~0.5% / 50 emails/s หรือ loyalty ภายใน 1 นาที         | ใช้ครบ: p95 `PlaceOrder` ≤ 500 ms (ปัจจุบัน ~300 ms), peak 200 orders/s ช่วง 11.11, MailFast p95 1.5 s / p99 8 s / fail ~0.5% / 50 emails/s, loyalty ภายใน 1 นาที, ทีม 4 คน (2 คนเคยใช้ RabbitMQ), Kubernetes และ open-source self-hosted คำนวณส่วนต่าง 200−50 ≈ 150 อีเมล/วินาทีที่ต้อง buffer                                                              |
| ทางเลือกที่พิจารณา (ระบุ)                                                          | (1) synchronous orchestration เรียกทีละตัวผ่าน gRPC/HTTP (2) event-driven pub/sub ผ่าน message broker ที่ยังไม่ระบุเทคโนโลยี (3) post-order orchestrator service เลือก (2)                                                                                | (1) synchronous gRPC fan-out ใน `PlaceOrder` (2) goroutine ใน `checkoutservice` (3) publish `OrderPlaced` ไป RabbitMQ (4) Apache Kafka เลือก (3)                                                                                                                                                                                                             |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | แต้ม: ให้ consumer idempotent ด้วย event ID หรือ order ID เป็น dedup key แต่ไม่พูดถึงกำหนด 1 นาที และไม่แยกคิว loyalty ออกจากอีเมล Rate limit: พูดถึง retry + dead-letter queue เมื่อ MailFast rate-limit แต่ไม่มีเลข 50/s และไม่มี throttle ที่ consumer | แต้ม: queue แยกจากอีเมล เพื่อไม่ให้ติด backlog ของ MailFast; dedupe ด้วย `order_id` (unique constraint) ใน transaction เดียวกับให้แต้ม ให้ at-least-once กลายเป็น effectively exactly-once; ตั้ง validation ว่าจำนวน award = จำนวน order และ p99 ≤ 60 s Rate limit: consumer อีเมลจำกัดรวม ≤ 50/s, retry + DLQ และกำหนด global rate เมื่อ scale หลาย replica |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | ไม่แต่งตัวเลข แต่เสนอกลไกที่ scenario ไม่ได้ระบุ: transactional outbox, protobuf, exponential backoff, broker แบบ HA ชื่อ `checkoutservice` ติดป้าย [Inferred] เทคโนโลยี broker ปล่อยเป็น Open question                                                   | ตัวเลขหลักอ้าง scenario สิ่งที่ scenario ไม่ได้ให้แต่ใส่ในการตัดสินใจ: alert เมื่อ loyalty queue age > 30 s, RabbitMQ cluster 3 nodes + quorum queues, เครื่องมือ load test "Locust", backlog ≈ 9,000 ฉบับ/นาที (คำนวณเองจาก 200−50) ภาษาของ `loyaltyservice` เป็น Go ติดป้าย [Inferred] protocol ของ MailFast ไม่ได้แต่ง ไว้ใน Open questions               |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| #   | สิ่งที่เปลี่ยนจากร่างของ AI                                                                                     | เหตุผล                                                                                                                                                      |
| --- | --------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | เปลี่ยนการตัดสินใจจาก RabbitMQ (Option 3) เป็น Kafka (Option 4) และแยกงานด้วย consumer group                    | ต้องการ log ที่ replay ได้สำหรับ analytics และ recommendations ปีหน้า และไม่ให้ backlog อีเมลไปหน่วง loyalty ยอมรับว่าทีมมีประสบการณ์ RabbitMQ ไม่ใช่ Kafka |
| 2   | กำหนดความยาว peak เป็น 10 นาที แล้วคำนวณ backlog อีเมล 90,000 ฉบับ และความล่าช้าของอีเมลใบสุดท้ายประมาณ 30 นาที | Scenario ไม่ได้ระบุความยาว peak ต้องมีตัวเลขนี้ถึงจะประมาณขนาดคิวและเวลาที่ลูกค้าได้รับอีเมลได้                                                             |
| 3   | กำหนดให้ `loyaltyservice` ใช้ Go แทนการติดป้าย [Inferred]                                                       | ทีมถนัด Go อยู่แล้ว จึงตัดสินใจเลือกภาษา ไม่ใช่แค่เดาจากทักษะทีม                                                                                            |
