---
name: browser-hls-transcoding
description: Implement in-browser video transcoding to an adaptive HLS ladder (1080p/720p/480p, CMAF segments + m3u8) using WebCodecs hardware encoders via Mediabunny, streaming each segment to object storage (GCS/S3 or a local stand-in) as it is produced so nothing is kept in the browser. Use when a repo needs client-side video upload + transcoding, HLS generation in the browser, or replacing ffmpeg.wasm for long videos.
---

# Browser HLS transcoding

Turn a user-selected video file (any length, tested to 2h) into an HLS ladder entirely in the
browser, uploading segments as they are encoded. The source file is never uploaded and nothing
is written to browser storage.

## Decide first

- **Use WebCodecs, not ffmpeg.wasm.** ffmpeg.wasm encodes on the CPU in WASM (hours for a 2h
  film × 3 renditions) and keeps files inside a 2–4 GB WASM heap that can't be drained mid-run.
  WebCodecs uses the hardware encoder: measured ~5× realtime for a 1080p/720p/480p ladder on an
  Apple M5 in Chromium.
- **Upload size is the production bottleneck, not encoding.** A ladder is ~2.4× the size of a
  typical source file. At 5× realtime the encoder wants ~40 Mbps of upload; slower links pace the
  job (backpressure, see below). If users are on weak devices or slow uplinks, offer a fallback:
  upload the original and transcode server-side.
- **Codec:** default H.264 (`avc`) for universal playback. HEVC is hardware-encoded on Apple
  Silicon and typically 25–40% smaller, but fails on Firefox, older Android and many TVs. Ask the
  user who the viewers are before switching.
- **Browser support:** Chromium (Chrome/Edge/Brave) and Safari. WebCodecs requires a secure
  context (https or localhost).

## Dependencies (pin exact versions, avoid same-week releases)

```
mediabunny@1.61.6   # WebCodecs conversion + native HLS muxer (HlsOutputFormat)
hls.js@1.7.3        # playback (MSE); Safari can also play natively
client-zip@2.5.1    # optional: server-side streaming ZIP download of a job
```

TypeScript: add `"dom.asynciterable"` to `compilerOptions.lib` if you iterate
`FileSystemDirectoryHandle`.

## Architecture

```
<input type=file>  ──File handle──▶  Dedicated Web Worker
                                      Mediabunny Input(BlobSource)  — reads source lazily
                                      Conversion (fan-out: 3 video encodes + 1 audio)
                                      HlsOutputFormat(CmafOutputFormat)
                                      PathedTarget → BufferTarget per file
                                        onFinalize: PUT bytes → storage, then drop
                                                         │
Server: POST /api/jobs → { jobId, uploadBase }           ▼
        PUT  {uploadBase}{path}   (prod: V4 signed URLs to gs://bucket/videos/{jobId}/)
        GET  {uploadBase}{path}   (prod: CDN)
Player: hls.js loadSource(`${base}/master.m3u8`)
```

Output layout (relative URIs, so the folder works as-is behind any static host/CDN):

```
master.m3u8
1080p/index.m3u8  1080p/init.mp4  1080p/seg-00001.m4s …
720p/…  480p/…  audio/…          job.json  ← written last = completion marker
```

## Implementation

### 1. Rendition ladder — fit inside a 16:9 box, never by short side

A 1920×800 scope film must be "1080p" at 1920×800 (how YouTube/Netflix label it). Planning by
short side wrongly drops the 1080p rung. Scale bitrate by pixel count to keep bits-per-pixel
constant. Portrait sources use 9:16 boxes.

```ts
const LADDER = [
  { label: "1080p", long: 1920, short: 1080, videoBitrate: 5_000_000 },
  { label: "720p",  long: 1280, short: 720,  videoBitrate: 2_800_000 },
  { label: "480p",  long: 854,  short: 480,  videoBitrate: 1_400_000 },
];
export const SEGMENT_DURATION = 6;   // Apple HLS authoring spec
export const KEYFRAME_INTERVAL = 2;  // same on every rendition → aligned segments for ABR
const even = (n: number) => Math.max(2, Math.round(n / 2) * 2);

export function planRenditions(w: number, h: number) {
  const portrait = h > w;
  const boxes = LADDER.map((r) => ({ ...r, bw: portrait ? r.short : r.long, bh: portrait ? r.long : r.short }));
  let rungs = boxes.filter((b) => w >= b.bw * 0.95 || h >= b.bh * 0.95); // tolerate 1920x1072
  if (rungs.length === 0) rungs = [{ ...boxes[boxes.length - 1], label: `${Math.min(w, h)}p` }];
  return rungs.map((b) => {
    const scale = Math.min(1, b.bw / w, b.bh / h); // never upscale
    const width = even(w * scale), height = even(h * scale);
    const ratio = Math.min(1, (width * height) / (b.bw * b.bh));
    return { label: b.label, width, height, videoBitrate: Math.round(b.videoBitrate * ratio) };
  });
}
```

### 2. Worker: one Conversion, fan-out, upload-as-you-go

Run in a dedicated module worker. Bundler syntax (Vite, webpack 5, Next/Turbopack):
`new Worker(new URL("./transcode.worker.ts", import.meta.url), { type: "module" })`.

```ts
import { ALL_FORMATS, BlobSource, BufferTarget, CmafOutputFormat, Conversion, HlsOutputFormat,
  Input, Output, OutputTrackGroup, PathedTarget, Quality, canEncodeAudio, type OutputTrack } from "mediabunny";

async function transcode(file: File, uploadBase: string, signal: AbortSignal) {
  const input = new Input({ source: new BlobSource(file), formats: ALL_FORMATS });
  try {
    const video = await input.getPrimaryVideoTrack();
    if (!video) throw new Error("No video track");
    // Use display dims: they include rotation and pixel aspect ratio.
    const renditions = planRenditions(await video.getDisplayWidth(), await video.getDisplayHeight());

    // One group per video rendition; audio joins all groups. Mediabunny then emits ONE shared
    // #EXT-X-MEDIA audio rendition referenced by every variant (audio stored once, not 3×).
    // Groups are also the only reliable way to name playlists by rendition.
    const groups = renditions.map(() => new OutputTrackGroup());
    const dirOf = (t: OutputTrack) =>
      t.type === "audio" ? "audio" : renditions[groups.indexOf(t.metadata.group as OutputTrackGroup)].label;

    const output = new Output({
      format: new HlsOutputFormat({
        segmentFormat: new CmafOutputFormat(),
        targetDuration: SEGMENT_DURATION,
        getPlaylistPath: ({ tracks }) => `${dirOf(tracks[0])}/index.m3u8`,
        getInitPath: () => "init.mp4",
        getSegmentPath: ({ n, format }) => `seg-${String(n).padStart(5, "0")}${format.fileExtension}`,
      }),
      // Each file is built in RAM and uploaded when complete. Mediabunny AWAITS onFinalize, so a
      // slow network pauses the encoders instead of piling segments up in memory (verified:
      // ≤3 segments in flight on a throttled link).
      target: new PathedTarget("master.m3u8", ({ path }) =>
        new BufferTarget({ onFinalize: (data) => upload(new URL(path, uploadBase), data, signal) })),
    });

    const conversion = await Conversion.init({
      input, output, tracks: "primary",
      video: renditions.map((r, i) => ({
        width: r.width, height: r.height, fit: "fill" as const, codec: "avc" as const,
        quality: new Quality({ bitrate: r.videoBitrate }),
        keyFrameInterval: KEYFRAME_INTERVAL,
        allowTransformationMetadata: false, // MSE ignores rotation metadata; bake it into pixels
        group: groups[i],
      })),
      // AAC copied untouched; anything else re-encoded to AAC. Do NOT set quality/bitrate here,
      // that forces a transcode of AAC sources.
      audio: async (track) => ({
        codec: (await track.getCodec()) === "aac" || (await canEncodeAudio("aac")) ? "aac" : undefined,
        group: groups,
      }),
      showWarnings: false,
    });
    if (!conversion.isValid) throw new Error(conversion.discardedTracks.map((d) => d.reason).join(", "));
    conversion.onProgress = (progress, processedTime) => { /* throttle + postMessage */ };
    await conversion.execute(); // on cancel: abort the signal, then conversion.cancel()

    // Completion marker, uploaded last.
    const manifest = new TextEncoder().encode(JSON.stringify({ sourceName: file.name, renditions }));
    await upload(new URL("job.json", uploadBase), manifest.buffer as ArrayBuffer, signal);
  } finally {
    input.dispose();
  }
}
```

### 3. Uploads that survive hour-long jobs

Wi-Fi drops are expected. Offline time must not count as failure; retry 5xx/429/network errors
with backoff capped at 30 s; give up only after ~20 failures while online. Never retry 4xx.

```ts
async function upload(url: URL, data: ArrayBuffer, signal: AbortSignal) {
  let onlineFailures = 0;
  for (let attempt = 0; ; attempt++) {
    let error: unknown, retryable: boolean;
    try {
      const res = await fetch(url, { method: "PUT", body: data, signal });
      if (res.ok) return;
      error = new Error(`PUT ${url.pathname} → ${res.status}`);
      retryable = res.status >= 500 || res.status === 429;
    } catch (e) {
      if (signal.aborted) throw e;
      error = e; retryable = true;
    }
    if (!retryable) throw error;
    if (navigator.onLine && ++onlineFailures >= 20) throw error;
    // Wait for backoff OR the worker's "online" event, whichever is first; reject on abort.
    await waitToRetry(navigator.onLine ? Math.min(30_000, 1000 * 2 ** attempt) : 30_000, signal);
  }
}
```

Post a `{ type: "network", waiting }` message when any upload starts or stops failing, and show
"connection lost, will resume" in the UI.

### 4. Server contract

Minimum endpoints (local stand-in shown; production replaces PUT/GET with signed URLs + CDN):

| Endpoint | Purpose |
|---|---|
| `POST /api/jobs` | Authenticate user, create `jobId` (UUIDv4), return `uploadBase` (prod: signed URLs scoped to `videos/{jobId}/`, short TTL) |
| `PUT /bucket/{jobId}/{path}` | Store object |
| `GET /bucket/{jobId}/{path}` | Serve object with correct Content-Type |
| `GET /api/jobs/{jobId}` | List objects + parsed `job.json` (null = incomplete) |
| `DELETE /api/jobs/{jobId}` | Remove job (cancel/failure/user delete) |
| `GET /api/jobs/{jobId}/zip` | Optional: stream a STORE zip server-side (client-zip `downloadZip` works in Node) |

Validation the PUT handler must enforce:
- `jobId` matches UUIDv4 and the job exists.
- Each path segment matches `^[A-Za-z0-9_-][A-Za-z0-9._-]{0,63}$` (no leading dot, so no
  `.`/`..`), depth ≤ 2.
- Extension allowlist: `.m3u8` (`application/vnd.apple.mpegurl`), `.m4s` (`video/iso.segment`),
  `.mp4` (`video/mp4`), `.json`. Size cap (64 MB is plenty for 6 s segments).
- Reject writes once `job.json` exists (job immutable).
- Create rendition subdirs with **non-recursive** mkdir: an upload landing after a DELETE then
  fails with ENOENT instead of resurrecting the job. Write to a temp file and rename.
- Delete with retries (`rm(..., { recursive: true, force: true, maxRetries: 3 })`).
- Sweep (or GCS lifecycle rule) incomplete jobs older than 24 h: tabs get closed mid-job.
- Treat uploaded output as untrusted: validate playlists/segments server-side before publishing.

### 5. Browser cleanup (nothing left behind)

- Terminate the worker on done, cancel and error, and create a fresh one per file. This is the
  only hard guarantee that decoders, encoders and GPU frames are released.
- After success, clear the File reference and reset `<input>.value`.
- Send cancel/cleanup `DELETE` with `fetch(..., { keepalive: true })` so it completes if the tab closes.
- Keep the job id in the URL (`?job=`), not in localStorage/OPFS.
- Serve segments with `Cache-Control: no-store` if the requirement is "nothing on disk"
  (a CDN would normally edge-cache them via `s-maxage`).
- `hls.destroy()` on unmount.
- During a job: Screen Wake Lock (re-acquire on `visibilitychange`) and a `beforeunload` guard.

### 6. Playback

```ts
const hls = new Hls(); hls.loadSource(`${base}/master.m3u8`); hls.attachMedia(video);
```

Label quality buttons from the ladder (match `level.width/height` to renditions), not
`${level.height}p`: a 1920×800 rendition would otherwise show as "800p".

## Pitfalls already hit

- **Brave reports a fixed 2³¹-byte storage quota** (anti-fingerprinting) and does not enforce
  it. Don't warn on `quota === 2 ** 31`.
- **Brave disables the File System Access API** (`showDirectoryPicker`/`showSaveFilePicker`).
  Don't rely on it; server-side ZIP via a plain `<a href>` works everywhere.
- Each fan-out rendition gets its own decoder (sinks run in lockstep, sharing the read cache).
  That's acceptable: hardware decode is cheap.
- Encoder pickers and save dialogs need user activation: call them before any `await`.
- `VideoEncoder` is undefined outside secure contexts. Test pages must be https or localhost.
- Untested: whether a WebCodecs encode survives macOS system sleep (lid closed). Advise users to
  stay plugged in with the lid open.

## Verify

Generate fixtures with ffmpeg, then drive a real Chromium-with-codecs browser (Chrome/Brave, not
Playwright's bundled Chromium, which lacks H.264/AAC) via `playwright-core` `executablePath`:

```bash
ffmpeg -f lavfi -i testsrc2=size=1920x1080:rate=30 -f lavfi -i sine=frequency=440 -t 180 \
  -c:v libx264 -preset veryfast -pix_fmt yuv420p -c:a aac test-1080p.mp4
ffmpeg -f lavfi -i testsrc2=size=1920x800:rate=24 -f lavfi -i sine=frequency=440 -t 45 \
  -c:v libx264 -pix_fmt yuv420p -c:a libopus scope.mkv     # widescreen + non-AAC audio
```

Check:
1. `master.m3u8`: shared `#EXT-X-MEDIA:TYPE=AUDIO`, one `STREAM-INF` per rung with CODECS,
   RESOLUTION and BANDWIDTH.
2. Playback starts; forcing the lowest rung changes `video.videoHeight`; seeking to 70% works.
3. Throttle uploads (`context.route` delaying PUTs): encoding speed drops and in-flight
   segments stay ≤ number of playlists.
4. `route.abort("internetdisconnected")` for 25 s, then `context.setOffline(true)` for 20 s:
   the job completes with every segment present.
5. Cancel mid-job: the server job directory is gone and `page.workers().length === 0`.
6. After success: OPFS root is empty, `input.files.length === 0`, and there are 0 workers.
