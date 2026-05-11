# OpenReel Video — Danh mục đầy đủ các loại Video Animation

> Tài liệu chi tiết tới mức **re-implement được ở dự án khác**. Mọi công thức/giá trị mặc định được trích trực tiếp từ source.
> Liên quan: [`ARCHITECTURE.md`](./ARCHITECTURE.md), [`TECHNICAL.md`](./TECHNICAL.md).
> Quy ước viết mỗi mục: **(1) Đầu vào cần tổng hợp** · **(2) Mô tả hiệu ứng** · **(3) Output** · **(4) Dùng ở đâu trong frame video** · **(5) Ghi chú re-implement** (công thức, default, edge case).

---

## Mục lục

0. [Mô hình thời gian & khái niệm chung](#0-mô-hình-thời-gian--khái-niệm-chung)
1. [Easing functions (bảng tra cứu đầy đủ)](#1-easing-functions-bảng-tra-cứu-đầy-đủ)
2. [Keyframe property animation (animate bất kỳ thuộc tính nào)](#2-keyframe-property-animation)
3. [Transform animation (12 thuộc tính + ma trận)](#3-transform-animation)
4. [Motion path animation (GSAP — đường đi cong)](#4-motion-path-animation)
5. [Transitions giữa clip (7 loại)](#5-transitions-giữa-clip)
6. [Text animation — cấp clip (19 preset)](#6-text-animation--cấp-clip)
7. [Text animation — cấp đơn vị: character / word / line + stagger (19 preset)](#7-text-animation--cấp-đơn-vị-character--word--line)
8. [Caption / subtitle animation (6 style)](#8-caption--subtitle-animation)
9. [Emphasis animation — nhấn mạnh lặp lại (25 loại)](#9-emphasis-animation--nhấn-mạnh-lặp-lại)
10. [SVG / Graphic entry-exit animation (20 loại)](#10-svg--graphic-entryexit-animation)
11. [Particle effects (10 loại + 12 preset)](#11-particle-effects)
12. [Speed ramping / speed curves (6 preset)](#12-speed-ramping--speed-curves)
13. [Expression-driven animation (wiggle / noise / linear / ease)](#13-expression-driven-animation)
14. [Hiệu ứng video phụ thuộc thời gian (grain, …)](#14-hiệu-ứng-video-phụ-thuộc-thời-gian)
15. [Bảng tổng hợp nhanh](#15-bảng-tổng-hợp-nhanh)

---

## 0. Mô hình thời gian & khái niệm chung

Mọi animation trong OpenReel quy về một hàm thuần `f(state, time) → state'`. Các khái niệm dùng xuyên suốt:

- **`time` (giây)**: thời gian timeline (preview/export gọi `renderFrame(project, t)`).
- **`localTime` / `relativeTime`**: `time - clip.startTime` — thời gian tính từ đầu clip. Đa số animation clip dùng localTime.
- **`progress` (0..1)**: tỉ lệ tiến của một đoạn animation. Luôn được clamp `[0,1]` trước khi đưa vào easing.
- **`easing(t)`**: hàm biến đổi progress (xem §1).
- **3 phase của animation in/out** (text & graphic entry/exit):
  - **in phase**: `relativeTime ≤ inDuration` → `progress = relativeTime / inDuration`, `isIn = true`.
  - **out phase**: `relativeTime ≥ clipDuration - outDuration` → `progress = (relativeTime - outStart) / outDuration`, `isIn = false`.
  - **middle (hold) phase**: ở giữa → `progress = 1` (trạng thái ổn định).
- **`stagger` (giây)**: độ trễ giữa các đơn vị animation kế tiếp (char/word/line). Đơn vị thứ `i` bắt đầu trễ `i * stagger`, có thời lượng `unitDuration = max(0.1, duration - (totalUnits-1)*stagger)`, và `unitProgress = clamp((progress*duration - i*stagger) / unitDuration, 0, 1)`. Khi out phase: `unitProgress = 1 - unitProgress`.
- **`loop`**: animation continuous lặp vô hạn — `cycleTime = (adjustedTime * speed) % 1`; nếu không loop: `cycleTime = min(adjustedTime * speed, 1)`.
- **`intensity` / `speed`**: hệ số nhân biên độ & tốc độ cho animation continuous.
- **Hệ toạ độ**:
  - Video/image clip: `position` theo **pixel canvas project**; nhiều text-animation preset cấp-clip dùng **normalized 0–1** cho offset (`slideDistance: 0.2` = 20% kích thước).
  - Graphic clip: `position` thường **normalized 0–1** (`DEFAULT_GRAPHIC_TRANSFORM = {0.5, 0.5}`); offsetX/offsetY của SVG animation là **hệ số normalized** nhân với canvas size.
  - `anchor`: luôn normalized 0–1 (0.5,0.5 = tâm).
  - `rotation`: **độ** (degrees).
- **Random/noise**: một số animation (`glitch`, `vibrate`, `flicker`, `grain`) dùng `Math.random()` → **không deterministic theo time** (mỗi lần render ra khác — chấp nhận được cho preview, nhưng export có thể nhấp nháy nếu render lại; phiên bản unit-level `glitch` dùng hash deterministic `sin(seed*12.9898)*43758.5453`).

---

## 1. Easing functions (bảng tra cứu đầy đủ)

File: `packages/core/src/animation/easing-functions.ts`. Tất cả nhận `t ∈ [0,1]` trả `[0,1]` (một số overshoot ngoài `[0,1]` như Back/Elastic).

### 1.1 `EasingFunction` = một trong:
- **Tên** (`EasingName`): 30 hàm dưới đây.
- **Cubic bezier**: `{ type: "cubicBezier", points: [x1,y1,x2,y2] }`.
- **Spring vật lý**: `{ type: "spring", stiffness, damping, mass }`.

`getEasingFunction(easing) → (t)=>number`. `interpolate(a, b, progress, easing)` = `a + (b-a) * easingFn(clamp(progress))`.

### 1.2 Bảng 30 hàm có tên

| Tên | Công thức |
|---|---|
| `linear` | `t` |
| `easeInQuad` | `t²` |
| `easeOutQuad` | `1 - (1-t)²` |
| `easeInOutQuad` | `t<0.5 ? 2t² : 1 - (-2t+2)²/2` |
| `easeInCubic` | `t³` |
| `easeOutCubic` | `1 - (1-t)³` |
| `easeInOutCubic` | `t<0.5 ? 4t³ : 1 - (-2t+2)³/2` |
| `easeInQuart` | `t⁴` |
| `easeOutQuart` | `1 - (1-t)⁴` |
| `easeInOutQuart` | `t<0.5 ? 8t⁴ : 1 - (-2t+2)⁴/2` |
| `easeInQuint` | `t⁵` |
| `easeOutQuint` | `1 - (1-t)⁵` |
| `easeInOutQuint` | `t<0.5 ? 16t⁵ : 1 - (-2t+2)⁵/2` |
| `easeInSine` | `1 - cos(tπ/2)` |
| `easeOutSine` | `sin(tπ/2)` |
| `easeInOutSine` | `-(cos(πt)-1)/2` |
| `easeInExpo` | `t==0 ? 0 : 2^(10t-10)` |
| `easeOutExpo` | `t==1 ? 1 : 1 - 2^(-10t)` |
| `easeInOutExpo` | `t==0?0 : t==1?1 : t<0.5 ? 2^(20t-10)/2 : (2 - 2^(-20t+10))/2` |
| `easeInCirc` | `1 - sqrt(1 - t²)` |
| `easeOutCirc` | `sqrt(1 - (t-1)²)` |
| `easeInOutCirc` | `t<0.5 ? (1 - sqrt(1-(2t)²))/2 : (sqrt(1-(-2t+2)²)+1)/2` |
| `easeInBack` | `c3·t³ - c1·t²` với `c1=1.70158`, `c3=c1+1` |
| `easeOutBack` | `1 + c3·(t-1)³ + c1·(t-1)²` |
| `easeInOutBack` | với `c2 = c1·1.525`: `t<0.5 ? ((2t)²·((c2+1)·2t - c2))/2 : ((2t-2)²·((c2+1)·(2t-2)+c2)+2)/2` |
| `easeInElastic` | `t==0?0 : t==1?1 : -2^(10t-10)·sin((10t-10.75)·c4)`, `c4 = 2π/3` |
| `easeOutElastic` | `t==0?0 : t==1?1 : 2^(-10t)·sin((10t-0.75)·c4) + 1` |
| `easeInOutElastic` | với `c5 = 2π/4.5`: `t==0?0 : t==1?1 : t<0.5 ? -(2^(20t-10)·sin((20t-11.125)·c5))/2 : (2^(-20t+10)·sin((20t-11.125)·c5))/2 + 1` |
| `easeInBounce` | `1 - bounceOut(1-t)` |
| `easeOutBounce` | `bounceOut(t)` (xem dưới) |
| `easeInOutBounce` | `t<0.5 ? (1 - bounceOut(1-2t))/2 : (1 + bounceOut(2t-1))/2` |

`bounceOut(t)` (4 đoạn parabol, `n1=7.5625`, `d1=2.75`):
```
t < 1/d1     → n1·t²
t < 2/d1     → n1·(t-1.5/d1)²  + 0.75
t < 2.5/d1   → n1·(t-2.25/d1)² + 0.9375
else         → n1·(t-2.625/d1)²+ 0.984375
```

### 1.3 Cubic bezier — `cubicBezier(x1,y1,x2,y2)`
Chuyển bezier 2D thành hàm easing 1D: với input `x`, giải `sampleCurveX(t) = x` rồi trả `sampleCurveY(t)`.
```
cx = 3x1;  bx = 3(x2-x1) - cx;  ax = 1 - cx - bx   (tương tự cho y)
sampleCurveX(t) = ((ax·t + bx)·t + cx)·t            // Horner
sampleCurveDerivativeX(t) = (3·ax·t + 2·bx)·t + cx
solveCurveX(x):
  t2 = x
  lặp ≤8 lần Newton-Raphson: t2 -= (sampleCurveX(t2)-x) / sampleCurveDerivativeX(t2)
                              (dừng nếu |err|<1e-6 hoặc |derivative|<1e-6)
  fallback bisection trên [0,1] cho tới |err|<1e-6
return (t) => sampleCurveY(solveCurveX(t))
```
(`AnimationEngine.cubicBezier` dùng cùng thuật toán, có cache theo `"x1,y1,x2,y2"`, 4 vòng Newton, precision 1e-7, 10 vòng bisection.)

### 1.4 Spring vật lý — `springEasing(stiffness=100, damping=10, mass=1)`
Mô hình dao động điều hoà tắt dần:
```
w0   = sqrt(stiffness / mass)                 // tần số tự nhiên
zeta = damping / (2·sqrt(stiffness·mass))     // tỉ số tắt dần
nếu zeta < 1 (underdamped, có nảy):
  wd = w0·sqrt(1 - zeta²)
  b  = zeta·w0 / wd
  progress(t) = 1 - e^(-zeta·w0·t) · (cos(wd·t) + b·sin(wd·t))
ngược lại (critically/overdamped):
  progress(t) = 1 - (1 + w0·t)·e^(-w0·t)
```
`zeta<1` bouncy · `zeta=1` không overshoot · `zeta>1` chậm chạp.

### 1.5 `EASING_PRESETS` (preset đặt tên thân thiện)
| Key | easing |
|---|---|
| `linear` | `linear` |
| `ease` | cubicBezier `[0.25, 0.1, 0.25, 1]` ("smooth start & end") |
| `easeIn` | `easeInQuad` |
| `easeOut` | `easeOutQuad` |
| `easeInOut` | `easeInOutQuad` |
| `bounce` | `easeOutBounce` |
| `elastic` | `easeOutElastic` |
| `back` | `easeOutBack` |
| `snappy` | cubicBezier `[0.19, 1, 0.22, 1]` |
| `smooth` | cubicBezier `[0.4, 0, 0.2, 1]` (Material) |

### 1.6 Easing dạng chuỗi (`EasingType`) — alias dùng trong keyframe & text
`"linear" | "ease-in" | "ease-out" | "ease-in-out" | "bezier"` + toàn bộ 30 tên camelCase ở §1.2.
- `ease-in` → `easeInQuad`, `ease-out` → `easeOutQuad`, `ease-in-out` → `easeInOutQuad`, `bezier` → cubic-bezier theo handle (mặc định `[0.25,0.1,0.25,1.0]`).
- `KeyframeEngine` còn có 3 preset đặc biệt map sang bezier handle: `bounce` `[0.34,1.56,0.64,1]`, `elastic` `[0.68,-0.55,0.27,1.55]`, `spring` `[0.5,1.5,0.5,1]`. Hàm fallback giản lược trong `KeyframeEngine.applyEasingPreset`: `ease-in=t²`, `ease-out=t(2-t)`, `ease-in-out=t<0.5?2t²:-1+(4-2t)t`, `bounce`= 4 đoạn parabol như §1.2, `elastic = 2^(-10t)·sin((t-s)·2π/p)+1` với `p=0.3, s=p/4`, `spring = 1 - cos³(4.5πt)·(1-t)^2.2·0.4 - (1-t)`.

> **Re-implement**: copy nguyên file `easing-functions.ts` — không phụ thuộc gì khác. GSAP map (nếu dùng GSAP): `linear→none`, `ease-in→power2.in`, `easeInQuad→power1.in`, `easeInCubic→power2.in`, `easeInQuart→power3.in`, `easeInQuint→power4.in`, `easeInSine→sine.in`, `easeInExpo→expo.in`, `easeInCirc→circ.in`, `easeInBack→back.in`, `easeInElastic→elastic.in`, `easeInBounce→bounce.in` (và biến thể `.out`/`.inOut`).

---

## 2. Keyframe property animation

Cốt lõi của mọi animation "thủ công". File: `packages/core/src/video/animation-engine.ts` (generic), `keyframe-engine.ts` (mở rộng + bezier handle + motion path + clipboard).

**(1) Đầu vào cần tổng hợp:**
- `keyframes: Keyframe[]` — mỗi keyframe `{ id, time(s), property(string), value(unknown), easing(EasingType) }`. Nhiều property cùng tồn tại trong một mảng; lọc theo `property` để xử lý độc lập.
- `time` (s, thường là localTime của clip).
- (tuỳ chọn) `bezierHandles` per keyframe nếu `easing === "bezier"`.

**(2) Mô tả hiệu ứng:** nội suy giá trị một thuộc tính bất kỳ theo thời gian, có easing per-segment. Hỗ trợ `number`, `object` (nội suy đệ quy từng field — ví dụ `{x,y}`, màu `{r,g,b}`), và non-numeric (step tại 0.5).

**(3) Output:** giá trị đã nội suy cho property tại `time` + `{keyframeA, keyframeB, progress, easedProgress}`. Cộng dồn các property → một `Transform` (hoặc tham số effect) đã animate.

**(4) Dùng ở đâu trong frame:** trước khi compositing — `TransformAnimator.getTransformAtTime` (video/image clip), `TextAnimationEngine.applyKeyframeAnimation` (text clip), `GraphicsEngine.getAnimatedTransform` (graphic clip), và nội suy tham số effect/audio automation. Property phổ biến: `position.x/y`, `scale.x/y`, `rotation`, `opacity`, `anchor.x/y`, `rotate3d.x/y/z`, `perspective`, `blur`, `brightness`, `contrast`, `saturation`, `volume`, `pan`...

**(5) Ghi chú re-implement:**
- Thuật toán `getValueAtTime` xem [`TECHNICAL.md` §2.2]. Tóm tắt: sort → clamp 2 đầu → tìm cặp bao quanh `time` → `linearT = (time - A.time)/(B.time - A.time)` → `easedT = easing(linearT, A.easing)` → `interpolateValue(A.value, B.value, easedT)`. Easing thuộc keyframe **trái** của cặp.
- 1 keyframe → giá trị hằng. 0 keyframe → `undefined` (caller giữ giá trị gốc).
- CRUD: `addKeyframe` thay thế nếu trùng `(time, property)`; `updateKeyframe` re-sort nếu đổi `time`.
- Bezier handle mặc định theo preset: xem §1.6 / [`TECHNICAL.md` §2.3].
- Copy/paste keyframe: `copyKeyframes` → clipboard `{keyframes, sourceClipId, sourceProperty, copiedAt}`; `pasteKeyframes` normalize về `timeOffset` (`time - minTime + timeOffset`), đổi `property` đích, sinh `id` mới.
- `AnimatableProperty` đầy đủ: `position | position.x | position.y | scale | scale.x | scale.y | rotation | opacity | anchor | anchor.x | anchor.y | fill | stroke.color | stroke.width | fontSize | letterSpacing | blur | brightness | contrast | saturation | <string bất kỳ>`.

---

## 3. Transform animation

File: `packages/core/src/video/transform-animator.ts`. `Transform` đầy đủ xem [`TECHNICAL.md` §1 / ARCHITECTURE §4].

**(1) Đầu vào cần tổng hợp:**
- `baseTransform: Transform` (giá trị tĩnh của clip).
- `keyframes: Keyframe[]` cho **12 property** sau (mỗi cái độc lập): `position.x`, `position.y`, `scale.x`, `scale.y`, `rotation`, `opacity`, `anchor.x`, `anchor.y`, `rotate3d.x`, `rotate3d.y`, `rotate3d.z`, `perspective`.
- `time` (s).
- (để vẽ/compositing) `width`, `height` của khung.

**(2) Mô tả hiệu ứng:** sinh `Transform` tại thời điểm `time` — vị trí, scale, xoay 2D, opacity, anchor (pivot), xoay 3D (perspective), từ base + override bởi keyframes có mặt. `opacity` clamp `[0,1]`, `perspective` clamp `≥0`. Nếu không có keyframe nào → trả base, `isAnimated=false`.

**(3) Output:** `{ transform: Transform, isAnimated: boolean, animatedProperties: string[] }`. Cộng thêm: `computeTransformMatrix(transform, w, h)` → ma trận affine 2D `{a,b,c,d,e,f}` (CSS `matrix()`):
```
anchorX = anchor.x·w;  anchorY = anchor.y·h
rad = rotation·π/180;  cos = cos(rad);  sin = sin(rad)
a = cos·scaleX
b = sin·scaleX
c = -sin·scaleY
d = cos·scaleY
e = position.x + anchorX - anchorX·cos·scaleX + anchorY·sin·scaleY
f = position.y + anchorY - anchorX·sin·scaleX - anchorY·cos·scaleY
applyMatrixToPoint(m, p) = { x: m.a·p.x + m.c·p.y + m.e,  y: m.b·p.x + m.d·p.y + m.f }
```
Thứ tự biến đổi tương đương: dịch tới anchor → xoay → scale → dịch ngược anchor → cộng position.

**(4) Dùng ở đâu trong frame:** bước "Per-clip transform" trong pipeline render (sau decode, trước effects/compositing). Áp cho **mọi loại clip** (video/image/text/graphic). 3D (`rotate3d`, `perspective`, `transformStyle: 'preserve-3d'`) được preview bằng THREE.js / CSS 3D.

**(5) Ghi chú re-implement:**
- `DEFAULT_TRANSFORM = { position:{0,0}, scale:{1,1}, rotation:0, anchor:{0.5,0.5}, opacity:1, rotate3d:{0,0,0}, perspective:1000, transformStyle:'flat' }`.
- Mỗi property gọi `AnimationEngine.getKeyframesForProperty` + `getValueAtTime` riêng → chỉ override khi `typeof value === 'number'`.
- Helper tạo nhanh keyframe-set: `createPositionKeyframes(startPos, endPos, startTime, endTime, easing)`, `createScaleKeyframes`, `createRotationKeyframes`, `createOpacityKeyframes(start, end, ...)` (opacity tự clamp [0,1]).
- `rotatePointAroundAnchor(point, anchor, deg)`: xoay điểm quanh anchor (cho hit-testing / handles UI).
- `mergeWithDefaults(partial)` để điền field thiếu.

---

## 4. Motion path animation

File: `packages/core/src/animation/gsap-engine.ts` (dùng GSAP `MotionPathPlugin`) + `keyframe-engine.ts` (sampling để vẽ path overlay).

**(1) Đầu vào cần tổng hợp:**
- `MotionPathConfig`: `{ clipId, enabled, pathType: 'linear'|'bezier'|'catmull-rom', points: GSAPMotionPathPoint[], showPath, autoOrient, alignOrigin:[number,number] }`.
- `GSAPMotionPathPoint`: `{ x, y, time(normalized 0–1), controlPoints?: { cp1:{x,y}, cp2:{x,y} } }`.
- `time` + `clipDuration` (để chuẩn hoá `normalizedTime = time/clipDuration`).

**(2) Mô tả hiệu ứng:** clip di chuyển dọc theo một đường (thẳng nối điểm / bezier per-segment / Catmull-Rom mượt). `autoOrient` → clip tự xoay theo tiếp tuyến đường đi (`rotation = atan2(dy, dx)·180/π`).

**(3) Output:** `samplePositionAtTime(clipId, time, clipDuration) → { x, y, rotation? } | null`. Ghi đè `position` (và `rotation` nếu autoOrient) trong `Transform` của clip. Cũng có `getSVGPath(clipId)` → chuỗi `M…C…` để vẽ overlay; `sampleFrameTransforms(clipId, start, end, fps)` → mảng `{time,x,y,rotation?}` từng frame cho export.

**(4) Dùng ở đâu trong frame:** thay thế/bổ sung cho keyframes `position.x/y` ở bước transform. Hai chiều chuyển đổi: `keyframesToMotionPath(keyframes, clipDuration)` ↔ `motionPathToKeyframes(points, clipDuration, easing='easeInOutCubic')`.

**(5) Ghi chú re-implement:**
- **Linear segment**: tìm 2 điểm `p0,p1` bao quanh `normalizedTime`, `t=(time-p0.time)/(p1.time-p0.time)`, `x = p0.x + (p1.x-p0.x)·t` (tương tự y). Clamp 2 đầu.
- **Cubic bezier per-segment** (khi cả 2 điểm có `controlPoints`): `B(t) = (1-t)³·p0 + 3(1-t)²t·cp2_of_p0 + 3(1-t)t²·cp1_of_p1 + t³·p1`. Rotation = atan2 của `(B(t+dt) - B(t))`, `dt=0.001`.
- **Catmull-Rom** (`tension=0.5`): với `numSegments = points.length-1`, `segment = floor(t·numSegments)`, `localT = (t·numSegments) % 1`, dùng 4 điểm `p[seg-1..seg+2]` (clamp index), tiếp tuyến `m1 = tension·(p2-p0)`, `m2 = tension·(p3-p1)`, basis Hermite `a=2t³-3t²+1, b=t³-2t²+t, c=-2t³+3t², d=t³-t²` → `pos = a·p1 + b·m1 + c·p2 + d·m2`.
- **Sinh control point mặc định**: `tangent = prev&&next ? (next-prev)/4 : next ? (next-point)/3 : (point-prev)/3`; `cp1 = point - tangent`, `cp2 = point + tangent`.
- `generateBezierPath`: nếu có controlPoints → `C cp2_p0 cp1_p1 p1`; nếu không → control point 1/3 & 2/3 đoạn thẳng.
- GSAP easing map xem §1.6.

---

## 5. Transitions giữa clip

File: `packages/core/src/video/transition-engine.ts`. **7 loại**: `crossfade`, `dipToBlack`, `dipToWhite`, `wipe`, `slide`, `zoom`, `push`.

**(1) Đầu vào cần tổng hợp (chung):**
- `outgoingFrame: ImageBitmap`, `incomingFrame: ImageBitmap` (đã render đầy đủ ở kích thước canvas — `width × height`).
- `transition: { id, clipAId, clipBId, type, duration(s), params: Record<string, unknown> }`.
- `progress ∈ [0,1]` — tính từ `calculateTransitionProgress(transition, clipA, currentTime)`:
  ```
  transitionStart = clipA.startTime + clipA.duration - transition.duration/2
  transitionEnd   = transitionStart + transition.duration
  progress = clamp((currentTime - transitionStart) / transition.duration, 0, 1)
  ```
  (Transition đặt **giữa** ranh giới 2 clip — overlap đối xứng.)
- `params.curve` (chỉ vài loại): `'linear' | 'ease' | 'ease-in' | 'ease-out' | 'ease-in-out'` — easing riêng của transition engine (đơn giản hoá): `linear=t`, `ease=t²(3-2t)` (smoothstep), `ease-in=t²`, `ease-out=t(2-t)`, `ease-in-out=t<0.5?2t²:-1+(4-2t)t`.

**Validate** (`validateTransition(clipA, clipB, duration)`): 2 clip phải kề nhau (gap < 0.001s) & cùng track. `maxDuration = min(clipA.duration, clipB.duration, clipA.duration + (clipA.outPoint - clipA.duration), clipB.duration + clipB.inPoint)` — giới hạn bởi handle frames. Nếu vượt → vẫn valid nhưng kèm warning + clamp xuống `maxDuration`.

**(3) Output (chung):** `{ frame: ImageBitmap, processingTime, gpuAccelerated: false }` — một frame mới đã blend. Render bằng `OffscreenCanvas` 2D.

**(4) Dùng ở đâu trong frame:** sau khi compositing toàn bộ layer của clipA và clipB riêng rẽ; thay frame cuối khi `currentTime` nằm trong `[transitionStart, transitionEnd]`.

### 5.1 `crossfade` — params `{ curve: 'ease' }`
- **Hiệu ứng:** outgoing mờ dần, incoming hiện dần, chồng lên nhau.
- **Công thức:** `drawImage(outgoing, ..., alpha = 1 - p); drawImage(incoming, ..., alpha = p)`.

### 5.2 `dipToBlack` / `dipToWhite` — params `{ holdDuration: 0.1 }`
- **Hiệu ứng:** outgoing fade về màu (đen/trắng) → giữ màu một lúc → incoming fade từ màu lên.
- **Công thức:** `totalPhases = 2 + holdDuration`; `fadeOutEnd = 1/totalPhases`; `holdEnd = (1+holdDuration)/totalPhases`.
  - `p < fadeOutEnd`: vẽ outgoing, đè lớp màu alpha `= p/fadeOutEnd`.
  - `fadeOutEnd ≤ p < holdEnd`: fill toàn màu.
  - `p ≥ holdEnd`: fill toàn màu, vẽ incoming alpha `= (p - holdEnd)/(1 - holdEnd)`.

### 5.3 `wipe` — params `{ direction: 'left'|'right'|'up'|'down'|'diagonal', softness: 0 }`
- **Hiệu ứng:** incoming "quét" che dần outgoing từ một cạnh.
- **Công thức:** vẽ outgoing làm nền; `ctx.clip()` một hình chữ nhật mở rộng theo `progress` rồi vẽ incoming trong vùng clip:
  - `left`: clip `rect(p·width, 0, width - p·width, height)` ... (engine dùng `rect(x=p·w, 0, w-x, h)`).
  - `right`: clip vùng `[width·(1-p), width]` (invert).
  - `up`: clip `rect(0, p·height, width, height - p·height)`.
  - `down`: clip vùng `[height·(1-p), height]` (invert).
  - `diagonal`: clip đa giác chéo, `offset = (width+height)·p`, polygon `(offset,0) → (offset-height, height) → (width,height) → (width,0)`.
  - `softness > 0`: giảm `globalAlpha` mép xuống ~0.8 (làm mềm thô).

### 5.4 `slide` — params `{ direction: 'left'|'right'|'up'|'down', pushOut: false }`
- **Hiệu ứng:** incoming trượt vào từ một cạnh; nếu `pushOut` thì outgoing trượt ra cùng lúc, ngược lại outgoing đứng yên bên dưới.
- **Công thức** (gọi `renderSlide`): với `direction='left'`: `inX = width·(1-p)`, nếu pushOut `outX = -width·p`. `'right'`: `inX = -width·(1-p)`, `outX = width·p`. `'up'`: `inY = height·(1-p)`, `outY = -height·p`. `'down'`: `inY = -height·(1-p)`, `outY = height·p`. Vẽ outgoing tại `(outX,outY)`, incoming tại `(inX,inY)`.

### 5.5 `push` — params `{ direction }`
- = `slide` với `pushOut = true` (cả 2 frame luôn di chuyển cùng nhau).

### 5.6 `zoom` — params `{ scale: 2, center: {x:0.5, y:0.5} }`
- **Hiệu ứng:** outgoing phóng to & mờ dần; incoming thu từ nhỏ về bình thường & hiện dần, lấy `center` (normalized) làm tâm zoom.
- **Công thức:** `outScale = 1 + (scale-1)·p`, `outAlpha = 1-p`; `inScale = 1/scale + (1 - 1/scale)·p`, `inAlpha = p`. `centerX = width·center.x`, `centerY = height·center.y`. Vẽ mỗi frame với `translate(center) → scale(s,s) → translate(-center) → drawImage`, đặt `globalAlpha` tương ứng.

> **Re-implement transition**: tất cả thuần Canvas2D, không GPU. `getDefaultParams(type)` trả default ở trên. `isTimeInTransition`, `updateTransitionDuration`, `removeTransition`, `findAdjacentClipPairs(track)` là helper quản lý. `getAvailableTransitionTypes() = ['crossfade','dipToBlack','dipToWhite','wipe','slide','zoom','push']`. README còn nhắc "fade/dip/wipe/slide" — đó là cách gọi marketing; type kỹ thuật là 7 cái trên.

---

## 6. Text animation — cấp clip

File: `packages/core/src/text/text-animation.ts` (`TextAnimationEngine.getAnimatedState(clip, time)`). Áp animation cho **toàn bộ text clip** (transform/opacity/style/visibleText), tuỳ chọn xuất `characterStates[]` cho vài preset đặc biệt.

**(1) Đầu vào cần tổng hợp:**
- `clip: TextClip` — `{ text, style:TextStyle, transform:Transform, duration, animation?:TextAnimation, keyframes?:Keyframe[] }`.
- `TextAnimation`: `{ preset:TextAnimationPreset, params:TextAnimationParams, inDuration(s), outDuration(s) }`.
- `time` (đây là **localTime trong clip** — caller truyền `currentTime` đã trừ `startTime`; lưu ý: hàm `getAnimatedState` nhận `time` so với đầu clip nhưng vài preset continuous dùng trực tiếp `time` toàn cục cho dao động — xem ghi chú).
- (ưu tiên) nếu `clip.keyframes.length > 0` → bỏ qua preset, dùng keyframe animation (§2) cho `opacity/position.x/y/scale.x/y/rotation`.

**(2) Mô tả + công thức từng preset** — `inProgress` & `outProgress` được tính rồi qua easing (`params.easing ?? 'ease-out'`):
```
holdEnd     = duration - outDuration
inProgress  = time < inDuration  ? time/inDuration            : 1
outProgress = time > holdEnd     ? (time - holdEnd)/outDuration: 0
easedIn  = easing(inProgress);   easedOut = easing(outProgress)
```

| Preset | params (default) | Công thức (state trả về) |
|---|---|---|
| `none` | — | trả nguyên transform/style/text |
| `typewriter` | `easing:'linear'` | `visibleChars = floor(inProgress · text.length)`; `visibleText = text[0..visibleChars]`; `opacity = transform.opacity·(1-outProgress)` |
| `fade` | `fadeOpacity:{start:0,end:1}, easing:'ease-out'` | `opacity = (start + (end-start)·inProgress) · (1-outProgress)` |
| `slide-left` | `slideDistance:0.2, easing:'ease-out'` | `offsetX = -d·(1-inProgress) + d·outProgress`; `opacity = transform.opacity·inProgress·(1-outProgress)`; cộng offsetX vào `position.x` |
| `slide-right` | nt | `offsetX = d·(1-inProgress) - d·outProgress` |
| `slide-up` | nt | `offsetY = -d·(1-inProgress) + d·outProgress` |
| `slide-down` | nt | `offsetY = d·(1-inProgress) - d·outProgress` |
| `scale` | `scaleFrom:0, scaleTo:1, easing:'ease-out'` | `scale = (from + (to-from)·inProgress)·(1-outProgress) + from·outProgress`; áp `scale.x = scale.y = scale`; `opacity = ·inProgress·(1-outProgress)` |
| `blur` | `blurAmount:10, easing:'ease-out'` | `currentBlur = blurAmount·(1-inProgress) + blurAmount·outProgress`; set `style.shadowColor = style.color, shadowBlur = currentBlur, shadowOffset = 0` (giả lập blur bằng shadow); `opacity = ·inProgress·(1-outProgress)` |
| `bounce` | `bounceHeight:0.1, bounceCount:3, easing:'ease-out'` | in: `t = inProgress·π·bounceCount`; `damping = 1-inProgress`; `offsetY = -|sin(t)|·bounceHeight·damping`. out: `offsetY = sin(outProgress·π)·bounceHeight·outProgress`. `opacity = transform.opacity·min(inProgress·2,1)·(1-outProgress)` |
| `rotate` | `rotateAngle:360, easing:'ease-out'` | `rotation = transform.rotation + rotateAngle·(1-inProgress) - rotateAngle·outProgress`; `opacity = ·inProgress·(1-outProgress)` |
| `wave` | `waveAmplitude:0.02, waveFrequency:2, easing:'linear'` | **per-character**: `phase = (i/len)·2π·frequency`; `offsetY_i = sin(time·5 + phase)·amplitude·inProgress`. Xuất `characterStates[]` với offsetY khác nhau từng char. (continuous) |
| `shake` | `shakeIntensity:0.01, shakeSpeed:20, easing:'linear'` | `shakeX = sin(time·speed)·intensity·inProgress·(1-outProgress)`; `shakeY = cos(time·speed·1.3)·intensity·…`; cộng vào position. (continuous) |
| `pop` | `popOvershoot:1.2, easing:'ease-out'` | `inProgress<0.5 → scale = inProgress·2·overshoot`; `<0.7 → scale = overshoot - (inProgress-0.5)·(overshoot-1)/0.2`; `else → 1`. `scale ·= (1-outProgress)`; `opacity = ·min(inProgress·2,1)·(1-outProgress)` |
| `glitch` | `glitchIntensity:0.02, glitchSpeed:10, easing:'linear'` | `glitchActive = sin(time·speed) > 0.7`; nếu active: `offsetX += rand(-0.5,0.5)·intensity·inProgress`, `rotation += rand(-0.5,0.5)·5·inProgress`, `style.letterSpacing += rand·2`. (random, không deterministic) |
| `split` | `splitDirection:'horizontal', easing:'ease-out'` | `splitOffset = 0.1·(1-inProgress) + 0.1·outProgress`. **per-character**: nửa đầu chuỗi offset `-splitOffset`, nửa sau `+splitOffset` (theo trục direction). Xuất `characterStates[]` |
| `flip` | `flipAxis:'y', easing:'ease-out'` | `flipProgress = (1-inProgress)·π + outProgress·π`; `scaleValue = |cos(flipProgress)|`; nếu axis='y' → `scale.x = scaleValue`, ngược lại `scale.y = scaleValue`; `opacity = ·inProgress·(1-outProgress)` |
| `word-by-word` | `wordDelay:0.2, easing:'linear'` | `words = text.split(' ')`; `visibleWordCount = ceil(inProgress·words.length)`; `visibleText = words[0..count].join(' ')`; `opacity = transform.opacity·(1-outProgress)` |
| `rainbow` | `rainbowSpeed:1, easing:'linear'` | `hue = (time·speed·100) % 360`; set `style.color = hsl(hue,100%,50%)`; xuất `characterStates[]`. (continuous) |

`createAnimationPreset(preset, inDuration=0.5, outDuration=0.5, params={})` merge với default. `getAvailablePresets()` = `['none','typewriter','fade','slide-left','slide-right','slide-up','slide-down','scale','blur','bounce','rotate','wave','shake','pop','glitch','split','flip','word-by-word','rainbow']` (19 mục).

**(3) Output:** `AnimatedTextState = { opacity, transform, style, visibleText, characterStates? }`.

**(4) Dùng ở đâu trong frame:** `TitleEngine` gọi `getAnimatedState(clip, localTime)` rồi render text với transform/opacity/style/visibleText kết quả, tại vị trí text clip trên canvas; nếu có `characterStates[]` thì render từng ký tự với offset/scale/rotation/màu riêng.

**(5) Ghi chú re-implement:**
- `easing` ở đây là chuỗi: `'linear' | 'ease-in' | 'ease-out' | 'ease-in-out' | 'bezier'` (hoặc tên camelCase) — map qua `getEasing` (xem `text-animation-presets.ts`): `linear→t`, `ease-in→easeInQuad`, `ease-out→easeOutQuad`, `ease-in-out→easeInOutQuad`, `bezier→easeInOutCubic`, còn lại tra `EASING_FUNCTIONS`.
- Các preset `wave/shake/glitch/rainbow` dùng `time` cho dao động — nếu muốn deterministic theo frame, truyền `time` = thời gian tuyệt đối ổn định (export) hoặc thay `Math.random()` bằng hash.
- `slideDistance/bounceHeight/...` ở engine cấp-clip là **normalized 0–1** (khác cấp-đơn-vị dùng pixel — xem §7).
- `blur` được giả lập bằng `shadowBlur` (Canvas2D `filter: blur()` đắt) — có thể đổi sang CSS `filter` nếu render DOM.

---

## 7. Text animation — cấp đơn vị: character / word / line

File: `packages/core/src/text/text-animation-presets.ts` (`calculateUnitAnimationState`) + `character-animator.ts` (đo layout & ráp). Đây là animation **per-unit có stagger** — nâng cao hơn §6, dùng khi `TextAnimation.unit` = `'character' | 'word' | 'line'`.

**(1) Đầu vào cần tổng hợp:**
- `clip: TextClip` với `animation: { preset, params, inDuration, outDuration, stagger, unit }`.
- `currentTime` toàn cục → tính `relativeTime = currentTime - clip.startTime`, `clipDuration`.
- **Layout text** (do `CharacterAnimator.measureText(text, fontFamily, fontSize, fontWeight, letterSpacing, lineHeight)` tạo): `TextLayout = { characters:CharacterInfo[], words:WordInfo[], lines:LineInfo[], totalWidth, totalHeight }`. Mỗi `CharacterInfo = { char, x, y, width, height, lineIndex, charIndexInLine, globalIndex }`; `WordInfo`/`LineInfo` tương tự. Đo bằng `ctx.measureText`; fallback nếu không có canvas: `charWidth ≈ 0.6·fontSize`, `lineHeightPx = fontSize·lineHeight`.
- Per unit `i`: `AnimatedUnit = { text, index(=globalIndex), totalUnits, x, y, width, height }`.
- `TextAnimationContext = { unit, progress, isIn, animation, totalDuration }`.

**(2) Mô tả + công thức** — phase & stagger:
```
isInPhase  = relativeTime ≤ inDuration
isOutPhase = relativeTime ≥ clipDuration - outDuration
isMiddle   = !isInPhase && !isOutPhase
nếu in:   progress = inDuration>0 ? relativeTime/inDuration : 1;  isIn = true
nếu out:  progress = outDuration>0 ? (relativeTime - (clipDuration-outDuration))/outDuration : 0;  isIn = false
nếu middle: progress = 1; isIn = true   (mỗi unit dùng progress=1 → trạng thái cuối)

// trong mỗi animation fn:
unitDelay    = unit.index · stagger        (với word-by-word: index · wordDelay)
duration     = isIn ? inDuration : outDuration
unitDuration = max(0.1, duration - (totalUnits-1)·stagger)
unitProgress = clamp((progress·duration - unitDelay) / unitDuration, 0, 1)
nếu !isIn: unitProgress = 1 - unitProgress
easedProgress = easing(unitProgress)      // easing mặc định khác nhau theo preset
```
`UnitAnimationState = { opacity, scale:{x,y}, rotation, offsetX, offsetY, blur, color?, skewX?, skewY? }` (default tất cả 1/0).

| Preset | easing default · params default · `defaultUnit` · `stagger/in/out` | Công thức `UnitAnimationState` |
|---|---|---|
| `none` | — | DEFAULT (no-op) |
| `typewriter` | — · char · 0.05 / 1 / 0.5 | `opacity = unitProgress ≥ 0.5 ? 1 : 0` (xuất hiện "khô") |
| `fade` | `easeOutQuad` · `fadeOpacity{0,1}` · char · 0.03 / 0.5 / 0.3 | `opacity = start + (end-start)·easedProgress` |
| `slide-up` | `easeOutCubic` · `slideDistance:50`(px) · **word** · 0.1 / 0.5 / 0.3 | `offsetY = -50·(1-easedProgress)`; `opacity = easedProgress` |
| `slide-down` | nt · **word** | `offsetY = +50·(1-easedProgress)` |
| `slide-left` | nt · **char** · 0.02 / 0.5 / 0.3 | `offsetX = -50·(1-easedProgress)` |
| `slide-right` | nt · **char** | `offsetX = +50·(1-easedProgress)` |
| `scale` | `easeOutBack` · `scaleFrom:0, scaleTo:1` · char · 0.03 / 0.5 / 0.3 | `s = from + (to-from)·easedProgress`; `scale={s,s}`; `opacity = easedProgress` |
| `blur` | `easeOutQuad` · `blurAmount:10`(px) · **word** · 0.1 / 0.5 / 0.3 | `blur = 10·(1-easedProgress)`; `opacity = easedProgress` |
| `bounce` | (hard-coded `easeOutBounce`) · `bounceHeight:30`(px) · char · 0.05 / 0.8 / 0.3 | `e = easeOutBounce(unitProgress)`; `offsetY = -30·(1-e)`; `opacity = unitProgress>0 ? 1 : 0` |
| `rotate` | `easeOutBack` · `rotateAngle:180`(deg) · char · 0.05 / 0.6 / 0.3 | `rotation = 180·(1-easedProgress)`; `scale={easedProgress, easedProgress}`; `opacity = easedProgress` |
| `pop` | (hard-coded `easeOutBack`) · `popOvershoot:1.2` · char · 0.05 / 0.5 / 0.3 | `e = easeOutBack(unitProgress)`; `s = unitProgress>0 ? e·(unitProgress<0.5 ? overshoot : 1) : 0`; `scale={max(0,s),max(0,s)}`; `opacity = unitProgress>0 ? 1 : 0` |
| `flip` | `easeOutBack` · `flipAxis:'y'` · char · 0.05 / 0.6 / 0.3 | `rotAngle = 90·(1-easedProgress)`; `scaleAxis = cos(rotAngle·π/180)`; `scale = axis==='x' ? {scaleAxis,1} : {1,scaleAxis}`; `opacity = easedProgress>0.1 ? 1 : 0` |
| `split` | `easeOutCubic` · `splitDirection:'horizontal'` · char · 0 / 0.5 / 0.3 | `centerIdx = (totalUnits-1)/2`; `dist = unit.index - centerIdx`; `offset = 100·dist·(1-easedProgress)/centerIdx`; `offsetX = horizontal ? offset : 0`, `offsetY = vertical ? offset : 0`; `opacity = easedProgress` |
| `word-by-word` | `easeOutQuad` · `wordDelay:0.2` · **word** · 0.2 / 1 / 0.5 | `unitDelay = index·wordDelay`; `unitDuration = 0.3` (cố định); `opacity = easedProgress`; `offsetY = 20·(1-easedProgress)`; `scale = {0.8+0.2e, 0.8+0.2e}` |
| `wave` | — (continuous) · `waveAmplitude:10, waveFrequency:2` · char · 0/0/0 | `phase = (i/totalUnits)·2π`; `offsetY = sin(progress·2π·frequency + phase)·amplitude` |
| `shake` | — (continuous) · `shakeIntensity:5, shakeSpeed:20` · char · 0/0/0 | `offsetX = sin(progress·2π·speed)·intensity`; `offsetY = cos(progress·2π·speed·1.3)·intensity·0.5` |
| `glitch` | — (emphasis) · `glitchIntensity:10, glitchSpeed:10` · char · 0/0/0 | `glitchPhase = floor(progress·speed) + i·0.3`; `r = fract(sin(glitchPhase·12.9898)·43758.5453)`; nếu `r>0.7`: `offsetX = (r-0.5)·intensity·2`, `skewX = (r-0.5)·5`, nếu `r>0.85`: `color = hsl(r·360,100%,50%)` (**deterministic**) |
| `rainbow` | — (continuous) · `rainbowSpeed:1` · char · 0/0/0 | `hue = ((i/totalUnits)·360 + progress·360·speed) % 360`; `color = hsl(hue,80%,60%)` |

`TEXT_ANIMATION_PRESETS[]` (19 mục, mỗi mục có `category`: `entrance` / `emphasis` (glitch) / `continuous` (wave, shake, rainbow) / `exit`). `createDefaultAnimation(preset)` lấy default từ metadata. `getPresetInfo(preset)` tra metadata.

**(3) Output:** `AnimatedTextLayout = { lines: AnimatedLine[], totalWidth, totalHeight }`, mỗi `AnimatedLine` chứa `state:UnitAnimationState` + `animatedWords[]`, mỗi word chứa `state` + `animatedChars[]`, mỗi char `{ ...CharacterInfo, state }`. Nếu `unit==='character'` thì char-level state khác nhau, word/line state = DEFAULT; nếu `unit==='word'` thì word-level animate, char giữ DEFAULT; tương tự `line`.

**(4) Dùng ở đâu trong frame:** renderer text vẽ từng line/word/char tại `(line.x + word relative + char.x)` cộng `state.offsetX/Y`, scale quanh tâm char, xoay `state.rotation`, độ mờ `state.opacity`, blur `state.blur`, màu override `state.color`, skew `state.skewX/Y`. Layout gốc do `measureText` cho ra để biết toạ độ từng đơn vị.

**(5) Ghi chú re-implement:**
- **Khác biệt với §6**: cấp-đơn-vị dùng **pixel** cho slideDistance/bounceHeight (50, 30...), có **stagger** thật, default unit khác nhau (slide-up/down/blur mặc định theo *word*). Cấp-clip (§6) dùng **normalized** và không có stagger thực.
- `getEasing(easingType)`: `undefined|'linear'→t`, `'ease-in'→easeInQuad`, `'ease-out'→easeOutQuad`, `'ease-in-out'→easeInOutQuad`, `'bezier'→easeInOutCubic`, còn lại tra `EASING_FUNCTIONS[name]`.
- Middle phase dùng `progress=1` cho mọi unit → ở trạng thái "đã vào hết". Continuous preset (wave/shake/rainbow) lờ phase, dùng `progress` trực tiếp như pha dao động (nên đặt `in=out=0` để chúng chạy suốt clip).
- `glitch` cấp-đơn-vị là **deterministic** (hash) — ưu tiên dùng bản này nếu cần export ổn định.

---

## 8. Caption / subtitle animation

File: `packages/core/src/text/caption-animation-renderer.ts` (`renderAnimatedCaption(subtitle, currentTime)`). **6 style**: `none`, `word-highlight`, `word-by-word`, `karaoke`, `bounce`, `typewriter`.

**(1) Đầu vào cần tổng hợp:**
- `subtitle: Subtitle` — `{ id, text, startTime, endTime, style?:SubtitleStyle, words?:SubtitleWord[], animationStyle?:CaptionAnimationStyle }`.
- `SubtitleWord = { text, startTime, endTime }` — **per-word timing** (sinh từ STT hoặc SRT). Bắt buộc cho mọi style trừ `none` (nếu thiếu `words` → fallback `none`).
- `SubtitleStyle`: `{ fontFamily, fontSize, color, backgroundColor, position:'top'|'center'|'bottom', highlightColor?, upcomingColor? }`.
- `currentTime` (toàn cục). Nếu ngoài `[startTime, endTime]` → `{ segments: [], visible: false }`.

**(2) Mô tả + công thức** — output là mảng `WordSegment = { text, style:'normal'|'highlighted'|'hidden'|'active', opacity, scale, offsetY, color? }`:

| Style | Hành vi |
|---|---|
| `none` | 1 segment = toàn `subtitle.text`, normal, opacity 1, scale 1, offsetY 0 |
| `word-highlight` | Tất cả từ luôn hiển thị; từ **đang active** (`startTime ≤ t < endTime`): `style='highlighted'`, `scale=1.15`, `offsetY=-2`, `color = style.highlightColor ?? '#ffff00'`. Từ **upcoming** (chưa tới): nếu có `upcomingColor` thì tô màu đó. Từ đã qua: bình thường |
| `word-by-word` | Chỉ hiển thị **đúng 1 từ đang active** (`style='active'`). Nếu chưa có từ nào active và `t < lastWord.endTime` → `visible:false` (ẩn caption). Nếu `t ≥ lastWord.endTime` → hiển thị từ cuối (normal) |
| `karaoke` | Tất cả từ hiển thị. Từ **upcoming**: `color = upcomingColor ?? 'rgba(255,255,255,0.5)'`. Từ **đã hoàn thành**: `style='highlighted'`, `color=highlightColor`. Từ **đang active**: `style='active'`, `scale=1.05`, `color = 'linear-gradient(90deg, ${highlightColor} ${progress·100}%, ${upcomingColor} ${progress·100}%)'` với `progress = clamp((t - word.startTime)/(word.endTime - word.startTime), 0, 1)` → hiệu ứng "tô màu chạy ngang" trong từ |
| `bounce` | Từng từ nảy vào khi tới `startTime` của nó. `animationDuration = 0.3s`. Trước `startTime`: `style='hidden'`, `opacity=0`, `scale=0`, `offsetY=20`. Sau: `animProgress = clamp((t - startTime)/0.3, 0, 1)`; `e = easeOutBounce(animProgress)`; `opacity=e`; `scale = 0.5 + e·0.5`; `offsetY = 20·(1-e)`; từ đang active thêm `style='active'` |
| `typewriter` | Chỉ hiển thị các từ có `startTime ≤ t`. Từ cuối cùng trong số đó fade in: `opacity = clamp((t - startTime)/0.1, 0, 1)` (0.1s); các từ trước opacity 1. Không có từ nào → `visible:false` |

`easeOutBounce` ở file này = bản 4-đoạn parabol giống §1.2.

**(3) Output:** `AnimatedCaptionFrame = { segments: WordSegment[], visible: boolean }`. Renderer ghép các segment thành dòng, tô màu/scale/offsetY theo từng segment, đặt theo `style.position` (top/center/bottom), kèm `backgroundColor` box.

**(4) Dùng ở đâu trong frame:** lớp phụ đề — vẽ **đè lên trên toàn bộ video**, thường ở dải dưới. Vị trí theo `style.position`. Là một loại text layer riêng (`Subtitle`), không phải `TextClip`.

**(5) Ghi chú re-implement:**
- `getAnimationStyleDisplayName`: `none→'Static'`, `word-highlight→'Word Highlight'`, `word-by-word→'Word by Word'`, `karaoke→'Karaoke'`, `bounce→'Bounce'`, `typewriter→'Typewriter'`. `CAPTION_ANIMATION_STYLES = ['none','word-highlight','word-by-word','karaoke','bounce','typewriter']`.
- Bắt buộc có `subtitle.words[]` per-word timing — nếu chỉ có 1 chuỗi text thì chỉ dùng được `none`. STT engine (`speech-to-text-engine.ts`) hoặc parser SRT (`subtitle-engine.ts`) sinh ra `words`.
- `karaoke` dùng CSS `linear-gradient` cho fill text — nếu render Canvas2D thuần phải tự vẽ 2 phần text với clip rect tại `progress·wordWidth`.

---

## 9. Emphasis animation — nhấn mạnh lặp lại

File: `packages/core/src/video/video-engine.ts` → `applyEmphasisAnimation(animation, time)`. **25 loại**. Áp cho **clip bất kỳ** (`Clip.emphasisAnimation`, `GraphicClip.emphasisAnimation`, text clip qua `TitleEngine`).

**(1) Đầu vào cần tổng hợp:**
- `EmphasisAnimation`: `{ type:EmphasisAnimationType, speed:number, intensity:number, loop:boolean, focusPoint?:{x,y}, zoomScale?:number, holdDuration?:number, startTime?:number, animationDuration?:number }`. Default: `{ type:'none', speed:1, intensity:1, loop:true }`.
- `time` (localTime của clip, giây).

**(2) Mô tả + công thức** — tiền xử lý:
```
animStart = startTime ?? 0
nếu time < animStart                              → no-op (tất cả 1/0)
nếu animationDuration>0 và time > animStart+animationDuration → no-op (animation đã kết thúc, về bình thường)
adjustedTime = time - animStart
cycleTime    = loop ? (adjustedTime·speed) % 1 : min(adjustedTime·speed, 1)
t            = cycleTime · 2π
```
Trả `{ opacity, scale, scaleX, scaleY, offsetX, offsetY, rotation }` — `offsetX/offsetY` là **hệ số normalized** (nhân với kích thước/khung khi áp), `rotation` độ. `scale` là scale đồng đều, `scaleX/scaleY` scale theo trục (kết hợp với `scale`).

| `type` | Công thức |
|---|---|
| `none` | tất cả `1`/`0` |
| `pulse` | `scale = 1 + sin(t)·0.1·intensity` |
| `shake` | `offsetX = sin(5t)·0.02·intensity`; `offsetY = cos(5t)·0.02·intensity` |
| `bounce` | `offsetY = |sin(t)|·(-0.05)·intensity` (nảy lên) |
| `float` | `offsetY = sin(t)·0.03·intensity` |
| `spin` | `rotation = cycleTime·360·intensity` |
| `flash` | `opacity = 0.5 + |sin(t)|·0.5` |
| `heartbeat` | `phase = cycleTime·4`; `phase<1 → scale = 1 + 0.15·intensity·sin(phase·π)`; `phase<2 → scale = 1 + 0.1·intensity·sin((phase-1)·π)`; còn lại `1` (hai nhịp đập) |
| `swing` | `rotation = sin(t)·15·intensity` |
| `wobble` | `rotation = sin(3t)·5·intensity`; `offsetX = sin(t)·0.02·intensity` |
| `jello` | `scaleX = 1 + sin(2t)·0.1·intensity`; `scaleY = 1 - sin(2t)·0.1·intensity` |
| `rubber-band` | `scaleX = 1 + sin(t)·0.2·intensity`; `scaleY = 1 - sin(t)·0.1·intensity` |
| `tada` | `rotation = sin(4t)·10·intensity`; `scale = 1 + sin(2t)·0.1·intensity` |
| `vibrate` | `offsetX = (rand-0.5)·0.02·intensity`; `offsetY = (rand-0.5)·0.02·intensity` (random) |
| `flicker` | `opacity = rand > 0.1 ? 1 : 0.3` (random) |
| `glow` | `scale = 1 + sin(t)·0.05·intensity`; `opacity = 0.8 + sin(t)·0.2` |
| `breathe` | `scale = 1 + sin(0.5t)·0.08·intensity` (chậm) |
| `wave` | `offsetY = sin(t + adjustedTime·2)·0.03·intensity`; `rotation = sin(t)·5·intensity` |
| `tilt` | `rotation = sin(0.5t)·10·intensity` (chậm) |
| `zoom-pulse` | `scale = 1 + sin(t)·0.15·intensity` |
| `focus-zoom` | dùng `focusPoint(0.5,0.5)`, `zoomScale(1.5)`, `holdDuration(0.3)`. `zoomInPhase=0.3`, `zoomOutPhase = 1 - holdDuration - zoomInPhase`. <br>• `cycleTime<0.3`: `e = 1-(1-cycleTime/0.3)³`; `scale = 1 + (zoomScale-1)·e·intensity` <br>• `0.3 ≤ cycleTime < 0.3+holdDuration`: `scale = zoomScale·intensity` <br>• còn lại: `e = ((cycleTime-0.3-holdDuration)/zoomOutPhase)³`; `scale = zoomScale - (zoomScale-1)·e·intensity` <br>• `offsetX = (0.5 - focusPoint.x)·(scale-1)`; `offsetY = (0.5 - focusPoint.y)·(scale-1)` (giữ focusPoint cố định khi zoom) |
| `pan-left` | `offsetX = -cycleTime·0.2·intensity` |
| `pan-right` | `offsetX = +cycleTime·0.2·intensity` |
| `pan-up` | `offsetY = -cycleTime·0.2·intensity` |
| `pan-down` | `offsetY = +cycleTime·0.2·intensity` |
| `ken-burns` | `scale = 1 + cycleTime·0.3·intensity`; `offsetX = cycleTime·0.1·intensity`; `offsetY = cycleTime·0.05·intensity` (zoom + pan chậm — hiệu ứng "ảnh động") |

**(3) Output:** delta transform `{ opacity, scale, scaleX, scaleY, offsetX, offsetY, rotation }` — **cộng/nhân vào** transform đã có của clip (scale tổng = `transform.scale · scale · {scaleX,scaleY}`; position += offset·khung; rotation += rotation; opacity *= opacity).

**(4) Dùng ở đâu trong frame:** áp ngay sau bước transform của clip, trước effects/compositing — trong `video-engine.ts` (`clip.emphasisAnimation` khi `type !== 'none'`). Dùng để "nhấn nhá" logo, sticker, text, hoặc cho hiệu ứng Ken Burns trên ảnh tĩnh. UI: `EmphasisAnimationSection.tsx`.

**(5) Ghi chú re-implement:**
- `loop=true` (mặc định) → dao động vô hạn theo `(adjustedTime·speed)%1`. `loop=false` → chạy một lần rồi giữ ở `cycleTime=1`.
- `startTime`/`animationDuration` cho phép giới hạn animation vào một khoảng thời gian con của clip; ngoài khoảng → no-op (về bình thường).
- `focus-zoom`/`ken-burns` thường dùng `loop=false` + `animationDuration` = độ dài clip để zoom một lần.
- `vibrate`/`flicker` random — không deterministic; nếu cần ổn định cho export, thay `Math.random()` bằng hàm hash của `time`.
- `offsetX/Y` là hệ số normalized — khi áp lên video clip nhân với `canvasWidth/Height`; trên graphic clip (position normalized) cộng trực tiếp.

---

## 10. SVG / Graphic entry-exit animation

File: `packages/core/src/graphics/graphics-engine.ts` → `applyGraphicAnimation(type, progress, easing, isEntry)`. Metadata: `packages/core/src/graphics/svg-animation-presets.ts`. Áp cho `SVGClip.entryAnimation` (lúc clip xuất hiện) và `.exitAnimation` (lúc clip biến mất). **20 loại** (type), trong đó core engine hiện thực hiện 6 dạng transform (fade/slide/scale/rotate/bounce/pop), các loại còn lại (draw/wipe/reveal/elastic/flip) khai báo trong metadata và xử lý ở tầng render khác (clip-path/stroke-dashoffset).

**(1) Đầu vào cần tổng hợp:**
- `GraphicAnimation`: `{ type:GraphicAnimationType, duration:number(s), easing:string }`. `DEFAULT_GRAPHIC_ANIMATION = { type:'none', duration:0.5, easing:'ease-out' }`.
- `progress ∈ [0,1]` của giai đoạn entry (từ đầu clip, `relativeTime/duration`) hoặc exit (`(relativeTime - (clipDuration - duration))/duration`).
- `isEntry: boolean`. `easing: string` (chuỗi `EasingType` hoặc `cubic-bezier(...)`).
- `width`, `height` khung (để quy đổi offset normalized → pixel).

**(2) Mô tả + công thức** — tiền xử lý:
```
easedProgress = applyEasing(progress, easing)
animProgress  = isEntry ? easedProgress : (1 - easedProgress)
// trả về { opacity, scale, offsetX, offsetY, rotation, blur }  (offsetX/Y normalized)
```

| `type` | Công thức (core engine) | Default duration / easing (metadata) |
|---|---|---|
| `none` | no-op | 0 / linear |
| `fade` | `opacity = animProgress` | 0.5 / ease-out |
| `slide-left` ("from right") | `opacity=animProgress`, `offsetX = (1-animProgress)·(-1.0)` | 0.6 / ease-out |
| `slide-right` ("from left") | `offsetX = (1-animProgress)·(+1.0)` | 0.6 / ease-out |
| `slide-up` ("from bottom") | `offsetY = (1-animProgress)·(-1.0)` | 0.6 / ease-out |
| `slide-down` ("from top") | `offsetY = (1-animProgress)·(+1.0)` | 0.6 / ease-out |
| `scale` | `opacity=animProgress`, `scale = animProgress` | 0.5 / ease-out |
| `rotate` | `opacity=animProgress`, `rotation = (1-animProgress)·(isEntry ? -180 : 180)` | 0.8 / ease-out |
| `bounce` | `bp = |sin(animProgress·3π)|·(1-animProgress)`; `opacity=animProgress`, `offsetY = bp·(-0.1)` | 0.8 / `cubic-bezier(0.68,-0.55,0.265,1.55)` |
| `pop` | overshoot 1.2: `animProgress<0.5 → s=animProgress·2·1.2`; `<0.7 → s=1.2-(animProgress-0.5)·0.2/0.2`; `else → 1`; `opacity=min(animProgress·2,1)`, `scale=s` | 0.5 / `cubic-bezier(0.175,0.885,0.32,1.275)` |
| `draw` | (SVG stroke reveal — `stroke-dasharray`/`stroke-dashoffset` từ 100%→0; không có trong `applyGraphicAnimation`, render bằng SVG attrs) | 1.2 / ease-in-out |
| `wipe-left` / `wipe-right` / `wipe-up` / `wipe-down` | (reveal bằng `clip-path` quét từ một cạnh; render tầng SVG/DOM) | 0.7 / ease-out |
| `reveal-center` | (expand `clip-path` từ tâm: scale clip 0→1) | 0.6 / ease-out |
| `reveal-edges` | (collapse về tâm: scale clip 1→0 hoặc inset) | 0.6 / ease-out |
| `elastic` | (scale với overshoot đàn hồi — dùng easing `cubic-bezier(0.68,-0.55,0.265,1.55)`) | 1.0 / `cubic-bezier(0.68,-0.55,0.265,1.55)` |
| `flip-horizontal` | (3D flip quanh trục Y: `scaleX = cos(angle)`, `angle = 90°·(1-animProgress)`) | 0.8 / ease-out |
| `flip-vertical` | (3D flip quanh trục X: `scaleY = cos(angle)`) | 0.8 / ease-out |

**(3) Output:** `{ opacity, scale, offsetX, offsetY, rotation, blur }` — nhân/cộng vào transform của graphic clip khi vẽ. (Graphic clip vẽ bằng `ctx.translate(pos·size) → rotate → scale → globalAlpha=opacity`.)

**(4) Dùng ở đâu trong frame:** `GraphicsEngine` khi render `SVGClip` kiểm tra `relativeTime` so với `entryAnimation.duration` / `exitAnimation`: trong vùng entry → áp `applyGraphicAnimation(entry.type, progress, entry.easing, true)`; trong vùng exit → `applyGraphicAnimation(exit.type, progress, exit.easing, false)`; ở giữa → không animate. Áp lên layer SVG/sticker trên canvas, tại vị trí graphic clip.

**(5) Ghi chú re-implement:**
- `getSVGPresetInfo(type)` / `createDefaultSVGAnimation(type)` lấy default từ `SVG_ANIMATION_PRESETS`.
- `applyEasing(progress, easing)` ở đây nhận chuỗi — cần parser cho `cubic-bezier(a,b,c,d)` (tạo hàm bằng `cubicBezier`) ngoài các tên `EasingType`.
- `offsetX/offsetY` normalized → nhân với `width/height` khi áp; `bounce` cho `offsetY` âm = nảy lên trên màn hình.
- Để re-implement đầy đủ `draw`/`wipe`/`reveal`/`flip`: dễ nhất render SVG trong `<svg>` DOM và animate bằng CSS (`stroke-dashoffset`, `clip-path: inset()/circle()`, `transform: rotateX/Y(...) perspective(...)`) — đúng cách OpenReel làm ở preview; còn export Canvas thì phải tự vẽ clip path/2 nửa text.
- 6 loại transform-based (fade/slide/scale/rotate/bounce/pop) hoạt động đầy đủ qua `applyGraphicAnimation` và an toàn dùng cho mọi backend.
- Cùng tập preset cũng được dùng cho **emphasis-style** liên tục trên graphic (`EmphasisAnimation` ở §9 áp riêng).

---

## 11. Particle effects

File: `packages/core/src/effects/particle-engine.ts`, `particle-presets.ts`, `particle-types.ts`. Hệ particle emitter-based, mô phỏng vật lý đơn giản (CPU).

**(1) Đầu vào cần tổng hợp:**
- `ParticleEffect`: `{ id, clipId, type:ParticleEffectType, startTime(s), duration(s), config:ParticleConfig, enabled:boolean }`.
- `ParticleConfig`:
  ```
  particleCount: số particle tối đa đồng thời
  speed, speedVariance: tốc độ phát ± dao động
  gravity: gia tốc theo y (px/s²)
  wind: {x,y,z}  (gia tốc thêm)
  turbulence: nhiễu ngẫu nhiên cộng vào velocity mỗi bước
  colors: string[]  (mỗi particle chọn ngẫu nhiên 1 màu)
  size: {min,max}   (px)
  opacity: {start,end}
  lifetime: {min,max}  (s)
  emissionRate: số particle phát ra mỗi giây
  emissionShape: 'point'|'line'|'circle'|'rectangle'|'sphere'
  emissionRadius: bán kính/độ rộng vùng phát
  rotationSpeed: tốc độ xoay particle (± ngẫu nhiên)
  fadeIn, fadeOut: tỉ lệ (0–1) của lifetime dùng để fade
  blendMode: 'normal'|'add'|'multiply'|'screen'
  ```
  `DEFAULT_PARTICLE_CONFIG`: count 100, speed 100±50, gravity 200, wind 0, turbulence 10, colors `['#ffffff']`, size 2–8, opacity 1→0, lifetime 0.5–2, emissionRate 50, shape point, radius 10, rotationSpeed 0, fadeIn 0.1, fadeOut 0.3, blendMode add.
- `currentTime`, `deltaTime` (gọi `update(currentTime, deltaTime)` mỗi frame).
- `canvasWidth`, `canvasHeight` (tâm phát = `(w/2, h/2)`).

**(2) Mô tả + công thức:**

*Vòng đời update mỗi effect (khi `startTime ≤ currentTime ≤ startTime+duration`):*
```
emissionAccumulator += emissionRate · deltaTime
while emissionAccumulator ≥ 1 and particles.length < particleCount:
    spawn particle; emissionAccumulator -= 1

cho mỗi particle active:
    velocity += (acceleration + wind) · deltaTime          // acceleration.y = gravity
    nếu turbulence>0: velocity.{x,y} += rand(-turb, turb) · deltaTime
    position += velocity · deltaTime
    rotation += rotationSpeed · deltaTime
    age += deltaTime
    lifeProgress = age / lifetime
    nếu lifeProgress < fadeIn:        opacity = opacity.start · (lifeProgress / fadeIn)
    nếu lifeProgress > 1 - fadeOut:   opacity = opacity.end + (opacity.start - opacity.end) · ((1-lifeProgress)/fadeOut)
    ngược lại:                        opacity = lerp(opacity.start, opacity.end, (lifeProgress - fadeIn)/(1 - fadeIn - fadeOut))
    nếu age ≥ lifetime: particle.active = false
xoá particle inactive
```

*Vị trí phát theo `emissionShape`* (quanh `center`):
- `point`: `center`.
- `circle`: góc `θ=rand·2π`, `r=rand·emissionRadius` → `(center + r·(cosθ, sinθ))`, z = center.z.
- `sphere`: `θ=rand·2π`, `φ=acos(2·rand-1)`, `r=rand·radius` → toạ độ cầu 3D.
- `rectangle`: `center + (rand(-radius,radius), rand(-radius,radius))`.
- `line`: `center + (rand(-radius,radius), 0)`.

*Vận tốc ban đầu theo `effectType`* (`speed = config.speed ± speedVariance`):
- `explode`: hướng ra xa tâm — `dir = normalize(pos - center)`, `v = dir·speed + rand(-20,20)`, `vz = rand(-speed/2, speed/2)`.
- `implode`: hướng vào tâm — `dir = normalize(center - pos)`, `v = dir·speed`.
- `confetti`: `vx = rand(-speed,speed)`, `vy = -|speed|·rand(0.5,1.5)` (bắn lên), `vz = rand(-0.3speed,0.3speed)`.
- `dust` / `sparkle`: `angle=rand·2π`, `v = (cosθ·0.3speed, sinθ·0.3speed - 0.2speed, rand(-10,10))`.
- `disintegrate`: `vx = rand(-0.5speed,0.5speed)`, `vy = -speed·rand(0.3,0.8)`, `vz = rand(-0.2speed,0.2speed)`.
- `shatter`: `angle=rand·2π`, `v = (cosθ·speed, sinθ·speed - 0.5speed, rand(-0.5speed,0.5speed))`.
- `dissolve` (và mặc định): `angle=rand·2π`, `v = (cosθ·0.5speed, sinθ·0.5speed, 0)`.
- `pixelate` / `morph`: dùng nhánh mặc định (`dissolve`), tạo "khối" bằng `size` lớn.

**(3) Output:** danh sách `Particle[]` đang sống — mỗi `{ id, position:{x,y,z}, velocity, acceleration, color, size, opacity, rotation, rotationSpeed, lifetime, age, active }`. Renderer vẽ mỗi particle (hình tròn/vuông/tam giác/sao tuỳ shape preset chọn — `ParticleEmitterConfig.particleShape` trong schema) tại `(position.x, position.y)`, xoay `rotation`, kích thước `size`, màu `color`, `globalAlpha = opacity`, với `globalCompositeOperation` theo `blendMode` (`add → 'lighter'`).

**(4) Dùng ở đâu trong frame:** `ParticleEngine` gắn vào `clipId` — particle layer được render **phủ lên** (hoặc thay thế) clip đó trong khoảng `[startTime, startTime+duration]`. Dùng cho hiệu ứng vỡ/biến mất/lễ hội/khói/lấp lánh. Preview render trong `Preview.tsx`.

**(5) `PARTICLE_PRESETS` (12 preset):**

| id | type | Cấu hình nổi bật |
|---|---|---|
| `dissolve-soft` | dissolve | count 200, speed 30±20, gravity 10, màu xám, size 2–6, op 0.8→0, life 1–3s, rate 100, fadeOut 0.5, blend normal |
| `explode-burst` | explode | count 300, speed 400±150, gravity 300, màu lửa, size 3–12, life 0.5–1.5s, rate 500, shape circle r20, fadeOut 0.4, blend **add** |
| `implode-gather` | implode | count 150, speed 200±50, gravity 0, màu magic, size 4–10, op 0→1, life 0.8–1.5s, rate 80, shape circle r300, fadeIn 0.3, blend add |
| `confetti-celebration` | confetti | count 200, speed 150±80, gravity 200, wind x30, turbulence 50, màu confetti, size 6–14, life 2–4s, rate 60, shape rectangle r400, rotationSpeed 5, fadeOut 0.3, blend normal |
| `dust-ambient` | dust | count 80, speed 15±10, gravity **-5** (bay lên), turbulence 20, xám, size 1–3, op 0.4→0.1, life 3–8s, rate 10, shape rectangle r500, fadeIn 0.3 fadeOut 0.5 |
| `sparkle-magic` | sparkle | count 100, speed 20±15, gravity 10, màu sáng, size 2–6, op 1→0, life 0.3–0.8s, rate 80, shape sphere r100, fadeIn 0.1 fadeOut 0.5, blend add |
| `disintegrate-thanos` | disintegrate | count 400, speed 60±40, gravity **-30**, wind x50 y-20, turbulence 30, xám, size 2–5, op 1→0, life 1–2.5s, rate 200, shape rectangle r200, fadeOut 0.6, blend normal |
| `pixelate-retro` | pixelate | count 150, speed 100±50, gravity 400, màu RGB rực, size 8–16, op 1→0.5, life 0.5–1.2s, rate 200, shape rectangle r150, rotationSpeed 3, fadeOut 0.3 |
| `shatter-glass` | shatter | count 250, speed 300±150, gravity 500, màu trắng/xám, size 3–12, op 0.9→0.3, life 0.4–1s, rate 400, shape circle r50, rotationSpeed 8, fadeOut 0.4 |
| `morph-transform` | morph | count 200, speed 50±30, gravity 0, màu magic, size 4–8, op 0.8→0.8, life 1–2s, rate 100, shape circle r150, fadeIn 0.2 fadeOut 0.2, blend add |
| `fire-trail` | dust | count 150, speed 80±40, gravity **-100** (bay lên), màu lửa, size 4–12, op 0.9→0, life 0.3–0.8s, rate 120, shape circle r20, fadeIn 0.05 fadeOut 0.6, blend add |
| `snow-fall` | dust | count 100, speed 30±20, gravity 50, wind x20, turbulence 30, màu trắng, size 2–6, op 0.8→0.3, life 4–8s, rate 20, shape rectangle r600, rotationSpeed 1, fadeIn 0.2 fadeOut 0.4 |

Bảng màu preset: confetti `#ff6b6b #ffd93d #6bcb77 #4d96ff #ff6b9d #845ef7`; lửa `#ff4500 #ff6b00 #ffb700 #ffd700`; sparkle `#ffffff #fffacd #fff8dc #ffffe0`; bụi `#a0a0a0 #808080 #606060 #404040`; magic `#9b59b6 #8e44ad #3498db #2980b9 #e91e63`.

`createEffectFromPreset(presetId, effectId, clipId, startTime, duration)` → `ParticleEffect`. `getParticlePresetsByType(type)` lọc theo loại.

> **Re-implement particle**: thuần CPU, không phụ thuộc gì. Lưu ý: vị trí phát quanh tâm canvas (cần dịch theo vị trí clip nếu muốn particle theo clip). `pixelate`/`morph` chưa có nhánh vận tốc riêng → dùng `dissolve`. `rotationSpeed` trong config là biên độ (mỗi particle random `±rotationSpeed`). Để deterministic cho export cần seed RNG theo `(effectId, particleIndex, spawnTime)`.

---

## 12. Speed ramping / speed curves

File: `packages/core/src/video/speed-engine.ts`, `speed-presets.ts`. Animate **tốc độ phát** của clip theo thời gian (slow-mo / fast-forward biến thiên).

**(1) Đầu vào cần tổng hợp:**
- `clip.speed?: number` (hằng số, ví dụ 0.5 = chậm 2×) **hoặc** keyframes property `"speed"` (đường cong tốc độ) — mỗi điểm `{ time(normalized 0–1 trong clip), speed, easing }`.
- `clip.reversed?: boolean` (phát ngược).
- `clip.smoothSlowMo?: boolean` + `clip.interpolationQuality?: 'low'|'medium'|'high'` → bật frame interpolation (optical flow / blend) khi `speed < 1`.
- `clip.inPoint`/`outPoint`, `fps`.

**(2) Mô tả + công thức:** thời gian timeline ↦ thời gian media qua tích phân tốc độ. Với tốc độ hằng: `mediaTime = inPoint + (timelineTime - clipStart)·speed`. Với đường cong: tích phân số `mediaTime(t) = inPoint + ∫₀ᵗ speed(τ) dτ` (sample đường cong tốc độ đã nội suy keyframe + easing). `reversed` → đảo chiều: lấy mẫu từ `outPoint` lùi về. Khi `speed < 1` và `smoothSlowMo` → sinh frame trung gian bằng `frame-interpolation/` (blend 2 frame gốc, hoặc optical flow tuỳ `interpolationQuality`). Phạm vi tốc độ UI: 0.25× – 4×; audio giữ cao độ (pitch preservation) khi đổi tốc độ.

**(3) Output:** ánh xạ `timelineTime → mediaTime` (và frame nội suy nếu có) — quyết định `decodeFrame(media, mediaTime)` lấy frame nào. Cũng đổi `duration` hiển thị của clip (`originalDuration / speed`).

**(4) Dùng ở đâu trong frame:** ngay sau khi xác định clip phủ `time`, trước `decodeFrame` — bước "SpeedEngine / FrameInterpolationEngine" trong pipeline. Áp cho video clip (và audio tương ứng).

**(5) `SPEED_CURVE_PRESETS` (6 preset)** — mảng keyframes `{time(0–1), speed, easing}`:

| id | mô tả | keyframes |
|---|---|---|
| `flash` | bùng tốc ở giữa | (0, 0.5, easeInQuad) → (0.3, 4, easeOutQuad) → (0.7, 4, easeInQuad) → (1, 0.5, linear) |
| `smooth-slow-mo` | chậm dần rồi nhanh lại | (0, 1, easeInOutCubic) → (0.3, 0.3, linear) → (0.7, 0.3, easeInOutCubic) → (1, 1, linear) |
| `jump-cut` | xen kẽ nhanh/thường | (0,1) (0.2,3) (0.4,1) (0.6,3) (0.8,1) (1,3) — linear |
| `montage` | tăng tốc dần | (0, 1, easeInQuart) → (0.5, 1.5, easeInQuart) → (1, 3, linear) |
| `hero-moment` | chậm kịch tính ở đỉnh | (0, 1.5, easeInCubic) → (0.35, 0.2, linear) → (0.65, 0.2, easeOutCubic) → (1, 1.5, linear) |
| `bullet-time` | gần như đóng băng giữa | (0, 2, easeInExpo) → (0.4, 0.1, linear) → (0.6, 0.1, easeOutExpo) → (1, 2, linear) |

> **Re-implement**: nội suy `speed(t)` bằng keyframe engine (§2) trên trục `t∈[0,1]`, rồi tích phân số (Riemann/trapezoid theo bước nhỏ ~1/fps) để được `mediaTime`. Frame interpolation là phần riêng (optical flow) — có thể bỏ ở bản đơn giản (chỉ lấy frame gần nhất → "stutter" nhẹ khi slow-mo cực mạnh).

---

## 13. Expression-driven animation

File: `packages/core/src/effects/expression-engine.ts` (`ExpressionEngine`). Cho phép drive một property bằng **biểu thức code** (kiểu After Effects) thay vì keyframe.

**(1) Đầu vào cần tổng hợp:**
- `expression: string` — biểu thức JS rút gọn, ví dụ `wiggle(2, 30)`, `value + Math.sin(time*5)*10`, `linear(time, 0, 2, 0, 100)`.
- `ExpressionContext`: `{ time, value, velocity, fps, width, height }` + các hàm helper.

**(2) Mô tả + helper functions:**
- `wiggle(freq, amp)`: nhiễu mượt — `seed = floor(time·freq)`, `t = (time·freq)%1`, nội suy smoothstep giữa `pseudoRandom(seed)·amp` và `pseudoRandom(seed+1)·amp`. `pseudoRandom(s) = fract(sin(s)·10000)`.
- `noise(x)`: Perlin 1D đơn giản — `xi=floor(x)`, `xf=x-xi`, `u=xf²(3-2xf)`, `lerp(pseudoRandom(xi), pseudoRandom(xi+1), u)`.
- `smooth(values, width=5, samples=5)`: trung bình `samples` giá trị gần nhất.
- `linear(t, tMin, tMax, v1, v2)`: clamp + nội suy tuyến tính (`progress = (t-tMin)/(tMax-tMin)`).
- `ease(t, tMin, tMax, v1, v2)`: như `linear` nhưng `eased = progress²(3-2·progress)` (smoothstep).
- `easeIn(...)`: `eased = progress²`. `easeOut(...)`: `eased = 1-(1-progress)²`.
- `clamp(v, min, max)`, `random(min?, max?)` (= `Math.random`).
- `Math` được phơi nguyên (sandbox an toàn — chỉ `Math` + helper, không truy cập DOM/global).

**(3) Output:** giá trị (number/object) cho property tại `time`. `evaluate(expression, context)` compile (`new Function`) lần đầu, **cache** theo chuỗi biểu thức; lỗi compile → trả `0`.

**(4) Dùng ở đâu trong frame:** thay thế kết quả keyframe của một property bất kỳ (position, rotation, scale, opacity, tham số effect...) tại bước transform/effect. Cho phép hiệu ứng "rung tự nhiên", "dao động", liên kết property...

**(5) Ghi chú re-implement:** an toàn — sandbox chỉ chứa `Math` + helper. `wiggle`/`noise` deterministic theo `time` (vì dùng `sin`-hash, không `Math.random`). Có thể mở rộng context (thêm `loopOut`, `valueAtTime`...). Lưu ý `new Function` cần CSP cho phép `unsafe-eval` — nếu không, phải interpret thủ công.

---

## 14. Hiệu ứng video phụ thuộc thời gian

Không phải "animation timeline" theo nghĩa keyframe nhưng **thay đổi theo từng frame** — cần biết để re-implement chính xác:

- **`grain` (film grain)** — `VideoEffectsEngine`: shader GLSL dùng `random(v_texCoord·u_size + u_time)` với `u_time = performance.now()/1000` → hạt nhiễu **thay đổi mỗi frame**. CPU path: `data[i] += (rand-0.5)·amount/100·50` mỗi pixel. Param: `amount`, `size`, (`roughness`, `colored` ở định nghĩa đầy đủ). → Hiệu ứng "phim cũ" động.
- **`chromatic-aberration`** (định nghĩa `EFFECT_DEFINITIONS`): tách 3 kênh R/G/B lệch nhau theo `amount(px)` & `angle(deg)` — tĩnh trừ khi animate `amount` bằng keyframe.
- **`motion-blur`** (`angle 0–360`, `distance 0–100px`): làm mờ theo hướng — thường keyframe `angle`/`distance` theo chuyển động.
- **`rainbow` text & `glitch` text** (§6/§7): màu/offset đổi theo `time` (hoặc theo `progress` & hash).
- **`emphasisAnimation` continuous** (§9): toàn bộ là hàm của `time`.
- **`flicker`/`vibrate`** (§9), **`grain`**: dùng `Math.random()` → không deterministic; export có thể nhấp nháy nếu render lại frame.
- **Color grading** (`ColorGradingEngine`): tĩnh, nhưng mọi param (lift/gamma/gain, curve point, HSL, LUT intensity) đều **animatable bằng keyframe** → có thể làm "color transition" theo thời gian.
- **`FILTER_PRESETS`** (`filter-presets.ts`, 18 preset gộp nhiều effect): bản thân tĩnh, nhưng dùng làm điểm xuất phát rồi keyframe từng effect.

---

## 15. Bảng tổng hợp nhanh

| # | Hệ animation | Số biến thể | Phạm vi áp dụng | File chính |
|---|---|---|---|---|
| 1 | Easing functions | 30 named + cubic-bezier + spring + 10 preset | nền tảng cho mọi animation | `animation/easing-functions.ts` |
| 2 | Keyframe property | bất kỳ property (number/object) | mọi clip | `video/animation-engine.ts`, `keyframe-engine.ts` |
| 3 | Transform | 12 property + ma trận affine + 3D | mọi clip | `video/transform-animator.ts` |
| 4 | Motion path | linear / bezier / catmull-rom + auto-orient | video/image/graphic/text | `animation/gsap-engine.ts` |
| 5 | Transitions (clip↔clip) | 7: crossfade, dipToBlack, dipToWhite, wipe, slide, zoom, push | giữa 2 clip kề nhau cùng track | `video/transition-engine.ts` |
| 6 | Text animation (clip-level) | 19: none, typewriter, fade, slide×4, scale, blur, bounce, rotate, wave, shake, pop, glitch, split, flip, word-by-word, rainbow | text clip toàn cục | `text/text-animation.ts` |
| 7 | Text animation (unit-level) | 19 (như trên) × unit {character/word/line} + stagger | text clip per-char/word/line | `text/text-animation-presets.ts`, `character-animator.ts` |
| 8 | Caption animation | 6: none, word-highlight, word-by-word, karaoke, bounce, typewriter | phụ đề (Subtitle, per-word timing) | `text/caption-animation-renderer.ts` |
| 9 | Emphasis animation | 25: pulse, shake, bounce, float, spin, flash, heartbeat, swing, wobble, jello, rubber-band, tada, vibrate, flicker, glow, breathe, wave, tilt, zoom-pulse, focus-zoom, pan-left/right/up/down, ken-burns | mọi clip (nhấn mạnh lặp lại) | `video/video-engine.ts` (applyEmphasisAnimation) |
| 10 | SVG/Graphic entry-exit | 20: none, fade, slide×4, scale, rotate, bounce, pop, draw, wipe×4, reveal-center, reveal-edges, elastic, flip-horizontal/vertical | SVG/sticker/shape clip | `graphics/graphics-engine.ts`, `graphics/svg-animation-presets.ts` |
| 11 | Particle effects | 10 loại (dissolve, explode, implode, confetti, dust, sparkle, disintegrate, pixelate, shatter, morph) + 12 preset | layer phủ lên clip | `effects/particle-engine.ts`, `particle-presets.ts` |
| 12 | Speed ramping | tốc độ hằng / đường cong + 6 preset (flash, smooth-slow-mo, jump-cut, montage, hero-moment, bullet-time) + frame interpolation | video clip (+audio) | `video/speed-engine.ts`, `speed-presets.ts` |
| 13 | Expression-driven | wiggle, noise, smooth, linear, ease/easeIn/easeOut, clamp, random + Math | property bất kỳ | `effects/expression-engine.ts` |
| 14 | Hiệu ứng theo thời gian | grain (động), chromatic-aberration, motion-blur, color-grading animatable, rainbow/glitch text... | filter cấp clip / color | `video/video-effects-engine.ts`, `color-grading-engine.ts`, `video/filter-presets.ts` |

**Easing được dùng bởi**: keyframe (mọi property), transform animation, motion path (qua GSAP), text animation (in/out + stagger), SVG entry/exit, transition (bản rút gọn), speed curves, caption bounce (`easeOutBounce` hard-coded). Emphasis animation **không** dùng easing — chúng là hàm sin/cos trực tiếp của `time`.

**Lưu ý re-implement chung:**
1. Tách `easing-functions.ts` ra trước — nền tảng, không phụ thuộc.
2. Keyframe engine (`getValueAtTime` + `interpolateValue`) là "lõi" — mọi thứ khác build lên trên.
3. Hệ toạ độ: thống nhất normalized vs pixel cho từng loại layer (xem §0). Sai chỗ này là sai toàn bộ vị trí.
4. Phân biệt `time` toàn cục vs `localTime` clip vs `progress` đoạn — mỗi hệ animation dùng loại khác nhau (đã ghi rõ ở từng mục).
5. Random vs deterministic: ưu tiên bản hash-based (`glitch` unit-level) cho export; bản `Math.random()` (`vibrate`, `flicker`, `grain`, `glitch` clip-level) chỉ hợp preview.
6. Transition đặt **giữa** ranh giới clip (overlap đối xứng `±duration/2`), không phải sau clip A.
7. Animation `in/out` của text & graphic chỉ chạy ở 2 đầu clip; phần giữa ở trạng thái "đã vào hết" (`progress=1`).
