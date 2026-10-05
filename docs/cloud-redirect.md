# Can a Local Server Stand In for the Cloud?

Whether TianEye can answer each cloud connection a Yoosee camera makes, so the camera keeps working
with no vendor cloud. Static analysis of the app (6.46.1), doc-only scope (te-99x.10).

**Big caveat.** This is assessed from the *app*. The camera's own trust decisions — which servers
it will accept, and how it checks them — live in the camera firmware, which has not been analysed
(deferred: te-99x.12) and not captured (deferred: te-2dc). Every "can impersonate" verdict below is
therefore a *possibility* from the app side, to be confirmed camera-side before relying on it.

## What makes impersonation possible or not

Two things decide whether a substitute server works for a given connection:

1. **Reaching it.** DNS redirection catches a connection only if the camera resolves a hostname it
   can be pointed at. Hardcoded IPs and non-DHCP resolvers escape it (see the open questions).
2. **Being accepted.** The camera has to trust the substitute. That depends on the protocol:
   - plain HTTP: nothing to forge;
   - TLS: depends on the camera's trust store and whether it pins (unknown, firmware);
   - Gwell/IoTVideo P2P: a mutual challenge-response handshake with keys (below).

## Request signing is not server authentication

The app signs its API requests with **HMAC-SHA1 keyed by the access token** (confirmed, see
[crypto-and-auth.md](crypto-and-auth.md)). That proves the *client* to the server. It does **not**
let the camera or app verify the *server*. A substitute server that knows the shared material can
produce valid responses; it doesn't need the vendor's private keys for the HTTP API. This is why the
HTTP services are the most impersonable.

## No certificate pinning (app side)

No pinning was found anywhere in the app or its native libraries — no fingerprints, no
`network_security_config` trust anchors, no code-level pinner. That removes one barrier. But the app
targets Android SDK 36, so it trusts only system CAs; and, more importantly, **the camera's** TLS
trust store is a separate, unexamined thing. Absence of pinning in the app does not prove the camera
accepts an arbitrary certificate.

## Per-connection assessment

| Connection | Protocol | How the camera likely checks the server | Impersonate? |
|------------|----------|------------------------------------------|--------------|
| HTTP API (device registration, config, OTA metadata) | HTTPS or the SDK's HTTP-over-P2P | TLS cert and/or signed responses; no pinning seen app-side | **Likely**, if the camera's trust store allows it (firmware) |
| Service discovery (`.../ListService/GetServiceList`) | HTTP(S) | same as HTTP API | **Likely** — and key: it tells the camera which relay to use, so answering it redirects the P2P layer too |
| P2P index (`list.iotvideo…`) | HTTP(S) to resolve device → relay | same | **Likely** |
| P2P media / control session | Gwell/IoTVideo over KCP/UDP | **R1/R2 mutual certification** (`CertifyReq`/`CertifyResp`/`CertifyAck`), RC5 frame cipher, RSA key exchange | **Unknown** — hinges on where the handshake keys come from (below) |
| Push delivery | FCM / vendor push cloud | n/a (runs on the phone, not the camera) | Not applicable — replace the mechanism, don't impersonate it ([alarm-delivery.md](alarm-delivery.md)) |
| Firmware download | HTTP(S) from a URL the cloud returns | TLS; maybe a signature on the image | **Partly** — can serve a URL, but flashing a custom image depends on an update signature check (firmware) |
| Telemetry / ads / analytics | HTTPS | n/a | Block, don't impersonate |
| NTP / connectivity ping (`114.114.114.114`) | UDP / ICMP | n/a | Allow or serve locally; not a vendor service |

## The P2P handshake is the crux

Live video, PTZ, two-way audio and SD playback all ride the P2P session, and that session opens
with a mutual certification handshake (`gutes_start_CertifyReq` → `CertifyResp` → `CertifyAck`, with
RC5 frames and RSA key exchange; see [crypto-and-auth.md](crypto-and-auth.md) and
[native-libs.md](native-libs.md)).

Whether TianEye can terminate that session as the "server"/peer depends on **what keys the camera
checks against**:

- If the handshake is keyed by the **device password / device ID** (which TianEye would know, since
  the user owns the camera), a local peer can likely complete it. The password-verification helpers
  (`p2pu_verifyEncPasswd`, `fgP2PGetMD5PasswordWithSrc`) and the anonymous-key derivation point this
  way, but don't prove it.
- If it also checks a **vendor key baked into the camera** that TianEye can't produce, the camera
  will refuse an unknown peer, and full P2P impersonation is out — leaving RTSP/ONVIF (which TianEye
  already uses, [live-view.md](live-view.md)) as the local path for media, and the P2P cloud simply
  blocked.

The IoTVideo handshake has since been decoded and the crux **resolved**
([protocol-iotvideo.md](protocol-iotvideo.md#the-crux-where-does-the-certify-key-come-from-resolved)):
the certify key is derived from the **cloud-issued access token**, not the device password. So full
IoTVideo P2P impersonation is **not** achievable with the device password alone — it needs a real
cloud token or a complete cloud stand-in. This settles the question for IoTVideo in favour of
**blocking**, not impersonating. (Legacy Gwell cameras are different: their auth password is
computable from the device password — [protocol-gwell.md](protocol-gwell.md) — so those may be
controllable P2P-direct.) One IoTVideo loophole is **partly promising**: the **LAN-password** path
(`iv_check_set_lan_device_pwd`) sends the device password to the camera under reproducible (mode-1)
encryption with no access token and gets a pass/fail, so the *verification* is cloud-free and
password-only. Whether passing it then opens a media/control session (vs. just verifying) is
unconfirmed and needs a capture (te-qgj.9).

This is the single most important open question for a cloud-free design, and it can only be answered
by analysing the camera's handshake (firmware, te-99x.12) or capturing a real one (te-2dc).

## Bottom line

- **HTTP-tier services** (registration, config, discovery, OTA metadata, index) look impersonable
  from the app side, pending the camera's TLS trust behaviour. Answering **service discovery** is
  especially valuable because it steers the camera's P2P layer.
- **The P2P media session** is the unknown. TianEye does not actually need to win it: it can serve
  live video and recording over **RTSP/ONVIF directly** (already proven), and only needs the cloud
  P2P either impersonated or cleanly blocked.
- **Push** is replaced, not impersonated.
- Nothing here is safe to build on until confirmed camera-side.

## Open questions

1. The camera's TLS trust store and whether it pins (firmware).
2. Where the P2P certification keys come from — device password vs. a baked-in vendor key.
3. Whether the camera falls back to hardcoded IPs or its own DNS/DoH when hostnames don't resolve
   (decides if DNS redirection alone is enough, or a firewall is required).
4. Whether firmware images are signature-checked (decides if custom firmware is possible).

---

# Verdict: stopping all China-bound traffic (te-99x.16)

The practical plan for a cloud-free setup, drawn from all the static findings. The **app** is easy
(just don't run it); the **camera** is the real target, and its exact behaviour with no cloud is
still unconfirmed (deferred: te-2dc, te-99x.12). This is the design to validate, not a proven result.

## The short answer

**Yes, in principle — by blocking, not by impersonating.** The simplest guarantee that nothing
reaches Chinese servers is to deny the camera all WAN egress and keep it on the LAN with TianEye.
Live view and recording need only RTSP/ONVIF, which stay on the LAN. Whether the camera stays happy
(no reboot loops, time sync, alarms) with the cloud blocked is the one thing that must be tested on
real hardware.

## Recommended configuration

Two layers, because DNS alone is not enough:

1. **Block all WAN egress for the camera** (firewall rule by the camera's MAC/IP). This is the hard
   guarantee and it catches hardcoded IPs and any DoH the camera might use.
2. **Run local DNS** that answers the vendor hostnames ([cloud-endpoints.md](cloud-endpoints.md))
   with TianEye's address, for two reasons: so the camera's lookups fail fast instead of hanging,
   and so TianEye can *optionally* answer service discovery to keep the camera from retrying.
3. **Allow only** what the camera genuinely needs locally: LAN RTSP/ONVIF (TianEye ↔ camera), and a
   **local NTP** server so time still syncs without `*.gwell` / public NTP.

## Per-destination rule

| Destination | Rule |
|-------------|------|
| All `cloudlinks.cn` / `cloud-links.net` (API, VAS, P2P relays, upgrade, telemetry, ads) | **Block** (and DNS-redirect to TianEye so lookups resolve locally) |
| Service discovery hostnames | **Redirect to TianEye** if emulating; otherwise block |
| Public DNS resolvers | **Block**; serve DNS locally |
| Public NTP | **Block**; serve NTP locally. Which time source the camera uses is not known yet (te-2dc) |
| Any hardcoded IPs in the firmware | **Block by IP**; DNS can't touch these. The hardcoded IPs in [cloud-endpoints.md](cloud-endpoints.md) are the *app's*; the camera's own list needs firmware or a capture |
| Firmware/OTA | **Block** — prevents vendor firmware changing hosts or behaviour under you |

## Why blocking beats impersonating here

- It needs no knowledge of the camera's trust decisions (the big unknown from the section above).
- It's verifiable: a firewall log showing zero allowed WAN packets from the camera *is* the proof.
- TianEye doesn't lose functionality by blocking the cloud, because it rebuilds video, recording and
  snapshots from the camera's LAN interfaces ([local-storage.md](local-storage.md)).

Impersonation (answering service discovery, standing in for the P2P server) is an *optional*
enhancement to keep the camera quiet or to receive its alarm reports — worth doing only after the
camera-side handshake is understood.

## Residual risks

1. **Unconfirmed camera behaviour with the cloud blocked** — the one must-test item (te-2dc / a LAN
   trial). A camera that hard-loops without its cloud would need impersonation after all.
2. **Hardcoded IPs or DoH in the firmware** that escape DNS — the egress block covers them, but
   confirm with firmware analysis (te-99x.12).
3. **The official app stops working** once the cloud is blocked — expected; TianEye replaces it.
4. **Firmware updates** could change hosts — blocking OTA prevents silent change, at the cost of no
   security updates.

## Bottom line

A LAN-only camera behind an egress block, with local DNS and NTP and TianEye for video/recording,
sends nothing to Chinese servers and keeps the core functions. The remaining work before building
is to confirm on real hardware that the camera tolerates the block (deferred capture/firmware tasks).
