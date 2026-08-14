# Reframe — Auto Portrait Cut

Turn a single-camera, landscape podcast recording into a portrait (9:16) cut that automatically follows whoever's talking — like a virtual multicam switcher, running entirely in your browser.

No upload, no backend, no account. Drop in a video, it detects faces, figures out who's speaking from mouth movement, and exports a reframed portrait video you can post straight to Shorts/Reels/TikTok.

## Why

Most podcasts are recorded with one static, wide camera. Turning that into portrait content usually means manually keyframing a crop for every cut in a video editor — slow, and it doesn't scale past a couple of episodes. Reframe automates the "who's talking, follow them" decision that a human editor (or a real multicam switcher) would normally make.

## Features

- **Active speaker detection** — tracks each face's mouth-movement variance over a rolling window to estimate who's talking, gated by overall audio level so it doesn't switch during silence.
- **Instant cuts** — switches between speakers as a hard cut, like a real camera switch, instead of a dragging/panning crop. Can be toggled off for a smooth pan style instead.
- **Reaction cutaways** — briefly cuts to the other person on a sharp mouth-movement spike (laugh, "wait, what?") before returning to the active speaker.
- **Manual override** — tap either face in the source preview to lock the crop on them; tap again or hit "Resume auto" to hand control back to the detector.
- **Adjustable sensitivity and zoom** — tune how eagerly it switches, and how tight the crop sits on a face.
- **100% client-side** — video is read and processed locally in the browser via `<video>`/`<canvas>`; nothing is uploaded anywhere.
- **Debug overlay** — optional face boxes + live crop window drawn over the source video so you can see exactly what it's tracking.

## How it works

1. **Face detection**: [face-api.js](https://github.com/justadudewhohacks/face-api.js) (TinyFaceDetector + 68-point landmarks, TensorFlow.js under the hood) runs on a downscaled copy of each video frame.
2. **Tracking**: detected faces are matched frame-to-frame by proximity to build up to two persistent "speaker" tracks.
3. **Speaking score**: for each track, the mouth-aspect-ratio (distance between inner lip points, normalized by mouth width) is sampled over a ~1.2s rolling window. Talking produces high variance in that signal; a still face doesn't.
4. **Audio gate**: a `Web Audio` `AnalyserNode` reads overall RMS level from the video's audio track, used to suppress switching during silence (since a single mixed track can't isolate who's making sound).
5. **Switching logic**: the leading speaker has to hold a margin over the current one for a short hysteresis window before the app commits to a switch — this avoids flicker on small mouth twitches.
6. **Rendering + export**: each frame, a 9:16 region of the source video is drawn to an output `<canvas>`. That canvas's `captureStream()` is combined with the original audio (routed through the Web Audio graph) and recorded with `MediaRecorder` into a downloadable `.webm` file, in real time as the video plays once through.

## Getting started

This is a single static HTML file — no build step, no dependencies to install.

```bash
git clone <this-repo>
cd reframe
open index.html   # or just double-click it / drag it into a browser
```

Face detection models load from a CDN at runtime (see [Models](#models) below), so an internet connection is required even though processing itself is local.

## Usage

1. Open `index.html` in a modern browser (Chrome recommended — best `MediaRecorder`/`captureStream` support).
2. Drop in a landscape video with one or two people in frame.
3. Adjust settings if you want:
   - **Switch sensitivity** — calm / medium / snappy
   - **Zoom / headroom** — how tight the portrait crop sits on a face
   - **Instant cuts** — hard cut vs. smooth pan between speakers
   - **Quick cutaways on reactions** — on/off
   - **Show detection boxes** — debug overlay
4. Click **Start & export**. The video plays through once in real time while it records the reframed output.
5. While it's running, tap either face in the source preview to manually lock focus on them; tap again or hit **Resume auto** to let the detector take back over.
6. Download the resulting `.webm` when it finishes.

### Converting the output

Output is WebM (VP9/Opus), which most platforms accept directly. If you need `.mp4` (e.g. for DaVinci Resolve or another NLE that doesn't import WebM cleanly), re-encode with [HandBrake](https://handbrake.fr/) or ffmpeg:

```bash
ffmpeg -i output.webm -c:v libx264 -crf 17 -preset slow -c:a aac -b:a 192k output.mp4
```

## Models

Face detection weights are loaded at runtime from a public CDN mirror of the face-api.js model repo, with fallbacks:

- `cdn.jsdelivr.net/gh/justadudewhohacks/face-api.js@master/weights`
- `justadudewhohacks.github.io/face-api.js/models`
- `raw.githubusercontent.com/justadudewhohacks/face-api.js/master/weights`

If you're forking this for offline/self-hosted use, download the `tiny_face_detector` and `face_landmark_68_tiny` model files from that repo and point `MODEL_URLS` in `index.html` at your own host.

## Known limitations

- **Not true lip-sync detection.** It's reading mouth-movement variance as a proxy for speech, so laughing, chewing, or exaggerated reactions can occasionally register as "talking."
- **Two speakers max**, tracked by proximity — works best for a static two-person setup. A third face in frame (crew, background people) may get picked up as a stray track.
- **Real-time export only.** Processing runs at 1x playback speed since detection and recording happen in the same pass; a 40-minute episode takes ~40 minutes to export.
- **WebM output only** — no in-browser MP4 muxing (browsers can't encode H.264 without licensing complications), so an external re-encode step is needed for MP4.
- Best results with **good, even lighting on both faces** and a **relatively static, wide-enough shot** that both people stay in frame.

## Roadmap ideas

- Two-pass mode (analyze first, then render) to decouple export speed from playback speed.
- Support for more than two speakers.
- Direct MP4 export via `ffmpeg.wasm`.
- Adjustable "shot" presets (e.g. wider two-shot vs. tight single) instead of one continuous crop.
- Optional stereo/multi-track input for cleaner per-speaker audio-based detection.

## Tech stack

- Vanilla JS, HTML, CSS — no framework, no build tooling.
- [face-api.js](https://github.com/justadudewhohacks/face-api.js) (TensorFlow.js) for face detection and landmarks.
- Web Audio API for audio-level gating.
- Canvas 2D + `MediaRecorder` for rendering and export.

## Contributing

Issues and PRs welcome. Since this is a single HTML file, most changes can be tested by just editing and reloading in a browser — no build step to worry about.

## License

MIT — do whatever you want with it. face-api.js and its models are used under their own respective licenses (MIT).
