# Security & Privacy Findings

Consolidated view of the security- and privacy-relevant observations from reverse-engineering the
Yoosee app (6.46.1) and one test camera. Each finding links to the doc with the detail. This is
**research on hardware we own**, to inform the self-hosted replacement and to let owners judge the
risk of the stock app/cloud.

Confidence: **static** = from decompilation/library analysis only; **tested** = observed on the
test camera; **needs capture** = requires on-wire or firmware work (deferred, te-2dc / te-99x.12) to
confirm.

## Credential & authentication weaknesses

| # | Finding | Impact | Where | Confidence |
|---|---------|--------|-------|------------|
| 1 | **Wi-Fi credentials sent in the clear during provisioning.** QR code, MediaTek "smartlink" and sound-wave all carry the SSID + Wi-Fi password unencrypted (on-screen or broadcast over the air); only BLE setup encrypts. | Anyone nearby during setup can recover the home Wi-Fi password. | [add-camera.md](add-camera.md), [data-inventory.md](data-inventory.md) | static |
| 2 | **Device/LAN password protected only by reversible obfuscation.** IoTVideo's LAN-password check frame uses "mode-1" RC5 whose key is derived from the cleartext header (not a secret) — effectively cleartext on the LAN. Legacy Gwell's on-wire password is `fold(MD5(pw))`, keyless and computable. | A passive LAN observer can recover or replay the device password. | [protocol-iotvideo.md](protocol-iotvideo.md), [protocol-gwell.md](protocol-gwell.md) | static |
| 3 | **Access token stored unencrypted at rest.** The IoTVideo access token is kept in an MMKV store opened with a null crypt key (no Keystore / EncryptedSharedPreferences); device passwords are stored per-camera (`contactPassword`). | Token/password theft on a rooted, backed-up or lost phone; the token is what gates P2P ([protocol-iotvideo.md](protocol-iotvideo.md)). | [app-overview.md](app-overview.md) | static |
| 4 | **Hardcoded vendor key** `www.gwell.cc` used for the P2P bootstrap frames. | Not a secret; same across all installs — useful to a reimplementer, and means those frames offer no real protection. | [protocol-iotvideo.md](protocol-iotvideo.md) | static |
| 5 | **No TLS certificate pinning** anywhere in the app or native libraries. | A system-CA-level MITM can read/modify the cloud API traffic. (Double-edged: this is also what makes authorised interception for research feasible.) The app does still trust only system CAs (SDK 36), so a user-added proxy CA is rejected without root. | [crypto-and-auth.md](crypto-and-auth.md), [app-overview.md](app-overview.md) | static |
| 6 | **RTSP password is `MD5("admin:HIipCamera:" + pwd)`**, pushed to the camera over the P2P channel. | Weak, unsalted KDF for the local stream credential; the HA1 transits the P2P link. | [crypto-and-auth.md](crypto-and-auth.md) | tested |

## Network & privacy exposure

| # | Finding | Impact | Where | Confidence |
|---|---------|--------|-------|------------|
| 7 | **Cleartext HTTP endpoints.** 4G SIM charging (`uc.api.china-m2m.com:80`) and customer-service run over plain HTTP; Tencent MTA (`pingma.qq.com`) is cleartext-permitted. | Data on these paths is interceptable/modifiable in transit. | [cloud-endpoints.md](cloud-endpoints.md) | static |
| 8 | **Cloud media uploads switched ON automatically at setup**, with no cloud-plan check: adding a camera enables alarm-snapshot upload (`uploadImgEna=1`) and cloud video upload (`pause=0`). | Snapshots (and possibly video) start leaving the camera to the vendor cloud by default, before any conscious opt-in. | [privacy-uploads.md](privacy-uploads.md), [camera-uploads.md](camera-uploads.md) | static |
| 9 | **GPS coordinates + camera LAN IP sent to the cloud at bind** (`latitude`/`longitude`/`ip`). | Precise location of the camera is disclosed to the vendor. | [data-inventory.md](data-inventory.md) | static |
| 10 | **Device-location upload endpoint present** (`collect.cloud-links.net`, `PositionInfo`), though no caller in this version. | Dormant capability to upload location; watch for it in a capture. | [cloud-endpoints.md](cloud-endpoints.md), [data-inventory.md](data-inventory.md) | static |
| 11 | **Heavy third-party telemetry/ad SDKs** bundled (ByteDance/Pangle, AppLovin, Mintegral, Tencent Bugly + MTA, Facebook, carrier one-click login). | Broad device/usage data to many parties beyond the camera vendor. | [data-inventory.md](data-inventory.md), [privacy-uploads.md](privacy-uploads.md) | static |

## Device access control

| # | Finding | Impact | Where | Confidence |
|---|---------|--------|-------|------------|
| 12 | **ONVIF answers without authentication.** Device info (model, firmware, MAC), capabilities, media profiles and stream URIs are all returned with no credentials on the test camera's port 5000. | Anyone on the LAN can fingerprint the camera and learn its stream paths; only the RTSP pull itself needs the password. | [onvif.md](onvif.md) | tested |
| 13 | **`forceBind=true` overrides existing ownership.** A bind request can take over a camera already bound to another account; what the server checks before allowing it is unknown. | Possible camera takeover if the server does not verify possession/ownership — to be confirmed server-side. | [add-camera.md](add-camera.md), [protocol.md](protocol.md) | needs capture |
| 14 | **No confirmed firmware-signature check.** The camera downloads and flashes its own firmware; whether it verifies a signature is unconfirmed. | If the update server or its DNS is controlled and images are unsigned, malicious firmware could be flashed — so TianEye blocks OTA by default. | [ota-firmware.md](ota-firmware.md) | needs capture |

## Transport crypto (observations, not necessarily weaknesses)

- **RC5-32** (a dated 64-bit-block cipher) protects P2P frames; the session key for IoTVideo derives
  from the cloud-issued access token, the legacy key from the device password
  ([protocol-iotvideo.md](protocol-iotvideo.md), [protocol-gwell.md](protocol-gwell.md)).
- **Request signing is HMAC-SHA1** keyed by the access token — proves the client to the server, not
  the server to the client, so a substitute server that holds the shared material can answer
  ([crypto-and-auth.md](crypto-and-auth.md)).

## What this means for TianEye / an owner

- The strongest privacy posture is the project's core recommendation: **block the camera's WAN
  egress** and run it LAN-only ([cloud-redirect.md](cloud-redirect.md)). That neutralises findings
  7–11 at a stroke.
- On the LAN, treat findings 1–3 and 12 as reasons to isolate cameras on their own VLAN and set a
  strong, unique device/RTSP password.
- Findings 13–14 are the open risks that need a capture or firmware pull to settle (deferred phase).

## Still unexamined (gaps for a fuller security review)

1. Whether the user's **Yoosee account login password** (not just the device password) is persisted
   on the phone, and in what form.
2. TLS versions / cipher suites the app and camera negotiate (needs a capture).
3. The server-side checks behind `forceBind` and the account **sharing/permission** model (privilege
   escalation surface).
4. Firmware signature verification and the camera's own update host (te-2dc / te-99x.12).
