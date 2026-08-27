# Dual Camera Viewer

A dependency-free, single-file web app that displays two webcam feeds at the
same time. Each feed has its own device selector and `MediaStream`, so either
camera can be changed or stopped without interrupting the other.

The complete application lives in [`index.html`](index.html).

## Run locally

Camera access is available only in a [secure context](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts).
Use `localhost` during development rather than opening the file directly:

```sh
python -m http.server 8000
```

Then open <http://localhost:8000> in a current version of Chrome, Edge,
Firefox, or Safari. Click **Enable cameras** and accept the browser permission
prompt. A production deployment must use HTTPS.

No build step, package manager, or third-party library is required.

## File structure

`index.html` is intentionally split into three conventional layers:

1. **Styles** - CSS variables define the color system and shared dimensions.
   The camera grid uses two columns on larger screens and one column below
   `760px`. Each video viewport keeps a stable `16 / 9` aspect ratio.
2. **Markup** - The `.camera-grid` contains two `.camera-panel` elements. Each
   panel has a labeled `<select>`, a muted inline `<video>`, an empty/error
   state, and an `aria-live` status line.
3. **Controller** - The module script discovers devices, starts and stops
   streams, updates panel state, and responds to browser and device events.

The JavaScript finds panels with `document.querySelectorAll('.camera-panel')`.
Panel order is therefore significant: panel index `0` corresponds to
`streams[0]`, panel index `1` to `streams[1]`, and so on.

## Runtime state

The controller keeps only two pieces of shared state:

```js
const streams = [null, null];
let cameras = [];
```

- `streams` stores the active `MediaStream` for each panel. This makes cleanup
  explicit and prevents abandoned camera tracks from keeping hardware active.
- `cameras` stores the latest `videoinput` devices returned by
  `navigator.mediaDevices.enumerateDevices()`.

`getControls(index)` is the DOM boundary. It returns the selector, video,
viewport, empty state, and status elements belonging to one panel. The rest of
the controller uses this helper instead of maintaining separate variables for
camera one and camera two.

## Camera lifecycle

The browser deliberately hides useful device labels until camera permission is
granted. The app follows this sequence:

1. The user clicks **Enable cameras**.
2. `enableCameras()` requests a temporary video stream with `getUserMedia()`.
3. The temporary stream is stopped immediately after permission is granted.
4. `refreshDevices()` enumerates devices and retains only `videoinput` entries.
5. `populateSelectors()` fills both selectors while preserving valid previous
   selections.
6. The first two discovered cameras are selected and started in parallel.

When a selection changes, `startCamera(index, deviceId)`:

1. Stops the panel's previous stream.
2. Requests the exact selected device using its `deviceId`.
3. Applies ideal, non-mandatory preferences of 1920x1080 at 30 FPS.
4. Assigns the stream to `video.srcObject` and starts playback.
5. Reads the negotiated settings from the video track for the status line.

Using `ideal` for resolution and frame rate allows the browser to fall back to
values the camera supports. Using `{ exact: deviceId }` ensures the selected
physical device is used or produces a clear error.

## Cleanup and device changes

`stopStream(index)` calls `stop()` on every track before clearing the stored
stream and video source. It runs before replacing a stream and for every panel
on the `pagehide` event.

The app also listens for the `MediaDevices` `devicechange` event. Connecting or
disconnecting a webcam refreshes both selectors and keeps selections that still
exist. If an active camera is physically removed, the browser ends that track;
the selector list is refreshed so another device can be chosen.

## Reusing the code

### Copy the controller into an existing page

Preserve these DOM relationships for each camera panel:

```html
<article class="camera-panel">
  <select aria-label="Camera"></select>
  <div class="viewport">
    <video autoplay playsinline muted></video>
    <div class="empty-state"><p>Waiting for camera</p></div>
  </div>
  <p class="panel-status" aria-live="polite"></p>
</article>
```

The classes are JavaScript hooks, not only styling hooks. If they are renamed,
update the selectors in `getControls()` and the initial `panels` query.

Keep `autoplay`, `playsinline`, and `muted` on each video. Together they avoid
fullscreen playback on iOS and satisfy common autoplay policies for local
media without audio.

### Extract into a module

For a component-based project, move the module script into a file such as
`dual-camera.js` and export the operations that the host needs:

```js
export { enableCameras, refreshDevices, startCamera, stopStream };
```

Initialize `panels` only after the host framework has mounted the panel DOM.
In React, Vue, or Svelte, retain stream objects outside render state and call
`stopStream()` from the component cleanup/unmount lifecycle.

### Change the number of feeds

Add or remove `.camera-panel` elements and size the registry from the DOM:

```js
const panels = [...document.querySelectorAll('.camera-panel')];
const streams = Array(panels.length).fill(null);
```

The current app uses a fixed two-entry registry because its UI guarantees two
panels. Browsers and operating systems may limit how many cameras can stream
simultaneously, especially at high resolutions.

### Change capture constraints

Edit the `video` constraints inside `startCamera()`:

```js
video: {
  deviceId: { exact: deviceId },
  width: { ideal: 1280 },
  height: { ideal: 720 },
  frameRate: { ideal: 60 }
}
```

Add `audio: true` only when sound is required. Doing so changes the permission
request and requires explicit handling to avoid feedback when multiple live
videos play through speakers.

## Web APIs used

- `navigator.mediaDevices.getUserMedia()` requests permission and streams.
- `navigator.mediaDevices.enumerateDevices()` lists available cameras.
- `navigator.mediaDevices.addEventListener('devicechange', ...)` observes
  hardware changes.
- `MediaStreamTrack.getSettings()` reports the negotiated resolution and FPS.
- `MediaStreamTrack.stop()` releases camera hardware.
- `HTMLVideoElement.srcObject` attaches a stream without creating object URLs.

## Integration considerations

- Camera permission must begin from a user action. Do not automatically call
  `enableCameras()` on page load.
- Device IDs can change between browsing sessions and should not be treated as
  permanent hardware identifiers.
- Embedded apps need an iframe `allow="camera"` permission policy, and the
  containing site must permit camera access through its HTTP headers.
- Two selectors may choose the same device, but some browsers or camera drivers
  cannot open one physical device twice. A host app can prevent duplicate
  selections if that behavior is undesirable.
- The application never records, uploads, or sends video data. Adding capture,
  recording, WebRTC transport, or frame analysis introduces separate privacy,
  storage, and cleanup responsibilities.

## Error handling

`describeError()` translates common `DOMException` names into user-facing
messages. Integrations can replace `showState()` with their own notification
system while retaining the error mapping. Common cases include denied
permission, disconnected hardware, another application holding the camera,
unsupported constraints, and an insecure origin.