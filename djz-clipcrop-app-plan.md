# djz-clipcrop

<!-- TINS Specification v1.0 -->
<!-- ZS:COMPLEXITY:HIGH -->
<!-- ZS:PRIORITY:HIGH -->
<!-- ZS:PLATFORM:WEB -->
<!-- ZS:LANGUAGE:TYPESCRIPT -->

## Description

**djz-clipcrop** is a browser-based video clip and crop tool built with React + Vite + TypeScript. It allows users to upload a video file directly in the browser, store it in IndexedDB, set IN/OUT trim points on a visual timeline, apply an aspect-ratio-locked crop overlay, optionally mute audio, choose an output resolution target (by megapixel), and export the result as a `.webm` video file. Frame batches are also exported as 100 MB ZIP archives of PNG images.

All processing is done entirely client-side using WebAssembly (ffmpeg.wasm) and WebGPU (for Lanczos-accelerated rescaling). No server upload is required. The app targets creative professionals and content creators who need fast, private, browser-based video trimming and cropping without installing desktop software.

Key differentiators: WebGPU Lanczos rescaling for quality downscaling, batch ZIP export of raw frames for further editing, pure client-side privacy, and a clean lock-before-process workflow that prevents accidental parameter changes.

---

## Functionality

### Core Features

1. **Video Upload & IndexedDB Storage** — User uploads a video file (mp4, mov, webm, mkv). The raw file is stored in IndexedDB under a `clips` object store keyed by a UUID. A blob URL is created from the stored bytes for preview playback.

2. **Video Preview Player** — An HTML5 `<video>` element shows the full source video. Playback is controlled by standard play/pause. The current playhead position drives the timeline scrubber in real time.

3. **Timeline with IN/OUT Points** — A horizontal timeline bar below the preview shows the full video duration. The user clicks "Set IN" to mark the current playhead position as the start of the clip, and "Set OUT" to mark the end. IN/OUT handles are draggable on the timeline. The selected region is highlighted. If IN >= OUT, OUT snaps to IN + 1 second.

4. **MUTE Toggle** — A toggle button labelled "MUTE AUDIO" / "KEEP AUDIO". Default is KEEP AUDIO. When MUTED, the exported webm will have no audio track.

5. **Aspect Ratio Selector** — A dropdown with the following options:
   - `Free` (no enforced ratio; crop box is fully resizable)
   - `16:9`
   - `9:16`
   - `4:3`
   - `3:4`
   - `1:1`
   - `21:9`
   - `2.35:1`
   Selecting a ratio constrains the crop overlay box to that ratio.

6. **Crop Overlay** — Rendered as an SVG or Canvas overlay on top of the video preview. The crop box:
   - Can be dragged to reposition (click-drag inside the box)
   - Is locked to the video frame boundaries (cannot be dragged outside)
   - Displays a semi-transparent dark mask outside the crop region
   - Shows drag handles at corners; if `Free` ratio is selected, corner handles resize the box
   - If a fixed ratio is selected, dragging corners resizes while maintaining ratio
   - Displays the crop region dimensions in pixels (e.g. `1920 × 1080`) in a small label

7. **Megapixel Resolution Selector** — A set of radio buttons or segmented control:
   - `Match Source` — no rescaling applied
   - `0.25 MP`
   - `0.50 MP`
   - `0.75 MP`
   - `1.00 MP`
   - `1.25 MP`
   - `1.50 MP`
   
   When a MP target is selected (and not Match Source), the crop region is rescaled to fit within that megapixel budget while preserving the crop aspect ratio. The computed output pixel dimensions are shown (e.g. `1333 × 750` for 1.0 MP at 16:9). Rescaling uses Lanczos resampling via WebGPU.

8. **Lock UI** — A prominent "🔒 LOCK & REVIEW" button. When clicked:
   - All controls (timeline handles, crop box, aspect ratio, MP selector, mute toggle) become non-interactive
   - A summary panel appears: "Clip: 00:12.3 → 00:45.8 | Duration: 33.5s | Crop: 1280×720 @ (320, 180) | Output: 1280×720 | Audio: Muted"
   - An "UNLOCK" button appears to revert to editing state
   - The "START PROCESSING" button becomes active only when locked

9. **START PROCESSING** — Triggers the export pipeline (see Technical Implementation). The button is disabled until UI is locked. While processing, a progress bar and status text are displayed (e.g. "Extracting frames 45 / 300…", "Creating ZIP batch 1 of 3…", "Encoding WebM…").

10. **Download Panel** — After processing completes, download links appear:
    - One or more ZIP files: `djz-clipcrop-frames-001.zip`, `djz-clipcrop-frames-002.zip`, etc.
    - One WebM file: `djz-clipcrop-output.webm`

---

### UI Layout

```
+------------------------------------------------------------+
|  djz-clipcrop                              [🔒 LOCK]       |
+------------------------------------------------------------+
|                                                            |
|   +--------------------------------------------------+    |
|   |                                                  |    |
|   |        VIDEO PREVIEW  (16:9 container)           |    |
|   |   +--------------------------------------+       |    |
|   |   |  CROP OVERLAY (draggable box)        |       |    |
|   |   |  [mask outside, handles at corners]  |       |    |
|   |   +--------------------------------------+       |    |
|   |                                                  |    |
|   +--------------------------------------------------+    |
|                                                            |
|  TIMELINE                                                  |
|  [====|----[##########]---------|====================]     |
|       IN handle           OUT handle                       |
|  00:00.0  [Set IN]  [Set OUT]  Duration: 1:23.4           |
|                                                            |
|  CONTROLS                                                  |
|  [MUTE AUDIO / KEEP AUDIO]                                 |
|  Aspect Ratio: [16:9 ▾]                                   |
|  Resolution:  ○ Match  ○ 0.25  ○ 0.50  ○ 0.75  ● 1.00   |
|               ○ 1.25  ○ 1.50                               |
|  Output size: 1333 × 750 px                               |
|                                                            |
|  ── LOCK SUMMARY (visible when locked) ───                |
|  Clip: 00:12.3 → 00:45.8 | Duration: 33.5s               |
|  Crop: 1280×720 @ x:320 y:180 | Audio: Muted             |
|  Output: 1333×750                                          |
|  [UNLOCK]           [START PROCESSING ▶]                  |
|                                                            |
|  PROGRESS (when processing)                               |
|  [============================----] 72%                    |
|  Extracting frames 216 / 300...                           |
|                                                            |
|  DOWNLOADS (when done)                                    |
|  [⬇ djz-clipcrop-frames-001.zip]                         |
|  [⬇ djz-clipcrop-frames-002.zip]                         |
|  [⬇ djz-clipcrop-output.webm]                            |
+------------------------------------------------------------+
```

---

### User Flows

**Happy Path:**
1. User opens app → sees upload area
2. User drags a video file onto the upload zone or clicks to browse
3. File is stored to IndexedDB; preview player loads the video
4. User plays the video, clicks "Set IN" at desired start frame
5. User scrubs to desired end, clicks "Set OUT"
6. User drags the crop overlay box to the desired region
7. User selects an aspect ratio from the dropdown (crop box snaps to ratio, maintaining centre)
8. User selects a MP resolution target
9. User clicks "MUTE AUDIO" if they want no audio
10. User clicks "🔒 LOCK & REVIEW"; summary is shown
11. User clicks "START PROCESSING"
12. Progress bar tracks frame extraction → ZIP batching → WebM encoding
13. Download links appear; user clicks to download ZIP(s) and WebM

**Edge Cases:**
- If no IN point is set, IN defaults to 0:00.0
- If no OUT point is set, OUT defaults to end of video
- If crop box is not moved, it defaults to centred at full video dimensions
- If source video has no audio track, MUTE toggle is hidden/disabled
- If the video is less than 1 second, OUT cannot equal IN (show error)
- If IndexedDB quota is exceeded, show error: "Video too large for browser storage. Try a smaller file."
- If WebGPU is unavailable, Lanczos falls back to a CPU WASM implementation (notify user with a yellow banner)
- Processing can be cancelled at any time with a "CANCEL" button (halts workers, clears temp data)

---

## Technical Implementation

### Stack

- **Framework:** React 18 + Vite 5 + TypeScript 5
- **Video Processing:** `@ffmpeg/ffmpeg` (ffmpeg.wasm) — frame extraction and WebM encoding
- **GPU Rescaling:** WebGPU API (navigator.gpu) for Lanczos kernel; CPU fallback via a custom WASM Lanczos module
- **ZIP Creation:** `fflate` library (fast WASM-accelerated zip in browser)
- **Storage:** IndexedDB via `idb` (typed wrapper)
- **Styling:** Tailwind CSS v3
- **State Management:** Zustand

---

### Project Structure

```
djz-clipcrop/
├── public/
│   └── ffmpeg/                  # ffmpeg.wasm core files (copied from node_modules)
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── store/
│   │   └── useClipStore.ts      # Zustand global state
│   ├── components/
│   │   ├── UploadZone.tsx
│   │   ├── VideoPreview.tsx
│   │   ├── CropOverlay.tsx      # SVG overlay, drag logic
│   │   ├── Timeline.tsx         # IN/OUT scrubber
│   │   ├── Controls.tsx         # Mute, aspect ratio, MP selector
│   │   ├── LockPanel.tsx        # Summary + START PROCESSING
│   │   └── DownloadPanel.tsx
│   ├── workers/
│   │   └── processWorker.ts     # Web Worker: ffmpeg frame extraction + encoding
│   ├── lib/
│   │   ├── indexedDB.ts         # idb wrapper: storeVideo, getVideo, clearVideo
│   │   ├── lanczos.ts           # WebGPU Lanczos; CPU fallback
│   │   ├── zipBatcher.ts        # Batch PNG frames into 100MB ZIPs with fflate
│   │   └── megapixel.ts        # Compute output dimensions from MP target + ratio
│   └── types.ts
├── package.json
├── vite.config.ts
└── index.html
```

---

### Data Models

```typescript
// types.ts

interface ClipState {
  // Source
  videoId: string | null;          // UUID key in IndexedDB
  videoUrl: string | null;         // blob URL for <video> src
  videoDuration: number;           // seconds, float
  videoWidth: number;              // source pixel width
  videoHeight: number;             // source pixel height
  hasAudio: boolean;

  // Trim
  inPoint: number;                 // seconds, default 0
  outPoint: number;                // seconds, default = videoDuration

  // Audio
  muteAudio: boolean;              // default false

  // Crop (in source pixels)
  cropX: number;                   // left edge, default 0
  cropY: number;                   // top edge, default 0
  cropWidth: number;               // default = videoWidth
  cropHeight: number;              // default = videoHeight

  // Aspect Ratio
  aspectRatio: AspectRatioOption;  // default 'free'

  // Output Resolution
  mpTarget: MpTarget;              // default 'match'
  outputWidth: number;             // computed
  outputHeight: number;            // computed

  // UI State
  locked: boolean;
  processing: boolean;
  progress: number;                // 0–100
  progressLabel: string;
  downloadItems: DownloadItem[];
  error: string | null;
}

type AspectRatioOption = 'free' | '16:9' | '9:16' | '4:3' | '3:4' | '1:1' | '21:9' | '2.35:1';

type MpTarget = 'match' | '0.25' | '0.50' | '0.75' | '1.00' | '1.25' | '1.50';

interface DownloadItem {
  label: string;     // e.g. "djz-clipcrop-frames-001.zip"
  url: string;       // blob URL
  type: 'zip' | 'webm';
}

// IndexedDB schema
interface StoredClip {
  id: string;        // UUID
  file: ArrayBuffer; // raw video bytes
  name: string;      // original filename
  size: number;      // bytes
  storedAt: number;  // Date.now()
}
```

---

### Key Algorithms

#### 1. Megapixel → Output Dimensions

```typescript
// lib/megapixel.ts
export function computeOutputDimensions(
  cropWidth: number,
  cropHeight: number,
  mpTarget: MpTarget
): { width: number; height: number } {
  if (mpTarget === 'match') return { width: cropWidth, height: cropHeight };
  const targetPixels = parseFloat(mpTarget) * 1_000_000;
  const aspectRatio = cropWidth / cropHeight;
  // height = sqrt(targetPixels / aspectRatio), width = height * aspectRatio
  const height = Math.round(Math.sqrt(targetPixels / aspectRatio));
  const width = Math.round(height * aspectRatio);
  // Ensure even dimensions for video codec compatibility
  return { width: width % 2 === 0 ? width : width + 1, height: height % 2 === 0 ? height : height + 1 };
}
```

#### 2. Aspect Ratio Crop Constraint

```typescript
// When aspectRatio changes, recompute cropWidth/cropHeight from current cropWidth,
// anchor at crop centre, clamp to video bounds.
export function constrainCropToRatio(
  currentCrop: { x: number; y: number; width: number; height: number },
  ratio: AspectRatioOption,
  videoWidth: number,
  videoHeight: number
): { x: number; y: number; width: number; height: number } {
  if (ratio === 'free') return currentCrop;
  const [rw, rh] = ratio.split(':').map(parseFloat);
  const r = rw / rh;
  let w = currentCrop.width;
  let h = Math.round(w / r);
  // If h exceeds video height, clamp and recompute w
  if (h > videoHeight) { h = videoHeight; w = Math.round(h * r); }
  if (w > videoWidth) { w = videoWidth; h = Math.round(w / r); }
  // Centre on previous crop centre
  const cx = currentCrop.x + currentCrop.width / 2;
  const cy = currentCrop.y + currentCrop.height / 2;
  const x = Math.max(0, Math.min(videoWidth - w, Math.round(cx - w / 2)));
  const y = Math.max(0, Math.min(videoHeight - h, Math.round(cy - h / 2)));
  return { x, y, width: w, height: h };
}
```

#### 3. WebGPU Lanczos Rescaling

```typescript
// lib/lanczos.ts
// Lanczos kernel size 3 (a=3). Two-pass (horizontal then vertical) compute shader.

const LANCZOS_SHADER = /* wgsl */`
  // Lanczos kernel function
  fn lanczos(x: f32, a: f32) -> f32 {
    if (abs(x) < 0.0001) { return 1.0; }
    if (abs(x) >= a) { return 0.0; }
    let px = 3.14159265 * x;
    return a * sin(px) * sin(px / a) / (px * px);
  }

  @group(0) @binding(0) var inputTex: texture_2d<f32>;
  @group(0) @binding(1) var outputTex: texture_storage_2d<rgba8unorm, write>;
  @group(0) @binding(2) var<uniform> params: ResampleParams;

  struct ResampleParams {
    srcWidth: u32,
    srcHeight: u32,
    dstWidth: u32,
    dstHeight: u32,
    pass: u32,   // 0 = horizontal, 1 = vertical
  }

  @compute @workgroup_size(8, 8)
  fn main(@builtin(global_invocation_id) gid: vec3<u32>) {
    // ... horizontal or vertical Lanczos pass depending on params.pass
    // Sample srcWidth -> dstWidth (pass 0) or srcHeight -> dstHeight (pass 1)
    // Write to outputTex
  }
`;

export async function lanczosResize(
  device: GPUDevice,
  sourceCanvas: OffscreenCanvas,
  targetWidth: number,
  targetHeight: number
): Promise<ImageBitmap> {
  // 1. Upload sourceCanvas pixels to GPUTexture
  // 2. Create intermediate GPUTexture for horizontal pass result
  // 3. Run horizontal pass compute shader
  // 4. Run vertical pass compute shader writing to output GPUTexture
  // 5. Copy output texture to OffscreenCanvas, return as ImageBitmap
}

// CPU fallback using a pure TypeScript Lanczos implementation
export function lanczosResizeCPU(
  pixels: Uint8ClampedArray,
  srcW: number, srcH: number,
  dstW: number, dstH: number
): Uint8ClampedArray { /* ... */ }
```

#### 4. Frame Extraction & ZIP Batching

```typescript
// workers/processWorker.ts  (runs in a Web Worker)
// Uses @ffmpeg/ffmpeg to extract frames as PNG, then fflate to batch into ZIPs.

import { FFmpeg } from '@ffmpeg/ffmpeg';
import { fetchFile } from '@ffmpeg/util';
import { zipSync } from 'fflate';

const ZIP_BATCH_BYTES = 100 * 1024 * 1024; // 100 MB

self.onmessage = async (e: MessageEvent<ProcessJob>) => {
  const { videoBuffer, inPoint, outPoint, cropX, cropY, cropWidth, cropHeight,
          outputWidth, outputHeight, muteAudio } = e.data;

  const ffmpeg = new FFmpeg();
  await ffmpeg.load({ coreURL: '/ffmpeg/ffmpeg-core.js' });

  // Write source video to ffmpeg virtual FS
  await ffmpeg.writeFile('input.mp4', new Uint8Array(videoBuffer));

  // Extract frames from IN to OUT, crop, then pass to lanczos resize
  // ffmpeg vf: trim, crop, scale to outputWidth x outputHeight
  // Use fps=native to extract every frame, or a fixed fps if desired (native recommended)
  const vf = `trim=start=${inPoint}:end=${outPoint},setpts=PTS-STARTPTS,crop=${cropWidth}:${cropHeight}:${cropX}:${cropY},scale=${outputWidth}:${outputHeight}:flags=lanczos`;
  
  // NOTE: ffmpeg built-in lanczos is used as primary scaler; WebGPU lanczos is
  // used for post-process quality pass when outputWidth < cropWidth (downscale only)
  // and WebGPU is available. For simplicity, ffmpeg's scale=flags=lanczos handles
  // frame-level rescaling; WebGPU Lanczos is the quality target for future iteration.

  await ffmpeg.exec([
    '-i', 'input.mp4',
    '-vf', vf,
    '-vsync', 'cfr',
    '-q:v', '1',
    'frame_%06d.png'
  ]);

  // Read frame filenames from ffmpeg FS
  const files = (await ffmpeg.listDir('/')).filter(f => f.name.startsWith('frame_'));
  const totalFrames = files.length;

  let batchIndex = 1;
  let currentBatchFiles: Record<string, Uint8Array> = {};
  let currentBatchBytes = 0;
  const zipBlobs: Blob[] = [];

  const flushBatch = () => {
    if (Object.keys(currentBatchFiles).length === 0) return;
    const zipped = zipSync(currentBatchFiles, { level: 0 }); // store, no compress (PNGs already compressed)
    zipBlobs.push(new Blob([zipped], { type: 'application/zip' }));
    currentBatchFiles = {};
    currentBatchBytes = 0;
    batchIndex++;
  };

  for (let i = 0; i < files.length; i++) {
    const name = files[i].name;
    const data = await ffmpeg.readFile(name) as Uint8Array;
    if (currentBatchBytes + data.byteLength > ZIP_BATCH_BYTES) flushBatch();
    currentBatchFiles[name] = data;
    currentBatchBytes += data.byteLength;
    self.postMessage({ type: 'progress', value: Math.round((i / totalFrames) * 70), label: `Extracting frame ${i + 1} / ${totalFrames}` });
  }
  flushBatch();

  self.postMessage({ type: 'progress', value: 75, label: 'Encoding WebM...' });

  // Encode WebM from extracted frames
  // Write frames back, run ffmpeg encode
  // (frames already in FS from extraction step, re-use them)
  const audioArgs = muteAudio ? ['-an'] : ['-i', 'input.mp4', '-map', '1:a?'];
  await ffmpeg.exec([
    '-framerate', '30',           // should match source FPS; detect it beforehand
    '-i', 'frame_%06d.png',
    ...audioArgs,
    '-c:v', 'libvpx-vp9',
    '-b:v', '0',
    '-crf', '33',
    '-c:a', 'libopus',
    'output.webm'
  ]);

  const webmData = await ffmpeg.readFile('output.webm') as Uint8Array;
  const webmBlob = new Blob([webmData], { type: 'video/webm' });

  self.postMessage({
    type: 'done',
    zipBlobs,
    webmBlob
  }, []);
};

interface ProcessJob {
  videoBuffer: ArrayBuffer;
  inPoint: number;
  outPoint: number;
  cropX: number; cropY: number; cropWidth: number; cropHeight: number;
  outputWidth: number; outputHeight: number;
  muteAudio: boolean;
}
```

#### 5. IndexedDB Helpers

```typescript
// lib/indexedDB.ts
import { openDB, IDBPDatabase } from 'idb';

const DB_NAME = 'djz-clipcrop';
const STORE = 'clips';

async function getDB(): Promise<IDBPDatabase> {
  return openDB(DB_NAME, 1, {
    upgrade(db) { db.createObjectStore(STORE, { keyPath: 'id' }); }
  });
}

export async function storeVideo(file: File): Promise<string> {
  const id = crypto.randomUUID();
  const buffer = await file.arrayBuffer();
  const db = await getDB();
  await db.put(STORE, { id, file: buffer, name: file.name, size: file.size, storedAt: Date.now() });
  return id;
}

export async function getVideoBuffer(id: string): Promise<ArrayBuffer> {
  const db = await getDB();
  const record = await db.get(STORE, id);
  if (!record) throw new Error('Video not found in IndexedDB');
  return record.file as ArrayBuffer;
}

export async function clearAllVideos(): Promise<void> {
  const db = await getDB();
  await db.clear(STORE);
}
```

---

### Component Specifications

#### `CropOverlay.tsx`
- Renders as an `<svg>` element absolutely positioned over the `<video>` element, matching its displayed dimensions (not natural dimensions — use the video element's `clientWidth` / `clientHeight`)
- Maintains a scale factor: `scaleX = clientWidth / videoWidth`, `scaleY = clientHeight / videoHeight`
- All crop coordinates stored in source pixels; displayed coordinates are multiplied by scale factors
- Mask: four `<rect>` elements covering areas outside the crop box with `fill="rgba(0,0,0,0.55)"`
- Crop box: a `<rect>` with `fill="transparent"` and `stroke="white"` with `strokeWidth=2`
- Corner handles: four 10×10 `<rect>` elements at corners, `fill="white"`, cursor-resizable
- Drag events use `onPointerDown`, `onPointerMove`, `onPointerUp` with `setPointerCapture`
- When locked, all pointer events on the overlay are disabled (`pointerEvents: 'none'`)

#### `Timeline.tsx`
- A `<div>` with `position: relative` spanning full width
- A `<div>` inner bar represents the full duration
- IN handle: a draggable `<div>` positioned at `(inPoint / duration) * 100%` from left
- OUT handle: a draggable `<div>` positioned at `(outPoint / duration) * 100%` from left
- Selected region: a `<div>` between IN and OUT handles with a highlight colour
- Playhead: a thin vertical line that tracks `<video>.currentTime` via a `timeupdate` listener
- Clicking anywhere on the timeline seeks the video to that position
- "Set IN" button: sets `inPoint = video.currentTime`
- "Set OUT" button: sets `outPoint = video.currentTime`

#### `VideoPreview.tsx`
- A container `<div>` with `position: relative; aspect-ratio: calculated from videoWidth/videoHeight`
- Contains the `<video>` element and the `<CropOverlay>` SVG as siblings
- Uses `ResizeObserver` to update displayed dimensions used by `CropOverlay`
- On metadata load (`onLoadedMetadata`), reads `videoWidth`, `videoHeight`, `duration` and writes to store

---

### Vite Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  optimizeDeps: {
    exclude: ['@ffmpeg/ffmpeg', '@ffmpeg/util'],
  },
  server: {
    headers: {
      'Cross-Origin-Opener-Policy': 'same-origin',
      'Cross-Origin-Embedder-Policy': 'require-corp',
    },
  },
});
// Note: COOP/COEP headers are required for SharedArrayBuffer used by ffmpeg.wasm
```

---

### package.json Dependencies

```json
{
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "@ffmpeg/ffmpeg": "^0.12.10",
    "@ffmpeg/util": "^0.12.1",
    "fflate": "^0.8.2",
    "idb": "^8.0.0",
    "zustand": "^4.5.4"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.1",
    "vite": "^5.4.0",
    "typescript": "^5.5.0",
    "tailwindcss": "^3.4.10",
    "autoprefixer": "^10.4.20",
    "postcss": "^8.4.45",
    "@types/react": "^18.3.3",
    "@types/react-dom": "^18.3.0"
  }
}
```

---

### Zustand Store

```typescript
// store/useClipStore.ts
import { create } from 'zustand';
import { ClipState, AspectRatioOption, MpTarget, DownloadItem } from '../types';
import { constrainCropToRatio } from '../lib/megapixel';
import { computeOutputDimensions } from '../lib/megapixel';

interface ClipActions {
  setVideo: (id: string, url: string, duration: number, w: number, h: number, hasAudio: boolean) => void;
  setInPoint: (t: number) => void;
  setOutPoint: (t: number) => void;
  setMuteAudio: (v: boolean) => void;
  setCrop: (x: number, y: number, w: number, h: number) => void;
  setAspectRatio: (r: AspectRatioOption) => void;
  setMpTarget: (mp: MpTarget) => void;
  lock: () => void;
  unlock: () => void;
  setProgress: (value: number, label: string) => void;
  setDone: (items: DownloadItem[]) => void;
  setError: (msg: string) => void;
  reset: () => void;
}

export const useClipStore = create<ClipState & ClipActions>((set, get) => ({
  videoId: null,
  videoUrl: null,
  videoDuration: 0,
  videoWidth: 0,
  videoHeight: 0,
  hasAudio: true,
  inPoint: 0,
  outPoint: 0,
  muteAudio: false,
  cropX: 0, cropY: 0, cropWidth: 0, cropHeight: 0,
  aspectRatio: 'free',
  mpTarget: 'match',
  outputWidth: 0, outputHeight: 0,
  locked: false,
  processing: false,
  progress: 0,
  progressLabel: '',
  downloadItems: [],
  error: null,

  setVideo: (id, url, duration, w, h, hasAudio) => set({
    videoId: id, videoUrl: url, videoDuration: duration,
    videoWidth: w, videoHeight: h, hasAudio,
    inPoint: 0, outPoint: duration,
    cropX: 0, cropY: 0, cropWidth: w, cropHeight: h,
    outputWidth: w, outputHeight: h,
  }),

  setAspectRatio: (r) => {
    const s = get();
    const newCrop = constrainCropToRatio(
      { x: s.cropX, y: s.cropY, width: s.cropWidth, height: s.cropHeight },
      r, s.videoWidth, s.videoHeight
    );
    const dims = computeOutputDimensions(newCrop.width, newCrop.height, s.mpTarget);
    set({ aspectRatio: r, cropX: newCrop.x, cropY: newCrop.y, cropWidth: newCrop.width, cropHeight: newCrop.height, ...dims });
  },

  setMpTarget: (mp) => {
    const s = get();
    const dims = computeOutputDimensions(s.cropWidth, s.cropHeight, mp);
    set({ mpTarget: mp, ...dims });
  },

  setCrop: (x, y, w, h) => {
    const s = get();
    const dims = computeOutputDimensions(w, h, s.mpTarget);
    set({ cropX: x, cropY: y, cropWidth: w, cropHeight: h, ...dims });
  },

  setInPoint: (t) => set({ inPoint: t }),
  setOutPoint: (t) => set({ outPoint: t }),
  setMuteAudio: (v) => set({ muteAudio: v }),
  lock: () => set({ locked: true }),
  unlock: () => set({ locked: false }),
  setProgress: (value, label) => set({ processing: true, progress: value, progressLabel: label }),
  setDone: (items) => set({ processing: false, progress: 100, downloadItems: items }),
  setError: (msg) => set({ processing: false, error: msg }),
  reset: () => set({ videoId: null, videoUrl: null, locked: false, processing: false, downloadItems: [], error: null }),
}));
```

---

### Processing Pipeline (App.tsx → Worker)

```typescript
// In LockPanel.tsx or App.tsx — startProcessing handler
import { getVideoBuffer } from '../lib/indexedDB';
import { useClipStore } from '../store/useClipStore';

async function startProcessing() {
  const s = useClipStore.getState();
  if (!s.videoId) return;

  s.setProgress(0, 'Reading video from storage...');

  const buffer = await getVideoBuffer(s.videoId);

  const worker = new Worker(new URL('../workers/processWorker.ts', import.meta.url), { type: 'module' });

  worker.onmessage = (e) => {
    const msg = e.data;
    if (msg.type === 'progress') {
      useClipStore.getState().setProgress(msg.value, msg.label);
    } else if (msg.type === 'done') {
      const items: DownloadItem[] = [];
      msg.zipBlobs.forEach((blob: Blob, i: number) => {
        items.push({ label: `djz-clipcrop-frames-${String(i + 1).padStart(3, '0')}.zip`, url: URL.createObjectURL(blob), type: 'zip' });
      });
      items.push({ label: 'djz-clipcrop-output.webm', url: URL.createObjectURL(msg.webmBlob), type: 'webm' });
      useClipStore.getState().setDone(items);
    } else if (msg.type === 'error') {
      useClipStore.getState().setError(msg.message);
    }
  };

  worker.postMessage({
    videoBuffer: buffer,
    inPoint: s.inPoint,
    outPoint: s.outPoint,
    cropX: s.cropX, cropY: s.cropY,
    cropWidth: s.cropWidth, cropHeight: s.cropHeight,
    outputWidth: s.outputWidth, outputHeight: s.outputHeight,
    muteAudio: s.muteAudio,
  }, [buffer]); // Transfer ownership to worker
}
```

---

## Style Guide

- **Background:** `#0e0e0e` (near black)
- **Surface:** `#1a1a1a` (dark card)
- **Accent:** `#e0ff4f` (acid yellow-green) for active handles, IN/OUT markers, and the LOCK button border
- **Text Primary:** `#f0f0f0`
- **Text Muted:** `#888888`
- **Danger/Error:** `#ff4f4f`
- **Font:** System UI stack (`-apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif`)
- **Timeline IN handle colour:** `#4fff91` (green)
- **Timeline OUT handle colour:** `#ff914f` (orange)
- **Crop overlay stroke:** `#ffffff` with `2px` stroke, `8px` dashed pattern on mask boundary
- **Buttons:** Rounded `4px`, uppercase labels, `0.5px` letter spacing
- **Transitions:** `150ms ease` for lock state changes; no animation on processing UI

---

## Performance Goals

- Video preview must be smooth (native HTML5 playback — no transcoding for preview)
- Frame extraction throughput target: ≥ 15 frames/second in ffmpeg.wasm on a modern laptop
- ZIP batching must not block the main thread (runs entirely in Web Worker)
- WebGPU Lanczos shader: process a single 1920×1080 frame in < 20ms
- IndexedDB read of a 500 MB video should complete in < 5 seconds on SSD-backed hardware
- Progress updates sent to main thread at most every 200ms to avoid UI jank

---

## Accessibility

- All interactive controls have `aria-label` attributes
- LOCK button uses `aria-pressed` state
- Progress bar uses `role="progressbar"` with `aria-valuenow`, `aria-valuemin`, `aria-valuemax`
- Crop overlay SVG handles have `role="slider"` and `aria-label="Crop handle [position]"`
- Keyboard: Tab navigates all controls; Enter/Space activates buttons; Arrow keys adjust IN/OUT points by ±0.1 seconds when focused on timeline handles
- Colour contrast ratio ≥ 4.5:1 for all text on backgrounds

---

## Testing Scenarios

1. **Upload and preview** — Upload a 720p MP4, verify preview loads, duration shown correctly
2. **Set IN/OUT** — Set IN at 5s, OUT at 20s; verify timeline highlight spans correct region
3. **Aspect ratio constraint** — Select 16:9 on a 4:3 source; verify crop box snaps and stays in bounds
4. **MP computation** — Set 1.0 MP on a 1:1 crop; verify output = 1000×1000
5. **MP computation edge** — Set 0.25 MP on a 16:9 crop; verify output ≈ 666×375 (both even)
6. **Lock/Unlock** — Lock UI, verify all controls non-interactive; unlock, verify restored
7. **Muted export** — Enable mute, process; verify output.webm has no audio track
8. **ZIP batching** — Process a long video generating > 100MB of frames; verify multiple ZIP files created
9. **WebGPU fallback** — Simulate WebGPU unavailable (`navigator.gpu = undefined`); verify CPU Lanczos fallback banner shown and processing completes
10. **Cancel processing** — Click cancel mid-process; verify worker terminated, no download links shown
11. **IndexedDB quota** — Simulate storage failure; verify quota error message shown
12. **No audio source** — Upload a silent video; verify MUTE toggle hidden/disabled

---

## Extended Features (Future / Optional)

- **Source FPS detection** — Use ffmpeg to detect source FPS and use it for WebM encoding (currently hardcoded 30fps)
- **Frame range preview** — Thumbnail strip on the timeline showing keyframes at regular intervals
- **Multiple crops / batch export** — Queue multiple crop regions from the same source
- **WebGPU Lanczos post-process** — Use GPU Lanczos as a post-processing step after ffmpeg scale for extra sharpness
- **Drag-to-upload from URL** — Paste a local file path or blob URL directly
- **Preset profiles** — Save and recall aspect ratio + MP combinations
- **Dark/Light theme toggle**
