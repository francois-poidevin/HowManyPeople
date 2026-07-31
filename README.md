# HowManyPeople

A website that counts people visible in a live public street camera using AI inference in the browser.

Hosted on GitHub Pages: https://francois-poidevin.github.io/HowManyPeople/

---

## Camera source

**Central Park South @ Columbus Circle East** — NYC DOT / 511NY public camera network

- Camera ID: 3356
- Endpoint: `https://511ny.org/map/Cctv/3356`
- Hardware: Axis Q6055-E
- CORS: `Access-Control-Allow-Origin: *` (confirmed)
- Format: JPEG snapshot, refreshed every ~60s by the camera, served via CloudFront CDN (max-age=60s)

The 511NY network exposes hundreds of public traffic cameras across New York State. City-level cameras (NYC DOT) serve JPEG snapshots; state highway cameras serve live HLS streams.

---

## How real-time AI analysis works

The key property is `Access-Control-Allow-Origin: *` on the JPEG endpoint.

Setting `crossorigin="anonymous"` on the `<img>` element tells the browser to make a CORS request. Because the server responds with `Access-Control-Allow-Origin: *`, the browser does **not** taint the canvas when `canvas.drawImage(img)` is called. TF.js can then call `model.detect(canvas)` to read pixel data.

```
511NY JPEG endpoint (CORS: *)
        │
        ▼
<img crossorigin="anonymous">     ← browser fetches with CORS, canvas stays untainted
        │
        ▼
canvas.drawImage(img)             ← works, no SecurityError
        │
        ▼
model.detect(canvas)              ← TF.js COCO-SSD MobileNetV2
        │
        ▼
people counter + bounding boxes   ← updated every ~60s (camera refresh rate)
```

The page polls the endpoint every 30 seconds. The CDN serves a cached frame for up to 60 seconds, so each poll either returns the current frame or a fresh one as soon as the camera updates.

## Why not a live video stream?

The Central Park South camera only provides JPEG snapshots — there is no HLS stream for it. NYC DOT city-level cameras are snapshot-only; NYSDOT state highway cameras have live HLS streams.

For live HLS with real-time per-frame inference, the NYSDOT highway cameras (e.g. `s9.nysdot.skyvdn.com`) also have `Access-Control-Allow-Origin: *` and can be used with a `<video crossorigin="anonymous">` element + `canvas.drawImage(video)` + `model.detect(canvas)` in a `requestAnimationFrame` loop.

## AI model

- **TF.js COCO-SSD MobileNetV2** — runs entirely in the browser, no server
- WebGL backend (GPU) preferred, CPU fallback
- Confidence threshold: 0.30
- No GitHub Actions, no server, no proxy required
