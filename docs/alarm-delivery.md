# Alarm Delivery

How a camera alarm (motion, sound, PIR, human detection, …) reaches the Yoosee app. Static
analysis of 6.46.1 (te-99x.14); not observed on the wire. What the camera emits is not visible in
the app, so the camera-to-cloud leg is inferred from what the app receives.

## Summary

The vendor cloud is the hub for every alarm. The app receives it one of two ways depending on
whether it is open:

| App state | Mechanism | Path |
|-----------|-----------|------|
| Foreground / connected | Real-time message from the P2P SDK | vendor server → SDK's persistent connection → app callback |
| Background / closed | Push notification | vendor server → FCM or a phone-vendor push service → app |

Neither path is camera-to-phone over the LAN. After the trigger arrives, the app fetches the
snapshot and event detail from the cloud by ID (`alarmId` / `EvtId`); see
[privacy-uploads.md](privacy-uploads.md) and [data-inventory.md](data-inventory.md).

## IoTVideo cameras

### App connected: SDK callbacks

The IoTVideo SDK (`libiotvideomulti.so`) keeps a connection to the vendor server and delivers
messages to `MessageMgrInternal` listeners:

| Callback | Carries |
|----------|---------|
| `onReceiveEvent(event, topic)` | System/model events by topic, e.g. `EVENT_SYS/NetCfg_OK` |
| `onReceiveOnlineMsg(msg, topic)` | Online messages while connected |
| `onReceivePassthroughMsg(bytes, deviceId)` | Device pass-through payloads |
| `onReceiveDataProxyMsg(bytes, deviceId)` | Proxied data |

Screens that care about live alarms register through `IBackstageTaskApi.addPassthroughMsgListener`.

### App closed: push notification

The payload is parsed into `IoTPush` (`compo_api_push`):

```
push_type : "DevAlmTrg"  = device alarm   |  "MsgCenter" = message-center item
content   : display text
push_data : { DevId, EvtId, TrgTime, TrgType }
```

`TrgType` is the alarm kind; `EvtId` is the event handle used to pull media and detail from the
cloud afterwards. A richer alarm bean, `PushAlarm`, adds `alarmId`, `alarmType`, `bucketName`
(the cloud storage bucket holding the snapshot), `deviceId`, `alarmRecordMinTime` and `devCfg`.

### Push registration

The app registers its push token with the cloud so the server can reach it when the app is gone:

| Endpoint | Purpose |
|----------|---------|
| `/openapi/app/user/pushTokenBind` | Bind the FCM/vendor push token to the account |
| `/openapi/offlinePush/uploadToken` | Upload the offline-push token |
| `/openapi/paasProxy/usr/push/setting/getAll` | Read offline-notification settings |
| `/openapi/paasProxy/usr/push/setting/setDevOfflineNotifyEna` | Toggle per-device offline notify |

There is also an SDK-side `registerOfflinePush` (`nRegisterOfflinePush` in `libiotvideomulti.so`),
which tells the vendor server how to reach the push service for this install.

### Push services bundled

`FcmPushService` (Google FCM) is the primary. The app also ships Huawei HMS Push and other
phone-vendor push SDKs, selected by handset so a notification still arrives on phones without
Google services. All of them route through the respective vendor's push cloud.

### Alarm types (`TrgType`)

`TrgType` is a **64-bit bitmask**; one alarm can set several bits. Bit names come from the SDK's
`PushData.EventType`. The app turns the mask into a single display type (`IotAlarmUtils.getAlarmType`),
checking bits in a fixed priority order, and the label is what the notification shows.

| Bit | SDK name | App type | Label shown |
|-----|----------|----------|-------------|
| 0 | Motion | 2 | Motion detected |
| 1 | Human | 63 | Person Detected |
| 2 | Volume (sound) | 2 | not mapped on its own; shows as motion |
| 3 | Linkage | 2 | not mapped on its own; shows as motion |
| 4 | AiFace | 2 | not mapped on its own; shows as motion |
| 6 | PressCall | 61 | Doorbell is ringing (call button) |
| 7 | FromCloud | 2 | Motion detected (event raised by the cloud) |
| 15 | VideoBit | — | |
| 16 | Infrared (PIR) | 63 | Person Detected |
| 17 | Doorbell | 54 | Doorbell is ringing |
| 18 | Keyboard | — | keypad |
| 19 | Urgent | 3 | Emergency alarm (only when bits 0 and 7 are clear) |
| 20 | Pet | 104 | Pet detected |
| 21 | Car | 103 | Vehicle detected |
| 23 | BabyCry | 106 | Crying Detection |
| 24 | FireCheck | 105 | Flame Detected |
| 25 | SmokeCheck | 40 | Smoke Alarm |
| 27 | (bird) | 110 | Birds |
| 28 | (squirrel) | 111 | Squirrels |
| 29 | (animal) | 112 | Animals |
| 32 | LowBattery | | device status |
| 33 | DeviceOffLine | 109 | device went offline; the app opens live view whatever the alarm's age |
| 34–37 | SavePower, ChangeNotSleep, DeviceShutdown, DevicePowerOn | | device status |
| 38 | CommonPushType | | generic push |

Priority, highest first: animal, bird, squirrel, smoke, fire, call, crying, doorbell, person,
pet, car; then motion. Types **107 Package/Delivery** and **108 License plate** have no bit; they come
only from the cloud's AI `labels` on the event record (`package`, `numberplate`). The same
priority applies to labels: `animal`, `bird`, `squirrel`, `smoke`, `fire`, `call`, `cry`, `ring`,
`person`/`humanact`, `pet`, `car`, `package`, `numberplate`, else motion.

So the camera itself reports motion, person/PIR, sound, doorbell and the basic AI classes. The
finer classes (package, plate) are added server-side.

## Legacy Gwell cameras

The older P2P stack delivers alarms through JNI callbacks into `com.p2p.core.MediaPlayer` while the
app holds a session:

| Callback | Carries |
|----------|---------|
| `vRetAlarm(id, …)` | Alarm trigger |
| `vRetAlarmWithTime(id, …, time, bytes…)` | Alarm with timestamp and extra data |
| `vRetAlarmCodeStatus(…)` | Alarm-code/sensor status |
| `vRetBindAlarmId(…)` | Alarm-ID binding |

Their alarm types are small integers, shown as: `1` external alarm, `2` motion, `3` emergency,
`5` wired alarm, `6` low voltage, `7` person, `8` armed, `9` disarmed, `10` low battery, `13`
doorbell, `15` recording failed, `40` smoke, `41` gas, `42` door sensor, `43` temperature, `44`
humidity, `99` no mask. Types 1, 5 and 40–44 are shown with a sensor/zone name ("defence area"
group and item), so they come from sensors paired with the camera, which acts as an alarm hub.

When the app is closed these cameras rely on the same cloud push path as above; the wire format of
the camera-to-cloud alarm report is not visible from the app and needs a capture (deferred:
te-2dc).

## What this means for TianEye

- **ONVIF events are not an option** on the test camera: it advertises an Events service but drops
  every Events call ([onvif.md](onvif.md)). Without the cloud, alarms must come from TianEye's own
  motion detection on the RTSP stream, or from the camera's alarm report (next point).
- To receive alarms **without the vendor cloud**, TianEye must stand in as the server the camera
  reports to. Whether the camera will accept a substitute server is the subject of te-99x.10, and
  depends on how it authenticates its server connection.
- If TianEye receives the camera's alarm report directly, it can raise its own notifications and does
  not need FCM or any phone-vendor push cloud — though a self-hosted push path to phones is then its
  own problem (web push, or a companion app).
- The snapshot for each alarm is uploaded separately by the camera to cloud storage
  (`uploadImgEna`, bucket in `PushAlarm.bucketName`); TianEye emulating that upload target would
  give it local alarm thumbnails.

## Open questions

1. The exact camera-to-cloud alarm report (protocol, fields). Camera side; needs a capture.
2. Whether any alarm path reaches the app over the LAN when both are on the same network, or if it
   is always cloud-relayed.
3. ~~How `TrgType` values map to alarm kinds?~~ See Alarm types above.
