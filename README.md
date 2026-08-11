# Wavetable Viewer

A browser-based viewer for classic single-cycle wavetable `.WAV` files. Pick a wavetable and see every waveform inside it laid out in a grid, click or arrow-key through them, and scrub through the whole bank with forward/backward/loop/bounce playback.

**[Live demo](https://todbot.github.io/wavetable_viewer/)** — replace `YOUR-GITHUB-USERNAME` once this is published (see [Deploying to GitHub Pages](#deploying-to-github-pages) below).

## Features

- Loads the bundled `samples/` bank automatically, or pick any other local folder of `.WAV` wavetables via the folder picker
- Renders every single-cycle waveform in a wavetable as a small canvas plot in a scrollable grid
- Click a waveform to preview it enlarged and loop its audio; click again to stop
- Arrow keys move the selection through the grid (←/→ step one waveform, ↑/↓ jump a row); Space/Enter toggles playback
- Transport controls to auto-scan through the wavetable: **Forward**/**Backward**, **Loop** (wrap at the ends) or **Bounce** (ping-pong at the ends), with an adjustable time-per-waveform speed slider

## Wavetable format

Each `.WAV` file is standard 16-bit PCM mono audio containing 64 single-cycle waveforms of 256 samples each, back-to-back (16,384 samples / 32,768 bytes of data total). This is the format used by [WaveEdit](https://synthtech.com/waveedit), the open-source wavetable editor for Synthesis Technology's [E352](https://synthtech.com/eurorack/E352/) Eurorack oscillator.

## Running locally

The app fetches `samples/manifest.json` and the individual `.WAV` files, so it needs to be served over `http://` rather than opened directly as a `file://` URL (browsers block `fetch()` of local files for security):

```sh
python3 -m http.server 8000
```

Then open <http://localhost:8000/>. The "Choose a different folder…" button still works when the page is opened directly from disk, letting you browse any local folder of `.WAV` files instead of the bundled samples.

## Samples

The bundled `samples/` directory is copied from [smpldsnds/wavedit-online](https://github.com/smpldsnds/wavedit-online), a community-contributed library of wavetables released under the CC0 1.0 Universal (public domain) dedication. Those wavetables were originally shared by users through the "Online" tab of WaveEdit, via a companion website at **waveeditonline.com** — which is no longer online. This project's bundled copy preserves a snapshot of that library.

## Deploying to GitHub Pages

This is a fully static site (`index.html` + `samples/`), so GitHub Pages can serve it with no build step:

1. Push this repo to GitHub (see commands below).
2. In the repo on GitHub, go to **Settings → Pages**.
3. Under "Build and deployment", set **Source** to "Deploy from a branch", **Branch** to `main`, folder `/ (root)`.
4. Save. The site will publish at `https://YOUR-GITHUB-USERNAME.github.io/wavetable_viewer/`.

A `.nojekyll` file is included so Pages serves the files as-is without running them through Jekyll.

## License

- Viewer code: no license specified yet — add one (e.g. MIT) if you want to make reuse terms explicit.
- Bundled wavetables in `samples/`: CC0 1.0 Universal (public domain), per [smpldsnds/wavedit-online](https://github.com/smpldsnds/wavedit-online).
