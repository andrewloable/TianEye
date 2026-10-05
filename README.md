<p align="center">
  <img src="assets/icon.png" alt="TianEye" width="160">
</p>

<h1 align="center">TianEye 天眼</h1>

<p align="center">
  Self-hosted control for Yoosee IP cameras, with no Yoosee cloud and no Yoosee app.
</p>

> **Status:** pre-alpha. This README describes what the project intends to do. Nothing here works yet.

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
- **Redirect the cloud to TianEye.** Point the hostnames the cameras call at TianEye with local
  DNS, and have TianEye answer in Gwell's place. The camera then behaves as if it is online,
  but nothing leaves the LAN.
- **Fallback: block the cloud.** For models that can't be redirected, document firewall/VLAN
  blocking that keeps local streaming working.

## Non-goals (for now)

- Remote access from outside the LAN. Use a VPN (WireGuard, Tailscale) instead.
- Supporting camera brands other than Yoosee/Gwell.

## What we know about the cameras

These are starting points and still need checking against real hardware:

| Item | Common value on Yoosee firmware |
|------|---------------------------------|
| RTSP main stream | `rtsp://<ip>:554/onvif1` |
| RTSP sub stream | `rtsp://<ip>:554/onvif2` |
| ONVIF port | `5000` |
| Username | `admin` |
| Password | the device password set in the Yoosee app |

Firmware differs a lot between models, and some units don't expose RTSP/ONVIF at all. Tested
models will be listed here.

## Reverse engineering

The cloud protocol isn't documented, so redirection depends on reverse engineering for
interoperability, done only on hardware we own:

1. Log the cameras' DNS queries and capture their traffic on an isolated network.
2. Decompile the Yoosee Android app (e.g. with jadx) to find server hostnames and the
   registration, P2P and alarm protocols.
3. Write the findings up in our own words under `docs/`.

The APK, decompiled code and raw packet captures are never committed. They are proprietary,
or they contain device IDs and passwords.

## Tested cameras

_None yet._

## Architecture

- **Server:** Go, shipped as a single binary with the web UI embedded.
- **Web UI:** Angular.
- **API:** [ConnectRPC](https://connectrpc.com) with a protobuf contract shared by the Go
  backend and the Angular frontend.
- **Video:** the camera's RTSP stream reaches the browser through WebRTC or MSE.

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
