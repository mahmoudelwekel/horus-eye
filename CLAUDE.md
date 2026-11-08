# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**Horus Eye** is a home CCTV viewer for an LG webOS TV. It has two parts:

1. **Bridge (Windows PC on the LAN):** `go2rtc/go2rtc.exe` pulls RTSP from the NVR (`192.168.1.222:554`) and serves the streams on port 1984 (REST API, WebSocket/MSE, HLS). Each stream is an `exec:` ffmpeg command that transcodes the cameras' HEVC to H.264 and G.711 to AAC, with a 1s keyframe interval (`-g 20`), so browsers and the TV can play it.
2. **TV app (`webos-camera-app/`):** a webOS "web" app titled "Horus Eye" (id `com.home.cameradashboard`, kept unchanged so updates replace the installed app and its saved settings) that finds cameras via go2rtc and plays them in a grid.

Other files: `out/` holds the built IPK; `webos_rsa` is a key (see below).

This is not a git repo, and there is no build system, linter or test suite.

## Sensitive files

- `go2rtc/go2rtc.yaml` contains NVR RTSP URLs with **plaintext credentials**. Never copy them into the app, docs or output; when inspecting the file, mask them (e.g. `sed -E 's#(rtsp://)[^@/ ]*@#\1***:***@#g'`), and edit it with a script rather than reading it whole.
- `webos_rsa` (TV SSH key) is a private key. Don't read, print or move it. The app is not signed; Developer Mode installs unsigned IPKs.

## The bridge

- **Runs at boot** from the scheduled task `\CameraDashboard-Bridge` (as SYSTEM, so it lives in session 0 and a normal account can't see its command line or stop it). Restarting it needs admin. `POST /api/restart` returns 200 but does **not** restart go2rtc on Windows.
- **Bridge PC addresses:** the PC has two LAN addresses. `192.168.1.4` has been stable; the second one changes through DHCP (seen as `.60`, then `.160`). The TV app's saved host must point at an address the TV can reach, so check `ipconfig` when the TV says it can't reach the bridge. A DHCP reservation on the router would fix this.
- **Streams:** each camera has `<Name>` (NVR substream, used in the grid) and `<Name>_fs` (main stream, used only in fullscreen). Stream names are the camera names the TV shows (`_`/`-` become spaces, title-cased). The ffmpeg path in the yaml is an absolute WinGet install path.
- **`preload:`** keeps the three grid streams (Door, Car, Street) connected at all times, so a viewer starts in about 1s instead of 6-10s. Preloads can also be added live with `PUT /api/preload?src=<name>&video&audio`, which rewrites the yaml. The `_fs` streams stay on demand.
- **NVR session limits:** the NVR can't keep all 6 variants open at once. Beyond about 4 simultaneous RTSP streams (3 preloaded substreams + 1 fullscreen `_fs`), it refuses connections and can **crash and reboot** (ports 554 and 80 both refuse for about 1.5 minutes); go2rtc reconnects on its own afterwards. Don't open extra RTSP sessions to the NVR (e.g. ffmpeg/ffprobe tests) while the bridge is running, and avoid anything that churns connections (reconnect loops, switching between protocols, polling scripts).
- **Car (NVR channel 2)** arrives from the NVR at only about 1 fps with gaps, so its live view and snapshots are slow regardless of the app.

```bash
cd go2rtc && ./go2rtc.exe            # manual run only; normally the scheduled task runs it
```

Useful API calls: `GET /api` (version, config path), `GET /api/streams` (producers/consumers per stream; strip URLs before printing), `GET /api/preload`.

## TV app architecture (`webos-camera-app/index.html`)

Everything lives in one file of plain JS with no framework or bundler. `hls.min.js` is a vendored hls.js (1.5.17).

- **Discovery:** polls `GET http://<host>:1984/api/streams` every 15s (`REFRESH_INTERVAL_MS`) and diffs the result against what is shown, adding or removing cells without restarting streams that are already playing. New cells start playback only after `reorderDom()` has placed them, and `reorderDom()` only moves cells that are out of order (see the TV quirks below).
- **Playback chain (`startPlayback`):** WebRTC → MSE → HLS.
  - `startWebRTC`: go2rtc's `/api/ws` signalling. Skipped when `RTCPeerConnection.prototype.addTransceiver` is missing (the TV), and only marked live once frames arrive.
  - `startMse`: go2rtc "MSE over WebSocket" (`/api/ws?src=`, send `{type:"mse", value:<codecs>}`, append fragmented MP4 to a SourceBuffer). This is what the TV uses. It retries "no reply" up to 3 times (a cold or refused camera) and falls back to HLS only if MSE is unsupported. Once go2rtc has replied it reconnects MSE rather than switching protocol. It has a stall watchdog (clock frozen for about 6s) and trims the buffer when it exceeds 20s. `entry.mseLastStop` records why the last session ended.
  - `startHls`: hls.js, the last resort.
- **Persistence:** `localStorage` keys `bridgeHost` (default `DEFAULT_HOST`) and `enabledCameras` (JSON array; `null`/missing means show all).
- **Live tile + snapshots:** the TV decodes only one video at a time, so only the focused tile (`liveName`) plays. `setLive()` stops the previous tile's stream before starting the new one, and `scheduleLive()` waits 400ms after a focus change so arrowing across tiles doesn't churn streams. Every other tile, and the live tile until its video shows frames (the `videoOn` class), shows an `<img class="snap">` from `/api/frame.jpeg?src=<name>`. Snapshots refresh once the previous one arrives (at most 1/s; they often take 1-3s because go2rtc waits for a keyframe). `startPlayback()` refuses any camera that isn't `liveName`, and a 3s supervisor restarts the live tile if it's left with no stream and no restart pending.
- **Layout:** `colsFor(n)` gives 1/2/3/4 columns; `sizeGrid()` sets an explicit pixel height on the grid.
- **Remote-control input:** one `keydown` handler that acts as a state machine across `settingsOpen` → `isFullscreen` → `focusZone` (`"grid"` | `"gear"`). Back is keyCode 461 (webOS), with 10009/27 also accepted. In fullscreen, Left/Right move through `FS_CONTROLS` (mute, pause, close) and OK triggers the selected one; Back also exits.
- The settings overlay keeps its own focus index (0 = host input, 1..N = camera rows, N+1 = Cancel/Save row).

## TV quirks (LG 50UN7240, webOS 5 = Chromium 68)

A desktop browser hides all of these, so check changes on the TV itself.

- **CSS:** no `inset` (use top/right/bottom/left), no flexbox `gap` (use sibling margins; CSS Grid `gap` works). Grid `1fr` tracks need `minmax(0, 1fr)` and an explicit grid height, or rows collapse or overflow.
- **WebRTC:** no `addTransceiver`; the legacy offer options connect but deliver no tracks.
- **One video decoder:** only one `<video>` decodes at a time. With several cameras playing, only one actually plays, which is why the grid uses one live tile plus snapshots.
- **MSE:**
  - It doesn't decode until it gets a seek plus `play()`; the app does this once 0.3s is buffered.
  - It never fires `loadeddata`/`playing`, and `readyState` stays at 1. The app treats decoded frames (`webkitDecodedFrameCount`) as live.
  - It pre-buffers about 3s before its clock starts, then plays about 3s behind live. Seeking at a smaller lag interrupts that pre-buffer and causes stutter, so the app only catches up beyond 6s of lag.
  - Moving a `<video>` in the DOM while its MediaSource attaches cancels the attach.

## Debugging on the TV

With the app running, the TV exposes the Chrome DevTools protocol at `http://192.168.1.10:9998/json` (the TV's address; `ares-inspect` itself times out). Connect to the page's `webSocketDebuggerUrl` and use `Runtime.evaluate` (with `awaitPromise` for promises) to read layout and playback state (`cells`, `cameraNames`, `BRIDGE_HOST`, video `buffered`/`currentTime`/`webkitDecodedFrameCount`). `Page.captureScreenshot` works but doesn't capture the video plane, so video always looks black in screenshots. Changing `BRIDGE_HOST`/`enabledCameras` in the page is temporary; reloading restores the saved settings.

To check UI changes on the desktop, open `index.html` in a browser (arrow keys, Enter and Esc stand in for the remote). The bridge must be reachable at the configured host.

## Packaging & deploying to the TV

Uses the webOS CLI (`@webos-tools/cli`, installed globally). The TV is registered as `MW55TV` (`192.168.1.10`, port 9922).

```bash
ares-package webos-camera-app -o out          # -> out/com.home.cameradashboard_1.0.0_all.ipk
ares-install -d MW55TV out/com.home.cameradashboard_1.0.0_all.ipk
ares-launch -d MW55TV com.home.cameradashboard
```

`ares-launch` occasionally times out; retrying works. Bump `version` in `appinfo.json` to change the IPK filename.

The app is installed through LG Developer Mode. The developer session expires (about 50 hours) unless extended in the TV's Developer Mode app, and the app is removed when it does.
