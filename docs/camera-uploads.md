# Camera Uploads: Alarm Snapshots and Cloud Video

What the Yoosee app (6.46.1) shows about the media cameras upload to the vendor cloud. Static
analysis (te-99x.13), doc-only scope.

**Limit:** the camera's own upload request (host, authentication, format) is not visible in the
app. This doc describes what the cloud *serves back* and the switches that control uploads; the
upload leg is inferred from that. Confirming it needs a capture or firmware (deferred: te-ab8,
te-99x.12).

## What controls uploads

Camera properties, set by the app (details in [privacy-uploads.md](privacy-uploads.md)):

| Upload | Property | On when |
|--------|----------|---------|
| Alarm snapshots | `_almEvtSetting.setVal.uploadImgEna` | `1` |
| Cloud video | `_cloudStoage.setVal.pause` | `0`, plus an active plan (`serviceType`, `utcExpire`) |
| Images for cloud AI | `_almEvtSetting.setVal.cloudAI` | not traced |

The app switches snapshot and video upload **on** for every newly added camera
([add-camera.md](add-camera.md), step 7).

## Alarm snapshots

The app gets alarm records from the event API and builds each image URL itself:

```
image URL = imgUrlPrefix (from the response) + per-event image suffix
thumbnail = imgUrlPrefix + per-event image suffix + thumbUrlSuffix
```

- Seen in `GLocalPlaybackVmImpl` (`info.imgUrlPrefix + alarmInfo.imgUrl`) and
  `GDevGuardServiceImpl` (`prefix + listBean.getImgurlSuffix()`).
- **No decryption step.** Alarm images are fetched and shown as plain image files over `http(s)`.
- **The storage host isn't hardcoded.** No Tencent COS, Aliyun OSS or AWS domain appears in the Java
  code or the native libraries; the prefix comes from the server at runtime.
- The push payload's `PushAlarm.bucketName` suggests the snapshot sits in an object-storage bucket.
  Which provider, and whether the camera uploads there directly, is unconfirmed.

Each event record also carries `labels`, `summary`, `duration` and `validVideoStartTime`, which
link the snapshot to the cloud video around it.

## Cloud video

Cloud recordings are **HLS with AES-128-encrypted segments**:

1. `/vas/playback/play` (or `playv2`, `/vas/speedplayback/play`, `/openapi/app/cloudstoage/speedPlay`)
   returns `CloudPlaybackAddress` = `{url, startTime, endflag}`, where `url` is an `.m3u8` playlist.
2. The playlist's `#EXT-X-KEY` line gives `METHOD=AES-128`, a key `URI` and an `IV`.
   `IoTHLSUtils` parses it; per segment the app tracks `tsUrl`, `tsDuration`, `encryptType`,
   `keyUri`, `encryptIv`, `encryptInfo` (`M3U8InfoEntity`).
3. The key comes from the cloud: `/vas/cloudstorage/getTsEncryptKey` (in `libiotvideomulti.so`),
   with a local key endpoint (`/key?localReq=`) used during download.

So the cloud keeps the video as encrypted transport-stream segments and holds the key. **Unknown
from the app:** whether the camera uploads segments it encrypted itself, or a stream the server
segments and encrypts.

## Cloud media API (app side)

The same `/vas/...` paths are reached two ways:

- **Plain HTTPS** on `openapi-iot.cloudlinks.cn` under the prefix `/openapi/vpaasPlayback`
  (e.g. `/openapi/vpaasPlayback/vas/event/list`). This is what the app's own screens use.
- **The Tencent SDK's `VasService`**, which sends bare `/vas/...` paths (`functionType` "vas") over
  the SDK's HTTP-over-P2P proxy; see [ota-firmware.md](ota-firmware.md) for that transport.

| Area | Endpoints |
|------|-----------|
| Status | `/vas/cloudstorage/describe` |
| Events | `/vas/event/list`, `query`, `datelist`, `completelist`, `completelistv2`, `remove`, `removebytime`, `clear` |
| AI events | `/vas/aievent/list`, `query`, `search`, `completelist`, `remove` |
| Recordings | `sdk/app/video/list`, `/sdk/app/video/play`, `/vas/playback/list`, `datelist`, `play`, `playv2`, `remove`, `removelist`, `removeall` |
| Fast playback | `/vas/speedplayback/play`, `/openapi/app/cloudstoage/speedPlay` |
| Album | `/vas/album/add`, `list`, `remove` |
| Faces | `/vas/aiface/list`, `edit`, `merge`, `remove` |

## What this means for TianEye

- Keeping these uploads locally would mean TianEye receiving them in place of the cloud, which
  depends on the camera-side upload protocol (unknown) and on whether the camera accepts a
  substitute server (te-99x.10).
- TianEye doesn't need them for its own recordings: it records the camera's RTSP stream directly
  ([live-view.md](live-view.md)), and can take alarm snapshots from that stream once it knows an
  alarm happened ([alarm-delivery.md](alarm-delivery.md)).
- Turning uploads off on a camera that stays online: `uploadImgEna = 0`, `pause = 1`, `cloudAI = 0`.

## Open questions

1. Where and how the camera uploads snapshots and video (host, auth, request format).
2. Whether the camera or the server does the AES-128 segment encryption.
3. Whether snapshots upload with no paid plan (te-ab8, deferred).
4. Which object-storage provider backs `bucketName`.
