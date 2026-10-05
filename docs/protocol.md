# Gwell/IoTVideo Cloud API Protocol

Findings from static analysis of the Yoosee Android app (com.yoosee 6.46.1, arm64).
All endpoints are on `openapi-iot.cloudlinks.cn:443` (HTTPS). No certificate pinning.

---

## Request Authentication

Every request carries these HTTP headers, added by `AddBaseParamsInterceptor`:

| Header | Value |
|--------|-------|
| `x-iotvideo-accessid` | User access ID obtained after login |
| `x-iotvideo-nonce` | Random non-negative integer (new per request) |
| `x-iotvideo-timestamp` | Unix timestamp in whole seconds |
| `x-iotvideo-area` | Area/region code (e.g. `CN`, `US`) |
| `x-iotvideo-appver` | App version string |
| `x-iotvideo-appid` | `d591b466644a0420e5f29aefb0cf0088` (Yoosee app ID) |
| `x-iotvideo-signature` | HMAC-SHA1 signature — see below |
| `Content-Type` | `application/x-www-form-urlencoded` (GET) / `application/json` (POST) |
| `Accept` | `application/json` (POST only) |

### Signature Algorithm

Source: `HttpUtils.iotVideoSignature` + `AddBaseParamsInterceptor.addSignatureHeader`

**Step 1 — Build the signing input** (sorted alphabetically by key):

```
host:<hostname>
payload:<sha256hex-of-body>      ← POST/PUT only; omit for GET
x-iotvideo-accessid:<accessId>
x-iotvideo-appid:<appId>         ← anonymous requests only
x-iotvideo-appver:<appVer>       ← anonymous requests only
x-iotvideo-nonce:<nonce>
x-iotvideo-timestamp:<ts>
<query-param-key>:<value>        ← GET requests: all query params, sorted
…
```

Each line is `key:value`, lines separated by `\n`, **no trailing newline**.
Keys come from a `TreeMap`, so ordering is always lexicographic (Java natural order).

**Step 2 — Compute signature**:

- *Anonymous / pre-login*: `HMAC-SHA1(key=secretKey, data=stringToSign)` → Base64 (no line wrap).
  `secretKey` is a provisioned anonymous key passed to the interceptor constructor.

- *Authenticated*: JNI call `IIoTVideoAbility.sha1WithBase256(stringToSign, accessToken)`,
  implemented in `libiotvideomulti.so`. **Confirmed by decompilation** to be
  HMAC-SHA1 keyed by `accessToken` with Base64 output (it calls `iv_hmac_encode_sha1_with_base64`).
  Same algorithm as the anonymous path, just keyed by the token instead of the anonymous secret.

If the result starts with `_` or contains a space it is additionally URL-encoded before being
set as the header value.

### POST Body Base Parameters

Every POST body includes these common fields (added by the same interceptor):

```
accessId, accessToken, productId, apiVersion, appId, appName,
appToken, appVersion, channel, funcSupport, language, pkgName,
platform, region, regRegion, sdkVersion, terminalOS, uniqueId
```

`channel` is `"googleplay"` for Google Play builds, `"china"` otherwise.

---

## API Endpoints

Base URL: `https://openapi-iot.cloudlinks.cn`

### Authentication Flow

#### Login

`POST /openapi/app/user/login/account`

Three login modes, distinguished by the `loginMode` field:

| Mode | Fields |
|------|--------|
| Email | `email`, `loginMode="email"`, `pwd`, `uniqueId`, `loginRegion` |
| Mobile | `mobile`, `loginMode="mobile"`, `mobileArea`, `pwd`, `uniqueId`, `loginRegion` |
| User ID | `userId`, `loginMode="userId"`, `pwd`, `uniqueId`, `loginRegion` |

Returns `accessId` and `accessToken` (short-lived).

#### Get / Regenerate Access Token

`GET /openapi/app/user/reGenUsrAcceccToken`

Query params: `uniqueId`

Called when `accessToken` needs rotation. Returns a fresh token.

#### Refresh User Token

`POST /openapi/app/user/refreshUserToken`

Overloads observed:
- No extra body (uses base params only)
- Body with `UniqueId`
- Body with arbitrary extra map

Token lifetime and refresh cadence unknown — need live capture to confirm.

---

### Device Management

#### List Devices

`GET /openapi/app/user/device/listDevice`

Query params: `lastDeviceId=0` (pagination cursor)

#### Bind Device

`POST /openapi/app/user/device/bind`

Full form:

| Field | Notes |
|-------|-------|
| `devId` | Camera device ID |
| `tid` | TID / serial from QR code |
| `remarkName` | User-assigned name |
| `permission` | Integer permission level |
| `bindToken` | Token from `GET_CONFIG_NET_TOKEN` (net config flow) |
| `devType` | Device type integer |
| `snCode` | Serial number |
| `timeArea` | Timezone area string |
| `timeZone` | Timezone offset integer |
| `latitude` | GPS (optional) |
| `longitude` | GPS (optional) |
| `ip` | LAN IP of device (optional) |
| `productID` | Product ID string |

Short form (minimal bind): `devId`, `tid`, `remarkName`, `permission`, `bindToken`, `devType`

Force bind (override ownership): `devId`, `forceBind=true`

#### Unbind Device

`POST /openapi/app/user/device/unbind`

Body fields not yet traced; source is `DeviceApiImpl`.

#### Net Config Token

`GET /openapi/netcfg/cloud/netcfg/genbindtoken`

Generates the `bindToken` used in the bind call above. Called during Wi-Fi provisioning.

---

## Other Known Paths

From `Protocol.java` constants (not yet traced):

```
/openapi/app/user/regist                     registration
/openapi/app/user/pushTokenBind              push notification registration
/openapi/app/device/...                      device-side endpoints (TBD)
```

Full list of ~150 paths is in `spike/yoosee-re/6.46.1/jadx/sources/com/yoosee/kmpsaas/accountmgr/constants/Protocol.java`.

---

## Open Questions

1. ~~Exact HMAC key material for authenticated requests?~~ The access token (see Signature Algorithm).
2. Token lifetime — how long before `refreshUserToken` is required.
3. Device keepalive mechanism — whether it is HTTP polling or P2P heartbeat.
4. ~~Alarm/push flow?~~ See [alarm-delivery.md](alarm-delivery.md).
5. Response envelope format — success/error codes, field names.
   Need a live capture (task te-2dc) to confirm.
