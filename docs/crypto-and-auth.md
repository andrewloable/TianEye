# Crypto and Authentication Findings

Static analysis of `libp2pav.so` and `libiotvideomulti.so` (arm64-v8a, Yoosee 6.46.1).

---

## Certificate Pinning and TLS Trust

**No pinning.** No hardcoded certificate fingerprints, no `BEGIN CERTIFICATE` blobs, no
`sha256//` pinning strings in any native library or in any `network_security_config.xml`.
The app uses standard TLS through `libcrypto_gw.so` (OpenSSL-derived) and `libmbedtls_gw.so`.

**But a plain proxy CA will NOT work on an unrooted phone.** The app targets `targetSdkVersion 36`
(`minSdkVersion 24`), and its `network_security_config.xml` declares **no `<trust-anchors>`**. So
the Android 7+ default applies: **only system CAs are trusted; user-installed CAs are not.** A
proxy CA added through Android settings will be rejected.

To intercept the app's HTTPS for analysis you therefore need one of:
- a **rooted** device with the proxy CA in the **system** store (e.g. Magisk move-cert), or
- **Frida** to disable TLS validation at runtime, or
- **repackaging** the APK with an NSC that trusts user CAs (`<certificates src="user"/>`).

The absence of pinning matters for the *local server* side: once a camera or app does reach
TianEye over TLS, TianEye can present its own cert without tripping a pin. It does **not** mean the
stock app is trivially MITM-able. Cleartext is allowed app-wide
(`base-config cleartextTrafficPermitted="true"`), but the APIs still use HTTPS.

---

## Request Signing (HTTPS API)

Covered in [protocol.md](protocol.md). The `nSha1WithBase256(content, token)` JNI function
in `libiotvideomulti.so` performs **HMAC-SHA1 with Base64 output**, keyed by the access token —
**confirmed** by decompilation: it calls `iv_hmac_encode_sha1_with_base64`. (The "Base256" in the
JNI name is a misnomer; the output is Base64.) The Java-side `SignatureCodec.hmacSha1Base64`
implements the same algorithm in pure Java for anonymous (pre-login) requests.

---

## P2P Frame Encryption (libp2pav — Gwell legacy)

Frames are encrypted at the gutes layer before transmission.

**Frame-level cipher**: **RC5** — `gute_frm_rc5_encrypt` / `gute_frm_rc5_decrypt`.
Versioned wrappers: `P2PEncryptGW1`, `P2PEncryptGW5` / `P2PDecryptGW5`.

**Password encoding**:
- `P2PGetTreatedPassword` — derives an encoded integer password from the raw device password.
  Log entry: `"check passwd is fail reqfrm->passwd=%u HostEncoded=%u GuestEncoded=%u"` confirms
  the on-wire password is an encoded (hashed) numeric, not plaintext.
- `fgP2PGetMD5PasswordWithSrc` — MD5 of (password ∥ source_device_id), used as the encoded password.
- `EncodeKey` / `DecodeKey` / `EncryptRKey` / `DecryptRKey` — key wrapping for P2P session keys.

**RSA key exchange**: `RSAPublicEncrypt`, `RSAPrivateDecrypt`, etc. — standard RSA, likely used
to exchange session keys during the initial handshake.

---

## P2P Certification Protocol (Gwell / IoTVideo)

Both `libp2pav.so` and `libiotvideomulti.so` implement a mutual certification handshake
over the P2P channel before any media flows. The sequence is:

```
Client → Camera:  CertifyReq         (contains R1: client random)
Camera → Client:  CertifyResp        (contains R2: device random, signed with device key)
Client → Camera:  CertifyAck         (verification of R2)
```

Functions: `gutes_start_CertifyReq` → `gutes_on_respfrm_CertifyResp` → `gutes_certify_ack`
(and `iv_gutes_*` variants in libiotvideomulti).

The IoTVideo handshake is decoded in detail in
[protocol-iotvideo.md](protocol-iotvideo.md#the-certification-handshake): the app sends a 32-byte
random `R1` encrypted with a per-device "certify key", and the session cipher is then keyed from
`R1`. A hardcoded vendor key (`www.gwell.cc`, non-secret) is used for the bootstrap frames. Whether
the certify key is derived from the device password (so TianEye could complete the handshake) or
issued by the cloud is the open crux there.

Challenge-response also used in `p2pu_verifyEncPasswd`, `p2pu_verifyDevPasswd`, `p2pu_verifyR1R2`.

**Anonymous key derivation**: `giote_cal_app_anonymous_secure_key` / `giote_gen_app_anonymous_secure_key`
using `giote_hmac_md5`. Called from `P2PAlgorithmProxy.getAnonymousSecureKey(appTag)`.
The anonymous key is used to sign HTTPS requests before login.

---

## RTSP Authentication

RTSP uses **HTTP Digest Authentication** (RFC 2617).

- **Realm**: `"HIipCamera"`
- **Username**: `"admin"` (hardcoded for NVR connection)
- **HA1 derivation**: `getRtspPassword("admin:HIipCamera:" + rawPassword)` → `MD5("admin:HIipCamera:rawPassword")`
- Source: `ConnectNVRViewModel.NVR_PWD_BEGIN = "admin:HIipCamera:"`, line 24

The `nGetRtspPassword` JNI function takes the pre-colon-concatenated string and returns the
HA1 digest as bytes. The caller converts it to a UTF-8 string and sends it to the camera via
`setRTSPpwd` over the P2P control channel.

**The RTSP password is set separately from the cloud device password (te-q34).** It is chosen by
the user on the app's "Connect NVR" screen (`ConnectNVRViewModel.setRTSPPwd`), which also enables
RTSP (`ProWritable.rtspEnable` / `onvifEn`). The app computes `MD5("admin:HIipCamera:" + pwd)` from
whatever the user types there and pushes it to the camera; the cloud login password is never used
for this unless the user deliberately types the same value. So an RTSP/ONVIF client (TianEye, an
NVR) authenticates with the **RTSP password**, which the owner must have set — not the Yoosee
account password. Newer firmware also has an account+password variant (`setRTSPInfo` /
`setNewRTSPPwd`, credentials encrypted in transit by a helper) so RTSP can use a username other than
`admin`.

---

## HLS Cloud Playback Encryption

For cloud-stored recordings served as HLS:
- Segment key fetched from `/vas/cloudstorage/getTsEncryptKey`
- Local key served from `/key` or `/key?localReq=`
- Segments are AES-encrypted; `aes dec token fail` indicates AES-128 CBC is standard HLS key wrapping

---

## Open Questions

1. Exact encoding algorithm for `fgP2PGetMD5PasswordWithSrc` — is it MD5(raw_pwd ∥ src_id) or
   HMAC-MD5? Ghidra disassembly needed for the exact call to confirm the mixing.
2. RC5 key schedule — what key is used for frame encryption? Is it per-session negotiated or
   derived from the device password?
3. RSA public key — is it hardcoded in the binary or fetched from the server during handshake?
   A Ghidra search for RSA key blobs in the data segment would confirm.
4. `libnms.so` crypto role — stripped, purpose unknown.
