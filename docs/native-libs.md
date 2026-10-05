# Native Library Inventory

Static analysis of arm64-v8a `.so` files from Yoosee 6.46.1.
Ghidra project at `spike/ghidra/yoosee-6461/` (gitignored). Symbol exports in `spike/ghidra/exports/`.

---

## Library Roles

| Library | Size | Role |
|---------|------|------|
| `libiotvideomulti.so` | 2.7 MB | **Primary P2P SDK** — Tencent IoTVideo, handles device init/register, A/V streaming, messaging, RTSP password derivation, request signing |
| `libgwmediaplayer.so` | 3.0 MB | **Legacy Gwell player**: JNI layer the app calls (`GwVideoPlayerNative`, PTZ, OTA, alarm settings); drives `libp2pav.so` |
| `libp2pav.so` | 1.0 MB | **Legacy Gwell P2P** — P2PAV protocol (P2PV1/V2), A/V encoding/decoding/recording, MTP sessions |
| `libelianjni.so` | 100 KB | MediaTek "Elian" smart connection: broadcasts Wi-Fi SSID + password during setup ([add-camera.md](add-camera.md)) |
| `liblarksmarkemtmfJNI.so` | 370 KB | Sound-wave Wi-Fi setup (`EMTMFSDK.sendWifiSet`) |
| `libsimpleconfiglib.so` | 70 KB | Realtek Simple Config; bundled but never called |
| `libgwbase.so` | 1.3 MB | Audio processing (WebRTC AGC, NS), logging infrastructure |
| `libnms.so` | 275 KB | Stripped (only `JNI_OnLoad` exported); role unknown — likely Network Management System |
| `libcrypto_gw.so` | 5.3 MB | OpenSSL/BoringSSL fork (custom Gwell build) |
| `libmbedtls_gw.so` | 850 KB | mbedTLS (custom Gwell build) |

---

## libiotvideomulti.so

Source tree (from debug symbols): `/home/lingyuanze/gitlab/iotvideop2p/jni/src/`
Key source files visible in binary: `iot_video_link_app.c`, `giot_eif.c`, `giot_encrypt.c`,
`gute_session.c`, `iv_push_channel.c`, `iv_push_session.c`, `p2pc_chnnel_v2.c`, `p2pc_comm.c`,
`p2pc_mtpchnnel.c`, `p2pc_mtpcomm.c`.

### JNI API Surface

**SDK lifecycle** (`IoTVideoInnerInitializer`):
```
nativeInit                  initialize SDK with config
nativeRegister              register app/user with SDK
nativeUnregister            deregister
nativeSetListener           set event callbacks
nativeGetState              query SDK state
nUpdateAccessToken          push new accessToken after refresh
```

**A/V player** (`AIoTBasePlayer`):
```
nCreateIoTPlayer(type, deviceId, chId, chSource)  create player session
nGetConnectState            0=PREPARING 1=ASSIGN_CHN 2=WAKE_UP_DEV 3=CONNECTING 4=CONNECTED 5=DISCONNECTING 6=DISCONNECTED
nGetConnectMode             -1=UNKNOWN 0=RELAY 1=P2P 2=LAN
nGetConnectProtocol         -1=UNKNOWN, 0=UDP, 1=TCP
nGetWatchingNum             number of concurrent viewers
nSetConnOptInt(key, val)    connection options: "dev_func_cfg", "app_state"
nSetConnOptStr(key, val)    connection options: "user_id_str"
nSetIoTPlayerListener       register callbacks (onConnectState, onRecvData, onError)
nShutdown                   disconnect and tear down
nSendUserData(cmd, data, ts, cb)  send arbitrary bytes to device
```

**Player types** (first arg to `nCreateIoTPlayer`):
| Value | Constant |
|-------|----------|
| 0 | CLOUD_PLAYBACK_PLAYER |
| 1 | LIVE_PLAYER |
| 2 | SD_PLAYBACK_PLAYER |
| 3 | NAS_PLAYBACK_PLAYER |

**Video quality** (negotiated after connect):
| Value | Constant |
|-------|----------|
| 1 | LD |
| 2 | SD |
| 3 | HD |
| 7 | AUTO |

**Messaging** (`MessageMgrInternal`):
```
nExecuteModelCmd(cmdOrdinal, deviceId, path, json, timeout, cb)
nSendMsgToDevice(deviceId, domain, cmd, data, timeout, reliable, encrypted, priority, cb)
nSendMsgToServer(url, data, timeout, cb)
nRefreshPropertyOfDevice    read current device property
nRegisterOfflinePush        register push endpoint
nUnregisterOfflinePush
nHttpProxyRequest           proxy HTTP request through SDK
```

**Model commands** (ordinals for `nExecuteModelCmd`):
| Ordinal | Name | Description |
|---------|------|-------------|
| 0 | WRITE_PROPERTY | Set a device property (path + JSON value) |
| 1 | READ_PROPERTY | Read a device property at path |
| 2 | TAKE_ACTION | Invoke an action on the device |
| 3 | ADD_PROPERTY | Add a new property |
| 4 | DELETE_PROPERTY | Remove a property |

**Message domains** (for `nSendMsgToDevice`):
- `BUILT_IN` = 0
- `USER` = 1

**Built-in command codes** (for `domain=BUILT_IN`; full list verified against `BuiltInCmd`):
| Value | Constant | Notes |
|-------|----------|-------|
| 0 | PLAYBACK_GET_LIST_V1 | |
| 1 | PLAYBACK_PAUSE | |
| 2 | PLAYBACK_RESUME | |
| 3 | PLAYBACK_SEEK | |
| 4 | PLAYBACK_STREAM_BEGIN | |
| 5 | SET_VIDEO_DEFINITION | change live quality |
| 6 | AP_NET_CONFIG | Wi-Fi provisioning via AP mode |
| 16 | PLAYBACK_GET_LIST_V2V3 | |
| 17 | PLAYBACK_END_OF_FILE | |
| 18 | PLAYBACK_GET_DATE_LIST | |
| 19 | DOWNLOAD_FILE_INFO | |
| 20 | DOWNLOAD_CANCEL | |
| 21 | DOWNLOAD_REQUEST | |
| 22 | DOWNLOAD_EXCEPTION | |
| 23 | DOWNLOAD_FILE_DATA | |
| 24 | PLAYBACK_SPEED | |
| 25 | PLAYBACK_STRATEGY | |
| 26 | THUMBNAIL_REQUEST | |
| 27 | THUMBNAIL_CANCEL | |
| 28 | DELETE_PLAYBACK_FILES | delete SD recordings |
| 29 | CANCEL_DELETE_FILES | |
| 48 | VIEWER_NUMBER_CHANGED | event: viewer count |
| 49 | TALKER_NUMBER_CHANGED | event: talker count |
| 50 | MICROPHONE_STATE_CHANGE | event |
| 80 | CAMERA_STATE_CHANGE | event |
| 112–121 | RES_QUERY_LIST (112), RES_QUERY_DETAIL, RES_DELETE, RES_CANCEL_DELETE, RES_DOWNLOAD_REQUEST, RES_DOWNLOAD_INFO, RES_DOWNLOAD_DATA (118), RES_DOWNLOAD_CANCEL (119), RES_DOWNLOAD_PAUSE (120), RES_EVENT_NOTIFY (121) | on-device resource (file) management |
| 255 | INVALID | |

**Crypto / key derivation** (`P2PAlgorithmProxy`):
```
nGetRtspPassword(inputParams) → byte[]      derive RTSP password from device params
nGetAnonymousSecureKey(appTag, ts) → str[]  get pre-login anonymous signing key
nSha1WithBase256(signContent, token) → str  HMAC-SHA1 for request signing (native impl)
nSha256WithHex(input) → str                SHA-256 hex
nCheckDevicePwd(deviceId, pwd) → int        verify device password
nSetDevicePwd(deviceId, oldPwd, newPwd)     change device password
nGetTerminalId() → long                     unique device/app terminal ID
nGetTidFromDid(did) → long                  convert device DID to TID integer
nLanDevConnectable(devId) → int             check LAN reachability
nGenH5SecureKey() → str                     generate H5/web secure key
nBleAesEncrypt(data, key, encType) → bytes  BLE provisioning AES encrypt
```

**Service discovery** path: `/iotvideo/service/ListService/GetServiceList`

**Transport**: KCP reliable UDP (IKCP_CMD_PUSH/ACK/WASK/WINS constants present)

---

## libp2pav.so

Older Gwell P2P protocol (pre-IoTVideo), still used alongside `libiotvideomulti.so`.

**Service discovery** path: `/gwellcloud/service/ListService/GetServiceList`

**Key function groups**:

*P2P channel management*:
```
p2pc_chnnel_new / _init / _free / _clear
p2pc_chnnel_new_v2 / _free_v2
p2pc_comm_new / _add_unit / _run / _exit
p2pc_close_peer_mtp_session
p2pc_close_tcpconnection_2_p2psrv
AllocChnByMtpSessionID
```

*MTP session (Gwell media transport)*:
```
mtp_session_new / _free
mtp_session_add_udp_relay / _add_tcp_relay
mtp_session_add_lan_or_nat / _add_tcp_lan / _add_tcp_nat_or_lan
mtp_chnnel_new / _free / _send_mtp_frm / _send_meter_frm
mtp_session_rcv_proc / _rcv_cmd_proc / _rcv_datafrm_proc
```

*AV control (A/V encode/decode/record)*:
```
avctl_CreateAVControl / _DestoryAVControl
avctl_StartAVEncAndSend / _StopAVEncAndSend      encode and send to camera (intercom)
avctl_StartRecvAndDec / _StopRecvAndDec          receive and decode from camera (live view)
avctl_GetVideoFrameToDisplay / _ReleaseVideoFrame
avctl_GetVideoStreamToDisplay
avctl_GetLastDisplayFrame / _GetLastDisplayVideoFrame
avctl_GetAudioDataToPlay
avctl_FillVideoRawFrame / _FillAudioRawData      inject local A/V (for calls)
avctl_StartRecordToFile / _StopRecord            local SD recording
avctl_GetDownloadProgress
avctl_SendUserData                               send arbitrary command bytes
avctl_SetPauseRecvData
avctl_P2PAVJump_C / _P2PAVNext_C / _P2PFastPlay_C  playback seek/next/fastplay
```

*Encryption* (versioned, keyed to Gwell protocol revisions):
```
P2PEncryptGW1
P2PEncryptGW5 / P2PDecryptGW5
P2PGetTreatedPassword        derive encoded password from raw password
```

*Protocol version negotiation*:
- `dstID is P2PV1` / `dstID is P2PV2` (logged during connect)
- `p2plib_version` field in frames
- `dev_type=%d version=%d subversion=%d`

*Transport*: KCP reliable UDP + libevent for async I/O.
*Hardcoded fallback domain*: `www.gwell.cc`

---

## Open Questions for Native Analysis

1. ~~What are the exact parameters of `nGetRtspPassword`?~~ It takes `"admin:HIipCamera:" + password` and returns the Digest HA1 ([crypto-and-auth.md](crypto-and-auth.md), RTSP Authentication).
2. ~~What does `avctl_SendUserData` carry for PTZ?~~ A 28-byte frame `FF FF FF 88 00 02 <cmd> <option> <20-byte payload>`; IoTVideo cameras get JSON via `nSendUserData` instead (camera-control.md, PTZ Control).
3. What is the `dev_func_cfg` connection option bitmask?
4. ~~What path does `TAKE_ACTION` use for PTZ?~~ None for movement: `Action.ptzCheck` is reset/calibration. Moves go through `nativePtzControl` (see camera-control.md).
5. How does `WRITE_PROPERTY` encode video quality change?
6. ~~Does the camera speak RTSP independently?~~ Yes: the test camera serves RTSP on 554 directly; `ProWritable.onvifEn` toggles it.
7. What is `libnms.so`'s actual role? (No exports beyond JNI_OnLoad — may be NMS telemetry or null stub)
