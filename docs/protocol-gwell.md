# Legacy Gwell Protocol (libp2pav / libgwmediaplayer)

Wire format of the older Gwell P2P stack, for a Go reimplementation (epic te-qgj, task te-qgj.6).
Static analysis (Ghidra). This is the stack used by **legacy** cameras (device id below 2³²,
[direct-control.md](direct-control.md)); the test camera is IoTVideo, so none of this is confirmed
against hardware. The IoTVideo `gute` layer ([protocol-iotvideo.md](protocol-iotvideo.md)) is a
fork of this one, so much of the framing is shared and only summarised here.

## Shared with IoTVideo

- **Frame format**: the same `gute` frame family — `gute_frm_rc5_encrypt`/`_decrypt`,
  `gute_frm_init_chkval`/`_verity_chkval`, RC5-32 in 8-byte blocks, XOR checksum. Read the
  [IoTVideo transport section](protocol-iotvideo.md#the-gute-frame-header-24-bytes) for the layout;
  the legacy header is the ancestor of it.
- **Transport**: MTP sessions over KCP/UDP with three routes — `mtp_session_add_lan_or_nat`,
  `mtp_session_add_tcp_lan`, `mtp_session_add_udp_relay` — same LAN/P2P/relay model.
- **Service discovery**: `GET /gwellcloud/service/ListService/GetServiceList` (vs the `/iotvideo/…`
  path), returning the `p2p1..p2p10` relay list.
- **LAN search**: UDP broadcast on **8899** — fully documented in
  [direct-control.md](direct-control.md#legacy-gwell-cameras-udp-8899).
- **A/V control**: the `avctl_*` function set ([native-libs.md](native-libs.md)) — encode/send,
  recv/decode, record, seek/fastplay.
- **PTZ**: the 28-byte user-data frame
  ([camera-control.md](camera-control.md#gwell-cameras-28-byte-user-data-frame)).

## Authentication — the key difference from IoTVideo

**The legacy device password is turned into a 32-bit number that is computable from the password
alone, with no vendor secret.** This is the important result: unlike the IoTVideo certify key (whose
source is unresolved, [protocol-iotvideo.md](protocol-iotvideo.md#the-crux-where-does-the-certify-key-come-from-resolved)),
a party that knows the device password can produce a valid legacy credential. So **TianEye can
likely authenticate to a legacy Gwell camera on the LAN with just the device password.**

How the on-wire password (`passwd`, a `uint32`) is derived (`P2PGetTreatedPassword` →
`P2PEncryptGW1`, gated by `IsNeedEncrypt`):

1. `IsNeedEncrypt(pw)` — true when the password is all digits (the common case). Non-numeric
   passwords take a different path.
2. `gw_MD5(pw)` → 16-byte digest, hex-encoded to 32 characters.
3. The 32 hex chars are read as **four 32-bit values**, folded together **modulo 999999999** and
   **XORed with a fixed constant table** baked into the library.
4. The result is the 32-bit `passwd` sent in the certify request.

The camera recomputes the same value and compares (`check passwd is fail … HostEncoded=%u
GuestEncoded=%u`; `App carry passwd=%d auth_result=%d`). There is also
`fgP2PGetMD5PasswordWithSrc(pw)` = the hex string of `MD5(pw)`, used where a password *hash* (not the
folded number) is needed, and `gw_EncodePassword(n)` — a keyless 17-round scramble
(`v = v·0xA2E39FD9 + i`) applied to a numeric field. All three are deterministic and key-free, so
all are reproducible in Go.

## Certification handshake

`gutes_start_CertifyReq` → `gutes_on_respfrm_CertifyResp` → `gutes_on_rcvfrm_CertifyReq_Ack`, with a
separate `gutes_start_UpdateEncKeyReq` / `gutes_on_respfrm_UpdateEncKey` for rotating the session
key mid-session. Session-key exchange uses **RSA** (`RSAPublicEncrypt` / `RSAPrivateDecrypt` and
`R_GeneratePEMKeys`) wrapped by `EncodeKey`/`DecodeKey`/`EncryptRKey`/`DecryptRKey`. The certify
request carries the encoded `passwd` above, the session id and the protocol version
(`p2plib_version`, with `P2PV1` and `P2PV2` branches logged).

Whether the RSA step needs a camera/vendor public key that TianEye lacks, or uses a key exchanged in
the handshake, is the one piece to confirm — but since the **password check is the auth gate** and
that is password-derived, the RSA layer looks like transport key agreement rather than an identity
TianEye couldn't satisfy. Confirm on a legacy camera if one is obtained.

## For the Go port

- Legacy cameras are the **easier** target for cloud-free control: compute `passwd` from the device
  password, speak the gute framing, and the LAN/MTP session should complete without the cloud.
- Priority is low because the test camera (and most current stock) is IoTVideo. Revisit if a legacy
  camera turns up.

## Open items

1. Confirm the four-word fold constants and the XOR table values (needed for a byte-exact `passwd`);
   read them from `P2PEncryptGW1`'s data table when implementing.
2. The non-numeric-password path (when `IsNeedEncrypt` is false).
3. Whether the RSA key exchange needs a pre-shared camera key — confirm on hardware.
4. Legacy certify/frame byte offsets — share the IoTVideo decode work, confirm with a capture.
