# Local Storage Feasibility

For each thing the vendor cloud stores, can TianEye keep it locally instead? Synthesis of the
other docs (te-99x.15), doc-only scope. "Rebuild from camera" means TianEye produces the data itself
from the camera's LAN interfaces rather than copying what the cloud holds.

## Verdict table

| Cloud-stored data | TianEye option | How | Confidence |
|-------------------|----------------|-----|------------|
| Live video | **Rebuild from camera** | RTSP/ONVIF pull, already working ([live-view.md](live-view.md)) | **Proven** on the test camera |
| Continuous recordings | **Rebuild from camera** | record the RTSP stream into segments with retention (see [Recording spike](#recording-spike-te-9m8) below) | **Proven** on the test camera |
| Alarm snapshots | **Rebuild from camera** | grab a frame from the RTSP stream when an alarm fires; or receive the camera's upload if impersonation works | Rebuild: feasible. Receive-upload: unknown (te-99x.10) |
| Alarm events (type, time, labels) | **Receive, or rebuild** | receive the camera's alarm report if TianEye stands in as its server; or do local motion/AI detection on the stream. ONVIF events don't work on the test camera ([onvif.md](onvif.md)) | Receive: unknown. Local detection: feasible, separate work |
| Cloud video recordings (HLS) | **Not needed** | TianEye's own continuous recording replaces it; the cloud copy is a paid convenience | n/a |
| Device settings model (`ProWritable`/`ProReadOnly`) | **Keep locally** | the model is read/written on the camera over the control channel, not stored only in the cloud; TianEye can cache the last-known model | Feasible if TianEye reaches the control channel (P2P or ONVIF) |
| Account ↔ device bindings | **Keep locally** | this is TianEye's own data (its camera list), not something to copy from the vendor | **Owned** |
| Face database (per-camera) | **Not possible as-is / rebuild** | the vendor runs face recognition server-side on uploaded images; TianEye can't get the vendor's model, but could run its own detector locally | Vendor copy: not possible. Local: separate, large work |
| Bird / AI detection | same as faces | server-side vendor feature | Vendor copy: not possible |
| PTZ presets | **Positions on camera; names/thumbnails local** | the camera stores preset positions; TianEye stores the names and thumbnails the cloud used to hold ([camera-control.md](camera-control.md)) | Feasible |
| Push tokens / offline-notify config | **Replace** | TianEye raises its own notifications; no vendor push state to keep ([alarm-delivery.md](alarm-delivery.md)) | Feasible; phone delivery is its own problem |

## What this means

TianEye can hold everything that matters by **rebuilding from the camera**, not by copying the
cloud:

- **Video and recordings** are already done over RTSP — this is the core and it works without the
  cloud at all.
- **Snapshots** come free from the same stream once an alarm is known.
- **Settings, presets, bindings** are either on the camera or are TianEye's own data.

The things TianEye **cannot** reproduce are the vendor's **server-side AI** (face/bird recognition)
— those need the vendor's trained models — and anything that requires **winning the camera's P2P
session** (receiving the camera's own alarm reports and settings over P2P), which is the open
question from [cloud-redirect.md](cloud-redirect.md). Neither blocks the core goal: local live view
and recording need only RTSP/ONVIF, which the test camera serves directly.

## Recording spike (te-9m8)

Built and run against the test camera, then reverted (planning phase). What it showed:

- **ffmpeg as the segmenter**, one process per camera. Video is stream-copied (no re-encode, low
  CPU). The camera's G.711 audio is transcoded to AAC (32 kbps) because browsers don't play G.711
  in MP4.
- **Fragmented MP4 segments.** A crash or power cut loses only the last fragment; a segment cut off
  mid-write still played in Chrome.
- **Restart with backoff** when the camera drops the stream.
- **Retention by age and by per-camera size**, never deleting the newest segment.
- Segments were listed through the API and served as plain files for playback in the browser.
- **Caveat:** the RTSP URL, password included, appears in ffmpeg's command line, so any local user
  can read it from the process list. A real implementation should pass credentials another way.
- Not done: motion-triggered recording.

## Dependencies on unresolved work

- Receiving the camera's **alarm reports** and **uploaded snapshots** locally needs cloud
  impersonation (te-99x.10 verdict: unknown until firmware/capture).
- Reading/writing the **settings model** without the cloud needs a working local control channel
  (ONVIF covers some of it; full `ProWritable` coverage needs the P2P control channel).
- **Local AI** (motion/person/face detection on the stream) is feasible but is its own project, not
  part of this epic.

## Open questions

1. ~~Does the camera expose its settings over ONVIF?~~ No. On the test camera ONVIF only serves
   stream information; imaging, time and PTZ queries are dropped ([onvif.md](onvif.md)).
2. Can TianEye receive the camera's alarm report directly (depends on te-99x.10)?
3. Is there a camera-side event/record the camera writes to its SD card that TianEye could read over
   the LAN, independent of the cloud?
