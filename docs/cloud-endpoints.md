# Domains and IP Addresses Used

**Source:** static analysis of com.yoosee 6.46.1 (jadx Java + native `.so` strings).
**Status:** app side, static only. Camera-side hosts and actual ports still need captures
(te-2dc) and firmware analysis (te-99x.12). Ports marked "TBD" are not in the app strings.

The production host list is authoritative: it comes from the app's own release-config class
`ci/a.java` (`ReleaseHostUrl`), which returns one host per purpose. Test/debug variants come from
a debug settings screen and are not used in a normal build.

---

## Production domains (app → cloud)

All operated by Gwell / Tencent-Cloud-Links unless noted. All resolve to Chinese-operated
infrastructure, so **all are redirect/block targets** for a no-China-traffic setup. Country here
means who operates the service, not the TLD — `cloud-links.net` is the same operator as
`cloudlinks.cn`.

Purposes marked **verified** come from the app's own purpose map (`WebHostUrlIndex` in
`DebugApiImpl.getGWebHostUrlMap()`) or from the class that consumes the host. Unmarked rows are
inferred from paths and names.

| Domain | Port | How it is used | Consumer / evidence | Operator |
|--------|------|----------------|---------------------|----------|
| `openapi-iot.cloudlinks.cn` | 443 | Main SaaS API: login, token refresh, device list/bind/unbind, settings, net-config token | `ReleaseHostUrl.a()`; Compose + main-process SaaS init — **verified** | Gwell |
| `list.iotvideo.cloudlinks.cn` | 443 | IoTVideo P2P index: device ID → relay node | `i()` → `IoTSdkInitor` (IoTVideo SDK init) — **verified** | Gwell/Tencent |
| `p2p1..p2p10` (see note) | UDP, TBD | Legacy Gwell P2P servers (registration, hole-punch, relay) | `l()` → `GSdkInitor.registerGSdk` → `P2PInitParam` — **verified** | Gwell |
| `vasapi.cloudlinks.cn` | 10443 | Value-added services: cloud storage, AI, orders; safe-check page | `p()` = `VAS` — **verified** | Gwell |
| `saas-playback.cloudlinks.cn` | 443 | Cloud recording playback | `b()`, passed to SaaS SDK config in `MainProcessApp` | Gwell |
| `upg1.cloudlinks.cn` | 443 | **Firmware and app** update lookups | `d()` = `DEVICE_UPDATE` and `APP_UPDATE` — **verified** | Gwell |
| `res.zhiduodev.com` | 443 | **Image upload** (`POST res/pic/upload`, multipart): the photo attached to in-app feedback | `c()` = `UPLOAD_IMAGE`; only caller `FeedBackActivity` — **verified** | Gwell (separate domain) |
| `collect.cloud-links.net` | 443 | **Device location upload** (`POST api/deviceinfo/position.ashx?PositionInfo=…`) — present but **no caller** in 6.46.1 | `f()` = `UPLOAD_LOCATION` — **verified** | Gwell |
| `uc.api.china-m2m.com` | **80 (plain HTTP)** | 4G SIM data-plan charging | `j()` = `CHARGE` (`ServicePath.CHARGE_BASEURL`) — **verified** | China Mobile IoT |
| `datasink.cloudlinks.cn` | 443 | Sensors Analytics (`/sa?project=default\|production\|dophigo`) | URL constants | Gwell (SensorsData) |
| `advertise.cloudlinks.cn` | 443 | In-app ads | `h()` → `ThirdModuleInitializer` — **verified** | Gwell |
| `trade.cloudlinks.cn`, `customer-service.cloudlinks.cn` | 443 | Purchases and customer service, loaded in the app's webview | `g()` → `WebViewFragment` — **verified** | Gwell |
| `customservicesystem.cloudlinks.cn` | **80 (plain HTTP)** | Customer-support system | `e()`, passed to SaaS SDK config in `MainProcessApp` | Gwell |
| `agent.cloudlinks.cn` | 443 | Purpose not traced | Compose module `init` config | Gwell |
| `personalinfo.cloudlinks.cn` | 443 | Privacy policy, device and permission notices (webviews) | `/pages/protocol/privacy/*.html` | Gwell |
| `faq.cloud-links.net` | 443 | Help center and the **firmware upgrade** info/result pages | `/help/index.html`, `/upgrade/upgrade*.html` | Gwell |
| `help.cloudlinks.cn` | 443 | WeChat help dialog page | `/help/wechatDialog.html` | Gwell |
| `api1.cloudlinks.cn`, `api2.cloudlinks.cn`, `api3.cloud-links.net`, `api4.cloud-links.net` | 443 | Gwell "web" SaaS API host list (tried in order) | `n()` = `SAAS_URL` — **verified** | Gwell |
| `abroad.cloud-links.net` | 443 | Third-party (social) login | `m()` = `THIRD_LOGIN` — **verified** | Gwell |
| `abroad-g.cloud-links.net` | 443 | Role not traced | `o()`, passed to SaaS SDK config in `MainProcessApp` | Gwell |
| `gwell.cc` | 443 | Notices; also the native libp2pav fallback domain and `support@gwell.cc` | `k()` = `NOTICE` — **verified** | Gwell |

**P2P host note:** the full relay list from `ReleaseHostUrl.l()` is
`p2p1.cloudlinks.cn, p2p2.cloudlinks.cn, p2p5..p2p10.cloudlinks.cn` plus
`p2p3.cloud-links.net, p2p4.cloud-links.net`.

**Service-discovery paths** (sent to the index hosts at session setup, must be answered to route
cameras locally):

| Stack | Path |
|-------|------|
| Legacy Gwell (`libp2pav.so`) | `/gwellcloud/service/ListService/GetServiceList` |
| IoTVideo (`libiotvideomulti.so`) | `/iotvideo/service/ListService/GetServiceList` |

---

## Test / debug-only domains (not used in release builds)

Reachable only by flipping keys in the hidden `gwell_debug` settings screen. Listed so a capture
that shows them is understood as a debug build, not normal behaviour.

`test-openapi-iot`, `testv1-openapi-iot`, `test-saas-playback`, `saas-trade-develop`,
`webapi-gw-test`, `vasapitest`, `test-gwell.list`, `test-help`, `customservicesystem-test`,
`test-iotvideo.list` — all `.cloudlinks.cn`; `test-abroad-thirdLogin.cloud-links.net`; plus hardcoded test IPs (below).

---

## Hardcoded IP addresses

DNS redirection does **not** affect these — only a firewall rule by IP does.

| IP[:port] | How it is used | Redirectable by DNS? | Notes |
|-----------|----------------|----------------------|-------|
| `114.114.114.114` | Target of a `ping -c 1` connectivity check (`NetPingKits.isNetworkAvailable`) | No | 114DNS, a Chinese public resolver. Ping only, no data sent. A local setup should still block or expect it |
| `123.125.99.31` | China Unicom one-click-login SDK (`com.unicom.online.account.kernel`) | No | Operator-auth SDK; only in the carrier-login path |
| `120.78.134.189:8080` | `TEST` entry in the app's host map (`https://`) | No | Debug/test only |
| `42.193.54.229:13570` | Ad SDK test server (debug builds) | No | Hardcoded; debug only |
| `39.108.193.125:7777` | Message/notice test server (debug builds) | No | Hardcoded; debug only |
| `127.0.0.1`, `0.0.0.0`, `255.255.255.255`, `192.168.1.1` | Loopback / any / broadcast / default-gateway guesses | n/a | Local only, not cloud |

Values like `2.5.4.x` / `2.5.29.x` in the strings are X.509 ASN.1 OIDs, and `4.0.20.301` etc.
are version numbers — not addresses.

---

## Third-party domains (telemetry / ads / login — block, not redirect)

Not Gwell infrastructure; carry analytics, ads or social login, not camera control.

| Domain | Purpose | Operator country |
|--------|---------|------------------|
| `android.bugly.qq.com` | Tencent Bugly crash reporting | China (Tencent) |
| `pingma.qq.com` / `182.254.116.117` | Tencent MTA (Mobile analytics), cleartext-permitted | China (Tencent) |
| `*.cmpassport.com`, China Unicom/Mobile/Telecom one-click login hosts | Carrier number auth | China |
| `mp.weixin.qq.com` | WeChat sharing | China (Tencent) |
| `beian.miit.gov.cn` | ICP filing lookup (webview) | China (gov) |
| `play.google.com`, `*.googleads.g.doubleclick.net`, `pagead2.googlesyndication.com`, `admob-gmats...appspot.com` | Google Play, AdMob ads | US (Google) |
| `*.facebook.com`, `developers.facebook.com` | Facebook login / SDK | US (Meta) |
| `*.applovin.com`, `*.tradplusad.com`, `sf16-static.i18n-pglstatp.com`, `*.bykv.vk.openvk...` (Pangle/ByteDance), Mintegral hosts | Ad networks | mixed (US / China) |
| `opencloud.wostore.cn` | China Unicom app store services | China |

---

## Open questions (need capture / firmware)

1. **UDP ports for P2P relay** — not in app strings; confirm with `tcpdump` (te-2dc).
2. **Which hosts the camera itself contacts** — the app list is not the camera list. The camera
   has its own update and P2P servers (te-99x.12 firmware, te-2dc capture).
3. **Whether the camera uses hardcoded resolvers or DoH** instead of DHCP DNS — decides if DNS
   redirection is enough (te-99x.9).
