# DJZ Clip-Crop

A browser-based video clip and crop tool. Upload a video, set trim points, define a crop region, and export both a WebM video and PNG frame sequences — all processed client-side with no server uploads.

**[Live Demo](https://scuffedepoch.com/clip-crop/)**

## Features

- **Video Upload** — Drag-and-drop or click to load MP4, MOV, WebM, or MKV files. Videos are stored in IndexedDB for persistence across sessions.
- **Timeline Trimming** — Set IN and OUT points with draggable handles on a visual timeline. Playhead tracks the current position in real-time.
- **Crop Overlay** — Interactive SVG overlay with drag-to-move and corner-resize handles. Supports aspect ratio locking (16:9, 9:16, 4:3, 1:1, 2.35:1, and more).
- **Resolution Targeting** — Choose from megapixel presets (0.25 MP to 1.50 MP) or match the source resolution. Output dimensions are computed automatically from the crop region.
- **Audio Control** — Toggle audio inclusion on/off before processing. Source audio is preserved in the WebM output when enabled.
- **WebM Video Export** — Encodes the trimmed and cropped clip as VP8/Vorbis WebM using ffmpeg.wasm.
- **PNG Frame Export** — Extracts every frame as high-quality PNGs, batched into ZIP archives (100 MB per archive).
- **Fully Client-Side** — All processing runs in the browser via Web Workers and ffmpeg.wasm. No data leaves your machine.

## How It Works

1. **Upload** a video file via drag-and-drop or file picker
2. **Set trim points** using the IN/OUT handles on the timeline
3. **Adjust the crop region** by dragging and resizing the overlay on the video preview
4. **Choose settings** — aspect ratio lock, megapixel target, audio on/off
5. **Lock & Review** — confirms your settings and shows a summary
6. **Start Processing** — ffmpeg.wasm encodes the WebM and extracts PNG frames in a Web Worker
7. **Download** — grab the WebM video and frame ZIP archives

## Tech Stack

- **React 18** + **TypeScript** — UI components and type safety
- **Vite 5** — Dev server and production bundler
- **Tailwind CSS 3** — Utility-first styling
- **Zustand** — Lightweight state management
- **ffmpeg.wasm 0.12** — Client-side video processing (VP8 video, Vorbis audio)
- **fflate** — Fast ZIP creation for frame archives
- **idb** — IndexedDB wrapper for video storage
- **WebGPU** — Lanczos rescaling (with CPU fallback)

## Getting Started

```bash
# Clone the repository
git clone https://github.com/MushroomFleet/djz-clipcrop.git
cd djz-clipcrop

# Install dependencies
npm install

# Start dev server
npm run dev

# Build for production
npm run build
```

The dev server runs at `http://localhost:5173/clip-crop/` with the required COOP/COEP headers for ffmpeg.wasm.

## Browser Requirements

- **Chrome 94+** or **Edge 94+** recommended (for WebGPU support and SharedArrayBuffer)
- Firefox and Safari work with CPU fallback for rescaling
- Requires Cross-Origin Isolation headers (configured automatically in the Vite dev server)

## Project Structure

```
src/
  components/    UI components (UploadZone, VideoPreview, CropOverlay, Timeline, Controls, LockPanel, DownloadPanel)
  store/         Zustand state management
  lib/           Utilities (IndexedDB, Lanczos rescaling, ZIP batching, megapixel math)
  workers/       Web Worker for ffmpeg.wasm processing
  types.ts       TypeScript interfaces
```

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{djz_clipcrop,
  title = {DJZ Clip-Crop: Browser-based video clip and crop tool},
  author = {Drift Johnson},
  year = {2025},
  url = {https://github.com/MushroomFleet/djz-clipcrop},
  version = {1.0.0}
}
```

### Donate:

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)

---

## Support This Project

If you found this useful, please **star the repo** — it helps others discover it!

[![Star on GitHub](https://img.shields.io/github/stars/MushroomFleet/djz-clipcrop?style=social)](https://github.com/MushroomFleet/djz-clipcrop)