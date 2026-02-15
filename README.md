# TP-7 Field Recorder

A web-based prototype that simulates the [Teenage Engineering TP-7](https://teenage.engineering/products/tp-7) field recorder. Built entirely as a single HTML file — no frameworks, no build step, no dependencies.

**[Launch the app](https://harryherbig.github.io/te-tp7/)**

---

## What is this?

A fully functional, skeuomorphic recreation of the TP-7's hardware interface in the browser. Record audio from your microphone, play it back, scrub through it by spinning the reel, manage a library of tapes, and export recordings as WAV files — all from a single self-contained HTML page.

The interface captures the TP-7's distinctive brushed aluminum body, OLED display, oversized spinning disc, and minimal transport controls. Every interaction is designed to feel tactile and physical.

---

## Features

### Recording & Playback
- **Microphone recording** via the Web Audio API — tap the red button, speak, stop
- **Instant playback** with real-time VU meter and waveform visualization
- **Volume control** via the rotary knob (drag to turn) or +/- buttons

### The Spinning Disc
The disc isn't just decorative — it's the primary interaction surface, just like on the real TP-7:

- **Scrub audio** — drag the disc to move through a recording and hear the audio at your scrub speed, like shuttling tape
- **Fast forward / rewind** — flick the disc quickly to jump forward or back. An orange or blue ring lights up to show direction
- **"Off the record" pause** — touch the disc while recording to instantly pause. Lift to resume. The display shows `OFF THE RECORD`
- **Library scrolling** — when the tape library is open, spin the disc to scroll through your tapes

### Tape Library
- **Multiple tapes** — create, name, switch between, and delete recordings
- **Persistent storage** — all recordings are saved to IndexedDB and survive browser refreshes
- **WAV export** — download any tape as a standard 16-bit PCM WAV file
- Open the library by tapping the **M** badge

### Display
- **OLED-style screen** showing current state (TODAY / REC / PLAY / SCRUB / CUE / OFF THE RECORD)
- **Time counter** in minutes, seconds, and centiseconds
- **Track number** for the current tape
- **Status indicators** — red dot for recording, green dot for playback

---

## How to Use

| Action | How |
|---|---|
| **Record** | Tap the red `REC` button. Grant mic permission if prompted. Tap `STOP` to finish. |
| **Play** | Tap the `PLAY` triangle. Audio plays through your speakers/headphones. |
| **Stop** | Tap the `STOP` square. Resets to the beginning of the tape. |
| **Scrub** | Drag the disc left/right during playback or while stopped on a tape. You'll hear the audio as you scrub. |
| **Fast forward / Rewind** | Flick the disc quickly. A colored ring indicates direction. |
| **Pause recording** | Touch and hold the disc while recording. Release to resume. |
| **Volume** | Drag the bottom-right knob, or use the `+` / `-` buttons. |
| **Switch tapes** | Use the `<` `>` arrows, or open the library with `M`. |
| **Open library** | Tap the `M` badge. Spin the disc to scroll. Tap a tape to select it. |
| **New tape** | Open the library and tap `+ New Tape`. |
| **Export** | Open the library, tap the download icon next to a tape. |
| **Delete tape** | Open the library, tap the `x` icon next to a tape. |

---

## Technical Details

### Architecture
Everything lives in a single `index.html` file (~1,400 lines) — HTML structure, CSS styling, and JavaScript application logic. No external dependencies except Google Fonts (Space Mono + Inter).

### Audio Pipeline
```
Microphone → MediaRecorder → Blob → IndexedDB
                                 ↓
                          decodeAudioData
                                 ↓
Playback:    AudioBufferSource → AnalyserNode → GainNode → Destination
Scrub:       AudioBufferSource → BiquadFilter → GainNode(envelope) → GainNode → Destination
```

- **Recording** uses `MediaRecorder` with codec negotiation (WebM/Opus → WebM → OGG/Opus → MP4)
- **Playback** creates a new `AudioBufferSource` each time, connected through an `AnalyserNode` for VU/waveform data
- **Scrub audio** uses granular synthesis — short 60-140ms audio grains with fade envelopes and a low-pass filter to prevent clipping
- **VU meter** reads real-time frequency data from the `AnalyserNode` and renders bars on a canvas
- **WAV export** encodes raw PCM 16-bit audio with a manually constructed RIFF header using `DataView`

### Canvas Rendering
Three `<canvas>` elements run in a `requestAnimationFrame` loop:
- **Disc** — radial gradient, rotating groove lines, center hub with screws, text overlays. Rotation speed smoothly interpolates toward a target with momentum physics (0.95 decay)
- **VU meter** — horizontal bar segments with dB scale labels, smoothed peak tracking
- **Waveform** — pre-computed 200-sample overview with a moving playhead line

### State Machine
```
IDLE ──→ REC ──→ IDLE
  │                ↑ (disc touch pauses/resumes)
  ├──→ PLAY ──→ IDLE
  │      │
  │      └──→ SCRUB ──→ PLAY (on release, resumes)
  │              ↑
  └──────────────┘ (disc drag on a tape in idle enters SCRUB)
```

### Disc Interaction Physics
- Angular velocity is tracked with a 6-sample rolling buffer
- Average velocity above threshold 8 triggers FF/REW flick behavior
- Momentum decays at 0.95× per frame after release, with a 0.05 cutoff
- Degrees-to-seconds mapping: `min(duration/120, 0.03)` seconds per degree

### Persistence
IndexedDB store `tp7` with object store `tapes`. Each tape record:
```js
{ id, name, created, blob, dur, wave }
```
- `blob` — raw audio blob from MediaRecorder
- `dur` — duration in milliseconds
- `wave` — pre-computed 200-point normalized amplitude array for waveform display

### Browser Compatibility
- **Desktop**: Chrome, Firefox, Safari, Edge
- **Mobile**: iOS Safari (requires HTTPS for mic access), Android Chrome, Brave
- Uses `webkitAudioContext` fallback for older WebKit browsers
- Pointer events for unified mouse/touch handling

---

## Run Locally

Just open `index.html` in a browser. No server needed for basic playback.

For **microphone recording**, you need a secure context:
```bash
# Simple HTTP server (mic works on localhost)
python3 -m http.server 8080

# For access from other devices (phone on same WiFi), use HTTPS:
# Option 1: mkcert (recommended)
brew install mkcert
mkcert -install
mkcert localhost 192.168.x.x
python3 -c "
import http.server, ssl, os
os.chdir('$(pwd)')
s = http.server.HTTPServer(('0.0.0.0', 8443), http.server.SimpleHTTPRequestHandler)
c = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
c.load_cert_chain('localhost+1.pem', 'localhost+1-key.pem')
s.socket = c.wrap_socket(s.socket, server_side=True)
s.serve_forever()
"
```

Or just use the hosted version: **https://harryherbig.github.io/te-tp7/**

---

## Credits

Inspired by the [Teenage Engineering TP-7](https://teenage.engineering/products/tp-7). This is an unofficial fan project — not affiliated with Teenage Engineering.
