# IoTVideo Protocol — Transport Layer

Wire format of the IoTVideo P2P transport in `libiotvideomulti.so`, for a Go reimplementation
(epic te-qgj, task te-qgj.2). From static analysis (Ghidra on `spike/ghidra`); source file names in
the binary are from Tencent's `iotvideop2p/jni/src`. **Nothing here is confirmed against a live
capture yet** (deferred: te-2dc). This is the "gute" framing that every control and media message
rides on; the session/handshake and application layers are separate tasks (te-qgj.3–.5).

All multi-byte integers are little-endian (the library is ARM64, reads are native).

## Where it sits

```
application  (model cmds, user data, A/V)   ← te-qgj.5
  gute frame (this doc): header + checksum + RC5
  reliability: KCP over UDP, or TCP relay
  socket:     UDP (v4/v6) direct/NAT/LAN, or TCP to a relay
```

A camera session holds up to three sockets: an IPv4 UDP socket, an IPv6 UDP socket, and optionally
a TCP connection to a relay. Each outgoing frame carries its own destination (`sockaddr`); the
sender picks TCP if a relay link is open, otherwise the UDP socket matching the destination's
address family. Reliable streams (A/V, bulk data) run **KCP** (the `IKCP_CMD_*` push/ack/window
constants are present) on top of UDP; small control frames are sent directly with their own
ACK/retransmit logic (`iv_gutes_send_proc` / `iv_gutes_resend_proc` / `iv_gutes_pkt_send_ack`).

## The gute frame header (24 bytes)

Every frame begins with a fixed 24-byte (`0x18`) header, then the payload. The total length field
counts header + payload.

| Offset | Size | Field | Notes |
|--------|------|-------|-------|
| 0 | 1 | magic | `0x7E` (`~`) for normal frames; `0x7F` seen for a second class |
| 1 | 1 | type | message/command type. Special values `0xAA`, `0xB2` are handled apart in the sender |
| 2 | 2 | total length | header + payload, in bytes |
| 4 | 8 | id | session/peer id — encrypted as one block (see below) |
| 12 | 4 | body-prefix word | first encrypted word; carries sequence/aux data |
| 16 | 4 | checksum | XOR check (below); computed before encryption |
| 20 | 4 | flags | option bits (below) |
| 24 | … | payload | may be compressed and/or encrypted |

### Flags word (offset 20)

| Bit(s) | Meaning |
|--------|---------|
| 0 | payload is zlib-compressed (`gute_frm_unzip` on receive) |
| 16–17 | encryption mode: `0` none, `1` per-frame-key RC5, `2` session-key RC5 |
| 18–19 | reliability: `1` or `3` ⇒ the frame must be ACKed; the receiver sends an ACK and drops duplicates |
| 21 | "response" flag — skip the checksum check on this frame |
| 22 | extended header variant: the encrypted region is `total − 0x68` instead of `total − 0x18` |
| 24 | an extra 16 bytes sit between header and payload (`total − 0x10` more) |
| 25 | detect/handshake frame — sent and received **without** encryption (the bootstrap path) |

The low byte of the flags word (offset 22) duplicates the encryption-mode bits, so a byte read there
gives mode `0/1/2` directly.

## Checksum (offset 16)

A 32-bit XOR, computed over the header and payload **before** encryption:

```
chk = (flags & 0x00FFFFFF) XOR word[0] XOR word[4] XOR word[8] XOR word[12]
for each 4-byte word w of the payload:   chk ^= w
```

where `word[n]` is the 4 bytes at header offset `n`. The checksum field itself (offset 16) is
excluded. The "payload length" for this loop is the encrypted-data length (below), rounded down to
4-byte words. On receive, the check is skipped when flag bit 21 is set.

**Encrypted-data length** = `total_length − 0x18`, minus `0x50` more if flag bit 22 is set, minus
`0x10` more if flag bit 24 is set.

## Encryption (RC5)

The cipher is **standard RC5-32** in 8-byte (64-bit) blocks, applied per-frame. The key schedule
uses the canonical RC5 constants (`Pw = 0xB7E15163`, `Qw = 0x9E3779B9`), so a stock RC5-32 library
will interoperate once the **round count** is pinned (set at context init; read in te-qgj.4). The
key can be up to 256 bytes. The implementation also carries 4- and 16-byte block variants, selected
by a byte in the context, but the gute frame always uses the 8-byte one.

A session keeps three RC5 key schedules:

- **id key** — encrypts the 8-byte `id` field (offset 4). After encrypting that block, the two
  32-bit halves are chained against the next two header words:
  `id[0..3] ^= word[12]; id[4..7] ^= word[16]`. Decrypt reverses the order.
- **per-frame key** (mode 1) — the RC5 key is built *from the frame header itself*: 4 bytes at
  offset 0 followed by 3 bytes at offset 20 (= 7 bytes, zero-padded to the key length). With this
  mode the key is derivable from the cleartext header, so **mode 1 is obfuscation, not
  confidentiality** — anyone who knows the scheme can decrypt it. It encrypts the 8 bytes at offset
  12 (body-prefix + checksum), then each 8-byte payload block.
- **session key** (mode 2) — a negotiated key established during certification (te-qgj.4). Same
  coverage as mode 1 (offset 12 onward), but the key is secret, so mode 2 is the real protection for
  session traffic.

Mode 0 frames are plaintext (used for the detect/handshake bootstrap, flag bit 25).

## Fragment/detect obfuscation

Detect and fragment frames apply a separate lightweight scramble to the header's id region: a random
16-bit seed is written into the frame, OR-folded across the eight 16-bit words from offset 4 to 19,
and each of those words is XORed with the seed. The receiver recovers the seed and reverses it. This
is a checksum-cum-obfuscation for the unencrypted bootstrap frames, not a cipher.

## Receive path (summary)

1. Read the frame; if the magic region isn't recognised or the id doesn't match this session's
   expected peer, drop it.
2. If it's a detect frame (flag 25), handle the handshake and stop.
3. Otherwise decrypt per the mode bits, decompress if flag 0 is set, verify the checksum unless flag
   21 is set.
4. If reliability bits require it, send an ACK and discard duplicates (tracked in a red-black tree
   by sequence).
5. Dispatch to the registered callback by frame type.

---

# Session Setup (te-qgj.3)

How a session gets from "I have a device id" to "I have an open gute channel to the camera". Four
stages: discover servers, register with an access server, call the device, pick a route. Server
roles and message identifiers are from the binary's strings and functions; exact wire bytes of each
control message need a capture (te-2dc).

## 1. Service discovery

`GET /iotvideo/service/ListService/GetServiceList` on the index host
(`list.iotvideo.cloudlinks.cn`, [cloud-endpoints.md](cloud-endpoints.md)), with query params
`serviceId` and `token`. The response lists the servers the terminal should use, in two roles:

- **Asrv (access server)** — the server the terminal keeps a standing connection to; it relays
  signalling and offline messages. Entries carry IPv4 and IPv6 address + port.
- **p2psrv (P2P/rendezvous servers)** — the `p2p1..p2p10` relays; used to locate and reach devices
  and, if direct fails, to relay media.

TianEye emulating discovery means answering this call with its own address in both roles — the most
valuable single interception point, since it steers everything after
([cloud-redirect.md](cloud-redirect.md)).

## 2. Register with the access server

The terminal connects to an Asrv (TCP, or UDP) and sends an **init-info / online** message
(`gat_send_init_info_msg`, acked by `..._ack`), which registers its terminal id and platform. It
then keeps the link alive with heartbeats — the code has 35 s, 40 s and 50 s heartbeat variants
(`gat_send_heart_frm_*`). Signalling after this point is a request/response layer keyed by
`{msg_id, tid, path, code, json}` (`tid` = the device id as a string): the same shape the
HTTP-over-P2P proxy and model commands ride on.

## 3. Call the device and find a route

To open media/control to a camera the terminal runs a **detect** phase and a **calling** exchange:

- **Detect** (`gat_send_detecReq_2_allp2psrv`, `gat_detect_fastest_p2psrv_v2`): probe every p2p
  server, score them by response, and pick the fastest (`P2PNetDetect`). Both IPv4 and IPv6
  interfaces are probed.
- **Calling** (`CALLING_REQ`): through the chosen server, signal the target device id (`srcID` →
  `dstID`). This wakes the device and triggers an address exchange, including NAT candidates
  (`ListNat` / `ListNatRsp`).
- The certification handshake (R1/R2) then runs over the selected path before media flows
  (te-qgj.4).

## 4. Route selection → MTP session

With addresses known, the terminal builds an **MTP session** on one of three routes, which maps to
the `ConnectionMode` the app reports ([camera-control.md](camera-control.md)):

| Builder | Route | Reported mode |
|---------|-------|---------------|
| `iv_mtp_session_add_lan_or_nat` | direct — same LAN, or NAT-traversed across the internet | `LAN` or `P2P` |
| `iv_mtp_session_add_udp_relay` | UDP via a p2p relay server | `RELAY` |
| `iv_mtp_session_add_tcp_relay` | TCP via a p2p relay server (last resort, NAT-hostile networks) | `RELAY` |

The LAN route is the one TianEye wants: if a session can be built LAN-direct without the servers'
help (using the broadcast table from [direct-control.md](direct-control.md) and the certification
keys from te-qgj.4), the cloud can be blocked entirely. Whether the calling/certify step can be
driven locally is the crux question, carried in te-qgj.4 and [cloud-redirect.md](cloud-redirect.md).

## Open items (session setup)

1. The JSON body of the `GetServiceList` response (server list schema) — capture or a Frida hook.
2. The byte layout of the init-info, calling and ListNat control messages — capture (te-2dc).
3. Whether a LAN session can be established with no Asrv/p2psrv reachable (the key cloud-free
   question).

---

# Crypto and Certification (te-qgj.4)

The handshake that opens a session, and the keys behind it. This decides the central cloud-free
question: can a server that knows only what the camera's owner knows (the device password) complete
the handshake, or does it need vendor/server-issued key material? **Static analysis has mapped the
mechanism but not yet resolved that one point** — see the crux below.

## Keys a session holds

A `gute` session (`iv_gute_session_new`) sets up several RC5-32 key schedules:

- **Bootstrap key** — a *hardcoded* key, the 12 bytes `www.gwell.cc` (stored Base64 as
  `d3d3Lmd3ZWxsLmNj`). Used for the earliest frames, before a session key exists. Being hardcoded
  and vendor-wide, it is **not a secret** — TianEye can use the same constant.
- **Certify key** — a 16-byte per-device key, taken from the device-link struct (offset `+0x330`).
  This encrypts the handshake's random (below). **Where this 16 bytes comes from is the crux.**
- **Session key** — set *after* the handshake, keyed directly from the 32-byte random `R1` the app
  generates. This is the mode-2 cipher that protects the live session.
- **Frame-local keys** — the per-frame (mode 1) and id-field keys from the transport section; mode 1
  is header-derived, so it's obfuscation only.

## The certification handshake

Confirmed shape (`iv_gutes_start_active_certify_req` → `..._on_respfrm_certify_resp` →
`..._certify_ack`):

1. **App → camera, CertifyReq** (frame type `0x0C`, magic `0x7F`, extended header):
   - The app generates a **32-byte random `R1`** (regenerated at most once per 2-hour window),
     stores it, and includes it plus a 32-bit hash of it in the frame.
   - `R1` is **RC5-encrypted with the certify key** (the `+0x330` per-device key) before sending.
   - Optional MTU and prior-session id are appended.
2. **Camera → app, CertifyResp:** carries a session id. (The camera must have decrypted `R1`, which
   it can only do if it holds the same certify key.)
3. On success the app **keys the session cipher from `R1`** (`rc5_ctx_setkey(session_ctx, R1, 32)`),
   switches to mode-2 encryption, and sends its init-info message. Both sides now share `R1` and
   talk under the session cipher.

So security rests entirely on the **certify key**: anyone who holds it can produce a valid
`CertifyReq`, learn `R1`, and run the session.

## The crux: where does the certify key come from (resolved)

**The certify key is derived from the cloud-issued access token, not the device password**
(`iv_set_access_token`). At login the server returns an `accessToken` (a hex string); the SDK
hex-decodes it into a 64-byte blob stored at the terminal struct `+0x300` (the decoded token starts
with a `0x02` version byte). The **certify key is the 16 bytes at offset `0x30` of that blob**
(`+0x330`), and it is re-keyed every time the access token is set or refreshed
(`iv_update_access_id_token`).

So the handshake is gated by **possession of a valid access token**, which only the vendor cloud
issues (and which expires and must be refreshed, [protocol.md](protocol.md)). The device password
does **not** feed this path.

**Consequence for TianEye:** full IoTVideo P2P impersonation is **not possible with the device
password alone**. It would require either logging into the real vendor cloud to obtain a token (not
cloud-free), or TianEye completely standing in as the login/token **and** the server the camera
registered against — a far larger undertaking than the LAN media path. Therefore, for IoTVideo
cameras the verdict holds firm: **block the cloud and serve media over RTSP/ONVIF**
([live-view.md](live-view.md), [cloud-redirect.md](cloud-redirect.md)), rather than impersonate P2P.

The **one remaining loophole** is the LAN-password path below, which does not obviously go through
the token-keyed certify and might authenticate locally with just the device password.

## A separate LAN-password path

`iv_check_set_lan_device_pwd` is a distinct, promising mechanism: in a specific link mode it stores
a device password (and optionally a new one) into the device-link struct and sends a **check-LAN-
password** or **modify-LAN-password** message straight over the LAN link
(`giot_eif_send_check_lan_pwd` / `..._modify_lan_pwd`), waiting for the device's verdict. This looks
like password-authenticated LAN access that does **not** go through the full cloud certification —
exactly the path a cloud-free TianEye would want. Its message format and when the mode is active are
not yet traced.

## Remaining lead: the LAN-password path (te-qgj.9)

With the main certify path shown to be token-gated, the LAN-password path is the only candidate for
cloud-free IoTVideo control. What static analysis now shows (`giot_eif_send_check_lan_pwd`,
`gat_on_rcvpkt_GATFRM_ChecklanpwdRsp`):

- The **check-LAN-password** frame carries the **device password** (up to 128 bytes) to the camera
  and gets back a 1-byte **result** (pass/fail), which `iv_check_set_lan_device_pwd` polls.
- Crucially, this frame is sent with **mode-1 encryption** (the header-derived, reproducible
  obfuscation), **not** the token-derived session key. So **a client can send it with no access
  token** — only the device password and the (public) framing.
- It runs in a specific link mode (the global state at `+0xE90 == 1`, to be identified — looks like
  a LAN/direct mode).
- A `modify-LAN-password` variant sets a new password the same way.

**So the auth *verification* is cloud-free and password-only — promising.** The password travels
under mode-1 obfuscation only, i.e. effectively in the clear on the LAN (a security note for
TianEye's own use).

**What's still unconfirmed (needs hardware/capture):** whether passing the LAN-password check
actually **opens a media/control channel** — i.e. whether model commands and A/V then flow without a
token-keyed certify — or whether it's only a verification step. If it opens a session, TianEye could
drive IoTVideo cameras on the LAN with just the device password; if not, RTSP remains the path. That
final step is carried in te-qgj.9 and gated on a capture (te-2dc).

---

# Application Layer (te-qgj.5)

What rides inside the gute frames once a session is up: device-model commands, free-form user data,
the built-in command set, and the A/V stream. All of these are gute frames (above), so they inherit
its header, checksum and encryption; this section is the payload each carries.

## Device-model commands (settings, actions)

`nExecuteModelCmd` (read property / write property / take action — ordinals in
[native-libs.md](native-libs.md)) maps to `MessageMgr::{read,write,take_action}_of_device`, which
sends a **GDM data-object** message: a gute frame with magic `0x7F`, **type `0xAA`**, carrying

- the session/device id (8 bytes),
- a 32-bit **message id** (matches the response; the ack is `gat_on_ackfrm_send_data_object…`),
- a **path** string with its length (e.g. `ProWritable.videoParm.setVal.flip`,
  `Action._otaUpgrade`), and
- a **JSON** value string with its length.

Both lengths are stored as (length − 1) in 16-bit fields; the frame's total-length field covers
path + json + a fixed `0x2A`-byte prefix. The response comes back keyed by the same message id,
with `code` and a JSON body (`req_id=%u tid=%s code=%u data=%s`). This one message type carries all
settings reads/writes and all actions (PTZ reset, OTA, zoom, format SD, …).

## User data (live PTZ, etc.)

`iv_send_user_data(data, len, channel)` sends free-form bytes on an **open media channel** rather
than as a model command — this is the path the PTZ "shake head" JSON and zoom/focus actions take
during live view ([camera-control.md](camera-control.md)). It routes through the channel's push-data
path (`giot_send_push_data` / `giot_eif_send_user_data`), so it needs an active player session; when
none is open, the app falls back to the model-command path above.

## Built-in commands

`sendMsgToDevice` with domain `BUILT_IN` carries a `BuiltInCmd` code and a byte payload — the SD
playback, file download, thumbnail and resource-management sub-protocols. The full code table is in
[native-libs.md](native-libs.md); the resource (file) messages have their own pack/unpack codec
(`res_pack_*` / `res_unpack_*`, `res_codec.c`): query, delete, download-info and download-chunk.

## A/V stream

Media rides a **push channel**. Each media frame is wrapped in a 20-byte (`0x14`) push-frame header,
then the encoded payload:

| Offset | Size | Field |
|--------|------|-------|
| 0 | 1 | class = `3` (push A/V) |
| 1 | 1 | subtype = `8` |
| 2 | 1 | flags: bit 0 = **is-video**; bit 2 = **key/I-frame** (set when the frame is a keyframe) |
| 4 | 2 | payload length |
| 6 | 2 | checksum (a hash of the frame XORed with the length field) |
| 8 | 4 | channel/source id |
| 12 | 8 | session/device id |
| 20 | … | encoded media payload |

The media payload itself carries the per-frame metadata the SDK logs as `RtcFrm_AvData`:
**opt_video** (video vs audio), **opt_videoI** (keyframe), **source_id** (lens/stream), **sqnum**
(sequence number) and **pts** (presentation timestamp, microseconds). Frames larger than a packet
are fragmented (`RtcFrm_FragBeg` with a `frag_id` and the total `rawdat_len`, then data fragments);
the receiver reassembles by sequence. Codec and resolution are negotiated at connect (definition
LD/SD/HD/AUTO, [camera-control.md](camera-control.md)); video is H.264/H.265 and audio is G.711,
matching what the camera also exposes over RTSP ([live-view.md](live-view.md)).

For a Go port, the practical consequence: **once a session is open, pulling A/V means reading push
frames, checking the video/keyframe flags, reassembling fragments by `sqnum`, and handing the
payload (with its `pts`) to a decoder or RTP packetizer** — the same shape as the RTSP path TianEye
already uses, so the live-view pipeline ([live-view.md](live-view.md)) applies once frames are in
hand.

## Open items (application layer)

1. The exact byte offsets of `opt_video` / `opt_videoI` / `sqnum` / `pts` inside the media payload
   (needs `iv_rcv_rtc_data` decode or a capture).
2. The JSON schemas of model-command responses (`code` values, error envelope) — capture.
3. Whether SD playback and user-data frames reuse the push-channel header or have their own.

---

## Open items for the Go port (transport)

1. **Confirm endianness and exact type codes on a capture** (te-2dc). Static analysis gives the
   layout; a capture pins the byte order of the length/flags and the type enumeration.
2. **RC5 round count** — it's standard RC5-32 with the canonical P/Q constants (confirmed); the
   round count (the `r` in RC5-32/r) is set at context creation and still needs reading (te-qgj.4).
3. **KCP tuning** (window, interval, nodelay) — from `iv_mtp_kcp_create`.
4. The exact fields in the body-prefix word (offset 12) and the extended (`0x68`) header variant.
