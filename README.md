<p align="center">
  <img src="assets/icon.png" alt="TianEye" width="160">
</p>

<h1 align="center">TianEye 天眼</h1>

<p align="center">
  Self-hosted control for Yoosee IP cameras, with no Yoosee cloud and no Yoosee app.
</p>

> **Status:** planning / reverse-engineering. The Yoosee app and camera protocols have been
> analysed and written up under [`docs/`](docs/); implementation hasn't started. This README
> describes what the project intends to do and what the analysis found.

## Why

Yoosee cameras are cheap and common. Out of the box they depend on the Yoosee app and the
Gwell P2P cloud for setup, viewing, playback and alerts. That means:

- your video goes through servers you don't control,
- the camera stops being useful if the cloud or the app goes away,
- there is no clean way to put the camera in your own NVR or home automation setup.

TianEye replaces that dependency. The camera talks only to your own network, and TianEye
handles setup, live view, recording and alerts locally.

## Goals

- **Local discovery.** Find Yoosee cameras on the LAN.
- **Provisioning without the cloud.** Get a factory-reset camera onto Wi-Fi and set its
  password without the Yoosee app.
- **Live view.** Pull the camera's RTSP/ONVIF streams and serve them to a browser.
- **Recording.** Continuous and/or motion-triggered recording to local disk, with retention limits.
- **PTZ and settings.** Pan/tilt, IR/night mode, and basic image settings where the model supports them.
- **Alerts.** Motion events without phoning home.
- **Keep all traffic off the vendor cloud.** The reverse engineering settled *how*: **block** the
  camera's internet egress (firewall + local DNS/NTP) and serve media locally over RTSP/ONVIF,
  rather than impersonate the Gwell/IoTVideo P2P cloud. On the newer (IoTVideo) cameras the P2P
  handshake is keyed by a cloud-issued token, so impersonation isn't feasible with just the device
  password; blocking is. See [docs/cloud-redirect.md](docs/cloud-redirect.md).

## Non-goals (for now)

- Remote access from outside the LAN. Use a VPN (WireGuard, Tailscale) instead.
- Supporting camera brands other than Yoosee/Gwell.

## What we know about the cameras

Common values on Yoosee firmware; the first entry in [Tested cameras](#tested-cameras) has been
confirmed against real hardware, the rest still vary by model:

| Item | Common value on Yoosee firmware |
|------|---------------------------------|
| RTSP main stream | `rtsp://<ip>:554/onvif1` |
| RTSP sub stream | `rtsp://<ip>:554/onvif2` |
| ONVIF port | `5000` |
| Username | `admin` |
| Password | the **RTSP password** set on the app's "Connect NVR" screen (separate from the cloud account password; see [docs/crypto-and-auth.md](docs/crypto-and-auth.md)) |
| Discovery | firmware 32.01.37 does not answer ONVIF WS-Discovery; such cameras must be found by probing RTSP/ONVIF ports and spotting the `HIipCamera` RTSP realm |

Firmware differs a lot between models, and some units don't expose RTSP/ONVIF at all. Tested
models are listed under [Tested cameras](#tested-cameras).

## Reverse engineering

The cloud protocol isn't documented, so redirection depends on reverse engineering for
interoperability, done only on hardware we own:

1. Log the cameras' DNS queries and capture their traffic on an isolated network.
2. Decompile the Yoosee Android app (e.g. with jadx) to find server hostnames and the
   registration, P2P and alarm protocols.
3. Write the findings up in our own words under `docs/`.

The APK, decompiled code and raw packet captures are never committed. They are proprietary,
or they contain device IDs and passwords.

Steps 2–3 (decompile and document) are done; step 1 (on-wire capture) is the pending hardware
phase — several findings below are marked as needing a capture to confirm.

Findings so far (Yoosee app 6.46.1, static analysis plus one test camera):

| Topic | Doc |
|-------|-----|
| Can the cloud be cut off? Verdict: block it, don't impersonate it | [cloud-redirect.md](docs/cloud-redirect.md) |
| Every domain and IP the app uses | [cloud-endpoints.md](docs/cloud-endpoints.md) |
| What data leaves the network | [data-inventory.md](docs/data-inventory.md), [privacy-uploads.md](docs/privacy-uploads.md), [camera-uploads.md](docs/camera-uploads.md) |
| What TianEye can keep locally instead | [local-storage.md](docs/local-storage.md) |
| Live view in the browser (WebRTC, measured) | [live-view.md](docs/live-view.md) |
| Finding cameras on the LAN | [discovery.md](docs/discovery.md) |
| What the camera's ONVIF actually supports (streams only; no events, no PTZ) | [onvif.md](docs/onvif.md) |
| PTZ, settings, SD playback | [camera-control.md](docs/camera-control.md) |
| Adding a camera; Wi-Fi provisioning | [add-camera.md](docs/add-camera.md) |
| Alarms and push | [alarm-delivery.md](docs/alarm-delivery.md) |
| Firmware updates | [ota-firmware.md](docs/ota-firmware.md) |
| Cloud API, signing, crypto | [protocol.md](docs/protocol.md), [crypto-and-auth.md](docs/crypto-and-auth.md) |
| The app and its native libraries | [app-overview.md](docs/app-overview.md), [native-libs.md](docs/native-libs.md) |
| Controlling a camera directly on the LAN (no cloud) | [direct-control.md](docs/direct-control.md) |
| Go-rewrite build guide for the camera SDKs | [protocol-go-guide.md](docs/protocol-go-guide.md) |
| IoTVideo P2P protocol (transport, session, crypto, app layer) | [protocol-iotvideo.md](docs/protocol-iotvideo.md) |
| Legacy Gwell P2P protocol | [protocol-gwell.md](docs/protocol-gwell.md) |

## Tested cameras

Addresses are placeholders ([TEST-NET-1](https://en.wikipedia.org/wiki/Reserved_IP_addresses));
real LAN addresses live in a gitignored `*.local.yaml`.

| Model | Firmware | Hardware | Open TCP ports | RTSP | ONVIF | Notes |
|-------|----------|----------|----------------|------|-------|-------|
| "IPC" (mfr reports "Technology"), **IoTVideo** SDK family | 32.01.37 | Ver 2.1 | 21, 554, 5000, 50000 | `rtsp://192.0.2.10:554/onvif1`, `/onvif2` — HTTP Digest, realm `HIipCamera`, user `admin` | port 5000, SOAP | FTP (21) present but broken (`inetd` can't exec ftpd); 50000 speaks an unknown binary protocol (silent on connect, not HTTP); ONVIF device info, capabilities, media profiles and stream URIs all answer **without auth**, but Events, PTZ, Imaging and time are advertised and don't work ([docs/onvif.md](docs/onvif.md)); both streams H.264 Main (1920×1080 and 640×360) at a measured **15 fps** with G.711 A-law **16 kHz** audio (ONVIF advertises 30 fps and 8 kHz); RTP over UDP only |

## Architecture

- **Server:** Go, shipped as a single binary with the web UI embedded.
- **Web UI:** Angular.
- **API:** [ConnectRPC](https://connectrpc.com) with a protobuf contract shared by the Go
  backend and the Angular frontend.
- **Video:** the camera's RTSP stream reaches the browser through WebRTC, re-packetized but not
  transcoded ([docs/live-view.md](docs/live-view.md)).

## Getting started

_TBD. There is no runnable code yet._

## Contributing

Issues and PRs are welcome. If you own a Yoosee camera, a report with the model, firmware
version and the ports it has open (`nmap -p- <camera-ip>`) is very useful.

## Disclaimer

TianEye is an independent project and is not affiliated with or endorsed by Yoosee or Gwell.
Use it only on cameras you own.

## License

[MIT](LICENSE)
