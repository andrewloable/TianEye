# Yoosee App Overview — RE Notes
**Source:** com.yoosee 6.46.1 (build 6454), arm64, pulled from owner device 2026-10-05  
**Tools:** apktool 3.0.2, jadx 1.5.5  
**Status:** Static analysis only. No traffic captured yet.

---

## Package Identity

| Field | Value |
|---|---|
| Package | com.yoosee |
| Vendor application class | com.jwkj.app.YooseeApplication |
| Version | 6.46.1 (build 6454) |
| SDK | minSdk 24 (Android 7), targetSdk 36 |
| Compiled SDK | 36 |
| Launcher activity | com.jwkj.activity.LogoActivity |

---

## Component Counts

| Type | Count |
|---|---|
| Activities | 253 |
| Services | 26 |
| Broadcast receivers | 16 |
| Content providers | 18 |

Notable app-owned service: `com.jwkj.device_setting.device_update.CheckDeviceUpdateService`  
Push: Firebase (FcmPushService) plus Huawei Push, OPPO, Vivo, Xiaomi, Heytap — all third-party push SDKs included.

---

## Network Security Config

File: `res/xml/network_security_config.xml`

```xml
<network-security-config>
    <base-config cleartextTrafficPermitted="true" />
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">android.bugly.qq.com</domain>
    </domain-config>
</network-security-config>
```

**No certificate pinning declared in any of the three config files.**  
Cleartext traffic is permitted globally, but the production base URLs are hardcoded `https://`, so the app
still uses TLS to the cloud. There is no `<trust-anchors>` block and the app targets SDK 36, so only
**system** CAs are trusted; a user-installed proxy CA is rejected. No code-level pinning was found either
(see [crypto-and-auth.md](crypto-and-auth.md)).

---

## Permissions Relevant to Networking / Camera Access

- INTERNET
- ACCESS_NETWORK_STATE, ACCESS_WIFI_STATE, CHANGE_WIFI_STATE, CHANGE_WIFI_MULTICAST_STATE
- BLUETOOTH, BLUETOOTH_ADMIN, BLUETOOTH_ADVERTISE, BLUETOOTH_CONNECT, BLUETOOTH_SCAN
- ACCESS_FINE_LOCATION (required for Wi-Fi scan on Android 10+)
- CAMERA, RECORD_AUDIO

---

## Cloud Server Hostnames (from static Java analysis)

These are extracted from the decompiled Java sources. The app stores the active server config in a SharedPreferences key named `gwell_debug`.

### Production

The main hosts only. The full list, with ports and the code that uses each host, is in
[cloud-endpoints.md](cloud-endpoints.md).

| Purpose | Hostname / URL |
|---|---|
| **Main API** | `https://openapi-iot.cloudlinks.cn` |
| Notices | `https://gwell.cc/` |
| P2P index / device list | `list.iotvideo.cloudlinks.cn` |
| P2P relay nodes | `p2p1.cloudlinks.cn` through `p2p10.cloudlinks.cn` |
| P2P relay nodes (alternate) | `p2p3.cloud-links.net`, `p2p4.cloud-links.net` |
| Value-added services (VAS) | `https://vasapi.cloudlinks.cn:10443/` |
| Analytics / telemetry | `datasink.cloudlinks.cn` |
| Advertising | `advertise.cloudlinks.cn` |
| Custom service / support chat | `customservicesystem.cloudlinks.cn` |
| Privacy policy / user agreement | `personalinfo.cloudlinks.cn` |
| FAQ / firmware update pages | `faq.cloud-links.net` |

### Test / Debug (visible in debug menu)

| Purpose | URL |
|---|---|
| API gateway (test) | `https://webapi-gw-test.cloudlinks.cn:443` |
| P2P index (test) | `test-gwell.list.cloudlinks.cn` |
| VAS API (test) | `https://vasapitest.cloudlinks.cn:10443` |
| Custom service (test) | `http://customservicesystem-test.cloudlinks.cn/` |

The app has a built-in debug screen where these can be toggled at runtime — useful for live-capture testing.

### Third-party / Analytics

| Purpose | Domain |
|---|---|
| Bugly (crash reporting) | `android.bugly.qq.com` |
| Sensors Data (analytics) | `datasink.cloudlinks.cn/sa` |
| Object storage | Tencent COS, used only to *download* PTZ preset images; the COS upload routine is never called ([privacy-uploads.md](privacy-uploads.md)). No Aliyun OSS SDK is bundled |

---

## P2P SDK

Two P2P SDKs are initialised at startup, each with its own pipe-delimited server string:

| SDK | Initialiser | Server string |
|-----|-------------|---------------|
| IoTVideo (`libiotvideomulti.so`) | `IoTSdkInitor` → `IoTVideoInitializer` | `|list.iotvideo.cloudlinks.cn` |
| Legacy Gwell (`libgwmediaplayer.so` → `libp2pav.so`) | `GSdkInitor.registerGSdk` → `P2PInitParam` | `|p2p1.cloudlinks.cn|p2p3.cloud-links.net|p2p2…|p2p10.cloudlinks.cn` |

Full host map: [cloud-endpoints.md](cloud-endpoints.md).

---

## Key Third-Party SDKs

| SDK | Source |
|---|---|
| Firebase (FCM push) | Google |
| Huawei Push, HMS | Huawei |
| Bytedance (TikTok ads) | 1590 classes |
| AppLovin | 749 classes |
| Tencent (Bugly, Weixin) | 538 classes |
| Facebook | 301 classes |
| Alipay | 258 classes |
| Gwell SDK | `com/gwell` (51 classes) |

---

## Open Questions for Follow-up Tasks

1. ~~Does `gwell.cc` handle auth?~~ No: auth and binding are on `openapi-iot.cloudlinks.cn` ([protocol.md](protocol.md)).
2. ~~Code-level pinning?~~ None found ([crypto-and-auth.md](crypto-and-auth.md)).
3. P2P session setup: see [native-libs.md](native-libs.md); wire format still needs a capture.
4. ~~Binding flow?~~ See [protocol.md](protocol.md).
5. Ports: the test camera serves RTSP on TCP 554 and ONVIF on 5000 and has TCP 50000 open
   (unknown binary protocol). P2P UDP ports still need a capture.
