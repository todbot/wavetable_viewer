# CLAUDE.md

Guidance for working on this repo.

## What this is

A single-file, client-side wavetable viewer. `index.html` contains everything (HTML/CSS/JS inline, no build step, no dependencies). It lists `.WAV` wavetable files, renders every single-cycle waveform inside a selected file as a grid of canvas plots, plays them back via Web Audio, auto-scans through a wavetable (forward/backward/stop, loop/bounce, adjustable speed), and encodes the current view (file, frame, scan state) into the URL hash so links are shareable.

`samples/` holds 705 bundled `.WAV` files plus `samples/manifest.json` (a flat JSON array of filenames, since browsers can't list a directory over `fetch`). The manifest was generated once with a throwaway Python script; regenerate it the same way if files are added/removed from `samples/`:

```python
import json, os
names = sorted(f for f in os.listdir('samples') if f.lower().endswith('.wav'))
json.dump(names, open('samples/manifest.json', 'w'), indent=0)
```

## Wavetable file format

Every bundled `.WAV` is identical in structure: standard 44-byte RIFF/WAVE header, 16-bit PCM, mono, 44.1kHz, followed by exactly 32,768 bytes of data — 16,384 samples — laid out as **64 single-cycle waveforms of 256 samples each**, back-to-back. This is the format used by WaveEdit / the E352 Eurorack module. `FRAME_SAMPLES = 256` in `index.html` encodes this; frame count is derived (`floor(samples.length / 256)`) rather than hardcoded to 64, so files with a different total length still work.

`parseWav()` walks RIFF chunks generically (looks for `fmt `/`data` by ID rather than assuming fixed byte offsets), so it isn't strictly dependent on this exact layout — but it does assume 16-bit PCM and will throw for other bit depths.

## Why the app needs to be served over HTTP

`loadDefaultSamples()` uses `fetch('samples/manifest.json')` and then `fetch('samples/' + name)` per file. Browsers block `fetch()` of `file://` URLs for security, so opening `index.html` directly (double-click) will fail to auto-load the bundled samples — you'll just see the "Bundled samples unavailable" status message. Always test with a local server:

```sh
python3 -m http.server 8000
```

The "Choose a different folder…" button (now in the sidebar, above the file list) is independent of this — it uses `showDirectoryPicker()` (or the `<input webkitdirectory>` fallback when that API isn't available, which is also the case for pages opened via `file://` since the File System Access API requires a secure context) to read files directly from disk via `File`/`FileReader`, which works with or without a server.

## Layout

Header is minimal: title + "View on GitHub" link. The folder-picker button and the bundled-file-count status live at the top of the sidebar (`#sidebar`), above the filter input and file list — they used to be in the header but were moved down to keep the header clean for the social-card screenshot. `#soundHint` (the "🔊 Click for sound" pill) is a fixed-position element outside both, at the top-center of the viewport.

## Key state and control flow (all in the one `<script>` block)

- `files`: array of `{ name, load: () => Promise<ArrayBuffer> }` — a uniform interface over both the fetch-based default samples and picker-based `File` objects, so `selectFile()` doesn't need to know which source it came from.
- `selectFile(f)` tears down and rebuilds `#content` from scratch on every file switch (`content.innerHTML = ''`), including the preview canvas, transport bar, share button, and the full waveform grid. Global element refs (`previewCanvasEl`, `btnForwardEl`, `btnShareEl`, etc.) get reassigned each time — this is why `stopScan()`/event listeners are re-wired per file rather than once at page load.
- `currentFrames` / `currentCells`: parallel arrays (Int16Array subarrays and their corresponding grid `<div>`s) for the currently displayed WAV. Most navigation/playback functions index into both by the same `i`.
- `selectedIndex`: the single source of truth for "which waveform is selected." Arrow keys, clicks, and the scan transport all funnel through `showFrame(i)` to update it consistently (preview canvas, `.selected` class, scroll-into-view, and the URL hash via `updateHash()`).
- Playback (`playFrame`/`stopPlayback`) is independent of selection — `activeCell`/`activeSource` track what's *audibly playing*, separate from what's *selected*. A waveform can be selected without playing (e.g. after an arrow key press with nothing already playing).
- The scan transport (Backward/Stop/Forward icon buttons, Loop, Bounce, Speed slider) is a `setInterval` loop (`startScan`/`stopScan`/`stepScan`) that advances `selectedIndex` and calls `playFrame` each tick. `scanMode` (`'once' | 'loop' | 'bounce'`) only matters at the array boundaries — see `stepScan()`. `scanMode` and `scanIntervalMs` intentionally persist across file switches (only `scanDirection`/the timer reset via `stopScan()` in `selectFile`), so a user's chosen speed/mode carries over to the next file they pick.
- `stopScan()` also calls `stopPlayback()` — it's the one function that fully silences audio and resets scan state together. **Gotcha:** every manual interaction (cell click, arrow keys, space/enter) calls `stopScan()` at the start of its handler, purely to stop any competing auto-scan before doing its own thing. This previously caused a real bug (see "Audio autoplay unlock" below) where a source started by one handler got silently killed by `stopScan()`'s `stopPlayback()` call fired moments later by a sibling handler on the same gesture. If you add new interaction handlers that call `stopScan()`, be aware it has this side effect now, not just "stop the timer."
- `gridColumnCount()` infers the grid's actual column count at runtime by comparing `offsetTop` of cells, since the CSS grid uses `auto-fill` and reflows with window width — there's no fixed column count to hardcode for Up/Down arrow navigation.

## URL state / sharing

`updateHash()` writes `#file=<name>&frame=<i>&dir=<1|-1>&mode=<loop|bounce>&speed=<ms>` via `history.replaceState` (never `pushState`, and never plain `location.hash =`) — this keeps the address bar live without spamming browser history or firing `hashchange`. `dir`/`mode`/`speed` are omitted when at their defaults to keep plain view-and-select links short. It's called from `showFrame()`, `startScan()`, `stopScan()`, and the Loop/Bounce/Speed control handlers — anywhere the shareable state changes.

`parseHash()` is the inverse, read once in `loadDefaultSamples()` after the file list loads. If the hash names a bundled file, it's auto-selected, jumped to the given frame, and — if `dir` was set — scanning starts automatically.

The **Share** button (top-right of the preview panel) copies `location.href` via the Clipboard API (`copyShareLink()`, with an `execCommand('copy')` fallback via `fallbackCopy()` for contexts where that's unavailable), since the hash rewrites live during scanning and can't be reliably selected by hand.

## Audio autoplay unlock

Browsers refuse to run audio without a prior user gesture, so a shared link with `dir` set arrives with the grid visibly scanning but silent — `audioCtx` is created `suspended`. `unlockAudio()` (wired to `document`'s `pointerdown`/`keydown`) calls `audioCtx.resume()` on the first interaction, and `soundHint` (the "🔊 Click for sound" pill) is shown while armed and hidden the instant `unlockAudio()` fires.

**Do not** try to replay audio inline inside `unlockAudio()` or inside `resume().then(...)`. That was the original approach and it was broken: `unlockAudio` runs on `pointerdown`, which fires *before* the `click`/`keydown` handler that follows it in the same gesture — and that handler almost always calls `stopScan()` (see the gotcha above), which kills whatever source `unlockAudio` just started, moments after it started it, before the context even finishes transitioning to `running`. The result looked like a Safari/Firefox-specific autoplay quirk but reproduced identically in Chrome once actually verified (checking `audioCtx.state` alone isn't enough — it can read `running` while the source that mattered has already been silently stopped).

The actual fix: `playFrame()` attaches `audioCtx.onstatechange` once, when the context is first created. It re-triggers `playFrame()` for whatever's currently selected (`activeCell` non-null, `selectedIndex >= 0`) whenever the context reaches `running` — decoupled from whichever gesture handler happened to fire and in what order. If audio-unlock bugs resurface, verify with actual state, not assumptions: check `activeSource` truthiness *and that it stays truthy a second or two later*, not just that `audioCtx.state === 'running'` once.

## Testing changes — including with a real (headless) browser

There's no test suite. After editing the inline `<script>`, at minimum:
1. Syntax-check the extracted JS with `node --check` (regex out the `<script>...</script>` contents to a temp file).
2. Serve locally (`python3 -m http.server`) and manually exercise the change in a browser.

For anything involving async/timing/audio behavior, manual eyeballing isn't enough — verify programmatically. This environment has no browser-automation extension available, but if the machine has any Chromium-family browser installed (Chrome, Chrome Canary, etc.), it can be driven headlessly via the raw DevTools Protocol with **zero npm installs** — Node ≥18 has native `fetch`; Node ≥22 has native `WebSocket`:

```sh
"/Applications/Google Chrome Canary.app/Contents/MacOS/Google Chrome Canary" \
  --headless=new --remote-debugging-port=9333 --window-size=1200,800 \
  --hide-scrollbars --disable-gpu about:blank &
curl -s http://localhost:9333/json/version   # confirms it's up, gives webSocketDebuggerUrl
```

Then from a `.mjs` script: `PUT http://localhost:9333/json/new?<url>` to open a tab at a target URL (returns that tab's own `webSocketDebuggerUrl`), connect a `WebSocket` to it, and send CDP commands as `{id, method, params}` JSON, matching responses by `id`. Useful methods used so far: `Runtime.evaluate` (poll app state, e.g. `document.querySelectorAll('#grid .cell').length` to know the grid actually rendered), `Input.dispatchMouseEvent` (mousePressed+mouseReleased — CDP-dispatched input is treated as a **trusted** gesture, so this is how the autoplay-unlock fix above was actually verified, not just eyeballed), `Emulation.setDeviceMetricsOverride` (fixed viewport/DPR for screenshots), and `Page.captureScreenshot`. Always `pkill -f "remote-debugging-port=9333"` when done.

This is also how `og-image.png` was produced (viewport set to exactly 1200×630 @2x, whole app captured with the sidebar visible) — regenerate it the same way if the layout changes and the social card goes stale.

## Deployment

Static site, deployed via GitHub Pages from `main` branch root. `.nojekyll` is present so Pages skips the Jekyll build. No CI/build step exists or is needed — pushing to `main` is the entire deploy.

`index.html` has Open Graph / Twitter Card meta tags pointing at `og-image.png` (root of the repo, absolute URL `https://todbot.github.io/wavetable_viewer/og-image.png`). If the title, tagline, or `og:url` ever change, update both the meta tags and regenerate the image to match — they're independent and won't warn you if they drift apart.

## Provenance / licensing

`samples/` is a copy of the wavetable bank from [smpldsnds/wavedit-online](https://github.com/smpldsnds/wavedit-online) (CC0 1.0 Universal), originally distributed via the now-defunct waveeditonline.com. See `README.md` for the full attribution — don't change licensing claims there without checking the source repo.
