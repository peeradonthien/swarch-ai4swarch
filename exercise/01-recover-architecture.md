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
| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| checkoutservice → paymentservice | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml:61–62` (`PAYMENT_SERVICE_ADDR=paymentservice:50051`); `system/online-boutique/protos/demo.proto:177–179` (`PaymentService.Charge`) | ✅ configuration และโค้ดการเรียก gRPC | `system/online-boutique/src/checkoutservice/main.go:116,123` อ่าน address และสร้าง connection; `chargeCard:369–376` เรียก `NewPaymentServiceClient(...).Charge(...)`; `PlaceOrder:252–255` เรียก helper และคืน error ถ้า charge ล้มเหลว |
| checkoutservice → emailservice | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml:63–64` (`EMAIL_SERVICE_ADDR=emailservice:5000`); `system/online-boutique/kubernetes-manifests/emailservice.yaml:80–83` (`grpc`, `5000 → 8080`); `system/online-boutique/protos/demo.proto:199–201` | ✅ configuration, port mapping และโค้ดการเรียก gRPC | `system/online-boutique/src/checkoutservice/main.go:115,122,379–383` สร้าง client และเรียก `SendOrderConfirmation`; `PlaceOrder:273–279` บันทึก warning ถ้าส่งไม่สำเร็จแต่ยังคืน response สำเร็จ ส่วน port 5000 ที่ checkout ใช้ถูกต้อง เพราะเป็น Service port ซึ่งส่งต่อไป container port 8080 |
| checkoutservice → cartservice | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml:67–68` (`CART_SERVICE_ADDR=cartservice:7070`); `system/online-boutique/kubernetes-manifests/cartservice.yaml:79–82` (`grpc`, port 7070); `system/online-boutique/protos/demo.proto:23–27` | ✅ configuration และโค้ดการเรียก gRPC | `system/online-boutique/src/checkoutservice/main.go:113,120` อ่าน address และสร้าง connection; `getUserCart:324–329` เรียก `GetCart`; `emptyUserCart:332–336` เรียก `EmptyCart` จึงพบทั้งการอ่านและล้าง cart |
| checkoutservice → currencyservice | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml:65–66` (`CURRENCY_SERVICE_ADDR=currencyservice:7000`); `system/online-boutique/protos/demo.proto:140–143` (`CurrencyService.Convert`) | ✅ configuration และโค้ดการเรียก gRPC | `system/online-boutique/src/checkoutservice/main.go:114,121,359–366` อ่าน address สร้าง connection และเรียก `Convert`; จุดใช้งานอยู่ใน `prepareOrderItemsAndShippingQuoteFromCart:302` และ `prepOrderItems:348` สำหรับค่าขนส่งและราคาสินค้า |
| checkoutservice → shippingservice | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml:59–60` (`SHIPPING_SERVICE_ADDR=shippingservice:50051`); `system/online-boutique/protos/demo.proto:107–110` | ✅ configuration และโค้ดการเรียก gRPC | `system/online-boutique/src/checkoutservice/main.go:111,118` อ่าน address และสร้าง connection; `quoteShipping:313–321` เรียก `GetQuote`; `shipOrder:386–393` เรียก `ShipOrder` จึงไม่ได้มีเฉพาะการขอราคาขนส่ง |
| checkoutservice → productcatalogservice | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml:57–58` (`PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550`); `system/online-boutique/protos/demo.proto:71–75` (`ProductCatalogService.GetProduct`) | ✅ configuration และโค้ดการเรียก gRPC | `system/online-boutique/src/checkoutservice/main.go:112,119` อ่าน address และสร้าง connection; `prepOrderItems:339–355` สร้าง `ProductCatalogServiceClient` และเรียก `GetProduct` สำหรับแต่ละ cart item |
| cartservice → redis-cart | `system/online-boutique/kubernetes-manifests/cartservice.yaml:50–51` (`REDIS_ADDR=redis-cart:6379`), `118–120` (`redis:alpine`), `143–156` (Service `redis-cart`, port 6379) | ✅ dependency ตาม configuration | address ที่ cartservice ได้รับตรงกับ Redis Service ที่ประกาศในไฟล์เดียวกัน; `/data` ใช้ volume `redis-data` ที่กำหนดเป็น `emptyDir` (`cartservice.yaml:129–141`) การตรวจนี้ยังไม่ได้อ่าน implementation ของ cartservice หรือทดสอบการอ่านเขียนข้อมูล |
| loadgenerator → frontend | `system/online-boutique/kubernetes-manifests/loadgenerator.yaml:49,69–70,82–83` (`http://${FRONTEND_ADDR}`, `frontend:80`) | ✅ configuration และ startup check; เป็น optional workload | init container `frontend-check` มีคำสั่ง HTTP ไป frontend และ main container ตั้ง address เดียวกัน แต่ `system/online-boutique/kubernetes-manifests/kustomization.yaml:24` comment resource นี้ไว้ จึงยืนยันได้ว่ามี configuration และคำสั่งตรวจ frontend ไม่ใช่ว่า workload รันอยู่เสมอ |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): ไม่พบ dependency ที่ขัดกับหลักฐานที่อ่าน

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

   จากผลลัพธ์จะเห็นได้ว่า `.proto` ไฟล์เดียวบอกได้ว่ามี service interfaces, RPC และรูปแบบข้อความอะไร แต่ยังยืนยันไม่ได้ว่าใครเรียกใคร ใช้ฐานข้อมูลอะไร หรือ deploy บริการแยกกันอย่างไร เพราะตัวไฟล์ proto หน้าที่มันจะประกาศเเค่เป็นเหมือนสัญญาของ api ที่ service จะคุยกัน ไม่ได้ประกาศ topology ของการ deploy เมื่อเพิ่ม manifests ใน B ถึงจะมีหลักฐานเพียงพอของปลายทาง เช่น `PAYMENT_SERVICE_ADDR` และ `REDIS_ADDR` ขณะที่ A ยังไม่มี dependencies ที่ยืนยันได้ เเต่จริง ๆ B ก็ยังไม่ได้พิสูจน์พฤติกรรมจริงทั้งหมด

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

   น่าจะคือการมอง `loadgenerator` เป็นส่วนที่เปิดใช้งานเสมอ เพราะในแผนภาพ B กล่องของมันอยู่ใน system boundary เช่นเดียวกับบริการอื่น แม้ชื่อแผนภาพและข้อความประกอบจะระบุว่าเป็น optional แล้ว เเต่ resource entry ของมันใน `kustomization.yaml` ถูก comment ไว้ จึงไม่ควรสรุปจากการมีไฟล์ Deployment ว่ามันถูก deploy ด้วยชุด configuration นี้ ถ้า member คนอื่น ๆ มามองเฉพาะกล่องและเส้นเชื่อม อาจเข้าใจว่าระบบมี traffic จำลองทำงานอยู่ตลอด