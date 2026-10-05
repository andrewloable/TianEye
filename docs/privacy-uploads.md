# Media and Data Uploads

What the Yoosee app and cameras send to remote servers. App-side (te-8rv) and camera-side (te-e0t) findings are from static analysis of
6.46.1. Wire measurements (te-ab8) are still open.
API paths are relative to `openapi-iot.cloudlinks.cn` unless noted.

## App side: media and files

| What leaves the phone | Where | Trigger | Opt-in? |
|-----------------------|-------|---------|---------|
| PTZ preset snapshot (JPEG of the camera view at a saved preset) | PaaS file slot from `/openapi/paasProxy/resfile/getuploadpath` (resource type 5), via `PaasPresetImgImpl`. A Tencent COS upload routine also exists (`TencentCosServer`, region default `ap-guangzhou`) but **nothing calls it** in 6.46.1; COS is only used to *download* preset images (`SaasPresetImgImpl`) | User saves a PTZ preset | Implicit in saving a preset |
| Feedback pictures and log file | Uploaded first as resource files, then referenced by `picResId` / `logResId` in `/app/feedback/commit`. The older feedback screen (`FeedBackActivity`) uploads its photo to `res.zhiduodev.com` `res/pic/upload` | User submits feedback | Yes, user action |
| App log file | `/openapi/app/user/uploadLog` (multipart file) | Not traced (only called via SDK adapters) | Unknown |
| Visit log (free-form map) | `/openapi/visitlog/upload` | Not traced | Unknown |

No path was found where the app uploads **live video, recordings, or local screenshots**.
Video reaches the cloud only through the camera (cloud storage, below).

## App side: references to media already on the server

These calls carry IDs, not media. They show the server already holds media that came from the camera:

| Endpoint | Fields | Implies |
|----------|--------|---------|
| `/app/feedback/aimisreportv2` | deviceId, alarmId, misreportTags, devVersion, … | Alarm events, and probably their media, are stored server-side by alarmId |
| `/openapi/vpaasPlayback/vas/aiface/list`, `edit`, `merge`, `remove` | devId, faceId, remarkName, iconUrl | The server keeps a **per-camera face database** with face thumbnails (iconUrl) |
| `/openapi/vpaasPlayback/vas/birdrec/alarm`, `image` | devId, alarmId | Server-side bird recognition on camera images |
| `/openapi/vpaasPlayback/vas/cloudstorage/describe`, `/openapi/app/cloudstorage/playback` | — | Cloud recording (paid plan) |

## Camera side

From the device model the app writes and the cloud records it reads (te-e0t, static). Two
separate camera properties control uploads, and **they use opposite polarity**:

| Media leaving the camera | Controlled by | Values | Notes |
|--------------------------|---------------|--------|-------|
| Alarm snapshots | `ProWritable._almEvtSetting.setVal.uploadImgEna` | 1 = upload, 0 = paused | Written together with `enable = 3` |
| Video (cloud recording) | `ProWritable._cloudStoage.setVal.pause` | 0 = upload, 1 = paused | Same object also has `serviceType`, `serviceParm`, `utcExpire` (the cloud plan) |
| Images for cloud AI (faces, birds, labels) | `ProWritable._almEvtSetting.setVal.cloudAI` | not traced | Server keeps a per-camera face DB and bird images (see above) |

- The app's "alert" switch writes the video switch first, then on success the snapshot switch, so
  one toggle turns both on or off. The app counts alerts as "on" when `enable == 3` and either
  upload is active.
- The cloud returns alarm records (`getEventList`) with `imgUrl`, `thumbUrlSuffix`, `labels`,
  `summary`, `duration` and `validVideoStartTime`. So the server stores a snapshot for each alarm,
  plus AI labels and a summary, and links it to cloud video when recording is on.
- **Adding a camera switches both uploads on.** After the setup flow binds a new IoTVideo camera
  and writes its time zone, it calls `openAlert`, which writes `pause = 0` (video upload on) and then
  `uploadImgEna = 1` with `enable = 3` (snapshot upload on). It also sets the push interval and a
  default push schedule. There is no check for a cloud plan (`compo_impl_confignet`, `IoTDeviceReadHttp`).
- **Likely, not confirmed:** video upload needs an active plan (`serviceType` and an unexpired
  `utcExpire`). Snapshot upload is a separate switch, so snapshots may go up **without** a paid
  plan. The capture task te-ab8 must settle this.

**Legacy Gwell cameras can also email alarm snapshots.** `libgwmediaplayer.so` has alarm-email
settings (SMTP server, port, user, password, encryption type, subject, body) next to the alarm
capture fields (`alarmCapDir`, `capNum`). The user must configure this, and it sends to the user's own
SMTP server, not Gwell's. Check that it is off on any camera you add.

### For TianEye

- To stop uploads from a camera that is still online, write `uploadImgEna = 0`, `pause = 1` and
  `cloudAI = 0`. Blocking the cloud hosts is the stronger guarantee.
- When TianEye emulates the cloud, the camera will try to upload alarm snapshots to it. That gives
  TianEye local alarm thumbnails, which is useful.

## Third-party SDKs bundled in the app

Device, usage and ad telemetry, not camera media. They still contact many hosts, so a
cloud-free setup should not run the official app.

| SDK | Purpose |
|-----|---------|
| Tencent Bugly | Crash reports (stack traces, device info, logs) |
| Firebase / Google Analytics, AdMob | Analytics, ads |
| Facebook SDK | Login / analytics |
| Huawei HMS | Push, services on Huawei phones |
| ByteDance (Pangle), AppLovin, Unity Ads, Mintegral | Ad networks |
| Tencent MTA (`pingma.qq.com`) | Mobile analytics |
| China Unicom / carrier one-click login | Phone-number login |

## Open questions

1. When does `uploadLog` run: automatically on errors, or only from a support screen?
2. What does the visit log contain?
3. The app sets `uploadImgEna = 1` at setup (above). Still open: the `cloudAI` default, and whether
   snapshots actually upload with no plan. Settled by te-ab8 (capture).
