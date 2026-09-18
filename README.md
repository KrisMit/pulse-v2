# Invisible Threads — Biometric Experience

A single-page, phone-camera-based biometric companion for the *Invisible Threads* exhibition. Each physical painting is paired with a QR code linking to its own instance of this page. Visitors place a fingertip over the phone's rear camera and flash; the page measures their pulse in real time (PPG) and, once a heartbeat is locked in, lets them "enter" the painting through the video variant that matches their current biometric state — **calm**, **neutral**, or **elevated**.

No app, install, or backend required — it's a single self-contained HTML file that runs entirely in the mobile browser.

## How it works

1. **QR code → painting.** Each painting's QR code links to this page with a `?painting=N` query parameter (see [Painting config](#painting-config)). If the ID doesn't match a configured painting, a "Painting not found" screen is shown instead.
2. **Onboarding overlay.** A full-screen intro (English, with Italian subtext) instructs the visitor to hold the phone upright and cover the rear camera and flash with their index finger, then rotate to landscape once the painting opens.
3. **Camera + torch as a pulse sensor.** Tapping **Begin** requests the rear camera via `getUserMedia` (with a constraint fallback chain that avoids `facingMode: { exact: 'environment' }`, since some Samsung devices reject it) and repeatedly attempts to turn on the torch, since torch capability isn't always reported correctly.
4. **PPG signal processing.** Once the feed is live, each animation frame is drawn to an offscreen `<canvas>`, and the average red-channel intensity across the frame is sampled. A finger covering the lens produces a periodic rise and fall in that signal as blood pulses through the fingertip.
5. **Peak detection → BPM.** A rolling average of recent samples is used to detect peaks (signal exceeding the average by a small margin, with a minimum interval between peaks to reject noise). Peak-to-peak timing is converted to BPM, filtered to a plausible physiological range, and smoothed with a median filter over the last 10 detected beats.
6. **State classification.** Once enough consistent beats have been read, the median BPM is bucketed into a state:
   - `< 70 BPM` → **calm**
   - `70–90 BPM` → **neutral**
   - `> 90 BPM` → **elevated**
7. **Enter Painting.** Once the heartbeat is "locked," an **Enter Painting** button appears. Tapping it stops the camera stream and opens a fullscreen video (a Vimeo embed) — the variant of that painting's video corresponding to the visitor's current state.
8. **Reset for next visitor.** Closing the video resets all pulse state and automatically restarts the camera/calibration flow, ready for the next person.

## Live feedback UI

While reading pulse, the page shows:
- A live BPM counter, color-coded by state (blue for calm, warm gold for neutral, red for elevated)
- A state label (`resting · calm state`, `present · neutral state`, `elevated · activated state`)
- A pulsing pink/red ring around the camera preview and a full-screen red flash, both triggered on each detected beat
- A scrolling ECG-style waveform trace driven by the live red-channel signal
- Status text guiding the visitor (`waiting for finger`, `cover lens with finger`, `no torch — cover flash with finger`, `heartbeat locked`)

## Painting config

Painting names and per-state video URLs are defined in the `PAINTINGS` object near the top of the `<script>` block:

```js
const PAINTINGS = {
  1: {
    name: "Painting I",
    calm:     "https://player.vimeo.com/video/XXXXXXX?autoplay=1&loop=1&title=0&byline=0&portrait=0",
    neutral:  "https://player.vimeo.com/video/XXXXXXX?autoplay=1&loop=1&title=0&byline=0&portrait=0",
    elevated: "https://player.vimeo.com/video/XXXXXXX?autoplay=1&loop=1&title=0&byline=0&portrait=0"
  },
  // ...
};
```

The page currently ships with 8 paintings configured. To add, remove, or update a painting, edit this object directly — the `painting` query-string value (`?painting=3`) is the object key. BPM thresholds are configurable via the `CALM_MAX` and `ELEVATED_MIN` constants just below `PAINTINGS`.

> **Known issues to fix before all QR codes go live:**
> - **Painting V** URLs point to `vimeo.com/manage/videos/...` (private "manage" links, viewable only when logged into the owning Vimeo account) instead of public `player.vimeo.com/video/...` embed links — these need to be swapped out.
> - **Painting VI** URLs have a double slash (`.../video//1205554722...`) which will likely 404 in the embed — should be a single slash.

## Usage

1. Host the HTML file on any static web host over **HTTPS** (camera access requires a secure context; `localhost` also works for testing).
2. Generate one QR code per painting, each linking to:
   ```
   https://your-domain.example/index.html?painting=1
   https://your-domain.example/index.html?painting=2
   ...
   ```
3. Display each QR code alongside its corresponding physical painting.
4. Visitors scan the code, tap **Begin**, grant camera permission, and cover the rear camera with a fingertip until their heartbeat locks in.

## Browser & device requirements

- A modern mobile browser with `getUserMedia` support (Chrome/Safari on Android and iOS)
- HTTPS hosting
- Rear camera access; torch/flash support meaningfully improves signal quality but isn't required (the UI falls back to "cover flash with finger" using ambient/finger-covered light)
- Camera constraint handling was specifically tuned around Samsung devices (e.g. Galaxy A52), which can reject `exact: 'environment'` constraints or misreport torch capability

## Privacy

All pulse detection and video processing happens locally in the browser via `getUserMedia` and `<canvas>` — no camera frames or biometric readings are ever uploaded, transmitted, or stored. The only network activity after calibration is loading the chosen Vimeo embed.
