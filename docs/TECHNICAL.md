# OpenReel Video — Tài liệu kỹ thuật (engines & subsystems)

> Bổ sung cho [`ARCHITECTURE.md`](./ARCHITECTURE.md). Danh mục animation đầy đủ + công thức re-implement: [`ANIMATIONS.md`](./ANIMATIONS.md).
> Mọi đường dẫn file tương đối với gốc repo.

---

## 1. Module `packages/core/src/video`

| File | Vai trò |
|---|---|
| `video-engine.ts` | Orchestrator render frame (~2.4k LOC). `renderFrame(project, time) → RenderedFrame`. Sở hữu: GPU init, parallel decoder, composite buffer, frame cache (~500MB/~100 frame). Chứa `applyEmphasisAnimation()` (25 loại) và `decodeFrame(mediaItem, time)` (qua MediaBunny `Input`/`BlobSource`/`VideoSampleSink`). Ảnh tĩnh decode trực tiếp bằng `createImageBitmap`. |
| `playback-engine.ts` | `PlaybackController`: master clock, play/pause/stop/seek, scrub mode, playbackRate; phát event `timeupdate`/`statechange`. |
| `composite-engine.ts` | `CompositeEngine`: compositing 2D thuần bằng `OffscreenCanvas`. `compositeLayers(layers, backgroundColor)`. Blend mode qua `globalCompositeOperation` + pixel-blend cho mode không có sẵn. |
| `gpu-compositor.ts` | `GPUCompositor`: quản lý layer (add/update/remove), delegate render cho `Renderer`, track memory/stats. |
| `renderer-factory.ts` | `RendererFactory` (singleton). `createRenderer(config) → Renderer`. `isWebGPUSupported()`. Interface `Renderer`: `initialize`, `createTextureFromImage`, `createRenderTarget`, `renderLayer(LayerRenderInput)`, `applyEffects(texture, Effect[])`, `beginFrame`/`endFrame`, `resize`, `destroy`, `type`. |
| `webgpu-renderer-impl.ts` | `WebGPURenderer`: acquire `GPUDevice`, pipeline render/compute, dùng `.wgsl` shaders. |
| `canvas2d-fallback-renderer.ts` | `Canvas2DFallbackRenderer`: thuần Canvas2D — `globalAlpha`, transform, pixel blend. |
| `webgpu-effects-processor.ts` / `webgpu-types.d.ts` | Wrapper compute-shader effects + type defs WebGPU. |
| `unified-effects-processor.ts` | Lớp thống nhất chọn GPU/CPU cho effects. |
| `video-effects-engine.ts` | `VideoEffectsEngine`: filter cấp clip (xem §3). |
| `color-grading-engine.ts` | `ColorGradingEngine` (~26k LOC): color wheels (lift/gamma/gain per shadows/midtones/highlights), curves RGB+kênh, 3D LUT (parse `.cube`, áp với intensity), HSL 8-vùng (hue/sat/lum). |
| `keyframe-engine.ts` | `KeyframeEngine`: CRUD keyframe + nội suy + bezier handle + motion path sampling + clipboard copy/paste keyframe. |
| `animation-engine.ts` | `AnimationEngine`: nội suy keyframe generic + easing + cubic-bezier solver (Newton-Raphson + bisection, có cache). |
| `transform-animator.ts` | `TransformAnimator`: lấy `Transform` tại thời điểm t từ keyframes của 12 property; `computeTransformMatrix()` (anchor + rotation + scale + translate → ma trận affine 2D `{a,b,c,d,e,f}`); helper tạo keyframe set position/scale/rotation/opacity. |
| `transition-engine.ts` | `TransitionEngine`: 7 loại transition + validate handle frames + tính progress. |
| `speed-engine.ts` / `speed-presets.ts` | Speed ramping: map thời gian timeline ↔ thời gian media qua đường cong tốc độ; 6 preset (flash/smooth-slow-mo/jump-cut/montage/hero-moment/bullet-time). |
| `frame-interpolation/` | Slow-mo: optical flow (CPU/GPU) + frame blending. |
| `stabilization/` | VidStab: phân tích chuyển động → ma trận bù + crop; `StabilizationProfile`. |
| `upscaling/` | AI upscaling bằng WebGPU shaders. |
| `mask-engine.ts` | Mask cấp clip: rectangle/ellipse/polygon/bezier, feather, inverted, expansion. |
| `motion-tracking-engine.ts` | Track điểm/đối tượng qua frame (template matching) → trả quỹ đạo để gắn effect/text. |
| `multicam-engine.ts` | Multicam: nhiều góc quay, switch theo thời gian. |
| `adjustment-layer-engine.ts` | Adjustment layer: effect áp lên mọi layer bên dưới trong vùng thời gian. |
| `chroma-key-engine.ts` | Chroma key chuyên dụng (key color, tolerance, spill suppression, edge softness). |
| `frame-cache.ts` / `frame-ring-buffer.ts` / `texture-cache.ts` | Cache frame LRU / ring buffer / GPU texture cache. |
| `decode-worker.ts` / `parallel-frame-decoder.ts` | Decode video song song trong worker. |

### 1.1 Định dạng `RenderedFrame`
`{ image: ImageBitmap, width: number, height: number }` — đơn vị trao đổi giữa engine ↔ bridge ↔ canvas.

---

## 2. Hệ thống keyframe & nội suy (cốt lõi animation)

### 2.1 `Keyframe`
```ts
interface Keyframe { id: string; time: number; property: string; value: unknown; easing: EasingType; }
```
- `property`: chuỗi có thể có dấu chấm — `"position.x"`, `"position.y"`, `"scale.x"`, `"scale.y"`, `"rotation"`, `"opacity"`, `"anchor.x"`, `"anchor.y"`, `"rotate3d.x|y|z"`, `"perspective"`, hoặc property hiệu ứng (`"blur"`, `"brightness"`, `"contrast"`, `"saturation"`...). Đặc tả mở rộng: `AnimatableProperty` trong `animation/animation-schema.ts`.
- `easing` áp lên **đoạn từ keyframe này tới keyframe kế tiếp** (easing thuộc keyframe trái của cặp).

### 2.2 Thuật toán `getValueAtTime(keyframes, time)` (`AnimationEngine` / `KeyframeEngine`)
```
sort keyframes theo time
nếu time <= kf[0].time      → trả kf[0].value          (clamp đầu, progress 0)
nếu time >= kf[last].time   → trả kf[last].value        (clamp cuối, progress 1)
tìm cặp (A, B) sao cho A.time <= time <= B.time
duration   = B.time - A.time
linearT    = duration > 0 ? (time - A.time) / duration : 0
easedT     = applyEasing(linearT, A.easing)
value      = interpolateValue(A.value, B.value, easedT)
```
`interpolateValue(a, b, t)`:
- `number` ↔ `number`: `a + (b - a) * t`
- `object` ↔ `object`: nội suy đệ quy theo từng key (key chỉ có ở `a` thì giữ nguyên)
- ngược lại: `t < 0.5 ? a : b` (step)

### 2.3 Bezier easing handle (`KeyframeEngine`)
`ExtendedKeyframe` thêm `bezierHandles: { in:{x,y}, out:{x,y} }`. Khi `easing === "bezier"` dùng `cubicBezier(t, in.x, in.y, out.x, out.y)`. Handle mặc định theo preset:
| Preset | in | out |
|---|---|---|
| ease-in | (0.42, 0) | (1, 1) |
| ease-out | (0, 0) | (0.58, 1) |
| ease-in-out | (0.42, 0) | (0.58, 1) |
| bounce | (0.34, 1.56) | (0.64, 1) |
| elastic | (0.68, -0.55) | (0.27, 1.55) |
| spring | (0.5, 1.5) | (0.5, 1) |

Solver cubic-bezier (chung cho `easing-functions.ts` và `animation-engine.ts`): chuyển bezier 2D thành hàm 1D bằng cách giải `sampleCurveX(t) = x` (Newton-Raphson ≤4–8 vòng, slope min ~0.001) rồi fallback **bisection** (≤10 vòng, precision ~1e-7), trả `sampleCurveY(t)`. Hệ số Horner:
```
ax = 3x1 - 3x2 + 1;  bx = 3x2 - 6x1;  cx = 3x1   (tương tự cho y)
sampleCurveX(t) = ((ax*t + bx)*t + cx)*t
```
Kết quả được cache theo key `"x1,y1,x2,y2"`.

### 2.4 Motion path
- `KeyframeEngine.getMotionPath(clipId, keyframes, sampleCount=100)` lấy keyframe `position.x`/`position.y`, sample đều theo thời gian → mảng `{time,x,y}` để vẽ đường đi trên Preview.
- `animation/gsap-engine.ts` (`GSAPAnimationEngine`): motion path "chuyên nghiệp" — `MotionPathConfig{pathType:'linear'|'bezier'|'catmull-rom', points:GSAPMotionPathPoint[], autoOrient, alignOrigin}`. Hàm độc lập: `sampleMotionPath` (linear/cubic-bezier per segment, trả thêm `rotation` = `atan2` của tiếp tuyến cho auto-orient), `catmullRomInterpolate(tension=0.5)` (Catmull-Rom Hermite), `generateBezierPath` (xuất `M…C…` SVG path), `generateDefaultControlPoints` (sinh control point từ tiếp tuyến `(next-prev)/4`), `keyframesToMotionPath` / `motionPathToKeyframes` (chuyển đổi 2 chiều), `sampleFrameTransforms(clipId, start, end, fps)` (sample theo từng frame để export). GSAP easing map: `easingToGSAP(EasingType)` → tên GSAP (`power1..4.in/out/inOut`, `sine`, `expo`, `circ`, `back`, `elastic`, `bounce`).

### 2.5 `CompositionRenderer` (`animation/composition-renderer.ts`)
Renderer độc lập cho định dạng "composition/animation schema" (`animation/animation-schema.ts`) — render layer (shape/text/image/video) lên `OffscreenCanvas` tại thời điểm t, evaluate `PropertyKeyframes` (dùng `EASING_FUNCTIONS`, hỗ trợ tên easing kebab-case), áp transform (`translate→rotate→scale→-anchor`), blend mode → `transferToImageBitmap()`. Đây là đường render dành cho template/animation import-export, song song với pipeline editor chính. `AnimationImporter`/`AnimationExporter` chuyển JSON ↔ schema; `substituteVariables` thay `{{var}}` cho template; `validateAnimationSchema` kiểm tra.

---

## 3. `VideoEffectsEngine` (effect cấp clip)

`packages/core/src/video/video-effects-engine.ts`. `applyEffects(image: ImageBitmap, effects: Effect[]) → {image, processingTime, gpuAccelerated}`. Áp các effect theo thứ tự trong mảng (effect order = thứ tự áp).

Hiện trạng thực thi: **dùng CPU Canvas2D** (`applyEffectsCPU`) — ổn định nhất. Có sẵn 2 path khác (`_applyEffectsGPU` dùng WebGL2 ping-pong framebuffer; `_applyEffectsWithNewRenderer` dùng `RendererFactory`/WebGPU) nhưng tạm tắt vì lỗi render. `applyEffectsCPU` chia 2 nhóm:
- **CSS filter** (browser tăng tốc, gộp 1 `drawImage`): `brightness` → `brightness(1 + v/100)`, `contrast` → `contrast(v)`, `saturation` → `saturate(v)`, `hue` → `hue-rotate(rot deg)`, `blur` → `blur(radius px)`.
- **Pixel-level** (`getImageData`/`putImageData`): `sharpen` (kernel 3×3 `[0,-a,0; -a,1+4a,-a; 0,-a,0]`, `a=amount/100`), `vignette` (smoothstep theo bán kính chuẩn hoá, `factor = 1 - vignette*amount/100`), `grain` (cộng nhiễu `(rand-0.5)*amount/100*50`), `chromaKey` (alpha = smoothstep theo khoảng cách RGB tới keyColor, ngưỡng `tolerance*441.67`, mềm `softness*441.67`), `temperature` (warm: +R+G−B; cool: ngược), `tint` (+R−G+B), `tonal` (weight theo luma: shadow `1−smoothstep(0,0.33,luma)`, highlight `smoothstep(0.66,1,luma)`, midtone phần còn lại; `adj = Σ value*weight*0.3`).

WebGL2 shaders (mã GLSL ES 300 trong file) gồm: passthrough, brightness, contrast, saturation, hue (RGB↔HSV), blur (box 2r+1 kernel, r≤10), sharpen (cross kernel), vignette (smoothstep), grain (`fract(sin(dot(...))*43758.5453)`), chromaKey, temperature, tint, tonal. WebGPU compute (`shaders/effects.wgsl`) chain brightness→contrast→saturation→hue→temperature→tint→shadows/highlights trong 1 pass (workgroup 16×16), RGB↔HSV chuẩn.

`FilterType` hỗ trợ: `brightness | contrast | saturation | hue | blur | sharpen | vignette | grain | chromaKey | temperature | tint | tonal`.

Định nghĩa effect "layer-level" đầy đủ (cho UI param panel) tại `types/effects.ts` — `EFFECT_DEFINITIONS`: blur (Gaussian, radius 0–100px), shadow (offsetX/Y, blur, opacity, RGB), glow (radius, intensity 0–3), brightness/contrast/saturation (−100..100%), hue-saturation (hue −180..180, sat/light −100..100), color-balance (C/R, M/G, Y/B cho 3 dải), curves (blackPoint/whitePoint/gamma), motion-blur (angle 0–360, distance 0–100px), radial-blur (amount, centerX/Y %), vignette (amount/size/roundness/feather), film-grain (amount/size/roughness), chromatic-aberration (amount 0–50px, angle). Mỗi param có `min/max/step/unit/default`.

`FILTER_PRESETS` (`video/filter-presets.ts`): 18 preset gộp sẵn nhiều effect — 5 nhóm `cinematic` (Teal & Orange, Film Noir, Blockbuster), `vintage` (70s Retro, Polaroid, VHS, Sepia), `mood` (Dreamy, Moody, Golden Hour, Cold Blue), `color` (Vibrant, Muted, B&W Classic, B&W High Contrast), `stylized` (Cyberpunk, Comic Book, Soft Glow). Ví dụ "Film Noir" = `saturation 0` + `contrast 1.4` + `brightness −0.05` + `vignette{amount 0.4, midpoint 0.4, feather 0.6}` + `grain{amount 0.15, size 1.5}`.

---

## 4. Module `text` (chữ & phụ đề)

| File | Vai trò |
|---|---|
| `title-engine.ts` | `TitleEngine` (~25k LOC): tạo/sửa/render text clip — font family/size/weight/style, fill (solid/gradient), stroke (cap/join), shadow, letterSpacing, lineHeight, textAlign/verticalAlign, textTransform, padding/background; tích hợp `emphasisAnimation`. Render lên `OffscreenCanvas`. |
| `text-animation.ts` | `TextAnimationEngine.getAnimatedState(clip, time)` — animation cấp **toàn clip** (transform/opacity/style/visibleText + tuỳ chọn `characterStates[]`). Ưu tiên `clip.keyframes` nếu có; nếu không, áp `clip.animation.preset` với `inDuration`/`outDuration`. 19 preset. |
| `text-animation-presets.ts` | `calculateUnitAnimationState(ctx)` — animation cấp **đơn vị** (character / word / line) với `stagger` (lệch pha giữa đơn vị); `TEXT_ANIMATION_PRESETS[]` metadata mỗi preset (name, description, category entrance/emphasis/exit/continuous, defaultParams, defaultUnit, defaultStagger, defaultInDuration, defaultOutDuration). |
| `character-animator.ts` | `CharacterAnimator.measureText()` (đo layout char/word/line bằng `ctx.measureText`, có fallback `0.6*fontSize`) + `calculateAnimatedLayout(clip, time)` → `AnimatedTextLayout` (mỗi line/word/char kèm `UnitAnimationState`). Tính `progress` + `isIn` theo phase in/middle/out. |
| `caption-animation-renderer.ts` | `renderAnimatedCaption(subtitle, currentTime) → {segments: WordSegment[], visible}` — 6 style phụ đề: none / word-highlight / word-by-word / karaoke / bounce / typewriter. Dùng `subtitle.words[]` (per-word timing). |
| `subtitle-engine.ts` | `SubtitleEngine`: parse SRT, layout, style, vị trí top/center/bottom. |
| `audio-text-sync-engine.ts` | Gán timing chữ theo audio. |
| `speech-to-text-engine.ts` / `transcription-service.ts` | STT để sinh phụ đề tự động (per-word timing). |
| `types.ts` | `TextClip`, `TextStyle`, `TextAnimation`, `TextAnimationPreset`, `TextAnimationParams`. |

`TextAnimation`: `{ preset, params, inDuration, outDuration, stagger?, unit?: 'character'|'word'|'line' }`.

---

## 5. Module `graphics` (hình vẽ, SVG, sticker, emoji)

`packages/core/src/graphics/graphics-engine.ts` (`GraphicsEngine`, ~56k LOC):
- **Shape**: rectangle (cornerRadius), circle, ellipse, triangle, arrow (headWidth/headLength/tailWidth, curved, doubleHeaded), line, polygon (sides), star (points + innerRadius ratio). Style: `FillStyle` (solid/gradient linear-radial/none + opacity), `StrokeStyle` (color/width/opacity/dashArray/dashOffset/lineCap/lineJoin), `ShadowStyle`.
- **SVG**: import (`SVGImportResult` — content, viewBox, width, height), color tint (`SVGColorStyle`: `colorMode 'none'|'tint'|'replace'`, tintColor, tintOpacity), `entryAnimation` + `exitAnimation` (`GraphicAnimation{type, duration, easing}`), `emphasisAnimation`. `applyGraphicAnimation(type, progress, easing, isEntry)` trả `{opacity, scale, offsetX, offsetY, rotation, blur}` — 20 loại entry/exit (xem ANIMATIONS.md §7).
- **Sticker / emoji**: `StickerClip` với `imageUrl`. `sticker-library.ts` = thư viện built-in.
- `svg-animation-presets.ts`: `SVG_ANIMATION_PRESETS[]` metadata (name, description, category, defaultDuration, defaultEasing — một số dùng `cubic-bezier(...)`).
- `DEFAULT_GRAPHIC_TRANSFORM` dùng `position: {0.5, 0.5}` normalized.

---

## 6. Module `effects`

| File | Vai trò |
|---|---|
| `particle-engine.ts` | `ParticleEngine`: emitter-based particle system (~thuần CPU). `addEffect(ParticleEffect)`, `update(currentTime, deltaTime)` (Euler integration: velocity += (accel + wind ± turbulence)·dt; position += velocity·dt; rotation += rotationSpeed·dt; age += dt; opacity theo fadeIn/fadeOut), `getParticles(effectId?)`. Toạ độ tâm emit = giữa canvas. |
| `particle-presets.ts` | 12 preset (`PARTICLE_PRESETS`) + `createEffectFromPreset()`. |
| `particle-types.ts` | `ParticleEffectType` (10 loại: dissolve/explode/implode/confetti/dust/sparkle/disintegrate/pixelate/shatter/morph), `ParticleConfig`, `Particle`, `EmitterState`, `DEFAULT_PARTICLE_CONFIG`. |
| `blend-modes.ts` | `BlendModeEngine`: map 14 blend mode sang `globalCompositeOperation` Canvas2D + cung cấp GLSL `blend(base, blend)` cho shader. |
| `expression-engine.ts` | `ExpressionEngine`: eval biểu thức kiểu After Effects — `wiggle(freq,amp)`, `smooth()`, `linear/ease/easeIn/easeOut(t,tMin,tMax,v1,v2)`, `clamp`, `random`, `noise` (Perlin 1D). Compile bằng `new Function` với context an toàn (chỉ `Math` + helper), có cache. Dùng để drive property bằng code thay vì keyframe. |

---

## 7. Module `audio`

`packages/core/src/audio`: `AudioEngine` (Web Audio graph per clip: source → gain (volume + fade + automation) → pan → effect chain → master), effect chain: EQ (biquad nhiều band: lowshelf/highshelf/peaking/lowpass/highpass/notch — freq 20–20kHz, gain ±24dB, Q 0.1–18), compressor (threshold/ratio/attack/release/knee/makeupGain), reverb (convolution: roomSize/damping/wet/dry/preDelay), delay (time/feedback/wet/sync), chorus/flanger/distortion, noise reduction (3-pass: tonal/broadband/rumble). Beat detection (WASM FFT — `wasm/beat-detection`, `wasm/fft`): BPM + downbeat → `beatMarkers`. Audio ducking (giảm nhạc khi có thoại). Volume/pan automation = mảng `AutomationPoint{time, value}`, nội suy tuyến tính giữa các điểm. Waveform visualization. Audio export → MP3/WAV/AAC/FLAC/OGG.

---

## 8. WASM modules (`packages/core/src/wasm`)

AssemblyScript build sang WASM:
- `fft/` — FFT cho phân tích phổ.
- `beat-detection/` — onset detection / tempo estimation.
- `wav/` — đọc/ghi WAV PCM.

---

## 9. Templates & AI integrations

### 9.1 Templates
`packages/core/src/template` — engine template (project mẫu có biến chỉnh sửa, `editableVariables`). `apps/web/src/services/template-cloud-service.ts` = thư viện template cloud. `animation/animation-schema.ts` + `AnimationImporter/Exporter` = định dạng JSON cho template/animation.

### 9.2 AI on-device (chạy local trong trình duyệt, không gửi dữ liệu ra ngoài)
`packages/core/src/ai/`:
| File | Engine | Mô tả |
|---|---|---|
| `background-removal-engine.ts` | `BackgroundRemovalEngine` | Xoá nền video/ảnh — mask người/đối tượng → alpha matting. Chạy mô hình segmentation trên GPU/WebGL/WebGPU. |
| `person-segmentation-engine.ts` | `PersonSegmentationEngine` | Tách người khỏi nền (selfie segmentation) → mask alpha per-frame, dùng cho background removal & blur nền. |
| `auto-reframe-engine.ts` | `AutoReframeEngine` | Tự đổi tỉ lệ khung hình (vd 16:9 → 9:16) bám theo chủ thể/khuôn mặt — phát hiện vùng quan trọng → keyframe `crop`/`position` theo thời gian. UI: `AutoReframeSection.tsx`. |

Liên quan (không nằm trong `ai/` nhưng cùng nhóm "thông minh"): `video/upscaling/` (`UpscalingEngine` — AI upscaling bằng WebGPU compute shaders, quality proxy/2x/4x), `video/motion-tracking-engine.ts` (template matching), beat detection + silence-cut + auto-highlight (`apps/web/src/services/highlight-service.ts`, WASM FFT).

### 9.3 AI dịch vụ ngoài (cloud — cần API key người dùng tự cấu hình)

**KieAI — image generation** (`apps/web/src/services/kieai/`):
- Endpoint: `https://api.kie.ai` (generation) + `https://kieai.redpandaai.co` (file upload). File upload tự xoá sau 3 ngày.
- Client: `client.ts` (`kieaiPostJson` / `kieaiGet` / `kieaiPostForm`, Bearer token); key lấy từ `secure-storage` id `"kie-ai"`. Lỗi → `KieAIError{code,msg}`.
- 6 model image (`image-generation.ts` `IMAGE_MODELS`):

  | Constant | Model id | Loại |
  |---|---|---|
  | `SEEDREAM` | `seedream/5-lite-image-to-image` | image→image (basic=2K, high=4K, max 14 ảnh tham chiếu) |
  | `Z_IMAGE` | `z-image` | text→image |
  | `NANO_BANANA2` | `nano-banana-2` | text→image + tham chiếu (1K/2K/4K, png/jpg) |
  | `FLUX2` | `flux-2/pro-image-to-image` | image→image |
  | `GROK` | `grok-imagine/image-to-image` | image→image |
  | `QWEN` | `qwen/image-to-image` | image→image |

- Flow: `uploadFileStream(file)` → `fileUrl` → `createImageTask(model, input)` → `taskId` → `pollTask(taskId)` (poll loop) → `getResultUrl(record)`. Kết quả là ảnh → thêm vào media library.
- UI: `components/editor/kieai/KieAIImageDialog.tsx`, `ModelPicker.tsx`, `forms/{Flux2,NanoBanana2,Qwen,Seedream,ZImage,Grok}Form.tsx`, tab `components/editor/AIGenTab.tsx`; state: `stores/kieai-store.ts` (key, task status, history).

**Text-to-Speech (ElevenLabs)** — `components/editor/inspector/{TextToSpeechPanel,VoiceBrowser,ModelSelector}.tsx`, hooks `useElevenLabsApi.ts` / `useTtsActions.ts`, constants/types `tts-constants.ts`/`tts-types.ts`, store `stores/tts-store.ts`. Sinh audio từ text → thêm clip audio.

**Transcription / Auto-caption (STT)** — `packages/core/src/text/speech-to-text-engine.ts` + `transcription-service.ts`; dịch vụ OpenReel `https://transcribe.openreel.video` và `https://cloud.openreel.video` (bản GPU); UI `components/editor/inspector/AutoCaptionPanel.tsx`. Trả về phụ đề kèm **per-word timing** → dùng cho caption animation (karaoke/word-highlight...) ở [`ANIMATIONS.md` §8].

**OpenReel cloud** (`config/api-endpoints.ts`) — `OPENREEL_CLOUD_URL` (Cloudflare Workers, `openreel-cloud.*.workers.dev`): template cloud, project share (`services/share-service.ts`, `template-cloud-service.ts`).

### 9.4 Hạ tầng & bảo mật khoá
- `apps/web/src/services/secure-storage.ts` — IndexedDB mã hoá AES-256, mở khoá bằng master password; lưu API key (`getSecret(id)`).
- `apps/web/src/services/api-proxy.ts` (`apiFetch`) — dev: gọi thẳng dịch vụ ngoài; production: đi qua Cloudflare Pages Function proxy (`apps/web/functions/api/proxy/`). App code nên gọi `apiFetch()` thay vì hard-code URL.
- `config/api-endpoints.ts` tập trung mọi endpoint → swap được để self-host.
- `services/background-generator.ts` / `processing-manager.ts` — hàng đợi tác vụ nền (upscaling, background removal, AI gen...) với progress callback, không block main thread.

### 9.5 "AI-managed development" (không phải tính năng runtime)
README mô tả dự án dùng Claude AI cho issue triage / viết feature / code review / cập nhật docs — đây là **quy trình phát triển**, không có gì trong app build ra. Không nhầm với các tích hợp AI runtime ở trên.

> **Lưu ý**: README liệt kê **"Plugin system"** trong roadmap *In Progress* — hiện **chưa có** kiến trúc plugin mở rộng cho bên thứ ba. "Tích hợp AI" ≠ "plugin".

---

## 10. Build / chạy

```bash
pnpm install            # Node 18+
pnpm dev                # Vite dev server (apps/web) → http://localhost:5173
pnpm build && pnpm preview
pnpm typecheck && pnpm test && pnpm lint
```
Yêu cầu trình duyệt: Chrome/Edge 94+, Firefox 130+, Safari 16.4+ (WebCodecs). Khuyến nghị GPU rời cho 4K. Conventional commits; CI chạy typecheck/test/lint.
