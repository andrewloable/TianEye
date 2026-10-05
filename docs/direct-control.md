# Direct (LAN) Control

Every way the Yoosee app (6.46.1) reaches a camera without traffic going through the cloud
(te-n0n, te-mnx). Static analysis plus probes of the test camera. Addresses are placeholders.

## Summary

| Path | Cloud needed? | Transport |
|------|---------------|-----------|
| Normal use on the same Wi-Fi | **Yes, to set up the session** (login, device list, P2P index); then video and commands may run directly on the LAN | P2P session in LAN mode |
| AP mode (phone on the camera's hotspot) | **No** | P2P SDK straight to the camera's address |
| Wired setup | Only for the bind that follows | LAN search, then token hand-off |
| RTSP/ONVIF | **No** | Plain RTSP on 554, ONVIF on 5000 (the app only uses it to switch RTSP on) |
| Alarms, push, cloud video | Always | Cloud |

## 1. P2P sessions in LAN mode

All camera control (live video, PTZ, settings, SD playback) rides one P2P session per camera. Both
SDKs report the route a session took: **relay** (via Gwell's relay servers), **P2P** (direct over
the internet) or **LAN** (direct on the local network); see
[camera-control.md](camera-control.md#iotvideo-path).

Settings go through one path regardless of route: the app's thing-model layer
(`YooseeComposeModel` → `ThingModelRepository`) calls `MessageMgr.readPropertyOfDevice`,
`writePropertyOfDevice` or `takeActionOfDevice` with the device ID, the property path (e.g.
`ProWritable.videoParm.setVal.flip`), a JSON value and a timeout. The native SDK
(`nExecuteModelCmd`) decides how it travels. Nothing in the Java layer picks LAN or cloud.

**How the SDK learns a camera is on the LAN.** The IoTVideo SDK runs a broadcast module that keeps
a table of LAN devices (ID, IP, port, MAC, a 16-byte field, flags), logged as
`new lan dev=… ip=… port=… key_did=… platform=…`. The app can ask whether a device is reachable
locally (`nLanDevConnectable`). Whether a LAN session can start **without** the server's help is
not settled; it's part of the protocol work (epic te-qgj).

## 2. LAN search

### Legacy Gwell cameras: UDP 8899

The app finds older cameras by broadcasting a search packet and listening for answers (`dl.i`,
packet class `el.a`).

- Every second it sends the search to the subnet broadcast address (and optionally
  `255.255.255.255`) on **UDP 8899**, and listens on **UDP 8899** for replies. Each reply's source
  address is the camera's IP.
- All integers are **32-bit big-endian**.

**Search request** (padded with zeros to 1024 bytes):

| Offset | Field | Value |
|--------|-------|-------|
| 0 | cmd | `1` (search) |
| 4 | error code | `0` |
| 8 | left length | `28` |
| 12 | right count / flags | `0` |
| 16–31 | device ID, type, flag, sub-type | `0` |

**Reply** (`cmd` = 2; the app reads the first 200 bytes):

| Offset | Field |
|--------|-------|
| 0 | cmd = `2` |
| 4 | error code |
| 8 | left length |
| 12 | flags. Bit 0: a customer ID is present at 68. Bit 6: "new format", where bits 0, 4, 7 and 8 say which of the optional fields below are filled in |
| 16 | **device ID** (numeric) |
| 20 | device type |
| 24 | flag |
| 60–61, 64–67 | **MAC**: 2 + 4 bytes, each run stored byte-reversed |
| 68 | customer (OEM) ID |
| 80 | sub-type |
| 84 | low 16 bits: the camera's **local P2P port**; high 16 bits: a local P2P address field (old format only) |
| 88 | `1` if the camera has a new ID |
| 92 | the new ID |

The app drops cameras whose customer ID isn't on its allow-list; a `0` entry in the list accepts all.

**Test camera:** no reply to this search, unicast or broadcast, and no other device on the test
network answered either. That fits: the test camera is an **IoTVideo** camera.

**Telling the families apart:** the app treats a device ID above 4,294,967,296 (2³², so 10+ digits)
as IoTVideo, and anything smaller as legacy Gwell (`IoTDeviceUtils.isIoTDevice`). Legacy IDs fit
the 32-bit ID field of this search packet.

### IoTVideo cameras: UDP 8900

`DevNetConfigMgr.searchLanDevs` (8-second search; native `WiredNetConfig.nativeGetDeviceList`) uses
the SDK's broadcast module in `libiotvideomulti.so`:

- A receive socket on **UDP 8900** (it tries 8900–9099 if that's taken), IPv4 and IPv6.
- A send socket with broadcast enabled, bound to **8901–8910** or **8904–8913** depending on an
  SDK role setting.
- Results come back as `DeviceInfo`: device ID, product ID, serial number, firmware version, MAC,
  Tencent ID, whether it already has an owner, whether it's in AP or QR setup, and IP.

The packet format isn't in the strings; decoding it is part of te-qgj. A 30-second passive listen
on 8899 and 8900–8913 heard nothing from the test camera, so it doesn't announce itself unprompted.

## 3. AP mode (no cloud at all)

The only fully cloud-free path in the app.

- A camera in setup mode opens a Wi-Fi hotspot named **`GW_AP_<id>`**, **`GW_AP_0<id>`** or
  **`GW_IPC_<id>`**. The app reads the device ID from the SSID.
- The phone joins it, and the camera's address is the network's **DHCP server/gateway**.
- If the SSID doesn't give an ID, the app runs the IoTVideo LAN search (above) to find the camera.
- The "AP mode" device list then offers live view and settings. Settings writes send the **whole
  object** (e.g. all of `ProWritable.videoParm` as JSON) instead of one sub-path. Wi-Fi provisioning
  in this mode uses the `AP_NET_CONFIG` built-in command ([add-camera.md](add-camera.md)).

So the P2P SDK *can* talk to a camera with no server in the loop. How it authenticates in that
case (the device password check `iv_check_set_lan_device_pwd` is a candidate) is part of te-qgj.

## 4. Wired setup

A camera on Ethernet is found by the LAN search; the app then hands it the bind token directly
(`subscribeDevice(token, devId)`, native `nSubscribeDevice`). The bind itself still goes through the
cloud ([add-camera.md](add-camera.md)).

## 5. RTSP / ONVIF

The app never streams over RTSP. It only switches the camera's RTSP server on or off
(`ProWritable.onvifEn`) and sets its password for NVR use
([crypto-and-auth.md](crypto-and-auth.md#rtsp-authentication)). For TianEye this is the main
cloud-free path ([live-view.md](live-view.md), [onvif.md](onvif.md)).

## TCP 50000 on the test camera

Still unidentified: it accepts a connection and stays silent. It isn't the 8899/8900 UDP search.
It may be the camera's local P2P/TCP listener (the search reply carries a "local P2P port"), to be
checked once the P2P framing is decoded (te-qgj).

## What this means for TianEye

- **Fully local today:** RTSP/ONVIF (live view, recording), and settings in AP mode.
- **Local once the protocol is decoded:** LAN-mode P2P sessions for PTZ, settings and SD playback,
  which would let TianEye control the camera with its cloud blocked. That's the goal of te-qgj.
