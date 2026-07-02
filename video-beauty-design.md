# Design Doc — ฟิลเตอร์แต่งหน้าสวยในวิดีโอ (on-device, real-time, temporal-stable)

> สถานะ: **DESIGN / pre-implementation** (ค้นคว้าก่อนลงมือ) · วันที่ 2026-06-21 · **rev3** (on-device guarantee + ไฮบริด)
> ต่อยอดจาก image pipeline ที่ทำงานแล้ว (`pipeline.py`, `model.py`, `face_reshape.py`, `skin_mask.py`)
> เป้าหมาย: **มือถือ on-device · เรียลไทม์ (<33ms/เฟรม) · กันกระพริบเต็มสูบ**

---

## 0. TL;DR ของการตัดสินใจ

1. **เอาโมเดลภาพมารันทีละเฟรมไม่ได้** — กระพริบแน่นอน (inherent problem ที่ยืนยันในงานวิจัย)
2. สถาปัตยกรรม = **Hybrid**: reshape เป็น **GPU shader** (geometric warp, ไม่ใช้ NN) + skin beauty เลือกได้ระหว่าง **shader (v1)** หรือ **CNN tier (v2)**
3. กันกระพริบหลายชั้น: One-Euro บน landmarks **+ บน transform**, stabilized canonical-space, **explicit temporal loss (geometric+photometric)** — canonical space อย่างเดียว **ไม่พอ**; และ 🔴 **GroupNorm ในโมเดลเดิมเป็นแหล่ง flicker ที่ซ่อนอยู่** ต้องแก้ที่สถาปัตยกรรม (ดู §1)
4. Mobile runtime: **CoreML (iOS17+/ANE)** + **TFLite GPU/NNAPI (Android)**; ONNX contract เดิมใช้ตรงไม่ได้ ต้อง convert
5. **License เป็น gate ก่อน ship** — data ปัจจุบัน (FFHQ) commercial ไม่ได้

---

## 0.1 🔴 กฎเหล็ก: Inference 100% on-device — server โหลด = 0 (rev3)

**ข้อกำหนดบังคับ: ฟิลเตอร์ต้องคำนวณบนมือถือผู้ใช้ทุกเฟรม. ห้ามมี per-frame server compute เด็ดขาด** (ไม่งั้น server ตายเมื่อสเกล). ดีไซน์นี้ต้องการเช่นนั้นโดยโครงสร้าง:

| | รันที่ไหน | ตอนไหน | โหลด server/ผู้ใช้ |
|---|---|---|---|
| **Training** | L4 server (dev) | ครั้งเดียวตอนสร้างโมเดล | 0 — ไม่เกี่ยวผู้ใช้ |
| **Inference (ฟิลเตอร์)** | **มือถือผู้ใช้** | ทุกเฟรม | **0 — ไม่มี call server** |

- โมเดลที่เทรนเสร็จ = ไฟล์ `.mlmodel`/`.tflite` ~3MB → **ฝังในแอป** หรือโหลดครั้งเดียวจาก **static CDN** (ไม่ใช่ compute)
- server touchpoint ที่อนุญาต: (1) โฮสต์ไฟล์โมเดล static (2) analytics — **ไม่มีอันไหนเป็น per-frame compute**
- ❌ **ห้าม** reuse C++ server engine เดิมเป็น runtime ของวิดีโอ (นั่นคือ path ที่ทำ server ตาย)

**ผลต่อแผน — shader-first การันตี server 0 ตั้งแต่ v1:**
- **v1 (Phase A) = shader ล้วน ไม่มี NN** → ไม่มีแม้โมเดลให้โหลด, server ไม่เกี่ยวแม้ตอน train
- **v2 (Phase B) = เติม CNN skin tier** (เทรน L4 ครั้งเดียว → ฝังไฟล์) เฉพาะเมื่อต้องการคุณภาพเกิน shader; runtime ยัง 0
- ถ้ากังวล server มาก: **อยู่ที่ shader ตลอดก็ได้** (TikTok/Meitu ยุคแรก = shader 30fps, ดู §2 Stage [5] Route A)

## 0.2 ตัวเลือกไฮบริด — อันไหนได้ อันไหนตาย (rev3)

"ไฮบริด" มี 2 ความหมายที่ผลต่างกันสุดขั้ว:

| Path | live preview | export/HD | server โหลด | ใช้ได้ไหม |
|---|---|---|---|---|
| **A. On-device ล้วน** | มือถือ | มือถือ | **0** | ✅ default |
| **B. ไฮบริด export** | มือถือ | **server async + คิว** | **bounded** (คุมได้) | ✅ ถ้าต้องการ HD เกินมือถือ |
| **C. ไฮบริด live-loop** | server (ส่งเฟรมไป-กลับ) | — | ☠️ ตามจำนวนคนถ่ายสด | ❌ **ห้าม** |

- **C ตาย** เพราะ: latency ไป-กลับ >100ms → 30fps พัง + bandwidth สตรีมทุกเฟรม + โหลด = คนถ่ายสดพร้อมกัน (คือ path ที่ทำ server ตาย)
- **B ปลอดภัย** เพราะ: live = on-device (server 0); ขึ้น server เฉพาะตอนกด export เป็น **offline async + คิว + rate-limit** → โหลด bounded ตามจำนวน export ไม่ใช่จำนวนคนดู
- 💰 **B reuse C++ engine เดิมได้** — งาน still-image per-frame ที่มีอยู่แล้ว เอามาไล่เฟรม offline ได้ทันที (low effort)
- เงื่อนไขบังคับของ B: **queue + rate-limit + cost cap** ก่อนเปิด ไม่งั้นกลายเป็น C ทางอ้อม

→ **แผน: A เป็น default; B เป็น option เสริม (HD export); C ห้ามแตะ**

---

## 1. ปัญหาหลัก: ทำไมวิดีโอ ≠ ภาพนิ่ง × N

| ประเด็น | ภาพนิ่ง (ของเดิม) | วิดีโอมือถือเรียลไทม์ |
|---|---|---|
| Runtime | C++ engine + ONNX Runtime / L4 | CoreML / TFLite-GPU / ORT-Mobile |
| งบเวลา | วินาที/รูปได้ | **<33ms/เฟรม** ครบทั้ง pipeline |
| Flicker | ไม่เกี่ยว | **ตัวฆ่างาน** |
| รันเต็มเฟรม | ได้ | แพงเกิน → process แค่ face-crop ลด res |

**แหล่งกำเนิด flicker 4 ทาง (แก้คนละจุด):**
- **(a) landmark jitter** → reshape สั่น / หน้าเต้น
- **(b) skin-mask jitter** → เขตเนียน "หายใจ" เข้าออก
- **(c) NN output flicker** → input ต่างนิดเดียว output เด้งคนละทาง
- **(d) 🔴 GroupNorm / per-frame normalization** → โมเดลเดิม (`model.py`) ใช้ GroupNorm ที่ normalize ด้วยสถิติ**ของเฟรมนั้นเฟรมเดียว**; พอ auto-exposure/brightness กล้องมือถือขยับ → output เด้ง**ทั้งภาพ** = flicker ที่ **temporal loss กดลงยาก** เพราะมันเกิดที่ตัว normalization ไม่ใช่ที่ output. (review rev2 — จุดบอดเดิมที่ระบุไม่ครบ)
  → **แก้ที่สถาปัตยกรรม ไม่ใช่จูนทีหลัง**: เอา GroupNorm ออกใช้ residual-scale แทน / หรือใช้สถิติคงที่ (running/calibrated) ไม่ใช่ per-frame. ตัดสินก่อนเทรน Route B

> หลักฐาน: การแก้ภาพแบบ frame-by-frame ทำให้เกิด temporal flicker อย่างหลีกเลี่ยงไม่ได้ — [arXiv 2007.01466](https://arxiv.org/pdf/2007.01466)
> per-frame instance/group normalization เป็นแหล่ง temporal inconsistency ที่รู้กันในงาน video translation (ต้องระวังเป็นพิเศษเมื่อ port โมเดลภาพ→วิดีโอ)

---

## 2. สถาปัตยกรรม (Hybrid Shader + Tiny-NN ใน stabilized face space)

```
กล้อง/เฟรม
 └─[1] FaceLandmarker Tasks (478 จุด)         ← reuse: API นี้เป็น mobile-first อยู่แล้ว
   └─[2] One-Euro filter บน landmarks          ← กัน flicker (a) ที่ต้นตอ
     └─[3] warp face-crop → canonical space (นิ่ง)
        ├─[4] RESHAPE = GPU fragment shader     ← port face_reshape.py liquify → GLSL/Metal
        └─[5] SKIN BEAUTY  (Route A shader | Route B CNN)
           └─[6] warp กลับ + composite (skin-mask feather)
```

### Stage [1] Detection — reuse ได้เลย
MediaPipe **FaceLandmarker Tasks API** (478 จุด) มี build สำหรับ iOS/Android อยู่แล้ว ใช้ต่อได้ทันที
> หมายเหตุ production มักใช้ landmark น้อยจุดกว่าได้ (เช่น 106-point) ถ้าต้องการเบาลง

### Stage [2] One-Euro filter — กันกระพริบชั้นที่ 1
ปรับแค่ 2 พารามิเตอร์:
- `mincutoff` (ต่ำ = เนียนตอนนิ่ง แต่ lag มากตอนขยับ)
- `beta` (สูง = ลด lag ตอนขยับเร็ว)

แนะนำเริ่มจาก `mincutoff ≈ 1.0`, `beta ≈ 0.007` แล้วจูนต่อหน้างาน
> ⚠️ **ค่านี้ขึ้นกับหน่วยของสัญญาณ** — ใช้ได้เมื่อ landmark เป็น **pixel** ที่ fps ~30. ถ้าใช้พิกัด normalized (0–1) ค่า cutoff ต้อง re-derive ใหม่ทั้งชุด ไม่งั้น filter เพี้ยน (กรองแรง/อ่อนผิด). ต้องวัด jitter จริงก่อนล็อกค่า
> [Casiez "1€ Filter" CHI 2012](https://gery.casiez.net/1euro/) · [implementation note](https://jaantollander.com/post/noise-filtering-using-one-euro-filter/)
> เลือก One-Euro over Kalman: latency ต่ำกว่า เหมาะ interactive, จูนง่าย (2 ตัว)

### Stage [3] Stabilized canonical-space warp
rigid-align face-crop เข้า template คงที่ (จาก landmarks ที่ filter แล้ว) → process → warp กลับ
ลด flicker เชิงเรขาคณิตได้ดี **แต่ไม่ใช่ยาวิเศษ**:
> ⚠️ **canonical space อย่างเดียวไม่พอ** — ยังต้องมี explicit temporal-consistency loss ตอนเทรน
> [arXiv 2007.01466](https://arxiv.org/pdf/2007.01466) · [BFVR-STC arXiv 2411.16468](https://arxiv.org/html/2411.16468v1)
> 🔴 **ต้อง filter ตัว transform เองด้วย ไม่ใช่แค่ landmarks** (review rev2): rigid-align fit มาจาก landmarks — ถ้า fit จาก landmarks ดิบ jitter จะกลับเข้ามาทาง **transform matrix** (scale/rotation/translation) แม้ filter landmarks แล้ว. → ใส่ One-Euro อีกชั้นบนพารามิเตอร์ transform (หรือ fit จาก landmarks ที่ filter แล้วเท่านั้น + smooth matrix). ระวัง: การ resample warp ที่สั่น = แหล่ง shimmer เพิ่ม

### Stage [4] Reshape = GPU shader (ไม่ใช้ NN)
port logic `face_reshape.py` (local-translation/scale liquify) → fragment shader
ยืนยันว่าเป็นแนวทาง production: face-reshape/瘦脸 = **geometric warp** ไม่ใช่ NN
> [Moving Least Squares arXiv 1910.13671](https://arxiv.org/pdf/1910.13671) · Intel [US10152778B2](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/10152778)
> ⚠️ ห้ามอ้างว่า "Snap ทำแบบนี้" — ตก verify (1-2), ยืนยัน Snap-specific ไม่ได้; หลักการ MLS ยังยืน
- ฟรีเชิง compute, นิ่งโดยโครงสร้าง (deterministic)
- ตาโตแบบ TikTok = งานชั้นนี้ (geometric) **ไม่ใช่** skin model

### Stage [5] Skin beauty — 2 route

**Route A — Shader (แนะนำ v1 เรียลไทม์)**
bilateral/surface-blur + frequency-separation + even-tone, gate ด้วย skin-mask, ทั้งหมดใน shader
= สิ่งที่ classical target ของเราคำนวณอยู่แล้ว (`beautify_data.py:pro_beautify_target`, `glam_classical.py`) แค่ย้าย numpy/cv2 → GLSL/Metal
- ✅ deterministic → **ลด flicker (c) ได้มาก** (ไม่มี NN เด้ง)
- ⚠️ 🔴 **ไม่ได้หายสนิท** (review rev2): shader deterministic ก็จริง แต่ **input ไม่ deterministic** — sensor noise ของกล้องทำให้ bilateral/surface-blur ออกผลต่างทุกเฟรมแม้หน้านิ่ง → ผิว "ซ่า"/shimmer. → ยังต้องมี temporal denoise เบาๆ (EMA บน output ใน stabilized space) คู่กัน
- ✅ เร็วมาก, strength slider ฟรี
- ⚠️ เพดาน = classical (ไม่มี learned pore preservation)
> beauty pipeline ทำ 30fps ด้วย shader ได้ตั้งแต่ 2014 — Intel [US10152778B2](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/10152778)
> commercial neural retouch ก็แยก frequency-separation เป็น stage ต่างหาก — Adobe [US10593023B2](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/10593023)

**Route B — CNN tier (คุณภาพสูง / "โมเดล" ที่อยากได้)**
FCN เดิม แต่ต้อง **เทรนใหม่ให้ temporal-aware** + รันใน stabilized space ลด res (face-crop 256–384)
- 🔴 **เปลี่ยน GroupNorm ก่อน** (review rev2, ดูแหล่ง flicker (d)) — ไม่งั้น temporal loss สู้ exposure-shift ไม่ไหว
- เทรนด้วย **synthetic-motion equivariance loss**: เอา still รูปเดียว สุ่ม warp/homography → บังคับ `model(warp(x)) ≈ warp(model(x))` → ได้ temporal consistency **โดยไม่ต้องมี video dataset**
- 🔴 **เสริม photometric augmentation คู่เฟรม** (review rev2): equivariance loss คุมแค่ **geometric** (ขยับ/หมุน/ซูม). flicker จริงมาจาก lighting/exposure/motion-blur/occlusion ด้วย → สุ่ม brightness/gamma/noise/blur ระหว่างคู่เฟรมในตอนเทรน เพื่อให้ทน photometric change
- เติม **Laplacian-pyramid high-freq reinjection** (SOTA HGFR, level l=2) กัน plastic + เก็บ pore
  > ⚠️ pyramid หลาย level **ไม่ฟรีบน GPU มือถือ** — ต้อง cost เข้า latency budget (§3.1) ก่อนยืนยัน l=2
- One-Euro บน **alpha/strength + output blend** กัน flicker (c) ที่เหลือ
- ⚠️ ใช้ recipe กันพังเดิม: **bf16 (ไม่มี GradScaler) + grad-clip 1.0 + warmup 300, ปิด SSIM** (ดู "บทเรียน" §5)
> SOTA = feed-forward fully-conv + Laplacian high-freq reinjection — ICCV2025 [HGFR](https://arxiv.org/abs/2510.11613)
> diffusion-teacher → synthetic pairs → student พิสูจน์แล้ว (pref 83% vs 16%) — [ICCV2025 paper](https://openaccess.thecvf.com/content/ICCV2025/papers/Xu_Face_Retouching_with_Diffusion_Data_Generation_and_Spectral_Restorement_ICCV_2025_paper.pdf)

### Stage [6] Composite
warp กลับพิกัดเดิม + blend ด้วย skin-mask feather (reuse `skin_mask.py` logic → shader)

---

## 2.1 Scope (review rev2)
- **v1 = หน้าเดียว** — face-crop + canonical-space ออกแบบบนสมมติ 1 หน้า. หลายหน้า/วิดีโอกลุ่มจะคูณ cost (detect+warp+NN ต่อหน้า) → เลื่อนเป็น v2, ระบุ scope ให้ชัดตั้งแต่ต้น

## 3. Mobile runtime

### 3.1 Latency budget (33ms @30fps) — ต้องจัดสรร ไม่ใช่ก้อนเดียว 🔴
review rev2: ดีไซน์เดิมพูด 33ms เป็นก้อนเดียว เสี่ยงพังเงียบ. ต้องจัดงบต่อ stage + **วัดจริงก่อนคอมมิต Route B**:

| Stage | งบโดยประมาณ | หมายเหตุ |
|---|---|---|
| [1] FaceLandmarker | ~5–15ms | กินเยอะบน mid-range; เลือก 106-pt ถ้าจำเป็น |
| [2] One-Euro | <1ms | CPU เบา |
| [3] warp in/out | ~2–4ms | GPU resample |
| [4] reshape shader | ~2–3ms | geometric, ถูก |
| [5] skin | A: ~3–5ms (shader) / B: **?ms** (CNN @384² — **ยังไม่มีตัวเลข**) | ตัวแปรเสี่ยงสุด |
| [6] composite | ~1–2ms | blend |

> ❗ Route B จะฟิตหรือไม่ขึ้นกับ [5] CNN latency ที่ **ยังไม่ได้วัด** — เป็น gate ของ Phase B (§6)

### 3.2 Thermal throttling 🔴
รัน NN+shader 30fps ต่อเนื่อง → เครื่องร้อน → throttle → fps ตกหลัง 2–3 นาที (ปัญหา production จริงของ beauty app). → ต้องมี **dynamic quality fallback** (ลด res/สลับ Route B→A เมื่อ fps drop)

### 3.3 Runtime matrix

| Platform | Runtime | ข้อกำหนด | หมายเหตุ |
|---|---|---|---|
| iOS | **CoreML** (ANE) | **iOS 17+** ได้เต็มที่ | convert ด้วย coremltools |
| iOS (เก่า) | TFLite Core ML delegate | **Apple A12+** | ต่ำกว่า fallback GPU/CPU |
| Android | **TFLite** GPU/NNAPI delegate | GPU delegate กว้างสุด | NNAPI แตกต่างราย OEM |
| ทั้งคู่ (fallback) | ONNX Runtime Mobile | — | dynamic-shape pitfalls |

> [Apple coremltools](https://apple.github.io/coremltools/docs-guides/source/opt-palettization-perf.html) · [TFLite Core ML delegate](https://blog.tensorflow.org/2020/04/tensorflow-lite-core-ml-delegate-faster-inference-iphones-ipads.html)

**Convert pitfalls ที่ต้องเทสต์ก่อน:**
- **dynamic shape** — engine เดิม contract เป็น dynamic H/W; mobile หลายตัวชอบ fixed shape → fix face-crop เป็น res คงที่ (เช่น 384²) ตอน export
- **GroupNorm / dilated conv** — ต้องเช็ค op support ของ delegate ปลายทาง (อาจ fallback CPU = ช้า)
- **quantization** — int8 เร็วสุดแต่กระทบคุณภาพผิว; เริ่มที่ **fp16** ก่อน, วัด PSNR/identity ก่อนลง int8

---

## 4. License / Legal — **gate ก่อน ship** 🔴

| สิ่งที่ใช้ | License | ใช้ commercial ได้ไหม |
|---|---|---|
| **FFHQ** (data ปัจจุบันทั้งหมด) | CC BY-NC-SA | ❌ NonCommercial |
| CelebA-HQ | non-commercial | ❌ |
| **SFHQ** synthetic ~425K | MIT | ⚠️ MIT แต่มี StyleGAN2/FFHQ **taint** upstream |
| InsightFace **code** | MIT | ✅ |
| ArcFace **pretrained weights** | NC research-only | ❌ ใช้เป็น identity-loss/gate ใน commercial student = taint risk |

> [FFHQ LICENSE](https://github.com/NVlabs/ffhq-dataset/blob/master/LICENSE.txt) · [SFHQ](https://github.com/SelfishGene/SFHQ-dataset) · [InsightFace](https://github.com/deepinsight/insightface)

**ข้อควรระวังที่ตก verify (ห้ามเชื่อ):**
- ❌ "synthetic faces = ไม่มี BIPA/GDPR concern เลย" — **ผิด** (ตก verify 0-3). Face filter app เก็บ biometric ของ **ผู้ใช้จริง** → ยังต้องมี consent flow + privacy policy ไม่ว่า training data จะ synthetic หรือไม่

**Action ก่อน ship commercial:**
1. ย้าย training data ออกจาก FFHQ → SFHQ (รับ taint risk) หรือ dataset commercial จริง
2. หา identity embedder ที่ commercial-licensed (เลี่ยง ArcFace NC weights)
3. consent + biometric privacy flow ในแอป (BIPA/GDPR)
4. ปรึกษาทนายเรื่อง teacher-taint (diffusion teacher / GFPGAN / CodeFormer → student)

---

## 5. บทเรียนจาก image pipeline (carry over — อย่าพลาดซ้ำ)

- 🔴 **identity-collapse bug**: fp16 autocast + GradScaler → overflow → ข้าม optimizer.step ทุก step → weights แช่ init = โมเดล identity. **FIX = bf16 ไม่มี GradScaler + grad-clip + warmup**
- 🔴 **black-collapse**: SSIM loss variance ติดลบใต้ bf16 → ระเบิด → output ดำ. **ปิด SSIM, ใช้ VGG/LPIPS แทน**
- ✅ contract เดิม: input `"input"` / output `"output"`, NCHW [1,3,H,W], RGB, [0,1]
- ✅ ตาโต = geometric reshape ไม่ใช่ skin model; ตาต้อง carve-out กันโดน beautify (บทเรียน beauty_glam → glam2)

---

## 6. แผนเฟส

**Phase A — พิสูจน์เรียลไทม์ (Route A ล้วน)**
landmark + One-Euro + reshape shader + skin shader → ฟิลเตอร์นิ่งบนมือถือก่อน (ยังไม่แตะ NN)

**Phase B — คุณภาพ (Route B CNN tier)**
1. เติม Laplacian high-freq reinjection ใน `model.py`
2. เทรน temporal-aware (synthetic-motion equivariance loss) บน L4
3. convert ONNX → CoreML/TFLite, วัด fps/คุณภาพ fp16
4. One-Euro บน output blend

**Phase C — polish + ship**
strength sliders · glam/natural presets · **commercial-dataset gate** · consent flow

---

## 7. Open questions / ต้องทำต่อ
- 🔴 **วัด latency จริงของ FCN <1M params @384² บนเครื่องเป้าหมาย** — gate ของ Route B (§3.1), ยังไม่มีตัวเลข mid-range
- 🔴 **เลือกวิธีจัดการ GroupNorm** (remove+residual-scale vs running/calibrated stats) — ตัดสินก่อนเทรน Route B (§1(d))
- เลือก mesh-warp library/shader implementation สำหรับ reshape บน mobile
- ตัดสินใจ dataset commercial (SFHQ vs จัดหา licensed) — gate ของ Phase C
- temporal loss: synthetic-motion (geometric+photometric aug) เพียงพอ หรือต้องมี real-video validation set
- กลยุทธ์ dynamic quality fallback ตอน thermal throttle (§3.2)

## 8. Changelog
- **rev3** (2026-06-21) — on-device guarantee: เพิ่ม §0.1 (inference 100% on-device, server โหลด=0, shader-first การันตี v1) + §0.2 (ตัวเลือกไฮบริด A/B/C — live-loop ห้าม, export-hybrid ได้ถ้า bounded)
- **rev2** (2026-06-21) — architecture review: เพิ่มแหล่ง flicker (d) GroupNorm, transform-jitter filtering, แก้คำเคลม Route A (ไม่หายสนิท), photometric aug, latency budget table, thermal fallback, single-face scope, One-Euro unit caveat, Laplacian cost
- **rev1** (2026-06-21) — ฉบับแรกจาก deep-research (24 sources, verify 23/25)

---

## อ้างอิงหลัก
- Temporal: [2007.01466](https://arxiv.org/pdf/2007.01466) · [BFVR-STC 2411.16468](https://arxiv.org/html/2411.16468v1) · [1€ Filter](https://gery.casiez.net/1euro/)
- Reshape: [MLS 1910.13671](https://arxiv.org/pdf/1910.13671) · Intel [US10152778B2](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/10152778)
- Retouch NN: Adobe [US10593023B2](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/10593023) · ICCV2025 HGFR [2510.11613](https://arxiv.org/abs/2510.11613) / [paper](https://openaccess.thecvf.com/content/ICCV2025/papers/Xu_Face_Retouching_with_Diffusion_Data_Generation_and_Spectral_Restorement_ICCV_2025_paper.pdf)
- Runtime: [coremltools](https://apple.github.io/coremltools/docs-guides/source/opt-palettization-perf.html) · [TFLite CoreML delegate](https://blog.tensorflow.org/2020/04/tensorflow-lite-core-ml-delegate-faster-inference-iphones-ipads.html)
- License: [FFHQ](https://github.com/NVlabs/ffhq-dataset/blob/master/LICENSE.txt) · [SFHQ](https://github.com/SelfishGene/SFHQ-dataset) · [InsightFace](https://github.com/deepinsight/insightface)
