# Phase A — Implementation Spec (Flutter, on-device shader-first)

> สถานะ: **IMPLEMENTED (Phase A) — device-verified A1–A5b + multi-face** (อัปเดต 2026-07-02; เดิม 2026-06-21 = SPEC)
> · ต่อจาก [video-beauty-design.md](video-beauty-design.md) rev3
> เป้า Phase A: ฟิลเตอร์แต่งหน้าวิดีโอ **shader ล้วน ไม่มี NN** → on-device 100%, server โหลด 0, real-time 30fps
> Target: **Flutter (iOS + Android)** · 1 หน้า (scope v1)
>
> **⚠️ STATUS (2026-07-02):** A1–A5b + multi-face are **SHIPPED + device-verified** (iPhone 15 Pro,
> iOS 26.5, 2026-06-22) — only **A6** (fps/thermal 3-min) + iOS↔Android tuning reconciliation remain.
> Two staleness caveats for readers: (1) the **code lives in the plugin package
> `apps/flutter-app/packages/beauty_filter/`**, NOT the `lib/beauty/` + `ios/Classes/FilterEngine.swift`
> layout §1 assumes (real classes: `MetalBeautyRenderer.swift` / `CameraGlPipeline.kt`). (2) The §4/§5
> "TODO native A4/A5" passes are **done on iOS** (`ReshapeDisp/Blur/ReshapeSample` + `SkinGate/SkinSmooth/
> Ema` shaders shipped). **Android shipped a simpler direct-space pipeline** (`slim/eye/blur/facemask/
> smooth_simple` frags — no canonical-warp/reshape-split), so the canonical/split design below describes
> the **iOS** implementation, not Android. SoT for current status = `packages/beauty_filter/HANDOFF.md`.

---

## 0. หลักการสถาปัตยกรรม Flutter

Flutter **ไม่เหมาะ**ทำ per-frame pixel ที่ 30fps บน Dart (CameraImage YUV → CPU ช้า). วิธี production:

```
┌─ Flutter (Dart) ────────────────────────────┐
│  UI: พรีวิว (Texture widget) + sliders        │
│  One-Euro filter (Dart, เบา) [optional native]│
└──────────────┬───────────────────────────────┘
   MethodChannel (params) │ EventChannel (landmarks ออก ถ้าต้องการ)
┌──────────────┴───────────────────────────────┐
│  Native plugin (FilterEngine) — GPU zero-copy │
│  [1] Camera → GPU texture                      │
│       iOS: AVCaptureSession→CVPixelBuffer      │
│       Android: CameraX→SurfaceTexture           │
│  [1b] FaceLandmarker Tasks (GPU delegate)      │
│  [3] warp→canonical (Metal/GL)                  │
│  [4] reshape mesh-warp shader                   │
│  [5] skin smoothing shader                      │
│  [6] composite + skin-mask feather              │
│  → render to Flutter Texture (zero-copy)        │
└───────────────────────────────────────────────┘
```

**กฎ zero-copy:** เฟรมกล้องต้องอยู่บน GPU ตลอด (texture) ห้ามวิ่ง CPU↔GPU ทุกเฟรม → ใช้ Flutter `Texture` widget + `TextureRegistry` (Android `SurfaceTexture`, iOS `CVPixelBuffer`/`FlutterTexture`)

---

## 1. โครงไฟล์ที่จะสร้าง (ใน Flutter app เดิม)

```
apps/flutter-app/
├─ lib/beauty/
│  ├─ beauty_filter.dart        # facade เรียก native + ถือ state
│  ├─ one_euro.dart             # One-Euro filter (Dart)
│  ├─ beauty_params.dart        # struct slider (smooth, slim, eye, evenTone...)
│  └─ beauty_preview.dart       # Texture widget + sliders UI
├─ ios/Classes/
│  ├─ FilterEngine.swift        # pipeline Metal
│  ├─ shaders/Reshape.metal
│  ├─ shaders/SkinSmooth.metal
│  └─ FaceLandmarkerBridge.swift
└─ android/src/main/
   ├─ FilterEngine.kt           # pipeline GL ES
   ├─ shaders/reshape.glsl
   ├─ shaders/skin_smooth.glsl
   └─ FaceLandmarkerBridge.kt
```

---

## 2. Platform-channel contract (แชร์ iOS/Android)

```
MethodChannel "beauty/control":
  start(cameraId, resolution)        → texture_id (int)  # Flutter ผูก Texture(textureId)
  stop()
  setParams({smooth, slim, chin, eye, evenTone, masterStrength})  # 0.0–1.0 ทุกตัว
  setCamera(front|back)
  setOneEuro({minCutoff, beta})      # จูน runtime

EventChannel "beauty/stats" (optional):
  { fps, landmarkMs, renderMs, faceFound }   # ไว้ debug + dynamic-quality fallback (§3.2 design)
```

> native ทำ pipeline + render ลง texture เอง; Dart แค่ส่ง params. **landmarks ไม่ต้องข้ามไป Dart ทุกเฟรม** (แพง) — filter ใน native หรือถ้า filter ใน Dart ให้ส่งผ่าน shared buffer

---

## 3. One-Euro filter (Dart — concrete, ใส่ได้เลย)

`lib/beauty/one_euro.dart`:

```dart
import 'dart:math';

class _LowPass {
  double? _y;
  double filter(double x, double alpha) =>
      _y = (_y == null) ? x : alpha * x + (1 - alpha) * _y!;
  void reset() => _y = null;
}

/// One-Euro (Casiez CHI 2012). t = วินาที (เช่น CMTime/ms→s).
class OneEuro {
  double minCutoff, beta, dCutoff;
  final _x = _LowPass(), _dx = _LowPass();
  double? _tPrev, _xPrev;

  OneEuro({this.minCutoff = 1.0, this.beta = 0.007, this.dCutoff = 1.0});

  double _alpha(double cutoff, double dt) {
    final tau = 1.0 / (2 * pi * cutoff);
    return 1.0 / (1.0 + tau / dt);
  }

  double filter(double value, double t) {
    if (_tPrev == null) {
      _tPrev = t; _xPrev = value;
      return value;
    }
    final dt = (t - _tPrev!).clamp(1e-3, 1.0); // กัน dt=0 / กระโดด
    _tPrev = t;
    final dxRaw = (value - _xPrev!) / dt;
    final edx = _dx.filter(dxRaw, _alpha(dCutoff, dt));
    final cutoff = minCutoff + beta * edx.abs();
    _xPrev = value;
    return _x.filter(value, _alpha(cutoff, dt));
  }

  void reset() { _x.reset(); _dx.reset(); _tPrev = null; _xPrev = null; }
}

/// ห่อสำหรับ landmark set (478 จุด × x,y). 1 filter ต่อ 1 พิกัด.
class LandmarkSmoother {
  final List<OneEuro> _fx, _fy;
  LandmarkSmoother(int n, {double minCutoff = 1.0, double beta = 0.007})
      : _fx = List.generate(n, (_) => OneEuro(minCutoff: minCutoff, beta: beta)),
        _fy = List.generate(n, (_) => OneEuro(minCutoff: minCutoff, beta: beta));

  /// pts = พิกัด PIXEL (ไม่ใช่ normalized — §Stage[2] design: ค่า cutoff ผูกหน่วย)
  List<Point<double>> smooth(List<Point<double>> pts, double t) =>
      List.generate(pts.length, (i) =>
          Point(_fx[i].filter(pts[i].x, t), _fy[i].filter(pts[i].y, t)));
}
```

> ⚠️ ต้องป้อน **พิกัด pixel** (ดู design Stage [2]: ค่า `minCutoff/beta` ผูกหน่วย). ถ้า MediaPipe คืน normalized → คูณ W,H ก่อน
> 🔴 ต้อง smooth **transform matrix** ของ canonical-warp ด้วย ไม่ใช่แค่ landmarks (design Stage [3]) — ใช้ OneEuro ตัวเดียวกันบน 4–6 พารามิเตอร์ของ similarity transform

---

## 4. Stage [4] Reshape — mesh-warp shader (ไม่ใช้ NN)

วิธี production = **textured grid mesh + displaced vertices** (ไม่ใช่ per-pixel loop ที่แพง):
1. สร้าง grid เช่น 40×40 vertices คลุม face region
2. vertex shader เลื่อนแต่ละ vertex ตาม displacement field จาก landmarks (liquify ของ `face_reshape.py`: local-translation/scale ดึงกราม/แก้ม/คาง/ตา)
3. GPU interpolate ระหว่าง vertices = warp เนียนฟรี
4. ตัด temple points (234/454/93/323) กัน warp โดนผม (carry over จาก image pipeline)

```metal
// Reshape.metal (vertex) — เลื่อน grid ตาม control points
vertex VOut reshapeV(uint vid [[vertex_id]],
                     constant float2* grid    [[buffer(0)]],
                     constant Control* ctrl   [[buffer(1)]], // จุดดึง + รัศมี + เวกเตอร์
                     constant uint& nctrl      [[buffer(2)]]) {
  float2 p = grid[vid];
  float2 disp = 0;
  for (uint i = 0; i < nctrl; i++) {
    float2 d = p - ctrl[i].center;
    float r = length(d);
    float w = smoothstep(ctrl[i].radius, 0.0, r);   // local, ไม่กระทบไกล
    disp += w * ctrl[i].delta * ctrl[i].strength;   // strength = slider slim/chin/eye
  }
  VOut o; o.uv = (grid[vid]); o.pos = toClip(p + disp); return o;
}
```

> ตาโตแบบ TikTok = ทำตรงนี้ (geometric) **ไม่ใช่** skin shader (design Stage [4])

**✅ Implemented + parity-verified (2026-06-21):**
- shader จริง: `ios/.../Shaders/Reshape.metal` + `android/.../assets/shaders/reshape.frag` (per-pixel = ตรงกับ `cv2.remap`, member ส่งเป็น texture)
- parity test: `tool/reshape_parity.py` (numpy, synthetic landmarks, ไม่พึ่ง mediapipe) — รันได้ headless
- ผล: สูตร shader == `face_reshape.py` **disp diff = 0.00px** ทุก case
- 🔴 **ข้อค้นพบ: disp-blur pass จำเป็น** — face_reshape เดิม blur displacement field (sig=faceW·0.04) ก่อน remap. ถ้าทำ:
  - **single-pass (ไม่ blur)** → ต่างจากต้นฉบับ PSNR ~29–35dB (เห็นได้)
  - **ใส่ disp-blur pass** → PSNR 99dB (เท่ากันสนิท)
  - → pipeline จริงต้องเป็น **2 pass**: (1) compute analytic disp → disp texture (RG float), (2) separable Gaussian blur disp, (3) sample. หรือ fold blur+sample. **อย่าทำ single-pass**
> NOTE: ตอนนี้ Reshape.metal/.frag = analytic disp + sample ใน pass เดียว (sample ตรง). ต้องเพิ่ม disp-blur pass คั่นก่อน sample เพื่อ parity (TODO native A4)

---

## 5. Stage [5] Skin smoothing — shader (Route A)

frequency-separation + even-tone + skin-mask gate ใน fragment shader:

```glsl
// skin_smooth.glsl (fragment) — สเก็ตช์ core
vec3 base   = surfaceBlur(tex, uv, radius);        // low-freq (เนียน) — bilateral/guided
vec3 detail = texture(tex, uv).rgb - base;         // high-freq (pore/ขอบ) เก็บไว้
vec3 even   = evenTone(base);                       // ลดรอยแดง/เกลี่ยโทน (YCrCb)
float m     = skinMask(uv);                         // oval - ตา/คิ้ว/ปาก, YCrCb gate, feather
vec3 smoothed = mix(base, even, evenStrength) + detail * detailKeep;
fragColor = vec4(mix(origColor, smoothed, m * masterStrength), 1.0);
```

**สำคัญ (จาก review rev2):**
- 🔴 **ไม่ deterministic-stable สนิท** — sensor noise ยังทำผิว shimmer → เพิ่ม **temporal EMA เบาๆ บน output** ใน stabilized space: `out_t = mix(out_t, out_{t-1}_warped, ~0.3)` (ทำใน 1 GL/Metal pass, ถูก)
- carve-out ตา/คิ้ว/ปากให้เป๊ะ (บทเรียน beauty_glam → glam2: eye-brighten ทำตาพัง)
- ลด even-tone ที่หน้าผากกัน over-smooth (open issue เดิม)

**✅ Implemented + parity-verified (2026-06-21):**
- shaders: `SkinSmooth.metal`/`skin_smooth.frag` (composite) + `Blur.metal`/`blur.frag` (generic separable Gaussian, reuse: low/even + disp-blur A4 + temporal-EMA)
- pipeline = multi-pass: blur→low (s1=min·0.010), blur(low)→even (s2=min·0.035), composite(input,low,even,mask)→ freq-sep + tone + sat + warmth + mask-gate + smooth blend
- parity `tool/skin_parity.py` (numpy, synthetic skin, headless):
  - **composite (freq-sep+tone) = ต้นฉบับ 99dB** (เป๊ะ)
  - **blur shader == cv2.GaussianBlur 70–76dB** (port blur ถูก)
  - **mask-gate: นอกมาส์ก = input** (sub-LSB)
  - 🟡 **saturation = luma-lerp approx ของ cv2 HSV S·1.05** → full operator 41dB (documented approx; HSV เป๊ะ ทำได้แต่ต้อง port hexcone + ยอม cv2 uint8 round-trip). cosmetic, รับได้
- uniforms: evenW(0.65) detail(0.72) gamma(1.07) contrast(1.04) sat(1.05) warmth(2/255) + slider smooth/evenTone
> TODO native A5: wire 5 passes (blur H/V ×2 + composite), feed skin-mask texture (port `skin_mask.py`), + temporal-EMA pass

---

## 6. Stage order & coordinate flow

```
camera tex (pixel)
 → FaceLandmarker → pts(pixel)
 → One-Euro(pts) → fit similarity T → One-Euro(T)          [Dart/native]
 → warp by T → canonical 384²                               [Stage 3]
 → reshape mesh-warp                                        [Stage 4]
 → skin smooth + temporal-EMA                               [Stage 5]
 → warp by T⁻¹ → composite skin-mask feather               [Stage 6]
 → Flutter Texture
```

---

## 7. Dependencies

| ส่วน | iOS | Android |
|---|---|---|
| Camera | AVFoundation | CameraX |
| Landmark | MediaPipe Tasks (FaceLandmarker, .task model) | เหมือนกัน (AAR) |
| GPU | Metal (MSL shaders) | OpenGL ES 3.0 / Vulkan |
| Flutter texture | `FlutterTexture` (CVPixelBuffer) | `SurfaceTexture` + `TextureRegistry` |

> MediaPipe FaceLandmarker = Apache-2.0 (commercial OK). model `.task` bundle ในแอป

---

## 8. Milestones (verify ทีละขั้น)

| M | สิ่งที่พิสูจน์ | เกณฑ์ผ่าน |
|---|---|---|
| A1 | camera → Flutter Texture (passthrough) | เห็นภาพสด 30fps, ไม่มี copy CPU |
| A2 | FaceLandmarker + วาด overlay | จุดเกาะหน้า, วัด landmarkMs |
| A3 | One-Euro บน landmarks + transform | overlay นิ่ง ไม่สั่น |
| A4 | reshape mesh-warp + slider | เรียวหน้า/ตาโต ลื่น ไม่มี ghost ในผม |
| A5 | skin smooth + mask + temporal-EMA | ผิวเนียน ไม่กระพริบ ตา/ปากคม |
| A6 | วัด fps/thermal 3 นาที | ≥30fps, ไม่ throttle จนหลุด (ไม่งั้นทำ dynamic-quality §3.2) |

ผ่าน A6 = Phase A เสร็จ → ตัดสินใจว่าต้อง Phase B (CNN tier) ไหม

---

## 9. ของที่ port ตรงจาก image pipeline
- `face_reshape.py` liquify logic → §4 vertex shader (control points + radius + delta)
- `skin_mask.py` (oval − ตา/คิ้ว/ปาก, YCrCb, feather) → §5 `skinMask(uv)`
- `beautify_data.py:pro_beautify_target` (freq-sep + tone-curve) → §5 fragment
- carve-out ตา + ลด even-tone หน้าผาก = บทเรียน glam2 (อย่าทำซ้ำ)
