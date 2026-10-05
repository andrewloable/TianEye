# Data Inventory — What Leaves the Network

Every data item the app or camera sends off the LAN, from static analysis of com.yoosee 6.46.1.
"Confirmed on wire" is still empty — capture tasks te-2dc / te-ab8 / te-cyj fill it in.
Hosts are in [cloud-endpoints.md](cloud-endpoints.md); upload flows in
[privacy-uploads.md](privacy-uploads.md).

Sensitivity: **high** = identifies a person, place, or credential; **med** = device/usage
metadata; **low** = coarse or non-identifying.

## App → cloud

| Data | Host | Trigger | Sensitivity | Source |
|------|------|---------|-------------|--------|
| Account login: email / phone (+country) / userId, **password**, `uniqueId`, region | `openapi-iot` | login | **high** | static |
| Per-request identity: `accessId`, `accessToken`, `uniqueId`, app/SDK version, platform, region, `appId` | `openapi-iot` | every API call | med | static |
| Device bind: `devId`, `tid`, serial (`snCode`), remark name, **`latitude`/`longitude`**, device **LAN `ip`**, time zone, productID | `openapi-iot` | adding a camera | **high** | static |
| Device list / info / unbind | `openapi-iot` | app use | med | static |
| Push-notification token | `openapi-iot` / `offlinePush` | startup, token refresh | med | static |
| PTZ preset snapshot (JPEG of the view) | PaaS resfile slot (the Tencent COS upload code is never called) | saving a PTZ preset | **high** (image of the scene) | static |
| Feedback: description, contact, app/device version, **log file**, **pictures** | `openapi-iot` `/feedback`; photo itself to `res.zhiduodev.com` `res/pic/upload` | user submits feedback | **high** | static |
| App log file | `openapi-iot` `/uploadLog` | not traced (likely error/support) | med | static |
| Visit log (free-form map) | `/visitlog/upload` | not traced | med | static |
| Alarm misreport: deviceId, alarmId, labels, device version | `/feedback/aimisreportv2` | user marks an alarm wrong | med | static |
| Analytics events (Sensors Data) | `datasink` `/sa` | periodic / in-app events | med | static |
| Device location (`PositionInfo`) | `collect.cloud-links.net` | **no caller in 6.46.1** (dead code; watch for it in captures) | **high** | static |
| 4G SIM data-plan charging | `uc.api.china-m2m.com` (plain HTTP) | 4G cameras, buying data | med | static |
| Crash reports (stack, device, logs) | `android.bugly.qq.com` | on crash | med | static |
| Ad / login SDK traffic (AdMob, Facebook, AppLovin, Pangle, Mintegral, Unicom login) | third-party | app foreground / carrier login | med | static |

**Not sent by the app:** live video, local recordings, device screenshots, and **Wi-Fi
SSID/password** — provisioning (AP, BLE, QR, smartlink) hands Wi-Fi creds straight to the camera
over the LAN. No cloud API call in the app carries Wi-Fi fields (static check; confirm with te-ab8). They do leave the phone **in the clear locally**, though: in the setup QR code, and in MediaTek
smart-connection broadcasts the QR screen sends at the same time. Only Bluetooth setup encrypts them
(see [add-camera.md](add-camera.md)).

## Camera → cloud

The camera has its own connections; the app list is not the camera list. These are inferred from
the device model and the records the cloud serves back. Confirm hosts/ports with te-2dc and
firmware (te-99x.12).

| Data | Controlled by | Default | Sensitivity | Source |
|------|---------------|---------|-------------|--------|
| P2P registration: device ID, LAN + public address | always (to be reachable) | on | med | static/native |
| Alarm snapshots (JPEG per event) | `_almEvtSetting.uploadImgEna` (1=upload) | **on**: the app switches it on when a camera is added | **high** | static |
| Cloud video recording (HLS, AES-encrypted) | `_cloudStoage.pause` (0=upload) + active plan | switch set **on** at setup; actual upload likely needs a plan (te-ab8) | **high** | static |
| Images for cloud AI; server keeps a **per-camera face DB** + bird images | `_almEvtSetting.cloudAI` | unknown | **high** | static |
| Alarm events (type, time, image ref, AI labels) → push | always when alarms on | on | med | static |
| Firmware version check / download | camera's own update server | automatic | low | static |
| (Legacy cameras) alarm snapshots by **email**, to the user's own SMTP server | user-configured SMTP settings | off | **high** | static |

## Open

1. What `uploadImgEna` / `cloudAI` default to, and whether snapshots upload with no paid plan (te-ab8).
2. Which hosts/ports the camera actually uses (te-2dc, firmware).
3. What the visit log and `uploadLog` contain and when they fire.
