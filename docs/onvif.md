# ONVIF on the Test Camera

What the test camera's ONVIF server (port 5000, firmware 32.01.37) actually does, probed operation
by operation (te-2gw). Other firmware may differ. Addresses are placeholders.

## Summary

ONVIF here is a **thin shim for NVRs**: it hands out the RTSP URLs and little else. It advertises
Events and PTZ, but **neither works**. There is no Imaging service.

- **No local alarms over ONVIF.** Every Events call is dropped, so TianEye can't subscribe to the
  camera's motion events this way.
- **No ONVIF PTZ.** A move command is acknowledged, but nothing moves and Stop is dropped.
- **Streams work.** Profiles and stream URIs answer, which is all TianEye needs for live view and
  recording ([live-view.md](live-view.md)).

## Server quirks

- **Unknown operations get a TCP hang-up, not a SOAP fault.** The connection closes with no reply.
  A client must treat a dropped connection as "not supported".
- **The URL path is ignored.** The server routes by the request body, so any operation works on
  `/onvif/device_service`.
- **Capability addresses are wrong.** `GetCapabilities` lists each service at the address of the
  next one: Events at `media_service`, Media at `ptz_service`, PTZ at `deviceio_service`. DeviceIO
  points at the camera's **previous** LAN address, a value left over from an earlier network.
  Clients that trust these addresses break; use `device_service` for everything.
- **`GetServices` lists only Events** (version 0.3).
- **Settings don't match the stream.** The video source says 1920×1080 at 30 fps. Encoder config 0
  says 1280×720 at 15 fps. The real main stream is 1920×1080 at 15 fps, with 16 kHz audio where
  the profile says 8 kHz. ONVIF advertises RTP over TCP, but the camera rejects it.
- No authentication is needed for the operations that work; requests with WS-Security
  UsernameToken digest are accepted too.

## Operation by operation

**Answers:**

| Service | Operations |
|---------|------------|
| Device | `GetCapabilities`, `GetServices`, `GetDeviceInformation` (manufacturer "Technology", model "IPC", firmware, hardware "Ver 2.1"; serial is all zeros), `GetNetworkInterfaces` (MAC, DHCP) |
| Media | `GetProfiles` (MainStream, SubStream; each has a PTZ configuration), `GetStreamUri`, `GetVideoSources`, `GetVideoEncoderConfigurations`, `GetVideoEncoderConfigurationOptions` |
| DeviceIO | `GetVideoSources` |
| PTZ | `ContinuousMove` (returns a normal response; see below) |

**Dropped (connection closed):**

| Service | Operations |
|---------|------------|
| Device | `GetSystemDateAndTime`, `GetScopes`, `GetHostname`, `GetUsers`, `GetNTP`, `GetDNS`, `GetNetworkProtocols`, `GetDiscoveryMode`, `GetServiceCapabilities`, `GetWsdlUrl` |
| Media | `GetSnapshotUri`, `GetAudioSources`, `GetVideoAnalyticsConfigurations`, `GetMetadataConfigurations`, `GetServiceCapabilities` |
| Events | `GetEventProperties`, `CreatePullPointSubscription`, `Subscribe` (push), `GetServiceCapabilities`. Tried at every service path, with and without WS-Addressing action headers |
| PTZ | `GetNodes`, `GetNode`, `GetConfigurations`, `GetStatus`, `GetPresets`, `GetServiceCapabilities`, `Stop` |
| Imaging | `GetImagingSettings`, `GetOptions` |
| DeviceIO | `GetRelayOutputs` |

## PTZ test

`ContinuousMove` (pan at half speed for 1.5 s, then the reverse) returned an empty success response
both times. Frames grabbed before and after were compared: the difference after the move
(PSNR 33 dB) was the same as between two still frames (34 dB), so the view did not change. Either
this unit has no pan/tilt motor, or the command is accepted and ignored. Either way, **ONVIF PTZ
does not work here**. On a camera with a motor, PTZ still means the app's P2P path
([camera-control.md](camera-control.md#ptz-control)).

## What this means for TianEye

- **Alarms** can't come from ONVIF on this firmware. The options left are motion detection on the
  RTSP stream in TianEye, or receiving the camera's own alarm report, which needs the P2P/cloud
  impersonation work ([alarm-delivery.md](alarm-delivery.md)).
- **Snapshots** have to be cut from the RTSP stream; there's no snapshot URI.
- **Time:** `GetSystemDateAndTime` and `GetNTP` are dropped, so TianEye can't read or set the clock
  over ONVIF.
- **Discovery** can use `GetDeviceInformation` (no auth) to identify the model and firmware
  ([discovery.md](discovery.md)).
- An ONVIF client library for TianEye has to handle hang-ups and ignore the advertised addresses.

## Open questions

1. Does a camera that has a motor move on ONVIF `ContinuousMove`? Needs a PTZ unit.
2. Do other firmware versions implement Events?
