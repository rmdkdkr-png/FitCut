<div align="center">

# FitCut

**Cut it to fit.** Turn videos into animated WebP — or just split them — right in your browser,
automatically sliced so every piece lands under your size limit.

[**▶ Open FitCut**](https://rmdkdkr-png.github.io/fitcut/) &nbsp;·&nbsp;
[한국어 README](README.ko.md)

![MIT License](https://img.shields.io/badge/license-MIT-4ade80)
![No dependencies](https://img.shields.io/badge/dependencies-none-4ade80)
![Single file](https://img.shields.io/badge/single%20file-75%20KB-4ade80)
![No upload](https://img.shields.io/badge/uploads-never-4ade80)

<img src="docs/screenshot.png" width="380" alt="FitCut screenshot">

</div>

---

## Why

Every place you post has a file size cap — Discord, forums, chat apps, wikis. Existing converters
either hand you a 40 MB file and shrug, or make you guess how many seconds fit under the cap and
run the job five times.

FitCut takes the cap as an input. Tell it "20 MB per file" and it slices your video into as many
pieces as it takes, each one guaranteed to fit.

Everything runs in your browser. No upload, no server, no size limit on the input, no queue.
One HTML file, no build step, no dependencies — open it from disk and it works offline.

## Features

**Animated WebP**
- Up to **60 fps**, up to 1200 frames per piece
- **Auto-split by size** — closes a piece just before it would exceed your limit
- Or split at a fixed length (5/10/15/20/30/60 s), or don't split at all
- **Frame optimization** — only the changed rectangle of each frame is stored (up to 76 % smaller)
- Batch: drop in many videos, convert sequentially or 2–3 at a time
- Frame timing is preserved exactly, even when the source fps doesn't match your target

**Video cutting**
- Cuts to **mp4** (falls back to webm) with the **audio intact**
- Same splitting rules: by size, by fixed length, or a single clip
- Speed control (2× halves the processing time as well)
- Bitrate is measured live and the next piece's length is corrected to stay under the cap

**Both modes**
- Custom size limit, down to a fraction of a megabyte
- Per-piece preview, save, and share; **Save all** packs everything into a ZIP
- Partial results survive cancellation or an error — finished pieces are kept
- Screen wake lock during long jobs; auto-pauses when you switch tabs and resumes on return

## Benchmarks

Measured on a headless Chromium container, not a phone — treat the timings as relative.

**Frame optimization** — 5 s clip, 480×270, 10 fps, quality 80, deterministic seek capture,
detailed background with a small moving element:

| | Size | Quality (PSNR) |
|---|---:|---:|
| Optimization off | 477 KB | 26.97 dB |
| **Optimization on** | **113 KB** (−76 %) | 26.94 dB |
| ffmpeg `libwebp_anim` (reference) | 142 KB | — |

Quality cost is 0.03 dB — effectively nothing. High-motion footage saves less (3–15 %), since
there is little to reuse between frames.

**Throughput**

| Job | Result |
|---|---|
| 25 s source → 60 fps, 480 px, 20 MB cap | 2 pieces (20 s + 5 s), 25 s of processing |
| 70 s of 720p → 60 fps, 480 px, 20 MB cap | 4 pieces covering all 70 s, 11.6 MB total, 70 s |
| 10 s at 1280 px, 30 fps | 17.6 s (optimization on) vs 18.6 s (off) |

## Usage

Three ways, all equivalent:

1. **Hosted** — open [the demo](https://YOUR_USERNAME.github.io/fitcut/)
2. **Local** — download `index.html`, open it in Chrome. Works with no internet connection.
3. **Phone app** — open it in Chrome on Android → menu → *Add to Home screen*

Drop in one or more videos, set the size limit, hit the button.

## Browser support

| | Animated WebP | Video cutting |
|---|---|---|
| Chrome / Edge (desktop, Android) | ✅ | ✅ |
| Samsung Internet | ✅ | ✅ |
| Firefox | ✅ | ⚠️ webm output |
| Safari / iOS | ⚠️ falls back to seek-based capture, slower | ❌ |

WebP encoding relies on `canvas.toBlob('image/webp')`; video cutting relies on `MediaRecorder`
plus `captureStream`. FitCut detects what's missing and either falls back or tells you.

## How it works

**Capture.** Instead of seeking frame by frame (slow — several minutes for 1200 frames), FitCut
plays the video once and grabs frames through `requestVideoFrameCallback`, which reports the exact
`mediaTime` of every presented frame. Frames are encoded in parallel by a pool of Web Workers using
`OffscreenCanvas`; when the queue backs up, playback pauses itself until the workers catch up.
Browsers without `requestVideoFrameCallback` fall back to seek-based capture automatically.

**Timing.** Slots are computed from your target fps. If the source runs at 30 fps and you ask for
60, FitCut doesn't duplicate frames — it extends the per-frame duration instead, so the animation
keeps exact timing without paying for redundant data.

**Frame optimization.** Each frame is compared to the previous one; only the changed rectangle
(aligned to 16 px so no seams show at macroblock edges) is encoded, and placed with an ANMF offset
using no-blend/no-dispose. Identical frames are dropped entirely and the previous frame's duration
is extended. If more than 70 % of the frame changed, a full keyframe is cheaper and is used instead.

**Splitting.** Frames accumulate in GOPs bounded by keyframes. A GOP is committed to the current
piece only if it still fits the byte budget and the length cap; otherwise the piece is closed and
the GOP starts the next one. Because pieces can only begin at a keyframe, every output file is
independently decodable — the size guarantee is exact, not an estimate.

**Muxing.** The animated WebP container (RIFF / VP8X / ANIM / ANMF) is assembled by hand from
still WebP frames produced by the canvas encoder. No WASM, no libwebp.

**Video mode.** `captureStream` + `MediaRecorder`, with audio routed through a Web Audio graph that
is never connected to the speakers — so processing stays silent while the audio track is still
recorded. WebM output from `MediaRecorder` carries no duration in its header, so FitCut patches the
EBML `Info` element after recording to make players show and seek the clip correctly.

## Limitations

Stated plainly, because it's better than finding out later.

- **Video cutting runs in real time** and re-encodes. ffmpeg's `-c copy` is instant and lossless;
  doing that in a browser means parsing the MP4 container, which FitCut doesn't do.
- **Encoder quality** is whatever the browser's canvas WebP encoder gives. libwebp's advanced
  settings aren't reachable from JavaScript, so ffmpeg still wins on quality per byte.
- No GIF output, no timeline scrubber, no cropping or rotation.
- Long jobs on a phone will heat it up and drain battery. Lower the fps for anything long.

## Contributing

Issues and pull requests are welcome. The whole tool is one file with no build step — edit
`index.html`, open it in a browser, done.

## License

MIT © 2026 dudu
