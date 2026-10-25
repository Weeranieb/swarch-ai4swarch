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

| ผู้เรียก → ผู้ถูกเรียก                            | หลักฐานที่ AI อ้าง                                                                        | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ                                                                                                                                                                                                        |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------- | ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `checkoutservice` → `paymentservice`              | `checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR: paymentservice:50051`)                     | ✅                             | `src/checkoutservice/main.go` อ่าน `PAYMENT_SERVICE_ADDR` (บรรทัด 116) แล้ว `PlaceOrder` เรียก `chargeCard` (บรรทัด 252) ซึ่งเรียก `PaymentService.Charge` จริง (บรรทัด 370)                                          |
| `checkoutservice` → `emailservice`                | `checkoutservice.yaml` (`EMAIL_SERVICE_ADDR: emailservice:5000`)                          | ✅                             | `src/checkoutservice/main.go` อ่าน `EMAIL_SERVICE_ADDR` (บรรทัด 115) แล้ว `PlaceOrder` เรียก `sendOrderConfirmation` (บรรทัด 273) ซึ่งเรียก `EmailService.SendOrderConfirmation` จริง (บรรทัด 380)                    |
| `frontend` → `shippingservice`                    | `frontend.yaml` (`SHIPPING_SERVICE_ADDR: shippingservice:50051`)                          | ✅                             | `src/frontend/main.go` อ่าน `SHIPPING_SERVICE_ADDR` (บรรทัด 137) และ `handlers.go` บรรทัด 271 เรียก `getShippingQuote` ซึ่งเรียก `ShippingService.GetQuote` จริง (`rpc.go` บรรทัด 88)                                 |
| `checkoutservice` → `shippingservice`             | `checkoutservice.yaml` (`SHIPPING_SERVICE_ADDR: shippingservice:50051`)                   | ✅                             | `src/checkoutservice/main.go` อ่าน `SHIPPING_SERVICE_ADDR` (บรรทัด 111) แล้ว `PlaceOrder` เรียก `shipOrder` (บรรทัด 258) ซึ่งเรียก `ShippingService.ShipOrder` จริง (บรรทัด 387)                                      |
| `recommendationservice` → `productcatalogservice` | `recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR: productcatalogservice:3550`) | ✅                             | `src/recommendationservice/recommendation_server.py` อ่าน `PRODUCT_CATALOG_SERVICE_ADDR` (บรรทัด 131) สร้าง `ProductCatalogServiceStub` (บรรทัด 136) แล้ว `ListRecommendations` เรียก `ListProducts` จริง (บรรทัด 73) |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): …

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

   จากไฟล์ `.proto` เพียงไฟล์เดียว agent เห็นได้แค่ schema ของแต่ละ service ว่ามี RPC อะไร และ request/response มีรูปแบบใด แต่ไม่รู้ว่าตอนรันจริงใครเรียกใคร เพราะความสัมพันธ์นั้นอยู่ใน logic ของ service ไม่ได้อยู่ในไฟล์ประกาศ interface Run A จึงมี container เก้าตัวแต่ไม่มีเส้น dependency เลย ส่วน Run B เห็นเส้นเรียกได้เพราะ Kubernetes manifest ระบุที่อยู่ของ service ผ่าน environment variable เช่น `CHECKOUT_SERVICE_ADDR`

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

   ข้อที่ทำให้สมาชิกใหม่เข้าใจผิดมากที่สุดคือเส้นในแผนภาพ Run B ที่เขียนราวกับรู้ว่าเรียก RPC ใด เช่น `checkoutservice` ไป `cartservice` ด้วยป้าย "Reads and empties cart" ทั้งที่ manifest พิสูจน์ได้เพียงว่ามี address ของ cart ไม่ได้พิสูจน์ว่าเรียก `GetCart` หรือ `EmptyCart` จริง ถ้าเอาแผนภาพที่ relation ยังเป็นข้อสันนิษฐานไปใช้วางแผนต่อ การออกแบบที่สร้างบนเส้นที่ผิดจะคลาดเคลื่อนทั้งระบบ ไม่ใช่แค่เส้นเดียว
