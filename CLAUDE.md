# CLAUDE.md

Guidance for working on this repo.

## What this is

A single-file, client-side wavetable viewer. `index.html` contains everything (HTML/CSS/JS inline, no build step, no dependencies). It lists `.WAV` wavetable files, renders every single-cycle waveform inside a selected file as a grid of canvas plots, and plays them back via Web Audio.

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

The "Choose a different folder…" button is independent of this — it uses `showDirectoryPicker()` (or the `<input webkitdirectory>` fallback when that API isn't available, which is also the case for pages opened via `file://` since the File System Access API requires a secure context) to read files directly from disk via `File`/`FileReader`, which works with or without a server.

## Key state and control flow (all in the one `<script>` block)

- `files`: array of `{ name, load: () => Promise<ArrayBuffer> }` — a uniform interface over both the fetch-based default samples and picker-based `File` objects, so `selectFile()` doesn't need to know which source it came from.
- `selectFile(f)` tears down and rebuilds `#content` from scratch on every file switch (`content.innerHTML = ''`), including the preview canvas, transport bar, and the full waveform grid. Global element refs (`previewCanvasEl`, `btnForwardEl`, etc.) get reassigned each time — this is why `stopScan()`/event listeners are re-wired per file rather than once at page load.
- `currentFrames` / `currentCells`: parallel arrays (Int16Array subarrays and their corresponding grid `<div>`s) for the currently displayed WAV. Most navigation/playback functions index into both by the same `i`.
- `selectedIndex`: the single source of truth for "which waveform is selected." Arrow keys, clicks, and the scan transport all funnel through `showFrame(i)` to update it consistently (preview canvas, `.selected` class, scroll-into-view).
- Playback (`playFrame`/`stopPlayback`) is independent of selection — `activeCell`/`activeSource` track what's *audibly playing*, separate from what's *selected*. A waveform can be selected without playing (e.g. after an arrow key press with nothing already playing).
- The scan transport (`startScan`/`stopScan`/`stepScan`) is a `setInterval` loop that advances `selectedIndex` and calls `playFrame` each tick. `scanMode` (`'once' | 'loop' | 'bounce'`) only matters at the array boundaries — see `stepScan()`. `scanMode` and `scanIntervalMs` intentionally persist across file switches (only `scanDirection`/the timer reset via `stopScan()` in `selectFile`), so a user's chosen speed/mode carries over to the next file they pick.
- Every manual interaction (cell click, arrow keys, space/enter) calls `stopScan()` first — without this the scan timer would silently fight with manual navigation.
- `gridColumnCount()` infers the grid's actual column count at runtime by comparing `offsetTop` of cells, since the CSS grid uses `auto-fill` and reflows with window width — there's no fixed column count to hardcode for Up/Down arrow navigation.

## Deployment

Static site, deployed via GitHub Pages from `main` branch root. `.nojekyll` is present so Pages skips the Jekyll build (not needed for a plain static site, and avoids any Jekyll file-handling quirks). No CI/build step exists or is needed — pushing to `main` is the entire deploy.

## Provenance / licensing

`samples/` is a copy of the wavetable bank from [smpldsnds/wavedit-online](https://github.com/smpldsnds/wavedit-online) (CC0 1.0 Universal), originally distributed via the now-defunct waveeditonline.com. See `README.md` for the full attribution — don't change licensing claims there without checking the source repo.

## Testing changes

There's no test suite. After editing the inline `<script>`, at minimum:
1. Syntax-check the extracted JS with `node --check` (regex out the `<script>...</script>` contents to a temp file).
2. Serve locally (`python3 -m http.server`) and manually exercise the change in a browser — this is a UI-heavy app where most bugs are only visible interactively (grid layout, keyboard nav, audio playback).
