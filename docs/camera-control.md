# Camera Control Protocol

Covers live video, PTZ, recording playback, and settings. Based on static analysis of
Yoosee 6.46.1 Java source and native libraries.

---

## Two SDK Paths

The app contains **two parallel P2P SDKs**, used for different camera generations:

| SDK | Library | Service discovery | Used for |
|-----|---------|-------------------|----------|
| Legacy Gwell P2P | `libgwmediaplayer.so` → `libp2pav.so` | `/gwellcloud/service/ListService/GetServiceList` | Older cameras |
| IoTVideo (Tencent) | `libiotvideomulti.so` | `/iotvideo/service/ListService/GetServiceList` | Newer cameras |

For a local server to work with both generations it must serve both service-list endpoints.

---

## Live Video Feed

### Legacy P2P Path

```
GwVideoSdkNative.init()                      → loads libgwmediaplayer
GwVideoSdkNative.nativeRegister(P2PInitParam) → connects to P2P index server
GwVideoPlayerNative.nativeSetDataResource(deviceId, channel, ...)
GwVideoPlayerNative.nativePrepare()
GwVideoPlayerNative.nativePlay()             → avctl_StartRecvAndDec() in libp2pav
```

After `nativePlay()`, frames arrive via the registered `nativeSetVideoRender` Surface.
AV header (codec, resolution) is readable via `nativeGetAVHeader()`.

**Video quality change**: `nativeChangeDefinition(definition)` where definition is:
- 1 = LD, 2 = SD, 3 = HD, 7 = AUTO

**Player states**: PREPARING, PLAYING, PAUSED, STOPPED, STATUS_STOP=7

### IoTVideo Path

```
IoTVideoInnerInitializer.nativeInit(config)
IoTVideoInnerInitializer.nativeRegister(...)
AIoTBasePlayer.nCreateIoTPlayer(
    type=1 (LIVE_PLAYER), deviceId, channelId, sourceId)
→ P2P connection, stream via libiotvideomulti callbacks
```

Connection states via `nGetConnectState()` (mapped by ordinal in `IoTConnectState.transform`):
0=PREPARING, 1=ASSIGN_CHN, 2=WAKE_UP_DEV, 3=CONNECTING, 4=CONNECTED, 5=DISCONNECTING, 6=DISCONNECTED
(unknown values fall back to DISCONNECTED).
Transport protocol via `nGetConnectProtocol()`: -1=UNKNOWN, 0=UDP, 1=TCP

**Route** via `nGetConnectMode()` (`ConnectionMode`): -1 = unknown, **0 = RELAY** (through Gwell's
relay servers), **1 = P2P** (direct across the internet, NAT hole-punched), **2 = LAN** (direct on
the same network). The legacy library has the same three paths (`mtp_session_add_lan_or_nat`,
`add_tcp_lan`, `add_udp_relay`). So even on one Wi-Fi network the cloud sets up every session (index
lookup, service discovery, certification), and only then can video and commands flow directly on
the LAN. Which route the app actually gets at home is unconfirmed (te-2dc).

---

## PTZ Control

Static analysis of 6.46.1 (te-99x.18). Not exercised against a camera. The app picks the path by
camera family: legacy **Gwell** cameras ("G", `GMonitorPlayer`) and **IoTVideo** cameras ("T",
`TMonitorPlayer`).

### Directions (both families)

`0` = left, `1` = right, `2` = up, `3` = down (`USR_CMD_OPTION_PTZ_TURN_*`, `PenetrateShakeHead`).
The app swaps left/right and up/down when the user has flipped the picture in the app's own display
settings (`isHorizontalFlip` / `isVerticalFlip`, stored on the phone), so the command matches what
the user sees. This is separate from the camera's own `flip` setting (below).

### Gwell cameras: 28-byte user-data frame

`GwMonitorPlayer.ptzControl(dir)` → `GwVideoPlayer::ptzControl` (`libgwmediaplayer.so`) →
`fgSendUserData(cmd=0, option=dir, data=NULL, len=0, channel)` → `avctl_SendUserData`
(`libp2pav.so`). The frame is built by `setAVCtrlUsrDataBuf` and sent on the P2P link channel
(`fgP2PLinkChannelSendDataToCh`), retried up to 20 times, 10 ms apart:

| Offset | Size | Value |
|--------|------|-------|
| 0 | 4 | `FF FF FF 88` (marker) |
| 4 | 2 | `00 02` |
| 6 | 1 | command: `0x00` PTZ turn, `0x0C` HXST PTZ |
| 7 | 1 | option: direction 0–3 |
| 8 | 20 | payload, zero-padded (empty for PTZ; 4 bytes for HXST) |

There is **no stop frame**. Each frame is a step. When the player mode is 4, the HXST variant is
used: `fgSendUserData(0x0C, dir, <4 bytes>, 4, channel)`. It travels inside the P2P session, which is
encrypted (see [crypto-and-auth.md](crypto-and-auth.md)), so it isn't sent raw on the LAN.

### IoTVideo cameras: JSON "shake head" message

`MonitorViewVM.shakeHead` builds JSON and sends it as IoTVideo user data on the live connection:
`TMonitorPlayer.V(lensId, json)` → `LivePlayer.sendUserData(lensId, json, 15 s timeout)` →
`nSendUserData` (`libiotvideomulti.so`). The first argument (the user-data command byte) is the
**lens/camera index**, 0 on single-lens cameras.

```json
{"type": 2,  "data": {"dir": 0, "touchType": 0}}   // old protocol
{"type": 22, "data": {"dir": 0, "touchType": 0}}   // "new agreement", press
{"type": 22, "data": {"dir": 0, "touchType": 1}}   // "new agreement", release = stop
```

- **Old protocol (`type` 2):** a step. While a finger is held, the live-view screen repeats the
  message every **200 ms** (adjustable in debug builds), and the release is not sent.
- **New protocol (`type` 22):** start/stop. One message on press (`touchType` 0) and one on release
  (`touchType` 1). It's chosen per camera by the capability check `devSupportPtzNewAgreement(devId, lens)`.
- Multi-lens cameras pass the lens index when `supportPtzWithCamId` is set; otherwise it is 0.

### Presets (IoTVideo)

Same JSON channel, `type` 6. Sent either from live view (as user data) or through the device
pass-through API (`MessageMgr.sendMsgToDevice`) when no live session is open:

```json
{"type": 6, "msgId": 1234, "data": {"cmd": 2, "idx": [3], "camId": 0}}
```

| `cmd` | Meaning |
|-------|---------|
| 0 | delete preset(s) in `idx` |
| 1 | save the current position as preset `idx` |
| 2 | move to preset `idx` |

The camera's reply carries `ret`. Preset **names and thumbnails** are kept in the cloud
(`/openapi/app/user/device/preset/add|delete|modify|swap|list`, thumbnail upload in
[privacy-uploads.md](privacy-uploads.md)). The camera itself only knows positions.

### Zoom and focus (IoTVideo, motorised-lens models)

A `TAKE_ACTION` on `Action.zoomFocusA.stVal`, with a press/release pair just like PTZ:

```json
{"actionType": "zoom_add_dn", "currentZoom": 1.0}    // start zooming in
{"actionType": "zoom_add_up", "currentZoom": 1.0}    // stop
```

`actionType` is one of `zoom_add_*`, `zoom_reduce_*`, `focus_add_*`, `focus_reduce_*`, where `_dn`
means press and `_up` means release. The app limits zoom using the camera's min/max zoom values
(`ProWritable.zoomFocusW`).

### PTZ reset / calibration (IoTVideo model)

`Action.ptzCheck` is **not** movement. It is the "PTZ reset" action (`PtzControlVM.ptzReset`):
the camera re-runs its PTZ self-check and calibration. It is sent with `TAKE_ACTION` on
`Action.ptzCheck.stVal`, and the value is a small bitfield: the lens index in the high bits and
`01` in the low two bits. Gun-ball (dual-lens) devices send binary `0001`.

### ONVIF

The app never uses ONVIF. The test camera advertises ONVIF PTZ, but it doesn't work: `ContinuousMove`
is acknowledged and nothing moves, and `Stop` and every PTZ query are dropped
([onvif.md](onvif.md)). So PTZ needs the P2P path above, unless a camera that has a motor behaves
differently.

## Recording Playback (SD Card)

The IoTVideo SDK has a complete SD playback sub-protocol over the P2P channel.
Commands are sent via `MessageMgrInternal.sendMsgToDevice` with `domain=BUILT_IN`.

**Playback flow**:
```
1. PLAYBACK_GET_DATE_LIST (18)    → list days with recordings
2. PLAYBACK_GET_LIST_V2V3 (16)   → list recordings for a date range
   (or PLAYBACK_GET_LIST_V1 (0) for older cameras)
3. PLAYBACK_STREAM_BEGIN (4)     → start stream at a timestamp
4. Stream frames arrive like live view
5. PLAYBACK_SEEK (3)             → seek to timestamp
6. PLAYBACK_PAUSE (1) / PLAYBACK_RESUME (2)
7. PLAYBACK_SPEED (24)           → fast-forward multiplier
8. PLAYBACK_END_OF_FILE (17)     → camera signals end of clip
```

**File download** (for saving clips locally):
```
DOWNLOAD_FILE_INFO (19)    → get file metadata
DOWNLOAD_REQUEST (21)      → start download
DOWNLOAD_FILE_DATA (23)    → file data packets
DOWNLOAD_CANCEL (20)       → abort download
DOWNLOAD_EXCEPTION (22)    → error notification
```

**Thumbnails**: `THUMBNAIL_REQUEST (26)` / `THUMBNAIL_CANCEL (27)`

**Player types for playback**:
- `SD_PLAYBACK_PLAYER` = 2 (SD card)
- `NAS_PLAYBACK_PLAYER` = 3
- `CLOUD_PLAYBACK_PLAYER` = 0 (cloud storage)

For cloud storage, HLS is used with AES-128 encryption; segment keys come from
`/vas/cloudstorage/getTsEncryptKey` (see [crypto-and-auth.md](crypto-and-auth.md)).

---

## Camera Settings

Settings are read/written via IoTVideo model commands using `MessageMgrInternal`.
The data model uses `ProWritable` (writable properties) and `ProReadOnly` (status).

**Read a property**: `nExecuteModelCmd(READ_PROPERTY=1, deviceId, path, "", ...)`
**Write a property**: `nExecuteModelCmd(WRITE_PROPERTY=0, deviceId, path, jsonValue, ...)`

### Known Writable Property Paths

| JSON path | Description |
|-----------|-------------|
| `ProWritable.videoParm` | Video resolution and encoding parameters (Java field `videoParam`) |
| `ProWritable.recordParm` | Recording mode (continuous, motion, schedule) (Java field `recordParam`) |
| `ProWritable.csVideoRes` | Video resolution setting (Java field `recordRes`; "cs" prefix, likely cloud-storage) |
| `ProWritable.guardParm` | Motion detection / guard parameters |
| `ProWritable._almEvtSetting` | Alarm event settings |
| `ProWritable.motionZone` | Motion detection zones |
| `ProWritable.onvifEn` | **Enable/disable RTSP/ONVIF on the camera** (Java field `rtspEnable`; the "Connect NVR" screen writes `1`/`0`) |
| `ProWritable.volume` | Audio volume |
| `ProWritable.audioMode` | Audio mode |
| `ProWritable.antiFlickerSwitch` | Anti-flicker (50/60 Hz) |
| `ProWritable.indicatorLight` | Status LED on/off |
| `ProWritable.whiteLightCtrl` | White light / spotlight control |
| `ProWritable.workMode` | Camera work mode |
| `ProWritable.timeZone` | Timezone setting |
| `ProWritable.zoomFocusW` | Optical zoom/focus (PTZ cameras) |
| `ProWritable.aiModeId` | AI detection mode |
| `ProWritable.autoWhiteLight` | Auto white light schedule |
| `ProWritable._cloudStoage` | Cloud storage configuration |

### Image and night-vision settings (te-667)

From the app's "Image & Sound", "Night Vision" and "Screen Flip" screens. Each write targets one
sub-path (e.g. `ProWritable.videoParm.setVal.flip` with value `1`). In **AP mode** (phone on the
camera's own hotspot, no cloud) the app writes the whole object instead (`ProWritable.videoParm`
with the full JSON), so these settings can be changed over the LAN.

**`ProWritable.videoParm.setVal`**

| Field | Values |
|-------|--------|
| `flip` | `1` = picture flipped ("Screen Flip", for ceiling mounting), `0` = normal. Whether it's a 180° rotation or a vertical flip isn't visible in the app |
| `multiFlip` | multi-lens cameras instead of `flip`: bitmask, bit (lens index + 1) per lens; `-1` = not supported |
| `nightViewMode` | older night-vision setting: `0` = automatic, `1` = daytime (IR off), `2` = night vision (IR on) |
| `videoLevel` | "Image Quality": `0` very low, `1` low, `2` medium, `3` high, `4` highest |
| `videoSetting` | on-screen time display: bit 0 = supported, bit 1 = `0` 24-hour / `1` 12-hour |
| `closeUp` | dual-lens cameras: close-up view `0` = shown, `1` = hidden |

**`ProWritable.nightViewModeV2.setVal.enable`** is the newer night-vision setting, used when the
camera reports it. It's a bitfield that holds the camera's capabilities and the current choice:

| Bit | Capability bit (camera supports it) | Selection bit (one set) | Mode |
|-----|-----|-----|------|
| 1 / 8 | 8 | 1 | **Night Vision (B&W)**: infrared only, black-and-white at night |
| 2 / 9 | 9 | 2 | **Full Color**: spotlight on, colour all night |
| 3 / 10 | 10 | 3 | **Smart**: IR in low light, switches to colour lighting when it detects something |
| 4 / 11 | 11 | 4 | **Ambient Light**: no IR or spotlight; picture depends on ambient light |
| 5–6 / 12 | 12 | 5 or 6 | When to switch: `5` = dusk (low light), `6` = dark (very low light) |

To change mode, the app clears bits 1–4 (or 5–6), sets the new bit, and writes the whole value back,
keeping the capability bits. Example: Smart, switching at dusk, on a camera that supports all modes
= `(0b11111 << 8) | (1 << 3) | (1 << 5)`.

**Other picture and light settings**

| Path | Values |
|------|--------|
| `ProWritable.antiFlick` → `antiFlickerSwitch` | anti-flicker. The app **reads** `antiFlick` and **writes** the new value to `antiFlickerSwitch`. Bits: 0 = kept as read (likely "supported"), 1 = on, 2 = 50 Hz, 3 = 60 Hz. Off = only bit 0 kept; 50 Hz = bits 1 + 2 |
| `ProWritable.whiteLightPlan.setVal.planEn` | spotlight schedule on/off; plans `plan1`… with on/off times and weekdays |
| `ProWritable.autoWhiteLight` | automatic spotlight |
| `Action.whiteLightCtrl` | switch the spotlight now |
| `ProWritable.indicatorLight` | status LED |

ONVIF has no Imaging service on the test camera ([onvif.md](onvif.md)), so these settings are
reachable only through the device model.

### The complete device model

The app ships a generated schema of every camera property (`com.yoosee.kmpsaas.yooseeiotmodel`).
Top-level names by section:

| Section | Properties |
|---------|------------|
| `ProWritable` | `_cloudStoage`, `_otaMode`, `_logLevel`, `_almEvtSetting`, `timeZone`, `volume`, `onvifEn`, `recordParm`, `guardParm`, `antiFlick`, `antiFlickerSwitch`, `videoParm`, `csVideoRes`, `whiteLightPlan`, `autoWhiteLight`, `resFile`, `cloudCtrlCfg`, `pressKeyCall`, `motionZone`, `softPS`, `zoomFocusW`, `indicatorLight`, `pirThresdhold`, `audioMode`, `workMode`, `aiModeId`, `lteServiceStatus`, `screenSwitch`, `nightViewModeV2`, `autoWorkMode`, `smartService`, `eventParam`, `timeSwitchParm` |
| `ProReadOnly` | `_online`, `tfInfo` (SD card), `devInfo`, `power`, `simCard`, `aiModeDownL` |
| `ProConst` | `_productInfo`, `_versionInfo`, `devFuncCfg`, `zoomFocusC`, `devFunCode` |
| `Action` | `laserCtrl`, `expelCtrl`, `zoomFocusA`, `cameraOn`, `ptzCheck`, `_otaVersion`, `_otaUpgrade`, `formatTF`, `whiteLightCtrl` |
| `ProUser` | `_buildIn` |

### Action Paths

| Path | Description |
|------|-------------|
| `Action.ptzCheck` | PTZ reset / self-calibration (not movement) |
| `Action.zoomFocusA` | Optical zoom/focus action |
| `Action.formatTF` | Format SD card (used as a path string; no field in the `Action` bean) |
| `Action.cameraOn` | Toggle camera on/off |
| `Action.laserCtrl` | Laser control |
| `Action.expelCtrl` | Alert/deterrent action |
| `Action._otaVersion` | Check OTA version |
| `Action._otaUpgrade` | Start OTA upgrade |

---

## Two-Way Audio (Intercom)

Legacy P2P: `GwVideoPlayerNative.nativeStartTalk()` + `nativeSendAudioData()`
Stop: implicit on player release.

IoTVideo: `AudioIntercom` / `AudioIntercomProxy` classes handle encode/decode over the P2P channel.

---

## Open Questions

1. ~~How do PTZ moves reach the camera?~~ P2P user data (Gwell 28-byte frame, or IoTVideo JSON); see PTZ Control. Whether the camera's RTSP `USER_CMD_SET` method also accepts PTZ is unknown.
2. ~~Do cameras speak plain RTSP without the P2P tunnel?~~ **Yes.** The test camera serves RTSP on
   554 directly (HTTP Digest, realm `HIipCamera`); `ProWritable.onvifEn` switches it on or off.
3. What `P2PInitParam` fields are required? (Index server address, appId, secretKey?)
4. Are PLAYBACK_GET_LIST_V2V3 timestamps in Unix seconds or milliseconds?
5. Does the legacy P2P SDK support SD playback, or is that IoTVideo-only?
