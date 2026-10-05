# Adding a Camera to an Account

How the Yoosee Android app (6.46.1) provisions a camera onto Wi-Fi and binds it to the user's
account. Static analysis only (te-99x.19); not observed on the wire. Examples use placeholders.

## Overview

```
1. (optional) scan the camera's sticker QR / pick the model   → model + supported setup modes
2. GET  /openapi/netcfg/cloud/netcfg/genbindtoken              → bind token   (IoTVideo cameras)
3. hand Wi-Fi + account info (+ token) to the camera            → QR, AP, BLE, smartlink, sound wave or wired
4. camera joins Wi-Fi and reports to the cloud with the token
5. poll /openapi/netcfg/cloud/netcfg/devresult every 5 s (≤ 60×) → {devId, status, token}
6. POST /openapi/app/user/device/bind                          → camera added to the account
7. post-setup writes: time zone, alerts on (cloud uploads on), push schedule
```

Steps 4–5 mean **the cloud is in the loop for every add**. The app never learns the camera
directly; it waits for the cloud to say the camera checked in with the token.

## Step 3: provisioning modes

All modes carry the same fields. The two camera families encode them slightly differently.

### Config fields

Two encoders exist, and they number some fields differently:

| Tag | QR screen (`QRCodeInfo`, `QRCodeConfigVM`) | AP mode (`QRCode` class) |
|-----|---------------------------------------------|---------------------------|
| 0 | Wi-Fi SSID | Wi-Fi SSID |
| 1 | Wi-Fi password (**plain text**) | Wi-Fi password (**plain text**) |
| 2 | Wi-Fi security type | Wi-Fi security type |
| 3 | account user ID (hex) | account user ID |
| 4 | language | language |
| 5 | bind token (IoTVideo only) | bind token (`netMatchId`) |
| 6 | APN name (4G) | — |
| 7 | APN user (4G) | time zone |
| 8 | APN password (4G) | — |
| 9 | APN type (4G) | — |
| 32 | — | share token |

**Text encoding (QR screens):** each field is `<tag digit><2 hex digits = byte length><UTF-8 value>`,
concatenated. Example with placeholders: `007HomeNet` + `108password` + `2013` (type 3) + …

- **IoTVideo cameras** (`QRCodeInfo.createQRCodeStr`): tags 0–5, plus 6–9 for 4G.
- **Gwell cameras** (`QRCodeConfigVM.gDeviceGenerateQRCode`): tags 0–4 only, with **no token**.
  The camera learns the owner from tag 3, the numeric account ID.

**Binary encoding** (`QRCode.toQRContentByte`): `<tag byte><length byte><value>` (TLV), no encryption.

### QR code (most common)

The app shows a QR code on the phone; the camera's lens reads it. **The Wi-Fi password is in the QR
in plain text.**

At the same time, the QR screen runs **MediaTek "Elian" smart connection** (`ElianNative`,
`libelianjni`). `StartSmartConnection(ssid, password, "", type)` repeatedly broadcasts the SSID and
**Wi-Fi password** encoded in Wi-Fi packet patterns, so cameras that can't read the QR still join.
Any nearby receiver can usually recover credentials from these broadcasts.

### AP hotspot

The camera opens its own Wi-Fi network; the phone joins it. The app sends the same config string to
the camera with the IoTVideo LAN message path:
`MessageMgr.sendMsgToDevice(devId, BUILT_IN, AP_NET_CONFIG (6), <config>)`.

### Bluetooth (BLE)

The config is encrypted with the native `bleAesEncrypt(data, key, encType)` (`libiotvideomulti.so`)
before it's written to the camera over GATT. The GATT service details and key exchange live in a
separate BLE SDK and were not traced. This is the only mode where Wi-Fi credentials aren't sent in
the clear.

### Wired

Cameras already on Ethernet are found on the LAN (`WiredNetConfig.nativeGetDeviceList`, a native
LAN search), then the app hands the token over with `subscribeDevice(devId, token)`.

### 4G

`ApnQRCodeActivity` adds the APN fields (tags 6–9) to the QR string.

## Step 5: waiting for the camera

`ConfigNetOnlineStatusProxy` polls `openapi/netcfg/cloud/netcfg/devresult` with the token every
5 s, up to 60 times (~5 minutes). The reply (`NetConfigResult`) is `{devId, status, token}`.

## Step 6: bind

`POST /openapi/app/user/device/bind` (fields in [protocol.md](protocol.md)): `devId`, `tid`,
`remarkName`, `permission`, `bindToken`, `devType`, plus optionally `snCode`, `timeArea`, `timeZone`,
**`latitude`/`longitude`**, the camera's **LAN `ip`** and `productID`.

A camera already bound to another account is taken over with `forceBind=true`.

## Step 7: what the app changes right after adding

From `compo_impl_confignet` (`IoTDeviceReadHttp`):

1. writes the phone's time zone to `ProWritable.timeZone`;
2. calls `openAlert`, which sets **cloud video upload on** (`_cloudStoage.pause = 0`) and
   **alarm snapshot upload on** (`_almEvtSetting.uploadImgEna = 1`, `enable = 3`), with no plan
   check (see [privacy-uploads.md](privacy-uploads.md));
3. sets the push interval and a default push schedule.

## Other netcfg endpoints (purpose inferred from names)

`/openapi/netcfg/app/getDevInfoBySN`, `getDeviceBarcode`, `appearanceInfo`, `getLinkInfo`,
`getConnectInfo`, `getGiveInfo`. These are likely used to identify a scanned camera and choose its
setup mode. Not traced.

## Cloud-free provisioning of a reset camera (te-oh2)

The question for TianEye: can a factory-reset camera be put on Wi-Fi without the Yoosee app and the
cloud? Split the flow in two:

**Getting the camera onto Wi-Fi is entirely local.** None of the handoff channels touch the cloud —
the phone talks straight to the camera. The QR screen in fact drives **three channels at once** for
the same Wi-Fi credentials, so a camera joins however it can sense them:

| Channel | How it carries Wi-Fi SSID + password | Library |
|---------|--------------------------------------|---------|
| QR code | shown on the phone, read by the camera lens; TLV/text (see Step 3) | app code |
| Smart connection ("smartlink") | SSID + password encoded in Wi-Fi packet lengths/patterns, broadcast over the air | MediaTek `ElianNative` (`libelianjni`) |
| Sound wave | SSID + password modulated into audio played from the phone speaker | `EMTMFSDK.sendWifiSet(ssid, password)` (`liblarksmarkemtmfJNI`) |
| AP hotspot | phone joins the camera's own AP, sends config over the LAN | `AP_NET_CONFIG` built-in cmd |
| BLE | encrypted over GATT | `bleAesEncrypt` |

A Realtek Simple Config library (`libsimpleconfiglib`, `SCLibrary`) is bundled too, but nothing in
the app calls it — likely dead/legacy. All of these are reproducible without the vendor: the formats
are known (QR/AP) or handled by the same third-party libraries TianEye could call (Elian, EMTMF).

**Binding to an account still needs the cloud.** After the camera is on Wi-Fi it checks in with the
cloud using the bind token, and the app polls `devresult` and calls `device/bind` (Steps 4–6). A
truly app-free, cloud-free setup therefore needs TianEye to stand in for `genbindtoken` / `devresult`
and accept the camera's check-in — the same impersonation question as
[cloud-redirect.md](cloud-redirect.md). For **Gwell** cameras the QR carries **no token** (only the
numeric account ID in tag 3), so those may bind with less cloud involvement; unconfirmed.

**Simplest path for TianEye:** provision the camera once with the normal app (or AP mode), then move
it behind the egress block. The camera is already on Wi-Fi and bound; from then on it needs only the
LAN. Full cloud-free *first-time* provisioning is a larger effort gated on the impersonation work.

## What this means for TianEye

- A camera can only finish setup if something answers `genbindtoken` and `devresult`, and accepts
  the camera's own check-in. Cloud-free setup needs those emulated (te-cz1), or a camera
  that's already on Wi-Fi.
- Wi-Fi credentials leave the phone in the clear in QR, smartlink and sound-wave modes — any nearby
  receiver can usually recover them. BLE is the only encrypted channel.

## Open questions

1. What the camera sends to the cloud when it checks in with the token (camera side, not visible in the app).
2. How Gwell cameras (no token in the QR) become bound: does the camera bind itself to the account
   ID in tag 3, and could that happen without the cloud?
3. BLE GATT service UUIDs and key exchange.
4. The exact role of the `getLinkInfo` / `getConnectInfo` endpoints.
