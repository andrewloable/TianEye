# Reimplementing the Camera SDKs in Go — Build Guide

A build order for talking to Yoosee cameras from Go without the vendor SDKs (epic te-qgj, task
te-qgj.7). It ties together the layer specs; read those for the wire details:

- [protocol-iotvideo.md](protocol-iotvideo.md) — IoTVideo (newer cameras, incl. the test camera)
- [protocol-gwell.md](protocol-gwell.md) — legacy Gwell
- [direct-control.md](direct-control.md) — LAN paths and discovery
- [cloud-redirect.md](cloud-redirect.md) — the block-vs-impersonate verdict

Everything here is from static analysis; **no part has been run against a camera over P2P yet.**
Treat it as a design to validate, not a proven client.

## The one decision that gates everything

Whether a Go client can open a P2P session **without the vendor cloud** depends on the
authentication gate of each family:

| Family | Auth gate | Can a Go client do it with only the device password? |
|--------|-----------|------------------------------------------------------|
| **Legacy Gwell** | an on-wire `passwd` = `fold(MD5(password))`, keyless | **Likely yes** ([protocol-gwell.md](protocol-gwell.md)) |
| **IoTVideo** | a certify key derived from the **cloud-issued access token** | **No** — needs a cloud token ([protocol-iotvideo.md](protocol-iotvideo.md#the-crux-where-does-the-certify-key-come-from-resolved)) |

The IoTVideo question is resolved: the certify key comes from the access token the vendor cloud
issues at login, so the P2P handshake can't be completed with the device password alone. **For
IoTVideo cameras, use RTSP for media and block the cloud** ([live-view.md](live-view.md)); build the
P2P port only for legacy cameras, or if the LAN-password loophole
(`iv_check_set_lan_device_pwd`) opens a session: its password *check* is already shown to be
cloud-free and password-only; whether it unlocks media/control needs a capture (te-qgj.9).

## What you don't need P2P for

Before building any of this, note how much is reachable **without** the P2P stack at all:

- **Live video + recording**: RTSP/ONVIF on the LAN ([live-view.md](live-view.md)) — already proven.
- **Discovery**: port-probe + fingerprint ([discovery.md](discovery.md)).
- **Settings in AP mode**: whole-object writes over the LAN ([direct-control.md](direct-control.md)).

A useful TianEye exists on these alone. The P2P port below is what adds PTZ, settings and SD
playback on a camera that is **not** in AP mode and whose RTSP you'd rather not depend on.

## Build order

Each stage is independently testable. Stop wherever the product needs allow.

1. **gute frame codec** — encode/decode the 24-byte header, XOR checksum, and RC5-32 (standard, with
   the canonical P/Q constants; pin the round count, te-qgj.4). This is the foundation for both
   families and the most mechanical part. *Test:* round-trip a frame, and decrypt a captured one
   (needs a capture, te-2dc).
2. **KCP + UDP socket layer** — reliable streams over UDP, plus the direct-with-ack path for small
   frames. Use an existing Go KCP library; match the tuning from `iv_mtp_kcp_create` (te-qgj.2).
3. **Service discovery + session setup** — answer or call `GetServiceList`, register with the access
   server, run the detect/calling exchange, build the MTP session (te-qgj.3). *For LAN-only:* skip
   the servers and try the LAN route directly once auth is solved.
4. **Authentication** — legacy: compute `passwd` and send the certify request. IoTVideo: needs a
   cloud-issued access token (resolved), so not cloud-free unless the LAN-password path works.
5. **Application layer** — model commands (type `0xAA` data-object: path + JSON) for settings/actions,
   user-data frames for live PTZ, built-in commands for SD playback, and the push-frame A/V reader
   (te-qgj.5). *Reuse the live-view pipeline* once frames are in hand.

## First milestone worth aiming at

**LAN PTZ + settings on a legacy camera, cloud blocked**: stages 1–5 for the Gwell family only,
which has the simplest auth. If a legacy camera is available this is the cleanest end-to-end proof
that the cloud can be cut. For IoTVideo the same milestone is out unless the LAN-password loophole pans out, since the handshake needs a cloud token.

## Test vectors derivable now

- **Legacy `passwd`**: once the fold constants and XOR table are read out of `P2PEncryptGW1`
  (te-qgj.6 open item), `fold(MD5("12345"))` etc. can be checked offline — no camera needed.
- **RC5**: against a standard RC5-32 implementation once the round count is known.
- **RTSP HA1**: `MD5("admin:HIipCamera:" + pwd)` ([crypto-and-auth.md](crypto-and-auth.md)) — checkable now.

## Risks

1. **IoTVideo auth** — the certify key is cloud-token-derived (resolved), so LAN-only IoTVideo P2P
   is out; use RTSP + the egress block ([cloud-redirect.md](cloud-redirect.md)). Only the
   LAN-password path might change this.
2. **No capture yet** — every byte layout here is from static analysis; a single capture (te-2dc)
   would confirm or correct the frame/field offsets cheaply.
3. **RSA in the legacy handshake** may need a camera key (te-qgj.6 open item).
4. **Effort vs payoff** — if RTSP covers the product's needs, the whole P2P port may not be worth
   building. Decide per feature (PTZ and SD playback are the main P2P-only wins).
