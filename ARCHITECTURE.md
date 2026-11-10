# Architecture: image and GIF work without a server-side pipeline

BulkMerge is intentionally small on the server and busy in the browser. The deployed Worker serves HTML, JavaScript, CSS, and fonts. It does not receive image bytes, does not write temporary files, and does not run an image codec. That boundary is the architecture.

The public tools are:

- [Merge Images Online](https://bulkmerge.com/) for stacking, side-by-side, and grid layouts.
- [GIF Resizer](https://bulkmerge.com/tools/gif-resizer) for scaling every frame of an animation.
- [WebP to GIF Converter](https://bulkmerge.com/tools/webp-to-gif) for animated and still WebP input.
- [Compress GIF](https://bulkmerge.com/tools/compress-gif) for fitting an animation under a size limit.

## The request path

A tool page is a React 19 route rendered by TanStack Start on Cloudflare Workers. The route loader selects a tool specification: labels, accepted formats, limits, presets, FAQ copy, and the client runner key. The page renders a drop zone and a set of controls. There is no API route behind the action button.

```
File (disk)
  -> File API
  -> decode (createImageBitmap / gifuct-js / ImageDecoder)
  -> RGBA frames
  -> Canvas 2D or OffscreenCanvas
  -> encode (PNG/JPEG/WebP/GIF)
  -> Blob
  -> download or ZIP
```

The image-processing path makes no network requests. The page still loads its own assets, and the site may load analytics, but none of that carries image bytes. The processing graph is a set of pure-ish functions in `src/tools/local/`; the React component owns the file list, options, progress, errors, and object URLs.

## Decoding an animation

A GIF is not a sequence of full frames. It is a logical screen, a global palette, and a series of image descriptors. Each descriptor can carry a patch rectangle, a local palette, a delay, and a disposal method. `gifuct-js` exposes those pieces; the application composites them into full-canvas RGBA frames.

The compositing loop matters for correctness:

- A non-zero pixel in the patch is copied over the running canvas.
- A transparent pixel leaves whatever was underneath.
- Disposal type 2 ("restore to background") clears the canvas after the frame is captured.
- The delay is already in milliseconds; the UI clamps it to a minimum of 20 ms because browsers do not render faster frames consistently.

Animated WebP takes a different path. The browser's `ImageDecoder` exposes a track with a frame count and per-frame durations. Each frame is decoded to a `VideoFrame`, drawn to a canvas, and converted to RGBA. If `ImageDecoder` is unavailable, the page reports that animated WebP cannot be split rather than quietly returning the first frame.

## Pixels on the right surface

Canvas 2D is the common denominator. The pipeline uses `OffscreenCanvas` when the browser exposes it, and falls back to an HTML canvas otherwise. Both surfaces support `drawImage`, `getImageData`, `putImageData`, and the smoothing controls the tools need. `OffscreenCanvas.convertToBlob` and `HTMLCanvasElement.toBlob` are wrapped behind one `encode` function.

For resizing, the important detail is not the canvas type but the fit mode:

- `contain` scales to the target box and pads the rest with transparency. This is what a Discord emoji needs: 128×128, not 128×96.
- `cover` scales until the box is filled and lets the overflow clip.
- `stretch` maps the source rectangle to the target rectangle without preserving aspect ratio.

Every frame goes through the same path, so the animation canvas and the still-image canvas behave the same way.

## Encoding a GIF

The encoder is `gifenc`, a small JavaScript implementation of palette quantization and LZW compression. The application makes three choices that affect both quality and size:

1. **One palette per file.** Per-frame palettes can flicker when the same colour is quantized differently in consecutive frames. The palette is built from the middle frame, which is usually representative of the whole clip.
2. **`rgb444` quantization.** This is cheaper than `rgb565` and good enough for flat artwork, screenshots, and emoji. The interface lets the visitor choose 256, 128, 64, or 32 colours; 256 is the ceiling, not a requirement.
3. **Delays are data.** `gifenc.writeFrame` receives the frame delay in milliseconds. Dropping frames does not drop time: the kept frame inherits the sum of the delays it replaces, so the clip plays at the same speed with fewer samples.

Compression is a search, not a single pass. The compressor first applies the requested frame drop and scale, then encodes. If the result exceeds the target size, it halves the palette down to 32 colours, and after that scales the animation down by 0.8 and tries again. The loop stops after a few attempts and returns an explanatory error rather than a file that is still over the limit.

## Merging still images

`createImageBitmap` decodes each input once. The merge step then chooses a common dimension:

- Vertical: scale every image to the widest image's width, then stack.
- Horizontal: scale every image to the tallest image's height, then place side by side.
- Grid: fit every image into a common cell, then arrange rows and columns.

The layout math is deliberately literal. A vertical stack's height is the sum of the scaled heights plus gaps. A horizontal row's height is the tallest scaled image, not the sum of rows. A grid is the only layout that uses equal cells. Getting this wrong produces a canvas with large empty bands, which is visually obvious and easy to miss in a thumbnail.

JPEG has no alpha channel. When the export format is JPEG, the merge canvas is filled with white before drawing; otherwise transparent areas would become black.

## ZIP without a dependency

The "Download all as ZIP" button uses a store-only ZIP writer of about 70 lines. It computes CRC-32 for each entry, writes local file headers, a central directory, and the end-of-central-directory record. GIF, PNG, and WebP are already compressed, so deflate would add CPU and code for little gain.

The writer matters for the product promise: the archive is assembled from `Blob` bytes in the tab. No server round-trip is needed to bundle the results.

## Cost model

The server-side alternative looks simple in a diagram and expensive in production:

- A container or image service licensed for GIF/WebP work.
- A queue to absorb bursts and a worker to drain it.
- Object storage for uploads and results, plus lifecycle rules.
- Egress for every download.
- Rate limiting, abuse handling, and cleanup.
- A privacy surface: every uploaded file is now data you have to protect and delete.

BulkMerge removes all of those from the per-file path. The Worker serves a page; the visitor's CPU does the decode, scale, quantize, and LZW encode. The marginal infrastructure cost of a tool run rounds to zero, which is why the tools can stay free and accountless.

This is not a claim that hosting is free. The domain, the edge deployment, the fonts, and the engineering time are real costs. The claim is narrower and more useful: **there is no per-visit image-processing bill**. The expensive resource is the one the visitor already paid for.

## What the browser cannot hide

GIF is an old format. It has 256 colours per file, one-bit transparency, and no inter-frame compression beyond LZW. A photographic clip will band; a soft alpha edge will harden. Re-encoding an already optimized GIF can increase its size because the palette is rebuilt.

The tab is also a finite machine. A hundred full-resolution frames are held in memory while they are processed. The UI processes files one at a time to keep the page responsive, but it cannot make a low-memory device into a render farm. The tool pages state those limits instead of pretending the work is free in every sense.

Those constraints are acceptable for the jobs these tools target: a screenshot merge, a 128×128 emoji, a WebP that needs to become a GIF, and an animation that has to fit under a chat app's upload cap. For everything else, a dedicated desktop encoder is still the right tool.
