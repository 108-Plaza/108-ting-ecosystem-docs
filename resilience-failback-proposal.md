# ข้อเสนอ: Fail-back / Resilience ทั้งระบบ 108 (เพื่อ "งานไม่ติดขัด")
เวอร์ชัน 1.0 — 2026-06-19 — สำหรับพิจารณา

## 1. เป้าหมาย
ทุกระบบต้องมี **ทางสำรอง (fail-back)** เมื่อส่วนใดส่วนหนึ่งล่ม เพื่อให้ **หน้าร้านขายต่อได้** และ
ข้อมูลไม่หาย/ไม่ซ้ำ แล้วค่อย sync กลับเมื่อระบบคืนสภาพ. เอกสารนี้สรุป (ก) สิ่งที่ **มีอยู่แล้ว**
(ข) **ช่องโหว่** ตามสถานการณ์ล่ม (ค) ข้อเสนอ **จัดลำดับ P0/P1/P2**.

## 2. สิ่งที่มีอยู่แล้ว (ยืนยันจากโค้ด — ไม่ต้องทำซ้ำ)
| กลไก | อยู่ที่ไหน | ป้องกันอะไร |
|------|----------|------------|
| **Offline-first สถาปัตยกรรม branch↔cloud** | `api/src/application/services/sync/*` (checkpoint, pull, conflict_resolver, reconciliation, inbox_applier, duplicate_counter) | สาขามี DB ของตัวเอง ขายได้แม้คลาวด์/เน็ตหลุด แล้ว sync ทีหลัง พร้อม resolve conflict + กันซ้ำ |
| **Flutter POS local DB + health** | `apps/pos/lib/core/local/app_database.dart` (drift/sqlite), `core/network/branch_health.dart` | แอปแคชข้อมูล + เช็คสุขภาพ branch API |
| **Outbox (durable events)** | `pos-infra/messaging/outbox.rs` + ทุก use case (sale/shift/refund) | เหตุการณ์ไม่หายแม้ปลายทาง (AccountZing/consumer) ล่ม — ส่งซ้ำได้ |
| **Idempotency** | Payment `idempotency_service`, sale `sale_id_registry`, outbox dedup | กดซ้ำ/retry ไม่จ่ายซ้ำ ไม่สร้างซ้ำ |
| **GL posting async/sync toggle** | `gl_async_flags.rs` (pos-commerce/finance/shift) | ถ้า GL/AccountZing ล่ม สลับเป็น async (degrade) ขายไม่สะดุด |
| **Self-contained auth (HS256)** | pos108 verify token เองภายใน ไม่ต้องเรียก Identity ทุก request | Identity ล่ม → ตรวจ token เดิมได้ |
| **Shift close offline-first** | `pos-shift/accounzing_summary.rs` (no network, อ่าน local) | ปิดกะ/Z-report ได้แม้ GL ออฟไลน์ |
| **Healthcheck-gated startup** | `deploy/docker-compose*.yml` (Identity→pos108→AccountZing `service_healthy`) | ไม่สตาร์ตบริการก่อน dependency พร้อม |
| **Watchdog + auto-update guard** | `.68` `ensure-running.sh` (cron */3 + @reboot) + `update-app.sh` migration-guard + `AUTO_UPDATE` switch | กู้แอปเองเมื่อ crash/reboot; กัน auto-update กระทบ schema ตอนทำงาน |
| **Optimistic locking** | ทุก aggregate (`version`) | กัน race ตอนแก้พร้อมกัน |
| **Reprint queue** | `pos_print_jobs` | พิมพ์ใหม่ได้เมื่อปริ้นเตอร์มีปัญหา |

> สรุป: ระดับ resilience **สูงอยู่แล้ว** โดยเฉพาะ offline-first ของสาขา. ช่องโหว่ที่เหลือคือ
> **จุดล้มเดี่ยว (single point of failure) + บางทาง degrade ยังไม่มี + การมองเห็น (observability)**.

## 3. สถานการณ์ล่ม × ช่องโหว่ × ข้อเสนอ (จัดลำดับ)

### 🔴 P0 — ทำก่อน (กระทบ "ขายไม่ได้" / เงินผิด)
1. **DB ของสาขาล่ม (Postgres เดี่ยวบน .68/local)** — จุดล้มเดี่ยวที่ใหญ่สุด: DB ตาย = ทั้งสาขาขายไม่ได้.
   - *ปัจจุบัน:* มี local DB + sync แต่ **ไม่มี replica/standby/auto-restore**.
   - *เสนอ:* (ก) **automated backup ถี่ + restore script ทดสอบจริง** (RPO/RTO ชัด) ; (ข) ระยะถัดไป **PG streaming replica + auto-failover** (patroni/pg_auto_failover) ; (ค) ใช้ **Flutter local drift DB เป็น last-resort** ขายเงินสด offline เมื่อ branch API/DB ล่ม แล้ว replay เข้าเมื่อคืนสภาพ.
2. **Payment gateway (SCB/บัตร/QR) ล่มหรือ timeout** — ลูกค้าจ่ายบัตรไม่ได้ คิวค้าง.
   - *ปัจจุบัน:* มี idempotency + webhook verify; แต่ **ไม่มี circuit-breaker/timeout มาตรฐาน + fallback UX ชัด**.
   - *เสนอ:* timeout สั้น + **circuit breaker** ต่อ gateway, **fallback เป็นเงินสด/QR อื่น/บันทึกค้างจ่ายแล้วยืนยันภายหลัง**, ปุ่ม "ลองใหม่/เปลี่ยนวิธีจ่าย" ใน POS, reconcile ยอด pending.
3. **ออก token/login ใหม่ตอน Identity ล่ม** — ตรวจ token เดิมได้ (HS256) แต่ **ออก token ใหม่/เปิดกะใหม่** อาจต้องพึ่ง Identity.
   - *เสนอ:* **cached credential / local login fallback** ที่สาขา (เข้ากะได้แม้ Identity ออฟไลน์) + จำกัดเวลา + audit.

### 🟠 P1 — ทำต่อ (กระทบความน่าเชื่อถือ/ข้อมูล)
4. **Observability ของ fail-back** — ตอนนี้ "เงียบ": ไม่รู้ว่า **outbox ค้างกี่ event, sync lag เท่าไร, gateway error พุ่ง, บริการ unhealthy**.
   - *เสนอ:* metrics + **alert**: outbox backlog/age, sync checkpoint lag, payment error-rate, DB connection-pool exhaustion, shift cash-variance ผิดปกติ. (มี `/metrics` Prometheus อยู่บางที่แล้ว — ต่อยอด + dashboard + alert rule).
5. **Sync ค้าง/แตก (conflict วนไม่จบ, clock skew)** — มี reconciliation แต่ขาด **dead-letter + คนเข้าไปแก้ + alert เมื่อค้างเกิน threshold**.
   - *เสนอ:* dead-letter queue + UI ดู/แก้ outbox-stuck + alert.
6. **Redis ล่ม** (rate-limit/session/pubsub) — audit เคยพบ Redis rate-limiter mutex poison = ทั้งระบบ error.
   - *เสนอ:* **fail-open/degrade** เมื่อ Redis ล่ม (rate-limit ปิดชั่วคราวดีกว่าปฏิเสธทั้งหมด), แยก connection, ไม่ให้ poison ลาม.
7. **มาตรฐาน timeout/retry/backoff ต่อ outbound ทุกบริการ** — กระจัดกระจาย.
   - *เสนอ:* นโยบายกลาง (timeout, retry+exponential backoff+jitter, circuit breaker) เป็น lib ใช้ร่วม.

### 🟢 P2 — ทำเมื่อพร้อม (ลด blast radius / ops)
8. **.68 = ฮาร์ดแวร์เดี่ยวของ core** — เครื่องตาย = สาขาตาย.
   - *เสนอ:* **UPS** (กันไฟดับ), เครื่องสำรอง/standby + ขั้นตอนสลับ, image/restore เร็ว.
9. **Cloud services (public) ล่ม** — กระทบ dashboard/online order ไม่กระทบขายหน้าร้าน.
   - *เสนอ:* multi-instance + health-based routing + graceful degrade ของ public API.
10. **Notification/print/IoT** — outbox/queue มีแล้ว (notification SoT outbox, print jobs, IoT store-and-forward) → เพิ่มแค่ alert + retention cap (ทำ IoT buffer cap ไปแล้ว).

## 4. แนะนำลำดับลงมือ (roadmap)
1. **P0-1 DB backup+restore ที่ทดสอบจริง** (เล็ก, กันหายนะใหญ่สุด) → แล้วต่อ replica.
2. **P0-2 Payment circuit-breaker + fallback เงินสด/QR** (กระทบลูกค้าโดยตรง).
3. **P0-3 Local login fallback** (เปิดกะได้แม้ Identity ล่ม).
4. **P1-4 Observability/alert** (outbox/sync/payment/DB) — ทำให้ "เห็น" ก่อนพังจริง.
5. **P1-6 Redis fail-open**, **P1-7 timeout/circuit lib กลาง**.
6. **P2** ฮาร์ดแวร์ standby/UPS + cloud HA.

## 5. หมายเหตุ
- หลายอย่างเป็น **โค้ด** (circuit breaker, fallback, local login, alert) ทำได้ในรอบ ๆ; บางอย่างเป็น
  **infra/ops** (replica, UPS, standby) ต้องเจ้าของตัดสินใจงบ/ฮาร์ดแวร์.
- แนะนำกำหนด **RPO/RTO ต่อระบบ** (ยอมข้อมูลหายได้กี่นาที / กู้คืนภายในกี่นาที) เพื่อเลือกความเข้มของ fail-back ให้คุ้ม.
