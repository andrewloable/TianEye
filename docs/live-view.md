# Live View: Camera RTSP to Browser

Decision and measurements from the te-m6z spike, tested against a Yoosee camera on firmware
32.01.37. The spike code lives in `spike/live-webrtc/` (gitignored); this doc is the record.

## Decision

**WebRTC, done in-process with Go:** [gortsplib](https://github.com/bluenviron/gortsplib) (v5) pulls
RTSP, and [pion](https://github.com/pion/webrtc) (v4) serves WebRTC. Video is **not transcoded**,
only re-packetized. Signalling is a single WHEP-style `POST` of the SDP offer, which returns the answer.

Why not the alternatives:
- **go2rtc** would work, but it's a second binary or process. The single-binary goal favours a library.
- **MSE** (fMP4 over WebSocket) needs remuxing, and the camera's audio (below) would still need
  transcoding. WebRTC also gives lower latency.

## What the camera actually sends

The ONVIF media service advertises 30 fps and 8 kHz audio. The real streams differ:

| Path | Video | Audio |
|------|-------|-------|
| `rtsp://<cam>:554/onvif1` | H.264 **Main**, 1920×1080, **15 fps**, ~1.2 Mbps | G.711 **A-law, 16 kHz**, mono |
| `rtsp://<cam>:554/onvif2` | H.264 Main, 640×360, 15 fps | (same) |

- **RTP over UDP only.** The camera rejects `RTP/AVP/TCP` ("Nonmatching transport"), so the server
  must receive UDP from the camera. That's fine on a LAN, but RTSP-over-TCP clients won't work.
- Auth is HTTP Digest, realm `HIipCamera`, user `admin`.

## The packetization quirk (why plain forwarding fails)

Forwarding the camera's RTP packets to the browser unchanged connects and delivers packets, but
Chrome decodes **0 frames**. The camera puts **SPS + PPS + IDR into a single NAL unit** (sent as
FU-A with type 7) with Annex-B start codes embedded inside it. In the stream, the only NAL start
types are 1 (P-slice) and 7 (SPS); 8 (PPS) and 5 (IDR) never appear as separate units. FFmpeg
tolerates this, but browser WebRTC depacketizers don't.

**Fix:** decode RTP into access units (gortsplib `rtph264` decoder), write each one as an Annex-B
byte stream, and hand it to pion's sample track. pion's H.264 payloader splits on the start codes
(including the embedded ones) and sends proper SPS/PPS/IDR packets. The SDP's SPS/PPS are injected
once before the first frame.

## Measured (headless Chrome, same LAN, one viewer)

| Metric | Result |
|--------|--------|
| Decoded | 1920×1080 at 15 fps, steady |
| Packet loss | 0 |
| Time to first frame | 1.0–2.0 s (waiting for the next keyframe) |
| Server cost | 1–2% CPU, ~24 MB RSS |

Glass-to-glass latency was not measured; it needs a visible clock in the camera's view.

## Follow-ups for the real implementation

1. **Audio:** WebRTC PCMA is fixed at 8 kHz, so the 16 kHz A-law must be resampled to 8 kHz PCMA
   or transcoded to Opus.
2. **Instant start:** cache the last keyframe-onwards access units per camera, and send them to new
   viewers so they don't wait up to a GOP.
3. **One RTSP pull per camera,** fanned out to all viewers. Cheap cameras handle few RTSP sessions.
4. **Advertise Main profile** (`profile-level-id=4d001f`) or match the camera's SPS. The spike
   advertised Constrained Baseline and Chrome decoded Main anyway, but strict browsers may not.
5. Other Yoosee firmware (e.g. 38.04.31) may packetize correctly; keep the re-packetizer anyway,
   since it's harmless on a well-formed stream.
