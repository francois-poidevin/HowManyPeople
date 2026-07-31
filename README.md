# HowManyPeople

A website to count the number of people in a live webcam video stream, using AI inference directly in the browser.

The live webcam shows **Piazza di Spagna, Rome** (via SkylineWebcams). TensorFlow.js runs a COCO-SSD object detection model on each video frame and draws bounding boxes around detected people, with a real-time counter displayed on screen.

Hosted on GitHub Pages: https://francois-poidevin.github.io/HowManyPeople/

---

## Token & CORS Workaround

### The problem

The HLS video stream served by SkylineWebcams (`hd-auth.skylinewebcams.com`) requires a short-lived **authentication token** embedded in the stream URL:

```
https://hd-auth.skylinewebcams.com/live.m3u8?a=<token>
```

This token is not public — it is injected dynamically into the SkylineWebcams page HTML by their backend, inside a JavaScript player initialization block:

```js
source:'livee.m3u8?a=itdg8gvbcfj3rf5cn5g56rq3v5'
```

To obtain the token, the page HTML must be fetched and parsed. This is straightforward server-side, but from a browser it hits two walls:

1. **CORS** — browsers block cross-origin reads of `skylinewebcams.com` because that domain does not include `francois-poidevin.github.io` in its `Access-Control-Allow-Origin` header.
2. **Public CORS proxies are blocked** — services like `allorigins.win`, `corsproxy.io`, and `cors.sh` all return empty responses or errors when the request carries `Origin: https://francois-poidevin.github.io`. They rate-limit or block GitHub Pages domains.

### The solution

The workaround splits the token fetch into two layers:

#### Layer 1 — `token.json` (primary, always works)

A GitHub Actions workflow (`.github/workflows/refresh-token.yml`) runs **every 30 minutes** on a GitHub-hosted runner. Runners are not browsers — they send no `Origin` header and face no CORS restriction. The workflow:

1. Fetches the SkylineWebcams page with `curl`
2. Extracts the token with a regex (`grep -oP`)
3. Writes the result to `token.json` at the repo root:
   ```json
   { "token": "itdg8gvbcfj3rf5cn5g56rq3v5", "updated": "2026-07-31T10:07:21Z" }
   ```
4. Commits and pushes the file

Because `token.json` lives in the same GitHub Pages site, the browser fetches it as a **same-origin request** — no CORS header needed, no proxy needed.

#### Layer 2 — proxy fallback (backup)

If `token.json` is missing or stale, the page falls back to a chain of public CORS proxies. These may or may not work depending on the proxy's current policy toward GitHub Pages origins, but they provide a best-effort safety net.

```
token.json (same-origin)  →  proxy 1  →  proxy 2  →  proxy 3  →  error
```

#### Why the token expires

SkylineWebcams rotates tokens server-side. A token is typically valid for around 30–60 minutes. The 30-minute Action schedule ensures the committed token is always fresh enough for a visitor loading the page.
