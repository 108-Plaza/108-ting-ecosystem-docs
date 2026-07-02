# Phase A — Native Wiring Spec (build guide สำหรับ device)

> สถานะ: **DONE (Phase A wiring) — device-verified** (อัปเดต 2026-07-02; เดิม 2026-06-21 = BUILD SPEC)
> · ต่อจาก [phase-a-flutter-spec.md](phase-a-flutter-spec.md)
> สำหรับ: คน/agent ที่ build native บนเครื่องจริง (iOS Metal / Android GL ES)
>
> **⚠️ STATUS (2026-07-02):** native wiring is **largely complete + device-verified (A1–A5b, both
> platforms; iOS + Android multi-face merged 2026-06-22)** — this doc's premise "งานที่เหลือ = wire" is
> now historical. iOS routes frames through `MetalBeautyRenderer` (capture→warp→reshape→skin→composite,
> not passthrough); Android through `CameraGlPipeline`. Two divergences from the spec below: (1) the
> "iOS still passthrough → switch to process" note in §1 is done. (2) **Android shipped a simpler
> direct-space pipeline** (`slim/eye/blur/facemask/smooth_simple/oes` frags — no reshape-disp/sample
> split, no canonical-warp pair), so §0/§4/§5's canonical/split design describes the **iOS Metal**
> build; Android is a proven variant (see `packages/beauty_filter/docs/multiface-beauty.md`). Remaining
> = A6 fps/thermal + iOS↔Android tuning reconciliation. SoT = `packages/beauty_filter/HANDOFF.md`.

---

## 0. ภาพรวม pass graph (ต่อเฟรม)

```
[cap]   camera frame ──────────────► inputTex (RGBA8, frame size)
[lm]    FaceLandmarker(inputTex/CPU) ► 478 pts (pixel, frame space)
[smooth] One-Euro(pts)  + fit T(frame→canonical 384²) + One-Euro(T params)   ← CPU/native
[warp]  inputTex ⊗ T ───────────────► canonicalTex (RGBA8, 384²)        [Stage 3]
[mask]  สร้างจาก pts→canonical:
          memberTex (R8, oval hull + blur)            [ใช้โดย reshape]
          maskTex   (R8, oval − ตา/คิ้ว/ปาก, YCrCb, feather)  [ใช้โดย skin/composite]
[A4]    reshape:  dispTex = analytic(canonicalTex, member, U_reshape)   (RG16F)
          dispTex ⊗ blur H,V (σ=faceW·0.04) ► dispBlurTex
          canonicalTex sample@(uv+disp) ► reshapedTex (RGBA8, 384²)
[A5]    skin:  reshapedTex ⊗ blur H,V (s1) ► lowTex
               lowTex      ⊗ blur H,V (s2) ► evenTex
               composite(reshapedTex, low, even, mask, U_skin) ► beautyTex
[temporal] mix(beautyTex, prevBeautyTex, emaAlpha≈0.3) ► stableTex
               (canonical space = pose-normalized → EMA ตรงๆ ไม่ต้อง warp align)
               copy stableTex → prevBeautyTex  (เก็บไว้เฟรมหน้า)
[A6/out] stableTex ⊗ T⁻¹ → composite ลง inputTex ด้วย maskTex feather ► outputTex
          outputTex → Flutter Texture (registered)
```

**Textures ที่ต้องจอง** (ปลายทาง 384²; ping-pong สำหรับ blur):
`inputTex, canonicalTex, memberTex(R8), maskTex(R8), dispTex(RG16F), dispPing(RG16F), reshapedTex, lowTex, lowPing, evenTex, beautyTex, prevBeautyTex, stableTex, outputTex`
→ จัด pool/atlas; blur ใช้ ping-pong 2 ใบต่อชนิด. ทั้งหมด 384² = เบา

---

## 1. [cap] Camera → GPU texture

**iOS:** `AVCaptureVideoDataOutput` BGRA → `CVPixelBuffer` → `CVMetalTextureCacheCreateTextureFromImage` → `MTLTexture` (zero-copy). (โครงใน `BeautyFilterPlugin.swift` แล้ว — เปลี่ยนจาก passthrough เป็น process)
**Android:** `CameraX ImageAnalysis` (RGBA_8888) หรือ `Preview`→`SurfaceTexture`(OES) → GL texture. OES ต้อง sample ด้วย `samplerExternalOES` ใน pass แรก (copy เข้า RGBA2D ก่อน หรือ warp ตรง).
> A1 (passthrough) = ส่ง inputTex → Flutter texture ตรงๆ ก่อน (พิสูจน์กล้อง 30fps). A2+ ค่อยแทรก process.

## 2. [lm] FaceLandmarker (MediaPipe Tasks)

- iOS: `MPPFaceLandmarker` (Tasks, CocoaPods/SPM `MediaPipeTasksVision`); Android: `com.google.mediapipe:tasks-vision`
- model `face_landmarker.task` (float16) bundle ใน assets (Apache-2.0, commercial OK)
- mode = `.liveStream` (async) หรือ `.video` (sync ต่อเฟรม). คืน 478 normalized landmarks → คูณ frame W,H เป็น **pixel** (One-Euro ผูกหน่วย pixel)
- `numFaces = 1` (scope v1)
- ถ้าไม่เจอหน้า → ข้าม process, ส่ง inputTex ตรง (faceFound=false ใน stats)

## 3. [smooth] One-Euro + transform fit

1. **One-Euro บน landmarks** — port `lib/src/one_euro.dart` (มี Dart แล้ว) ไปฝั่ง native หรือคำนวณใน Dart แล้วส่งกลับ (แพง ทุกเฟรม → ทำ native). ค่าเริ่ม `minCutoff=1.0, beta=0.007` (pixel @30fps)
2. **fit similarity transform T** (frame→canonical) จาก anchor stable: ตา 2 ข้าง (33/263 หรือ eye centers) + จมูก 168 → Umeyama/Procrustes (scale+rot+trans) ไป canonical reference (จุดอ้างอิงคงที่ใน 384²)
3. **One-Euro บนพารามิเตอร์ T** (scale, θ, tx, ty) — 🔴 สำคัญ (review rev2): ถ้าไม่ smooth T เอง jitter กลับเข้าทาง warp
4. landmarks ใน canonical = T · pts (ใช้ทำ uniforms + mask)

## 4. [mask] สร้าง member + skin mask (canonical space)

port `skin_mask.py` → GPU passes:
- **memberTex** (reshape): raster `convexHull(FACE_OVAL)` เป็น R8=1 → `blur` (σ=faceW·0.05)
- **maskTex** (skin):
  1. raster oval hull=1 → carve ตา(LEFT/RIGHT_EYE) คิ้ว(BROW) ปาก(LIPS) hull=0 → membershipTex (native draw)
  2. **`SkinGate.metal`/`skin_gate.frag`** (✅ parity): membership × max(YCrCbGate, 0.3) → maskRaw  *(YCrCb gate port แล้ว — เหลือ raster + dilate native)*
  3. (optional) dilate skin 5×5 = max-filter pass ก่อน gate (skin_mask.py ทำ); ข้ามได้ถ้า feather พอ
  4. `blur` feather (σ=min·0.04) → maskTex
- raster: ส่ง landmark canonical เป็น vertex, draw triangle-fan/convex polygon ลง R8 FBO. carve = draw 0 ทับ.
> uniforms reshape/skin หลายตัว derive จาก landmarks canonical: `axisX=pt168.x, faceW=|pt234−pt454|, upperY=pt168.y, lowerY=pt152.y+(pt152.y−upperY)·0.15, chinC=pt152, chinM=[chinC.x+(axisX−chinC.x)·0.10·chin, chinC.y−faceW·0.06·chin], chinR=faceW·0.14·1.3, eyeLO=mean(33,133,159,145), eyeLR=|33−133|·1.6` (เหมือน parity scripts)

## 5. [A4] Reshape — uniform binding

**2-pass split = พร้อมแล้ว** (parity: 2-pass=99dB, single-pass=29–35dB → ใช้ split):
| pass | shader (มีแล้ว) | in → out |
|---|---|---|
| 4a | `ReshapeDisp.metal` / `reshape_disp.frag` | member,U_reshape → dispTex(RG16F, .rg=px) |
| 4b | `Blur.metal` / `blur.frag` ×2 (H,V) | dispTex → dispBlurTex (σ=faceW·0.04) |
| 4c | `ReshapeSample.metal` / `reshape_sample.frag` | canonicalTex + dispBlurTex → reshapedTex |
> `Reshape.metal`/`reshape.frag` = เวอร์ชัน single-pass (fold disp+sample) เก็บไว้เป็น **fallback คุณภาพต่ำ** เท่านั้น — production ใช้ split 3-pass ข้างบน

`ReshapeUniforms`: `resolution, axisX, faceW, upperY, lowerY, slim, chin, eye, chinC, chinM, chinR, eyeLO, eyeLR, eyeRO, eyeRR` (slim/chin/eye จาก `setParams`)

## 6. [A5] Skin — 5 passes

shader: `SkinSmooth.metal` + `Blur.metal` (✅ parity 99dB core). `s1=min·0.010, s2=min·0.035`:
| pass | shader | in → out |
|---|---|---|
| 5a,5b | blur H,V σ=s1 | reshapedTex → lowTex |
| 5c,5d | blur H,V σ=s2 | lowTex → evenTex |
| 5e | skin composite | reshaped,low,even,mask,U_skin → beautyTex |

`SkinUniforms`: `evenW=0.65, detail=0.72, gamma=1.07, contrast=1.04, sat=1.05, warmth=2/255, smooth(slider), evenTone(slider)`

## 7. [temporal] EMA — กัน shimmer (review rev2)

`stableTex = mix(beautyTex, prevBeautyTex, emaAlpha)`, `emaAlpha≈0.3`. canonical space = pose ถูก normalize → align ตรง ไม่ต้อง flow. แล้ว copy stableTex→prevBeautyTex.
> ระวัง ghosting: ถ้า expression เปลี่ยนเร็ว ลด alpha; advanced = EMA เฉพาะ low-freq (lowTex) เก็บ high-freq สด

## 8. [out] Warp back + composite

`outputTex = mix(inputTex, warp(stableTex, T⁻¹), maskTex_in_frame)`; mask feather ทำให้ขอบเนียน. → ลง Flutter texture (iOS `copyPixelBuffer` คืน outputTex's pixelbuffer; Android render ลง SurfaceTexture)

---

## 9. Permissions
- iOS `Info.plist`: `NSCameraUsageDescription`
- Android `Manifest`: `<uses-permission android:name="android.permission.CAMERA"/>` + `<uses-feature camera>` + **runtime request** (ก่อน start)

## 10. ลำดับ build + acceptance (ต่อ milestone, ดู spec §8)
| M | wire อะไร | acceptance |
|---|---|---|
| A1 | [cap] → Flutter texture (ข้าม process) | กล้องสด 30fps, ไม่มี CPU copy |
| A2 | + [lm] + วาด overlay จุด | จุดเกาะหน้า, log landmarkMs |
| A3 | + [smooth] One-Euro(pts)+T+One-Euro(T) | overlay นิ่ง ไม่สั่น |
| A4 | + [warp][mask member][A4 reshape 2-pass] + slider | เรียวหน้า/ตาโต ลื่น ไม่มี ghost ในผม |
| A5 | + [mask skin][A5 5-pass][temporal] | ผิวเนียน ไม่กระพริบ ตา/ปากคม |
| A6 | วัด fps/thermal 3 นาที | ≥30fps; ถ้า throttle → dynamic quality (ลด res / ข้าม temporal) |

## 11. Perf budget (ดู design §3.1) — วัดจริงต่อ pass
landmark ~5–15ms (mid-range) เป็นก้อนใหญ่สุด. GPU passes @384² รวมควร <8ms. ถ้าเกิน 33ms: ลด canonical เป็น 320²/256², หรือรัน landmark ทุก 2 เฟรม (interpolate ด้วย One-Euro).

## 12. สิ่งที่ "ไม่ต้องทำใหม่" (parity ผ่านแล้ว headless)
- reshape math (`tool/reshape_parity.py` = 0.00px, 2-pass split ReshapeDisp+Blur+ReshapeSample) · skin composite (`tool/skin_parity.py` = 99dB) · blur (70–76dB vs cv2) · skin YCrCb gate (`tool/skin_mask_parity.py` = ≥35dB feathered) · One-Euro (Dart 7/7)
- แตะแค่ **wiring** (capture, FBO/texture pool, pass dispatch, uniform upload, mask raster, transform fit). ถ้าเปลี่ยน shader math ต้องรัน parity ใหม่

## 13. GPU / feature availability + fallback matrix

**GPU มีทุกเครื่อง 100%** (Metal บน iOS / OpenGL ES บน Android = system framework บังคับ). ต่างกันที่ "เร็วแค่ไหน" + feature ปลีกย่อยบางตัว. เช็กตอน start แล้วเลือก path:

| feature ที่ pipeline ใช้ | iOS (Metal) | Android (GL ES) | fallback ถ้าไม่มี |
|---|---|---|---|
| GPU + shader API | ✅ ทุกเครื่อง (A7+/iOS8+) | ✅ ทุกเครื่อง (ES 2.0+) | — (ไม่มีทาง ไม่มี) |
| **float render target** (RG16F disp ของ A4) | ✅ เต็ม | 🟡 ต้อง `EXT_color_buffer_float` (มาตรฐาน ES 3.2; ~2016+) | เก็บ disp เป็น **RG8 + scale factor** (disp/maxDisp→[0,1]) หรือ half-float; precision ลดเล็กน้อย รับได้ |
| **half-float texture** (RGBA16F intermediates) | ✅ | 🟡 ES 3.0 core (RGBA16F) | RGBA8 (พอสำหรับ [0,1] color; disp ห้าม RGBA8 ตรงๆ) |
| linear filtering บน float tex | ✅ | 🟡 ต้อง `OES_texture_float_linear` | nearest + blur pass ชดเชย |
| camera→GPU zero-copy | ✅ `CVMetalTextureCache` | ✅ `SurfaceTexture`(OES) / `ImageReader`(HardwareBuffer) | ImageAnalysis→glTexImage2D (มี CPU upload, ช้าลงนิด) |
| MediaPipe GPU delegate | ✅ | 🟡 บางเครื่อง | CPU delegate (landmark ช้าขึ้น ~2×) |
| compute shader (ถ้าใช้) | ✅ Metal | 🟡 ES 3.1+ | ทำเป็น fragment pass แทน (เราใช้ fragment อยู่แล้ว) |

**ขั้นตอน detect ตอน start:**
1. query GL extensions (`GL_EXTENSIONS`) / Metal feature set
2. ตั้ง `dispFormat = RG16F | RG8(scaled)` ตามผล
3. ถ้า MediaPipe GPU delegate fail → CPU delegate + ลด landmark rate (ทุก 2 เฟรม + One-Euro interpolate)
4. log capability ออก stats channel (debug)

**Min spec แนะนำ:** iOS A10+ (iPhone 7+) / Android ES 3.1 + Vulkan-capable mid-range (~2017+). ต่ำกว่านี้: บังคับ Route A เบา (ลด canonical 256², ปิด temporal, skin smooth อย่างเดียว). **ห้าม assume RG16F — ตรวจก่อนเสมอ**.
