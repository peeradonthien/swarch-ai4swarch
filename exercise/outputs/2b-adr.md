เอเจนต์: Codex; โมเดล: GPT-6 (ไม่ทราบรุ่นย่อย)

# ADR-0002: ใช้ RabbitMQ แยกงานหลังสั่งซื้อออกจาก checkout

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** ยังไม่ระบุผู้มีอำนาจตัดสินใจ ดู Open questions

## Context and problem

R-12 ต้องส่งอีเมลผ่าน MailFast แทน mock `emailservice`, เพิ่มคะแนนผ่าน `loyaltyservice` ที่ต้องสร้าง และบันทึกเหตุการณ์สั่งซื้อให้การตลาด รวมถึงรองรับ SMS, fraud scoring และการปรับ recommendations ในปีถัดไป (`docs/scenario.md:7–14`)

| ประเด็น | หลักฐานและผลต่อการตัดสินใจ |
|---|---|
| เส้นทางปัจจุบัน | `PlaceOrder` เรียก `chargeCard` แล้ว `shipOrder` ก่อนสร้าง `OrderResult` และรอ `sendOrderConfirmation`; หากอีเมลผิดพลาดจะบันทึกคำเตือนและยังคืนผลสำเร็จ (`system/online-boutique/src/checkoutservice/main.go:252–279`) การส่งอีเมลใช้ gRPC `SendOrderConfirmation` (`main.go:379–383` ในไฟล์เดียวกัน) |
| งบ latency | p95 ของ `PlaceOrder` ต้อง ≤ 500 ms จากปัจจุบันประมาณ 300 ms ขณะที่ MailFast มี p95 1.5 s และ p99 8 s (`docs/scenario.md:24–26`) **[Inferred]** การรอ MailFast ในเส้นทางเดิมเสี่ยงเกินเป้าหมาย แต่ยังคำนวณ p95 รวมที่แน่นอนไม่ได้จาก quantile เหล่านี้ |
| ความจุอีเมล | โหลดสูงสุด 200 orders/s แต่ MailFast จำกัด 50 emails/s และผิดพลาดประมาณ 0.5% (`docs/scenario.md:25–26`) **[Inferred]** หากหนึ่งออเดอร์ใช้อีเมลหนึ่งครั้งและโหลดนี้ต่อเนื่อง คิวจะเพิ่มอย่างน้อย 150 รายการ/s ก่อนรวม retry; เพิ่ม worker อย่างเดียวแก้เพดานผู้ให้บริการไม่ได้ |
| คะแนนและความพร้อมใช้งาน | ต้องให้คะแนนภายใน 1 นาทีและ exactly once; ออเดอร์ต้องไม่ล้มเหลวเพราะอีเมล คะแนน หรือ analytics ใช้งานไม่ได้ (`docs/scenario.md:27–28`) |
| ทีมและการปฏิบัติการ | ทีม 4 คนถนัด Go/gRPC และ 2 คนเคยใช้ RabbitMQ; ระบบอยู่บน Kubernetes และองค์กรต้องการ open-source ที่ self-hosted (`docs/scenario.md:29–30`) |
| ช่องว่างด้านความทนทาน | `PlaceOrder` สร้าง UUID ใหม่ต่อการเรียก แล้วชำระเงิน จัดส่ง และตอบกลับ โดยไม่เห็นการบันทึกออเดอร์หรือ outbox ในฟังก์ชันนี้ (`system/online-boutique/src/checkoutservice/main.go:230–279`) กลไกเก็บออเดอร์ถาวรและกู้คืนงานของระบบทั้งหมด: **Not evidenced in the provided files** |

ADR-0001 เลือก Redis สำหรับตะกร้าซึ่งยอมรับการสูญหายได้เพราะยังไม่เกี่ยวกับเงิน (`docs/adr/0001-use-redis-for-cart-storage.md:11,38–45`) **[Inferred]** เหตุผลนี้ไม่เพียงพอสำหรับใช้ตะกร้าเป็นหลักฐานถาวรของออเดอร์ที่ชำระเงินแล้ว

## Decision drivers

1. **Availability:** ความขัดข้องของงานหลังสั่งซื้อต้องไม่ทำให้ออเดอร์ล้มเหลว (`docs/scenario.md:28`)
2. **Correctness และ timeliness:** คะแนนต้องเพิ่มเพียงครั้งเดียวภายใน 1 นาที (`docs/scenario.md:27`)
3. **Performance และ scalability:** รักษา p95 ≤ 500 ms ที่ 200 orders/s พร้อมจัดการเพดานและความผิดพลาดของ MailFast (`docs/scenario.md:24–26`)
4. **Modifiability:** เพิ่มงานหลังสั่งซื้อในปีถัดไปได้โดยลดการแก้ `checkoutservice` (`docs/scenario.md:14`)
5. **Operability และ cost:** ให้ทีม 4 คนดูแลได้บน Kubernetes ตามทักษะและข้อจำกัด self-hosted (`docs/scenario.md:29–30`)

## Considered options

ทุกทางเลือกต่อไปนี้เป็นข้อเสนอสำหรับ R-12 ไม่ใช่คำอธิบายระบบที่มีอยู่

1. เรียก MailFast, `loyaltyservice` และตัวบันทึก analytics แบบ synchronous จาก `checkoutservice`
2. บันทึกงานอย่างทนทาน แล้วให้ worker ส่วนกลางจัดลำดับและเรียกบริการแต่ละตัวผ่าน API
3. บันทึกออเดอร์พร้อม transactional outbox แล้วกระจาย `OrderPlaced` ผ่าน RabbitMQ ไปยังคิวและ consumer แยกตามงาน

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. เรียกแบบ synchronous | โครงสร้างใกล้โค้ดปัจจุบัน; ติดตามผลแต่ละการเรียกได้ตรงไปตรงมา | ต้องรอผู้ให้บริการที่ช้า; การขัดข้องกระทบเวลา checkout แม้กลืนข้อผิดพลาด; การ retry อย่างทนทานต้องเพิ่มกลไกอีก และทุกงานใหม่เพิ่ม coupling กับ checkout |
| 2. worker ส่วนกลางกับ durable job store | ตอบ checkout ก่อนทำงานภายนอกได้; รวมการ retry และติดตามสถานะไว้จุดเดียว; ไม่ต้องเพิ่ม message broker | ต้องสร้างการจองงาน การกู้คืน และสถานะรายงานย่อย; หากจัดลำดับร่วมกัน อีเมลค้างอาจหน่วงคะแนน; ทุกงานใหม่ต้องแก้ตัวประสานงาน |
| 3. RabbitMQ กับ outbox และคิวแยก | แยกความขัดข้องและ backlog ของแต่ละงาน; เพิ่ม consumer สำหรับงานใหม่ได้; สอดคล้องกับประสบการณ์ RabbitMQ ของสมาชิก 2 คน (`docs/scenario.md:29`) | เพิ่ม broker และ durable store ที่ต้องดูแล; มี eventual consistency, การส่งซ้ำ และความซับซ้อนในการติดตามเหตุการณ์; ต้องจัดการช่องว่างระหว่าง payment/shipping กับการบันทึกออเดอร์ |

## Decision

เลือก **ทางเลือก 3** เพื่อแยกเวลาทำงานของ MailFast และงานหลังสั่งซื้อออกจาก checkout และรองรับการเพิ่มงานตาม R-12 (`docs/scenario.md:7–14,24–30`) รายละเอียดต่อไปนี้เป็น **ข้อเสนอใหม่** ทั้งหมด:

1. **กำหนดจุดสำเร็จและบันทึกเหตุการณ์อย่างทนทาน:** หลัง payment และ shipping สำเร็จ ให้บันทึกสถานะออเดอร์ที่สำเร็จพร้อม outbox `OrderPlaced` ใน transaction เดียวของ durable store ก่อนตอบสำเร็จ แทนจุดเรียกอีเมลเดิม (`system/online-boutique/src/checkoutservice/main.go:252–279`) ให้ข้อมูลเหตุการณ์มี `event_id`, `order_id`, เวลาสั่งซื้อ, รุ่น schema และข้อมูลขั้นต่ำที่แต่ละงานต้องใช้ โดยไม่ส่งข้อมูลบัตรเครดิต เทคโนโลยีของ store และการกู้คืนยังต้องตัดสินใจตาม Open questions
2. **ส่งจาก outbox:** ให้ publisher ส่งเหตุการณ์เข้า RabbitMQ แล้วบันทึกว่าส่งสำเร็จเมื่อได้รับ publisher confirm; หากผลไม่แน่ชัดให้ส่งซ้ำด้วยตัวระบุเดิม กำหนด persistent messages, durable exchange และคิวแบบ replicated พร้อมการตรวจสอบ routing เพื่อให้ครบทุกงานที่บังคับทำ หาก broker ใช้งานไม่ได้ ให้เก็บงานไว้ใน outbox และส่งภายหลังโดยไม่รอ broker ในคำขอ checkout
3. **แยกคิวสำหรับทุกงาน:** ใช้ exchange กระจายสำเนาไปยังคิวอีเมล คิวคะแนน และคิว analytics แต่ละคิวมี consumer, retry และการติดตามแยกกัน เพิ่ม SMS หรืองานอื่นผ่านคิวใหม่เมื่อมี requirement ที่ชัดเจน ทุกงานต้องได้รับเหตุการณ์ของตนเอง ไม่แย่งรับจากคิวเดียวกัน
4. **แทนที่ mock emailservice:** ย้ายการส่งอีเมลไป consumer ที่เชื่อม MailFast และเลิกเรียก `SendOrderConfirmation` จากเส้นทาง `PlaceOrder` เดิม (`system/online-boutique/src/checkoutservice/main.go:273–277,379–383`) กำหนด rate limit รวมทุก replica และ retry ไม่เกิน 50 calls/s พร้อมจำกัดงานที่กำลังส่งตามผลทดสอบ; protocol และ API ของ MailFast ยังไม่ระบุ (`docs/scenario.md:26`) ใช้ `order_id` เป็น idempotency key หากผู้ให้บริการรองรับ
5. **ให้คะแนนครั้งเดียวที่ผลธุรกิจ:** ให้ consumer ของ `loyaltyservice` บันทึก `order_id` ที่ประมวลผลแล้วและเพิ่มคะแนนใน transaction เดียว โดยบังคับ uniqueness ตามออเดอร์ แล้วจึง ACK เหตุการณ์ การส่งซ้ำต้องคืนผลเดิมโดยไม่เพิ่มคะแนนอีก ข้อเสนอนี้ใช้การส่งแบบ at-least-once ร่วมกับ idempotent processing เพื่อให้ผลธุรกิจ exactly once ไม่ถือว่า broker ป้องกันข้อความซ้ำได้เอง (`docs/scenario.md:27`)
6. **บันทึก analytics และกู้คืนรายงานย่อย:** ตัวบันทึก analytics ต้องเก็บเหตุการณ์สำเร็จอย่างทนทานก่อน ACK และกำจัดรายการซ้ำด้วยตัวระบุเหตุการณ์ แต่ละ consumer ใช้ retry แบบหน่วงเวลาและ backoff; งานที่ยังแก้ไม่ได้เข้า dead-letter queue พร้อมแจ้งเตือนและ replay ด้วยตัวระบุเดิม จำนวน retry และเวลาหน่วงต้องกำหนดจากงบเวลาและผลทดสอบ

**ขอบเขตของการรับประกัน:** transaction ของออเดอร์กับ outbox ไม่ครอบคลุม payment และ shipping ที่เรียกภายนอกก่อนหน้า (`system/online-boutique/src/checkoutservice/main.go:252–271`) จึงต้องมีการกู้คืนกรณี process หยุดหลังเรียกบริการสำเร็จแต่ก่อน commit รวมถึงตัวระบุออเดอร์ที่คงเดิมเมื่อ client retry; UUID ปัจจุบันถูกสร้างใหม่ทุกคำขอ (`main.go:233–236` ในไฟล์เดียวกัน) ยังอ้างการรับประกันครบทั้งเส้นทางไม่ได้จนกว่าจะออกแบบและทดสอบส่วนนี้

หาก durable store ใช้งานไม่ได้ การรับออเดอร์อย่างทนทานอาจหยุดชะงัก จึงเป็น dependency ใหม่ที่ต้องประเมิน availability ส่วนการหยุดของอีเมล คะแนน หรือ analytics จะไม่ถูกนำไปทำให้ออเดอร์ที่บันทึกสำเร็จแล้วล้มเหลว เป้าหมายคะแนนภายใน 1 นาทีไม่สามารถรับประกันระหว่างการขัดข้องที่ยาวเกิน 1 นาทีได้ ต้องตกลงเงื่อนไขนี้ก่อนรับรองการออกแบบ (`docs/scenario.md:27–28`)

## Consequences

- **Positive:** checkout ไม่ต้องรอ MailFast หรือ consumer แต่ละตัว จึงลดการผูก latency และความพร้อมใช้งานเข้าด้วยกัน; ยังต้องวัดเวลาบันทึกออเดอร์/outbox จริง (quality attribute: performance, availability; `docs/scenario.md:24,26,28`)
- **Positive:** คิวคะแนนไม่ติดตาม backlog อีเมล และเพิ่มงานใหม่ผ่าน consumer ได้ (quality attribute: scalability, modifiability; `docs/scenario.md:14,25–27`)
- **Negative:** ผลอีเมล คะแนน และ analytics เกิดภายหลังการตอบ checkout; เพิ่ม broker, durable store, dashboard และภาระกู้คืนให้ทีม 4 คน (quality attribute: consistency, operability, cost; `docs/scenario.md:27,29–30`)
- **Risks and mitigations:** เหตุการณ์ซ้ำอาจเพิ่มคะแนนซ้ำ จึงต้องใช้ uniqueness และ transaction ภายใน `loyaltyservice` รวมถึงทดสอบการหยุดก่อน ACK (quality attribute: correctness; `docs/scenario.md:27`)
- **Risks and mitigations:** MailFast ระบายงานไม่ทันโหลดสูงสุด จึงต้องวัดอายุและขนาดคิว วางแผนพื้นที่เก็บ และตกลงความล่าช้าหรือเพิ่มโควตากับผู้ให้บริการ; retry ต้องใช้โควตาร่วมด้วย (quality attribute: capacity, timeliness, cost; `docs/scenario.md:25–26`)
- **Risks and mitigations:** outbox, broker หรือ consumer ขัดข้องอาจทำให้คะแนนเกิน 1 นาที จึงติดตามเวลาตั้งแต่สั่งซื้อถึงเพิ่มคะแนน พร้อมแจ้งเตือนก่อนพ้นกำหนดและมีขั้นตอนกู้คืน; replication เพียงอย่างเดียวไม่รับประกันกำหนดเวลา (quality attribute: availability, timeliness; `docs/scenario.md:27–28`)
- **Risks and mitigations:** process หยุดระหว่าง payment/shipping กับการบันทึกออเดอร์อาจทำให้งานตกหล่นหรือเกิดการทำซ้ำ ต้องออกแบบ reconciliation และ idempotency ของ checkout; จำกัดข้อมูลส่วนบุคคลใน event และสิทธิ์เข้าถึงแต่ละคิว (quality attribute: durability, correctness, security; `system/online-boutique/src/checkoutservice/main.go:233–279,379–389`)

## Validation

แผนต่อไปนี้เป็นเกณฑ์ตรวจสอบข้อเสนอ ยังไม่ได้ดำเนินการทดสอบ:

- ทดสอบโหลด 200 orders/s พร้อมจำลอง latency, ความผิดพลาด และ rate limit ของ MailFast ตาม scenario; วัด p95 `PlaceOrder` ให้ ≤ 500 ms และอัตราการเรียก MailFast รวม retry ให้ ≤ 50/s (`docs/scenario.md:24–26`) เก็บการกระจายเวลาจริงโดยไม่บวก p95 ของแต่ละบริการเข้าด้วยกัน
- วัดตั้งแต่จุดที่ถือว่าออเดอร์สำเร็จจนคะแนน commit ให้ไม่เกิน 1 นาทีสำหรับทุกออเดอร์ในเงื่อนไขการทดสอบที่ตกลงกัน ไม่ใช้เพียงค่าเฉลี่ย; ตรวจยอดคะแนนเทียบกับออเดอร์ไม่ให้ขาดหรือซ้ำ (`docs/scenario.md:27`)
- ส่งเหตุการณ์ซ้ำและ replay จาก dead-letter queue; หยุด `loyaltyservice` หลัง commit แต่ก่อน ACK แล้วตรวจว่าคะแนนเพิ่มครั้งเดียว ตรวจ analytics ไม่ขาดและไม่ซ้ำด้วยตัวระบุเดิม
- ปิด MailFast, `loyaltyservice` และตัวบันทึก analytics ทีละตัวและพร้อมกัน; ตรวจว่าออเดอร์ยังสำเร็จ จากนั้นเปิดกลับและตรวจการกู้คืนงานครบ (`docs/scenario.md:28`) บันทึกผลกรณีเกินกำหนดคะแนนด้วย
- หยุด broker และ publisher; ทดสอบการหยุดหลัง publish แต่ก่อนบันทึกสถานะ outbox และการเริ่มใหม่ ตรวจออเดอร์ที่ตอบสำเร็จเทียบกับเหตุการณ์และผลของทั้งสามงานทุกตัว
- ทดสอบการหยุดหลัง payment, หลัง shipping และก่อน/หลัง transaction ของออเดอร์; ทดสอบ client retry เพื่อยืนยันกลไก reconciliation และตัวระบุคงเดิมก่อนรับรองความทนทานทั้งเส้นทาง
- วัด backlog อีเมล พื้นที่เก็บ และเวลาระบายคิวตามระยะเวลาโหลดสูงสุดที่ได้รับคำตอบแล้ว พร้อมตรวจว่า backlog นี้ไม่ทำให้คิวคะแนนพ้น 1 นาที

## Open questions

ข้อมูลต่อไปนี้ **Not evidenced in the provided files** เว้นแต่ระบุว่าเป็นความขัดแย้งที่ต้องตกลง:

- ใครเป็นผู้ตัดสินใจและรับผิดชอบการดูแล broker, durable store, consumer และการ replay?
- จะใช้ durable store ใด และกำหนดจุดที่ถือว่าออเดอร์สำเร็จอย่างไร? จะกู้คืนหรือชดเชยเมื่อ payment/shipping สำเร็จแต่บันทึกออเดอร์ไม่ได้อย่างไร? บริการเหล่านั้นรองรับ idempotency และการตรวจสอบผลเดิมหรือไม่?
- จะคง `order_id` เดิมข้าม client retry อย่างไร? `PlaceOrder` ปัจจุบันสร้าง UUID ใหม่ทุกครั้ง (`system/online-boutique/src/checkoutservice/main.go:233–236`)
- ข้อกำหนดคะแนนภายใน 1 นาทีครอบคลุม outage ยาวเกิน 1 นาทีด้วยหรือไม่ และยอมรับการละเมิดกำหนดเวลาแบบใด? ต้องตกลงความขัดแย้งนี้กับการให้ออเดอร์สำเร็จขณะ `loyaltyservice` ล่ม (`docs/scenario.md:27–28`)
- สูตรคะแนน ฐานข้อมูลและขอบเขต transaction ของ `loyaltyservice` รวมถึงวิธีแก้คะแนนเมื่อคืนเงินหรือยกเลิกออเดอร์คืออะไร?
- โหลดสูงสุดอยู่นานเท่าใด? อีเมลและ analytics ยอมล่าช้าได้เท่าใด? เพิ่มโควตา MailFast ได้หรือไม่ และการจำกัด 50 emails/s นับ retry อย่างไร?
- MailFast ใช้ protocol/API ใด รองรับ idempotency key หรือการตรวจสถานะหลัง timeout หรือไม่? จะจัดการกรณีส่งสำเร็จแต่ไม่ได้รับคำตอบอย่างไร?
- analytics จะบันทึกที่ใด ใครเป็นเจ้าของ schema และต้องใช้ข้อมูลใด? นโยบายเก็บข้อมูลส่วนบุคคล ระยะเวลาเก็บ event/outbox และสิทธิ์เข้าถึงคืออะไร?
- งบทรัพยากร รูปแบบ replication/backup เป้าหมายการกู้คืน และเกณฑ์แจ้งเตือนที่ทีมรับผิดชอบได้คืออะไร?
- fraud scoring ที่คาดในปีหน้ามีผลปฏิเสธออเดอร์ก่อนสำเร็จหรือเป็นการตรวจภายหลัง? หากต้องตัดสินก่อนสำเร็จ จะต้องทบทวนขอบเขตงานหลังสั่งซื้อนี้ (`docs/scenario.md:14`)
