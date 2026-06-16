# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## GitHub Pages Deploy

The site is served from the `docs/` folder. `docs/index.html` is a one-line meta-refresh redirect to `docs/scanner.html`, which is the deployable copy of `scanner.html`.

GitHub Pages only serves files inside `docs/` — paths like `../scanner.html` do not work. When `scanner.html` is updated, `docs/scanner.html` must be updated to match.

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

- Chunk size is user-configurable (100–500 bytes, default 250), set at file-selection time and fixed for that file's lifetime — retry transfers must use the same `used_chunk_size` stored when the file was compressed
- QR dimensions scale to `min(window.innerWidth, window.innerHeight) / 1.5`; the QRCode object is destroyed and recreated (`innerHTML = ''` + `new QRCode(...)`) whenever error correction level changes
- The scanner holds decoded chunks in a plain object keyed by index; re-scanning a chunk overwrites the previous value harmlessly
- `scanner.html` initializes the canvas size 2 seconds after camera starts to allow video metadata to settle — adjusting this delay affects startup reliability
- QR type 40 is always used regardless of error correction level

## Generator Configuration

`generator.html` exposes three user-controlled parameters (Vue refs, all configurable before transfer):

| Ref | Default | Range | Effect |
|-----|---------|-------|--------|
| `error_correction` | `'M'` | L / M / Q / H | QR complexity; L = simplest, easiest phone scan |
| `chunk_delay_ms` | `50` | 20–500ms | Pause between QR frames |
| `chunk_size_config` | `250` | 100–500 bytes | Bytes per chunk; smaller = simpler QR |

File is compressed immediately on selection (`on_file_selected`) via `process_file(file, chunk_size)`, not on transfer start. This means `compressed_data`, `total_chunks_count`, and `used_chunk_size` are all populated before the user clicks Start Transfer. Changing `chunk_size_config` while a file is loaded triggers a `watch` that calls `process_file` again automatically — no re-selection needed.

## Partial Retry (generator.html)

The range input accepts print-dialog syntax: `"5-20"`, `"3,7,15"`, `"0-5,12,18-24"`. Parsed by `parse_range(input, max_index)` which returns a sorted array of unique valid indices. Empty input sends all chunks.

The metadata QR (`{name, chunks}`) is always sent first regardless of range — the receiver needs total chunk count to track completion.

To retry missed chunks: Stop Transfer → type missing indices into the range field → Start Transfer again. No file re-selection needed as long as `chunk_size_config` hasn't changed.

## Missing Chunk Display (scanner.html)

`missing_chunks` is a Vue `computed` that diffs `file_metadata.chunks` against `Object.keys(decoded_chunks)`. Updates reactively as chunks arrive. The "Kopyala" button writes `missing_chunks.value.join(',')` to the clipboard — paste directly into generator's range field.

Displayed below the video element. Starting the receiver does NOT clear previously received chunks — only a new file (different name or chunk count in the metadata QR) resets `decoded_chunks`. Stop receiver performs a full reset.
