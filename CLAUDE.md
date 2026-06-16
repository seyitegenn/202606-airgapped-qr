# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the App

No build step. Open the HTML files directly in a browser:

```sh
open generator.html   # sender interface
open scanner.html     # receiver interface
open index.html       # landing page / docs
```

`scanner.html` requires camera access and must run in a browser that grants `getUserMedia` — open via `file://` or a local server. Chrome and Firefox are supported.

## Architecture

Three standalone HTML files with no shared modules, build system, or package manager. All logic lives inline via `<script>` tags using CDN-loaded libraries.

**generator.html** — Sender side:
1. Reads a file, compresses it with `pako.gzip` (level 9)
2. Splits the compressed bytes into 250-byte chunks
3. Encodes each chunk as `{index},{base64}` and renders it as a QR code via `qrcode.js`
4. Cycles through QR codes every 50ms for the receiver to scan

**scanner.html** — Receiver side:
1. Opens the device camera via `getUserMedia` (rear-facing preferred)
2. Draws video frames to an off-screen canvas every 10ms
3. Passes canvas image data to `zbar-wasm` for QR decoding
4. First QR is a JSON metadata frame `{name, chunks}`; subsequent frames are `{index},{base64}` data chunks
5. Once all chunks arrive, reassembles them in order, decompresses with `pako.inflate`, and triggers a file download

**QR chunk payload format:**
- Metadata frame: `{"name":"filename.ext","chunks":N}` (detected by presence of `"chunks"` key)
- Data frames: `{index},{base64encoded}` (detected by comma-split length == 2)

## Key Constraints

- `chunk_size = 250` bytes (post-compression) per QR code — QR type 40 with error correction level M
- QR dimensions scale to `min(window.innerWidth, window.innerHeight) / 1.5`
- The scanner holds decoded chunks in a plain object keyed by index; re-scanning a chunk overwrites the previous value harmlessly
- `scanner.html` initializes the canvas size 2 seconds after camera starts to allow video metadata to settle — adjusting this delay affects startup reliability
