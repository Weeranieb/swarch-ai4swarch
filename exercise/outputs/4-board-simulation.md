**Agent / model:** Claude Code, Claude Opus 5.5 (`claude-opus-5-5`)

# การประชุมคณะกรรมการรีวิว (จำลอง): ADR-0002

> **หมายเหตุ:** การประชุมนี้ **จำลองโดย AI** ไม่ใช่การประชุมจริงของกลุ่ม AI **ไม่ได้อ่านการ์ดบทบาท** ใน `docs/review-board/` ข้อกังวลของ PO และ SRE จึงเป็นสิ่งที่ AI สมมติขึ้นเอง **[Assumption]** ไม่ใช่ข้อมูลจากการ์ด
>
> ADR: `docs/adr/0002-publish-order-placed-to-kafka.md` · ผลรีวิว AI: `exercise/outputs/3-review.md` · วันที่ประชุม: 2026-10-31 (เหลือ 11 วันก่อน 11.11)

| บทบาท | ผู้รับบท |
|---|---|
| สถาปนิก | Weerachai Panthueng (ผู้เขียน ADR) |
| AI analyst | สมาชิก B (จำลอง) |
| Product owner | สมาชิก C (จำลอง) |
| SRE | สมาชิก D (จำลอง) |

## 1. สถาปนิกนำเสนอ (3 นาที)

**สถาปนิก:** "R-12 ให้เราทำ post-order action 3 อย่าง ปัญหาคือ MailFast p95 1.5 วินาที แต่งบ checkout มีแค่ 500 ms เลยต้องเอาออกจาก critical path เราพิจารณา 4 ทาง: sync gRPC, goroutine, RabbitMQ และ Kafka ทาง sync ใช้ไม่ได้เพราะ latency ส่วน goroutine งานหายตอน pod restart ผมเลือก Kafka เพราะแต่ละ action เป็น consumer group แยก loyalty ไม่ต้องรอคิวอีเมล 90,000 ฉบับ และเก็บ log ไว้ replay ให้ recommendations ปีหน้าได้ ข้อเสียที่ผมยอมรับคือทีมไม่เคยใช้ Kafka และอีเมลใบสุดท้ายอาจช้าประมาณ 30 นาที"

## 2. AI analyst อ่านผลรีวิว (3 นาที)

**AI analyst:** "AI ให้ผลว่า *Accept with changes* มี Risk 4 ข้อ:
1. ถ้า publish ล้มหลังตัดเงิน event จะหาย และ loyalty ไม่ถูกให้เลย
2. replay ไม่อยู่ใน decision drivers ถ้าใช้เกณฑ์ของ ADR เอง RabbitMQ ชนะ
3. retry ใน consumer block partition
4. ทีมไม่มีประสบการณ์ Kafka

ข้อแก้ไขที่ AI เสนอคือ ปิดช่อง event หาย, ให้เหตุผลการเลือก broker ใหม่ และใช้ retry topic"

## 3. อภิปรายเปิด (7 นาที)

**PO:** "ผมขอเริ่มที่อีเมลก่อน ช่วง 11.11 ลูกค้าที่ไม่ได้อีเมลยืนยันภายใน 5–10 นาทีจะคิดว่าสั่งไม่สำเร็จ แล้วกดสั่งซ้ำหรือโทรหา call center ปีที่แล้ว ticket ประเภท 'ไม่ได้รับอีเมล' พุ่งขึ้นเยอะมาก ช้า 30 นาทีรับไม่ได้ครับ" **[Assumption]**

**สถาปนิก:** "rate limit 50/s เป็นข้อจำกัดของ MailFast ไม่ใช่ของ Kafka ไม่ว่าเลือก broker ไหนอีเมลก็ช้าเท่ากัน"

**PO:** "งั้นแสดงหน้า order confirmation ให้ชัด และบอกว่าอีเมลอาจมาช้าได้ไหม หรือเจรจาขอ quota MailFast เพิ่มช่วง sale" **[Assumption]**

**ทุกคนถาม AI analyst:** "AI เห็นประเด็นนี้ไหม" → **AI analyst:** "AI พูดถึงแค่ว่าตัวเลข 30 นาทีเป็นสมมติฐาน ไม่ได้พูดถึงผลต่อลูกค้าหรือ call center"

**PO:** "อีกข้อ ลูกค้าอยากเห็นแต้มบนหน้า order confirmation ทันที ถ้าต้องรอ 1 นาที หน้านั้นจะโชว์แต้มเป็น 0" **[Assumption]**

**สถาปนิก:** "หน้าเว็บคำนวณ 'แต้มที่จะได้รับ' จากยอดสั่งซื้อมาแสดงได้ ส่วนการบันทึกแต้มจริงยังเป็น async เหมือนเดิม" → AI analyst: "AI ไม่ได้พูดถึง"

**SRE:** "วันนี้ 31 ตุลา เหลือ 11 วันก่อน 11.11 ผมต้องตั้ง Kafka บน Kubernetes ให้ production-ready ทั้ง persistent volume, replication, monitoring และ runbook แล้วยังต้อง load test ทั้งที่ไม่มีใครในทีมเคย on-call Kafka เลย ผมไม่เซ็นรับ pager ให้ระบบนี้ช่วง 11.11 ครับ" **[Assumption]**

**AI analyst:** "AI เห็นว่าทีมไม่มีประสบการณ์ Kafka แต่ไม่รู้ว่าเหลือเวลาเท่าไร"

**SRE:** "ถ้าเป็น RabbitMQ ทีมเรามีคนเคยใช้ 2 คน และใช้ quorum queue กับ DLQ ได้เลยโดยไม่ต้องสร้าง retry topic เอง" **[Assumption]**

**สถาปนิก:** "แต่ recommendations ต้องใช้ replay"

**PO:** "recommendations เป็นแผนปีหน้า ยังไม่มีงบหรือ spec ไม่ควรเอามาเป็นเหตุผลหลักตอนนี้" **[Assumption]** → AI analyst: "ตรงกับ Risk ข้อ 2 ของ AI ว่า replay ไม่อยู่ใน drivers"

**SRE:** "ถ้า broker ล่มช่วง peak และ checkout ทำ fail-soft แปลว่าแต้มของ order ช่วงนั้นหายเงียบ ๆ ผมขอให้มี job reconcile จากข้อมูล payment กับ loyalty ได้ไหม" **[Assumption]** → AI analyst: "AI ยกเรื่อง event หายไว้เป็น Risk ข้อ 1 และเสนอ reconciliation แต่ไม่ได้บอกว่าจะใช้ข้อมูลจากไหน"

## 4. ตัดสินใจ (2 นาที)

**มติ:** อนุมัติโดยมีเงื่อนไข (3 ต่อ 1 สถาปนิกขอเก็บ Kafka ไว้เป็นทิศทางระยะยาว)

1. ใช้ **RabbitMQ** (quorum queues + DLQ) ก่อนสำหรับ 11.11 ส่วน Kafka ให้ทำ ADR ใหม่เมื่อมี requirement replay ที่มีหลักฐานจริง
2. ปิดช่อง event หาย: alert บน publish-failure และทำ reconciliation job ระหว่าง payment กับ loyalty
3. ปรับ UX และ SLA ของอีเมล: หน้า confirmation แสดงเลข order และแต้มที่จะได้ บอกลูกค้าว่าอีเมลอาจช้า และ PO เจรจา quota MailFast ช่วง sale
