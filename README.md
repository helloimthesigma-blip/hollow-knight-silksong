# Hollow Knight: Silksong

A webport of the game [Hollow Knight: Silksong](https://hollowknightsilksong.com/)!

Runs from GitHub Pages: <https://helloimthesigma-blip.github.io/hollow-knight-silksong/>

---
# How it works

The 21 large asset bundles can't live in the repo as-is - the biggest is 481 MB
and GitHub rejects files over 100 MB - so they ship as `WebGL.zip` split into
100 parts under `StreamingAssets/aa/`. On first load the page unpacks them into
the browser's Cache API, and every load after that reads from the cache.

Everything is served from the Pages origin, so there is no CDN dependency.

---
# Running it locally

Any static server with byte-range support works, for example:

```bash
python3 -m http.server 8000
```

Then open <http://localhost:8000>. Unity decompresses the `Build/*.unityweb`
files itself, so no special `Content-Encoding` handling is needed.

---
# Notes on running it on a weak machine

Chrome was killing the tab during the first load (*"Aw, Snap!"*, error code 9 -
the renderer being SIGKILLed for memory). Two things caused it, both fixed:

- **Unpacking used to happen in memory.** All 100 parts were downloaded at once
  and held as `ArrayBuffer`s, copied into a `Blob`, then handed to JSZip, which
  loads an entire zip into the heap to parse it - several GB live at once, and
  the largest single entry inflates to 481 MB.

  Now the parts are range-read, and each zip entry is piped through
  `DecompressionStream("deflate-raw")` directly into the Cache API. Nothing
  large is ever held in a JS variable, and JSZip is gone.

- **The wasm asked for 2 GB up front.** `Build/w-pt.wasm.unityweb` now requests
  512 MB at startup instead of 2048 MB. The build has memory growth enabled and
  the 4080 MB maximum is untouched, so it still grows when the game needs it.

If the framerate is poor, lower `RENDER_SCALE` in `index.html` (try `0.75`).
Leaving it at `1` already avoids HiDPI screens rendering at 2x, which is 4x the
pixels.

Requires Chrome 103+ (for `DecompressionStream`).
