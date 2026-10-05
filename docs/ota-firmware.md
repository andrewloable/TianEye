# OTA Firmware Updates

Static analysis of Yoosee 6.46.1. Nothing here has been confirmed against live traffic yet.

## Summary

**The camera downloads its own firmware.** The app only tells it to check and to install.
The two commands carry no URL, so the camera must get the download location from
its own update server. To find that host, or to block or redirect updates, capture the
camera's traffic (te-2dc). Capturing the app's traffic is not enough.

## IoTVideo cameras

Both commands are `TAKE_ACTION` model commands (see [camera-control.md](camera-control.md)); on the
wire they are type-`0xAA` data-object frames carrying the action path + JSON value
([protocol-iotvideo.md](protocol-iotvideo.md#device-model-commands-settings-actions)):

| Path | Payload | Meaning |
|------|---------|---------|
| `Action._otaVersion` | `{"stVal":""}` | Ask the camera for the newest firmware version it can install |
| `Action._otaUpgrade` | `{"stVal":1}` | Start the upgrade |

**Replies**

- `_otaVersion` returns `{"stVal": "<version string>", "t": <time>}` (Java `ActionStrValue.version`). The app stores this as the
  *available* version (the same store that backs `getNewDevVersion`) and shows it in the update
  dialog. The camera reports its current version separately in device properties.
- `_otaUpgrade` progress arrives as `{"stVal": <int>, "t": <time>}` (Java `ActionIntValue.intVal`):

| intVal | Meaning in the app |
|--------|--------------------|
| 0–99 | progress (the app rewrites 80 as 40 and multiplies by 3.3 for its progress bar) |
| 100 | done; the app re-reads the device version to confirm |
| -33 | battery too low (the threshold is a per-device setting, 25% by default) |
| other < 0 | failed; the code is shown to the user |

**Timing (app side)**

- Pressing "update" sends `_otaVersion`. If the camera does not acknowledge within 10 s, the app
  fails the update with its own code -2023. Battery cameras get a 30 s wake-up first.
- Once `_otaUpgrade` is acknowledged, the app starts a 20-minute watchdog. Every 30 s, if progress
  has not changed for over a minute (60,000 ms), it re-reads the version. At 20 minutes it declares success if
  the version changed, otherwise failure.

## App-side version lookup (display only)

A separate, `@Deprecated` path in the app asks for release info through the IoTVideo SDK's
HTTP-over-P2P proxy:

```
POST sdk/dev/upg/app/queryVersion     fields: tid, forceVersion, language, currentVersion
POST sdk/dev/upg/app/queryVersions    field:  devs (list, for many devices)
→ {"data": {"version": "...", "downUrl": "...", "upgDescs": "release notes"}}
```

The device-info screen uses only `version` and `upgDescs`. **`downUrl` is never read** anywhere in
the app, so it is not how the camera gets its firmware. The server still returns it, so it is one way
to learn firmware URLs.

**Transport (te-ejv).** These calls are not HTTPS from the phone. The Java proxy turns the annotated
method into `(method, url, data)` and calls native `nHttpProxyRequest`. `libiotvideomulti.so`
(`MessageMgr::http_proxy_request`) wraps it as

```
topic: HTTP_PROXY/REQ
{"http":{"url":"/sdk/dev/upg/app/queryVersion","type":"POST"},"data":{"tid":"…","forceVersion":"","language":"en-US","currentVersion":"…"}}
```

and sends it over the SDK's own encrypted server link to the IoTVideo access server, which forwards
it to an internal HTTP service. Any app-wide "public params" the proxy holds are merged into `data`
(not traced). So it cannot be replayed with curl. Calling it requires the SDK's registered server
session.

Don't confuse it with `/openapi/app/upg/queryVersion`, a plain HTTPS endpoint for the **app's** own updates.

**A second lookup** exists in the cross-platform `com.gw.gwiotapi` SDK:
`GWIotDevUpgradeComponent.checkDevUpgradeInfo` → `queryDevUpgradeInfo(devId, currentVer)` returns the
same `{version, downUrl, upgDescs}` (`DevUpgradeData`) and opens a batch-upgrade screen. Its endpoint
goes through a generic request component and was not traced. Nothing reads `downUrl` there either.

**Practical ways to read `downUrl`:**
1. On a test phone with a bound camera, open the camera's device-info screen (that triggers the
   query), and hook `IoTDeviceInfoMgrHttp`'s success callback or the `nHttpProxyRequest` callback
   with Frida to log the response JSON. The native library also has a `httpproxy[…] rsp: %s` log line,
   and the Java side logs the full JSON at debug level, so the app's own logs may already contain it.
2. Capture the camera's own update check (te-cyj). It downloads from whatever its update server returns.

## Legacy Gwell P2P cameras

Same model: the camera checks and downloads by itself (te-93g). The JNI calls in `libgwmediaplayer.so`
(`nGetDeviceVersion`, `nCheckDeviceUpdate`, `nDoDeviceUpdate`, `nCancelDeviceUpdate`) **take no
arguments**. Each only builds a P2P command packet for the camera (`fgCheckDeviceUpdate`,
`fgDoDeviceUpdate`, `fgCancelDeviceUpdate` in the native code). The app holds no update-server
hostname or firmware URL for these cameras.

| Reply from camera | Fields |
|-------------------|--------|
| check update | device ID, result code, **current version**, **available version**. Each version is packed in one 32-bit int, one byte per part, shown as `a.b.c.d` |
| do update | device ID, result code, one more int (likely progress) |

The legacy update server's host, like the IoTVideo one, has to come from a camera capture (te-cyj).

## What TianEye needs

- **Block updates:** stop the camera reaching its update server, by DNS or firewall, once
  te-2dc identifies the host.
- **Serve firmware locally:** emulate the update server the camera queries. Only attempt this
  after checking whether the camera verifies a signature on the image, because a bad image can brick it.
- TianEye can show update state by sending `_otaVersion` and listening for `_otaUpgrade`
  progress, but it should not start an upgrade unless the user explicitly asks.

## Open questions

1. Which host and request does the camera use to check for updates and download firmware?
2. Does the camera verify a signature on the firmware before flashing it?
3. Does `_otaVersion` only report the version, or does it also prepare or start the
   download? The update dialog treats an acknowledged `_otaVersion` as "update started".
4. Legacy path wire format.
