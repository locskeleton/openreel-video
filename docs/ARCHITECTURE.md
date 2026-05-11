# OpenReel Video — Tài liệu thiết kế kiến trúc

> Phiên bản tài liệu: 1.0 — viết từ source tại commit hiện tại của `Augani/openreel-video`.
> Tài liệu này mô tả kiến trúc tổng thể. Chi tiết kỹ thuật từng engine xem [`TECHNICAL.md`](./TECHNICAL.md). Danh mục đầy đủ các hiệu ứng animation xem [`ANIMATIONS.md`](./ANIMATIONS.md).

---

## 1. Tóm tắt

OpenReel Video là một **trình biên tập video chạy hoàn toàn trên trình duyệt (client-side)** — không upload, không server xử lý. Toàn bộ decode/encode/compositing/effects chạy bằng `WebCodecs`, `WebGPU`/`WebGL2`, `Web Audio API`, `OffscreenCanvas` và `Web Workers`. Dữ liệu project lưu local bằng `IndexedDB`.

Đặc tính kiến trúc cốt lõi:

| Nguyên tắc | Hiện thực |
|---|---|
| **Action-based editing** | Mọi thao tác sửa = một `Action` có thể undo/redo (`packages/core/src/actions`) |
| **Immutable state** | Toàn bộ data model timeline là `readonly`; cập nhật bằng spread/clone; UI state quản lý bằng Zustand |
| **Engine separation** | Video / Audio / Graphics / Text / Export là các engine độc lập trong `packages/core` |
| **Progressive enhancement** | WebGPU → WebGL2 → Canvas2D fallback tự động và im lặng |
| **Bridge layer** | UI (React) không gọi engine trực tiếp — đi qua tầng `bridges` để đồng bộ state ↔ engine |

---

## 2. Cấu trúc monorepo

Quản lý bằng `pnpm` workspace (`pnpm-workspace.yaml`), TypeScript base config tại `tsconfig.base.json`.

```
openreel/
├── apps/
│   ├── web/            # Trình biên tập VIDEO (React 18 + Vite)  — ~66k LOC
│   └── image/          # Trình biên tập ẢNH kiểu Photoshop (React + Vite)
├── packages/
│   ├── core/           # Toàn bộ engine xử lý video/audio/graphics/text/export — ~59k LOC
│   ├── image-core/     # Data model + operations cho trình biên tập ảnh
│   └── ui/             # Component UI dùng chung (shadcn/ui + Tailwind)
├── infra/              # Hạ tầng deploy (Cloudflare Pages / functions)
└── mediabunny.d.ts     # Type definitions cho thư viện MediaBunny (media I/O)
```

Tài liệu này tập trung vào `apps/web` + `packages/core` (sản phẩm chính). `apps/image`/`packages/image-core` là một editor ảnh tách biệt (layers, adjustment layers, retouch tools, liquify, background removal) — chỉ nhắc qua.

---

## 3. Sơ đồ tầng (layered view)

```
┌─────────────────────────────────────────────────────────────────────┐
│  PRESENTATION  (apps/web/src/components, apps/web/src/pages)         │
│  React 18 + Tailwind + shadcn/ui                                     │
│  EditorInterface · Timeline · Preview · InspectorPanel · Toolbar     │
│  AssetsPanel · KeyframeEditorPanel · AudioMixer · ExportDialog       │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ (đọc state qua hooks, gọi action qua store)
┌──────────────────────────────▼──────────────────────────────────────┐
│  STATE  (apps/web/src/stores) — Zustand                             │
│  project-store (Action/History) · timeline-store (playhead/zoom)     │
│  engine-store (lazy engines + currentFrame) · ui-store · settings    │
│  recorder-store · kieai-store · tts-store · notification-store       │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│  COORDINATION  (apps/web/src/bridges) — singleton bridges           │
│  playback-bridge · render-bridge · effects-bridge · media-bridge     │
│  text-bridge · graphics-bridge · photo-bridge · transition-bridge    │
│  audio-bridge (+effects) · beat-sync · silence-cut · motion-tracking │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│  CORE ENGINES  (packages/core/src)                                  │
│  video/    VideoEngine, PlaybackEngine, Composite/GPU compositor,    │
│            VideoEffectsEngine, ColorGradingEngine, TransitionEngine, │
│            KeyframeEngine, TransformAnimator, SpeedEngine, Mask,     │
│            Stabilization, FrameInterpolation, Upscaling, MotionTrack │
│  audio/    AudioEngine, effects (EQ/compressor/reverb/...), beat det.│
│  graphics/ GraphicsEngine (shapes/SVG/sticker/emoji), animation prst │
│  text/     TitleEngine, SubtitleEngine, TextAnimationEngine,         │
│            CharacterAnimator, CaptionAnimationRenderer, STT          │
│  animation/ EasingFunctions, AnimationEngine, GSAPEngine (motion path)│
│            CompositionRenderer, AnimationImporter/Exporter, schema   │
│  effects/  ParticleEngine, BlendModeEngine, ExpressionEngine         │
│  export/   ExportEngine + WebCodecs encoders + MediaBunny mux        │
│  storage/  IndexedDB serialization, project persistence              │
│  actions/  ActionExecutor, ActionHistory, validators, inverse gen    │
│  timeline/ data model: Timeline → Track → Clip → Effect/Keyframe     │
│  wasm/     AssemblyScript modules: FFT, beat-detection, WAV          │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│  PLATFORM APIs                                                       │
│  WebCodecs · WebGPU · WebGL2 · Web Audio · OffscreenCanvas ·         │
│  Web Workers · IndexedDB · File System Access · MediaRecorder        │
│  3rd party: MediaBunny (media I/O), GSAP (motion path), THREE.js     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Data model timeline (nguồn gốc của mọi thứ)

Định nghĩa tại `packages/core/src/types/timeline.ts`. Tất cả `readonly` (immutable).

```
Project
└── Timeline
    ├── tracks: Track[]
    │   ├── type: 'video' | 'audio' | 'image' | 'text' | 'graphics'
    │   ├── locked / hidden / muted / solo: boolean
    │   ├── transitions: Transition[]          # giữa các clip kề nhau cùng track
    │   └── clips: Clip[]
    │       ├── mediaId, startTime, duration, inPoint, outPoint   # trim
    │       ├── transform: Transform           # position/scale/rotation/anchor/opacity/3D/crop/borderRadius/fitMode
    │       ├── effects: Effect[]              # video effects (blur/saturation/chromaKey/...)
    │       ├── audioEffects: Effect[]         # EQ/compressor/reverb/...
    │       ├── keyframes: Keyframe[]          # animate BẤT KỲ property nào theo thời gian
    │       ├── blendMode?, blendOpacity?
    │       ├── volume, fade?{fadeIn,fadeOut}, automation?{volume[],pan[]}
    │       ├── speed?, reversed?, smoothSlowMo?, interpolationQuality?
    │       ├── stabilization?{enabled,strength,cropMode,profile}
    │       ├── emphasisAnimation?             # animation nhấn mạnh lặp lại (pulse/shake/ken-burns/...)
    │       └── audioTrackIndex?
    ├── subtitles: Subtitle[]                  # text + per-word timing + animationStyle (karaoke/...)
    ├── markers: Marker[]
    └── beatMarkers?, beatAnalysis?{bpm,confidence}
```

Các loại clip "phái sinh" không nằm trực tiếp trong `Clip`:
- **Text clip** (`packages/core/src/text/types.ts`): có `text`, `style`, `animation` (`TextAnimation`), `keyframes`. Render bởi `TitleEngine`.
- **Graphic clip** (`packages/core/src/graphics/types.ts`): `ShapeClip` / `SVGClip` / `StickerClip` (emoji). `SVGClip` có `entryAnimation` + `exitAnimation` (`GraphicAnimation`), cộng `emphasisAnimation`.
- **Particle effect** (`packages/core/src/effects/particle-types.ts`): `ParticleEffect` gắn vào một `clipId`, có `startTime`, `duration`, `config`.

`Transform` đầy đủ:
```ts
interface Transform {
  position: {x,y}; scale: {x,y}; rotation: number; anchor: {x,y}; opacity: number;
  borderRadius?: number; fitMode?: 'contain'|'cover'|'stretch'|'none';
  rotate3d?: {x,y,z}; perspective?: number; transformStyle?: 'flat'|'preserve-3d';
  crop?: {x,y,width,height};
}
```
> Lưu ý hệ toạ độ: với video/image clip, `position` thường tính theo **pixel của canvas project**; với graphic clip, `DEFAULT_GRAPHIC_TRANSFORM` dùng `position: {0.5, 0.5}` **normalized 0–1**. Anchor luôn normalized 0–1 (0.5,0.5 = giữa). Trong text-animation, một số preset dùng "normalized distance" cho offset (ví dụ `slideDistance: 0.2` = 20% kích thước).

---

## 5. Hệ thống Action / Undo-Redo

`packages/core/src/actions`. Mọi chỉnh sửa project đi qua `ActionExecutor`:

1. `execute(action, project)` → `ActionValidator` kiểm tra hợp lệ → clone snapshot project → `ActionInverseGenerator` sinh action nghịch đảo → áp mutation → push `{action, inverseAction, timestamp, description, groupId?}` vào `ActionHistory.undoStack`, clear `redoStack`.
2. `undo(project)` → pop `undoStack`, push sang `redoStack`, áp `inverseAction`.
3. `redo(project)` → ngược lại.
4. `groupId` cho phép gộp nhiều action thành một bước undo (ví dụ kéo nhiều clip).
5. `ActionSerializer` chuyển action sang JSON để lưu/khôi phục cùng project.

Họ action: `TrackAction`, `ClipAction`, `EffectAction`, `TransformAction`, `KeyframeAction`, `TransitionAction`, `AudioAction`, `SubtitleAction`, `MediaAction`, `ProjectAction`. (Trình biên tập ảnh `apps/image` dùng một cơ chế Command riêng — `packages/image-core/src/commands.ts`.)

Store `project-store.ts` (Zustand) bọc `ActionExecutor` + `ActionHistory`, expose các method như `addClip`, `moveClip`, `splitClip`, `addKeyframe`, `addEffect`... và tích hợp auto-save.

---

## 6. State management (Zustand stores)

| Store | Trách nhiệm |
|---|---|
| `project-store` | Project + media library + tracks/clips + text/shape/subtitle/photo CRUD + effects/keyframes; chứa `ActionExecutor`/`ActionHistory`; khởi tạo auto-save |
| `timeline-store` | Trạng thái playback & view: `playheadPosition`, `playbackState`, `playbackRate`, `pixelsPerSecond` (zoom), `scrollX/Y`, viewport, track heights, loop range, scrubbing; helper `timeToPixels`/`pixelsToTime`/`getVisibleTimeRange` |
| `engine-store` | Lazy-init tất cả engine (`VideoEngine`, `AudioEngine`, `PlaybackController`, `TitleEngine`, `SubtitleEngine`, `GraphicsEngine`, `PhotoEngine`, `ExportEngine`, `SpeechToTextEngine`, `TemplateEngine`, `ChromaKeyEngine`, `MultiCamEngine`, `MaskEngine`, `AdjustmentLayerEngine`...); lưu `currentFrame` (`RenderedFrame`) + playback stats |
| `ui-store` | Modal/panel đang mở, dialog settings, shortcuts panel |
| `recorder-store` | Trạng thái screen recording |
| `settings-store` / `theme-store` | Theme, autosave interval, keybindings, API keys |
| `kieai-store` / `tts-store` | Tích hợp dịch vụ AI generation (KieAI) và text-to-speech (ElevenLabs) |
| `notification-store` | Hàng đợi toast |

---

## 7. Bridge layer (cầu nối UI ↔ engine)

`apps/web/src/bridges`. Mỗi bridge là singleton với `getXxxBridge()` / `initializeXxxBridge()` / `disposeXxxBridge()`. `EditorInterface.tsx` khởi tạo tất cả bridge khi mount. Bridge dịch state-change của store thành lệnh engine và đẩy event engine ngược lại UI.

| Bridge | Engine wrap / nhiệm vụ |
|---|---|
| `playback-bridge` | Đồng bộ play/pause/stop/seek với `PlaybackController`; logic mute/solo audibility; chế độ scrub; playback rate |
| `render-bridge` | Pipeline render frame: gọi `VideoEngine`, frame cache LRU, áp color grading/adjustment layer, phát hiện & render transition; vẽ frame lên canvas; debounce khi scrub |
| `effects-bridge` | Wrap `VideoEffectsEngine` + `ColorGradingEngine`; quản lý effect cấp clip & color grading; serialize/deserialize effects; sinh scope (waveform/vectorscope/histogram) |
| `media-bridge` | Wrap `MediaImportService` + `WaveformGenerator`: import file, thumbnail, waveform, metadata |
| `text-bridge` | Wrap `TitleEngine` + `TextAnimationEngine`: tạo/sửa text clip, áp text animation preset với in/out timing |
| `graphics-bridge` | Wrap `GraphicsEngine`: shapes, SVG (+entry/exit animation), stickers, emoji |
| `photo-bridge` | Wrap `PhotoEngine`: photo layers, retouch (clone stamp, healing, dodge/burn, liquify) |
| `transition-bridge` | Quản lý transition giữa clip, áp params + duration, gọi `TransitionEngine.renderTransition` |
| `audio-bridge` / `audio-bridge-effects` | Volume/pan automation (nội suy theo thời gian), EQ/compressor/reverb/delay/noise reduction |
| `audio-text-sync-bridge` | Đồng bộ audio ↔ caption timing |
| `beat-sync-bridge` | Phân tích beat (BPM) và sync animation/effect theo beat marker |
| `silence-cut-bridge` | Phát hiện & cắt vùng im lặng theo threshold/duration |
| `motion-tracking-bridge` | Track đối tượng/người trong clip để gắn effect |

---

## 8. Pipeline render frame (preview)

Trace từ thời gian timeline → ảnh hiển thị. Các file chính trong `packages/core/src/video`.

```
playhead time (s)
  │
  ▼  PlaybackController (master clock) ──timeupdate──▶ playback-bridge ──▶ render-bridge.renderFrame(t)
  │
  ▼  VideoEngine.renderFrame(project, t)   (video-engine.ts — orchestrator ~2.4k LOC)
  │   ├─ ParallelFrameDecoder: decode frame từ clip media (WebCodecs, qua MediaBunny VideoSampleSink)
  │   │     · ảnh tĩnh → createImageBitmap trực tiếp
  │   │     · video → VideoSampleSink.getSample(localTime), localTime tính từ inPoint/outPoint + speed
  │   ├─ SpeedEngine / FrameInterpolationEngine: nếu clip có speed/slow-mo → nội suy frame (optical flow / blend)
  │   ├─ VidStab (stabilization/): nếu clip.stabilization.enabled → áp ma trận bù rung + crop
  │   ├─ Per-clip TransformAnimator/KeyframeEngine: lấy Transform tại thời điểm t (áp keyframes)
  │   ├─ emphasisAnimation: nếu có → cộng thêm scale/offset/rotation/opacity (xem ANIMATIONS.md §8)
  │   ├─ VideoEffectsEngine.applyEffects(image, clip.effects): blur/sharpen/vignette/grain/chromaKey/...
  │   │     (hiện chạy CPU Canvas2D — CSS filter cho effect đơn giản, pixel-loop cho effect phức tạp;
  │   │      WebGL2 + WebGPU paths có sẵn nhưng tạm dùng CPU vì ổn định hơn)
  │   ├─ ColorGradingEngine: color wheels (lift/gamma/gain), curves, 3D LUT, HSL 8-vùng
  │   ├─ Text clip → TitleEngine.render (font/stroke/shadow/gradient + TextAnimationEngine/CharacterAnimator)
  │   ├─ Graphic clip → GraphicsEngine.render (shape path / SVG / sticker; áp entry/exit GraphicAnimation)
  │   └─ Subtitle → SubtitleEngine + CaptionAnimationRenderer (karaoke/word-highlight/...)
  │
  ▼  CompositeEngine / GPUCompositor: xếp tất cả layer theo z-index, áp blendMode + opacity
  │     · RendererFactory chọn WebGPURenderer (webgpu-renderer-impl.ts + .wgsl shaders)
  │       hoặc Canvas2DFallbackRenderer (canvas2d-fallback-renderer.ts)
  │
  ▼  Transition (transition-engine.ts): nếu t nằm trong vùng transition giữa 2 clip → blend frame ra/vào
  │
  ▼  Frame cache LRU (frame-cache.ts, ~500MB / ~100 frame) → RenderedFrame {image: ImageBitmap, width, height}
  │
  ▼  render-bridge vẽ lên <canvas> trong Preview.tsx; playback-bridge cập nhật playheadPosition trong timeline-store
```

`renderer-factory.ts` định nghĩa interface `Renderer` (`initialize`, `createTextureFromImage`, `renderLayer`, `applyEffects`, `beginFrame/endFrame`, `resize`, `destroy`). WebGPU dùng compute shader `shaders/effects.wgsl` (brightness/contrast/saturation/hue/temperature/tint/shadows-highlights chained 1-pass, workgroup 16×16) và vertex/fragment `shaders/transform.wgsl` (ma trận 4×4 + bilinear). `shaders/blur.wgsl`, `border-radius.wgsl`, `composite.wgsl` lo blur Gaussian / bo góc / blend.

`frame-ring-buffer.ts` (`CompositeFrameBuffer`) tránh allocate liên tục khi playback. `texture-cache.ts` cache GPU texture. `decode-worker.ts` / `parallel-frame-decoder.ts` decode trong Web Worker.

---

## 9. Pipeline export

`packages/core/src/export`. `ExportEngine.exportVideo(project, settings)` chạy tuần tự:

1. **preparing** — validate settings, dò codec hỗ trợ (qua MediaBunny), khởi tạo encoder.
2. **rendering** — vòng lặp `for t = 0 → duration step 1/fps`: gọi `VideoEngine.renderFrame(project, t)` → `VideoFrame`. (Tuỳ chọn upscaling qua `video/upscaling` — WebGPU shader, quality proxy/2x/4x.)
3. **encoding** — `VideoEncoder` (WebCodecs) H.264/H.265/VP8/VP9/AV1/ProRes; song song encode audio (`AudioEncoder` → MP3/WAV/AAC/FLAC/OGG) từ AudioEngine render-to-buffer.
4. **muxing** — MediaBunny mux video+audio thành MP4/WebM/MOV; hoặc xuất image sequence (JPG/PNG/WebP).
5. **complete** — trả `Blob` + `ExportStats`; `downloadBlob()`.

Cấu hình tại `export/types.ts`: `VideoExportSettings` (format/codec/width/height/frameRate/bitrate/quality/colorDepth), `AudioExportSettings`, `ImageExportSettings`, `SequenceExportSettings`, `UpscalingSettings`. `VIDEO_QUALITY_PRESETS` có 4K@60/1080p/720p/480p. Encode chạy trong `export-worker.ts` để không block main thread; có progress callback + cancel.

---

## 10. Persistence & recovery

- `packages/core/src/storage`: serialize/deserialize toàn bộ project (gồm history actions) sang JSON; lưu vào `IndexedDB`.
- `apps/web/src/services/auto-save.ts` (`AutoSaveManager`): snapshot mỗi ~30s, 3 slot xoay vòng, debounce dirty ~2s; event `saved`/`restored`/`error`/`recoveryAvailable`.
- `apps/web/src/services/media-storage.ts`: lưu blob media vào IndexedDB; hỗ trợ File System Access API (`loadFileHandle`/`loadDirectoryHandle`) để gắn lại file gốc.
- `apps/web/src/services/secure-storage.ts`: IndexedDB mã hoá (AES-256, master password) cho API key (KieAI, ElevenLabs).
- `apps/web/src/services/service-worker.ts`: PWA offline.

---

## 11. Entry point & routing

- `apps/web/src/main.tsx` — khởi tạo analytics (PostHog), đăng ký service worker, mount `<App/>`.
- `apps/web/src/App.tsx` — router dựa trên URL (`hooks/use-router.ts`, không reload trang): `WelcomeScreen` (tạo project / template / recent) ↔ `EditorInterface` (editor chính) ↔ `pages/SharePage` (xem project chia sẻ, read-only) ↔ `RecoveryDialog` (khôi phục auto-save) ↔ `SearchModal` (Cmd+K).
- URL params: `route`, `preset`, `dimensions`, `shareId`, `fps`.
- `EditorInterface.tsx` — layout chính: `Toolbar` (top) · `AssetsPanel` (trái) · `Preview` (giữa) · `InspectorPanel` + `KeyframeEditorPanel` (phải) · `Timeline` (dưới) · `AudioMixer` (toggle). Khởi tạo toàn bộ bridge ở `useEffect`.

`InspectorPanel` là panel theo ngữ cảnh: ~40+ section con tuỳ loại clip — `VideoEffectsSection`, `ColorGradingSection`, `TextSection`, `TextAnimationSection`, `Transform3DSection`, `KeyframesSection`, `CropSection`, `MaskSection`, `SpeedSection`, `MotionTrackingSection`, `BeatSyncSection`, `AutoReframeSection`, `EmphasisAnimationSection`, `NestedSequenceSection`, `AudioEffectsSection`, ...

---

## 12. Công nghệ & thư viện

| Thành phần | Công nghệ |
|---|---|
| UI | React 18, TypeScript, Vite, TailwindCSS, shadcn/ui (Radix), Zustand |
| Media I/O | **MediaBunny** (`mediabunny.d.ts`) — đọc/ghi/mux container, dò codec |
| Decode/Encode | **WebCodecs** (`VideoDecoder`/`VideoEncoder`/`AudioEncoder`) |
| GPU compositing & effects | **WebGPU** (compute + render, WGSL shaders) → fallback **WebGL2** → fallback **Canvas2D** |
| Audio | **Web Audio API** + AudioWorklet; WASM (AssemblyScript) cho FFT / beat detection / WAV |
| 3D transforms | **THREE.js** (perspective / rotate3d / preserve-3d preview) |
| Motion path | **GSAP** + `MotionPathPlugin` (`animation/gsap-engine.ts`) |
| Storage | **IndexedDB**, **File System Access API** |
| Recording | **MediaRecorder** (screen + webcam) |
| Workers | **Web Workers** (decode, export, processing) |
| AI dịch vụ ngoài (tuỳ chọn) | KieAI (image gen), ElevenLabs (TTS), upscaling WebGPU shaders |

---

## 13. Tích hợp AI

OpenReel **chưa có "plugin system"** theo nghĩa kiến trúc mở rộng cho bên thứ ba (mục này nằm trong roadmap *In Progress*). Nhưng có tích hợp AI sẵn ở 3 dạng. Chi tiết file/flow xem [`TECHNICAL.md` §9](./TECHNICAL.md#9-templates--ai-integrations).

| Dạng | Thành phần | Chạy ở đâu |
|---|---|---|
| **AI on-device** | Background removal (`ai/background-removal-engine.ts`), Person segmentation (`ai/person-segmentation-engine.ts`), Auto-reframe / auto-crop bám chủ thể (`ai/auto-reframe-engine.ts`), AI upscaling (`video/upscaling/`, WebGPU shaders), motion tracking, beat detection / silence-cut / auto-highlight | Local trong trình duyệt (WebGPU/WebGL/WASM) — không gửi dữ liệu ra ngoài |
| **AI cloud — image generation** | **KieAI** (`api.kie.ai` + `kieai.redpandaai.co`) — 6 model: `seedream/5-lite-image-to-image`, `z-image`, `nano-banana-2`, `flux-2/pro-image-to-image`, `grok-imagine/image-to-image`, `qwen/image-to-image`. UI: `KieAIImageDialog`, `ModelPicker`, forms từng model, tab `AIGenTab`; service `services/kieai/`; store `kieai-store`. Flow: upload ảnh → createTask → poll → ảnh kết quả vào media library | Dịch vụ ngoài — cần API key người dùng tự nhập |
| **AI cloud — text/voice** | TTS (ElevenLabs — `TextToSpeechPanel`, `VoiceBrowser`, `tts-store`), Transcription/auto-caption STT (`text/speech-to-text-engine.ts`, dịch vụ `transcribe.openreel.video` / `cloud.openreel.video`, UI `AutoCaptionPanel`) → sinh phụ đề kèm per-word timing cho caption animation | Dịch vụ ngoài / OpenReel cloud |
| **OpenReel cloud** | Template cloud, project share (Cloudflare Workers `openreel-cloud.*.workers.dev`); endpoint tập trung `config/api-endpoints.ts` (swap được để self-host) | Cloudflare Workers |
| **AI-managed development** | Claude AI dùng cho issue triage / viết feature / code review / docs (README) — **quy trình phát triển**, không phải tính năng runtime | — |

Bảo mật khoá: `services/secure-storage.ts` (IndexedDB mã hoá AES-256, master password); gọi production qua Cloudflare Pages Function proxy (`apps/web/functions/api/proxy/`), dev gọi trực tiếp (`services/api-proxy.ts` → `apiFetch()`). Tác vụ AI nặng chạy qua hàng đợi nền (`services/background-generator.ts`, `processing-manager.ts`).

---

## 14. Điểm nối quan trọng cần nhớ khi đọc code

1. `EditorInterface` = trung tâm khởi tạo bridge.
2. `playback-bridge ↔ PlaybackController ↔ VideoEngine` = vòng đồng bộ audio/video + drive render.
3. `render-bridge ↔ VideoEngine ↔ effects-bridge` = pipeline render + effect.
4. `project-store ↔ ActionExecutor ↔ ActionHistory` = undo/redo.
5. `timeline-store` = nguồn duy nhất cho playhead/zoom; `engine-store` = nguồn duy nhất cho engine + `currentFrame`.
6. Tất cả mutation project là immutable — không bao giờ sửa trực tiếp `Clip`/`Track`, luôn qua action.
