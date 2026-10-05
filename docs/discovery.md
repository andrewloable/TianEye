# Finding Cameras on the LAN

Results of the discovery spike (te-bcf), run against the test camera (firmware 32.01.37) and its
network, then reverted (planning phase). Addresses are placeholders.

## WS-Discovery does not work

The test camera does **not** answer ONVIF WS-Discovery in any form tried: multicast probe, unicast
probe, and listening on UDP 3702 for unsolicited announcements. Replies from other devices did reach
the host (checked with SSDP), so the network wasn't the cause. Other Yoosee firmware may behave
differently.

## What works: port probe + fingerprint

1. Probe TCP **554** (RTSP) and **5000** (ONVIF) across a subnet the user gives, e.g. `192.0.2.0/24`.
2. **Fingerprint by the RTSP challenge.** An unauthenticated request gets a `401` with HTTP Digest
   realm **`HIipCamera`**. That realm identifies a Yoosee camera.
3. **Read model and firmware without a password.** ONVIF `GetDeviceInformation` on port 5000 answers
   with no authentication on this firmware.

A /24 scan took about 12 s. It found 3 Yoosee cameras and correctly skipped 2 non-Yoosee devices that
also serve RTSP/ONVIF.

## Not covered yet

1. **The Yoosee app's own LAN search.** The app finds cameras on the LAN natively
   (`WiredNetConfig.nativeGetDeviceList`, `nLanDevConnectable` in
   [native-libs.md](native-libs.md)). Its port and packet format were not traced. If cameras answer
   it, it would be a cleaner discovery method than a port scan.
2. **TCP 50000** on the test camera speaks an unknown binary protocol (silent on connect, not HTTP).
   It may be that LAN protocol.
3. Whether cameras broadcast anything on their own (needs a capture with sniffing privileges).
