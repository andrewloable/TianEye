# First On-Wire Capture — Results

The first real packet capture of the test camera (te-2dc), taken 2026-10-07. The camera and the
phone were put on a dedicated Raspberry Pi access point (Pi 3 B+: `wlan0` hostapd AP, `eth0` uplink
to the LAN, NAT), and the camera's traffic was captured on the Pi's `wlan0` while the Yoosee app did
a firmware update check, live view, PTZ, and SD-card playback. ~16.5k packets over ~4 minutes.

This validates and sharpens the static analysis. The capture itself (pcap) is never committed; the
camera's device id and MAC are kept out of this write-up.

## Headline findings

1. **The camera uses hardcoded IPs — no DNS at all.** Zero DNS queries in the entire capture, across
   a reboot onto a new network and heavy activity. **DNS redirection therefore cannot steer this
   camera** ([cloud-redirect.md](cloud-redirect.md)); only an **IP/firewall block** works. This
   settles the open question that was flagged there and in [cloud-endpoints.md](cloud-endpoints.md).
2. **The camera's entire cloud footprint is one UDP endpoint:** `124.243.181.72:9100` (UDP).
   WHOIS: **Huawei Cloud Singapore** (HUAWEI INTERNATIONAL PTE. LTD.). Hardcoded, reached directly by
   IP. Only ~23 small packets (32–76 bytes) camera→cloud across the session — a rendezvous +
   keepalive, spread over the whole window.
3. **No TCP and no TLS from the camera** during live view, PTZ, SD playback or the update check.
   Everything is **UDP** — consistent with the IoTVideo KCP/UDP P2P stack
   ([protocol-iotvideo.md](protocol-iotvideo.md)); the HTTPS cloud API is the *app's* path, not the
   camera's.
4. **Live view, PTZ and SD playback all went LAN-direct**, camera↔phone, because both were on the
   same subnet: 16,318 of 16,468 packets were camera↔phone, none of that via the internet. This is
   the `ConnectionMode LAN` path ([direct-control.md](direct-control.md), [camera-control.md](camera-control.md))
   proven on the wire: once the session is up, media never leaves the LAN.
5. **One multiplexed UDP socket.** The camera used source port **41878** for *both* the cloud
   rendezvous to `124.243.181.72:9100` *and* the LAN P2P to the phone. A second UDP port **49625**
   carried additional P2P (likely the second stream / SD playback), plus a little on 51010.
6. **The firmware "update check" contacted no separate update server.** It returned "already latest"
   and produced no distinct host — handled within the existing P2P channel, not via a camera→update
   -server fetch. (A real *download* might differ; still needs catching an actual upgrade, te-cyj.)
7. **LAN chatter:** the camera emits SSDP/UPnP (`239.255.255.250`) and IGMP (`224.0.0.2`).

## What the camera talked to

| Peer | Transport | Role | Packets |
|------|-----------|------|---------|
| `124.243.181.72:9100` (Huawei Cloud SG) | UDP | Hardcoded cloud rendezvous / keepalive | ~23 out |
| phone (same subnet) | UDP :41878, :49625 | LAN-direct P2P: live view, PTZ, SD playback, two-way audio | ~16,300 |
| `239.255.255.250`, `224.0.0.2` | UDP multicast | SSDP/UPnP, IGMP on the LAN | ~90 |

No other public IP, no DNS, no TCP.

## Why this matters for TianEye

- **Blocking is confirmed as the right strategy, and DNS is not enough.** The camera skips DNS and
  dials a hardcoded Huawei-Cloud-SG IP, so a firewall rule by **IP** (or blanket WAN-egress block for
  the camera) is required — exactly the per-destination plan in
  [cloud-redirect.md](cloud-redirect.md), now with a concrete IP/port.
- **The LAN media path is real and self-sufficient.** With the camera and the viewer on the same
  network, live view and playback are pure LAN UDP P2P — no cloud in the media path. TianEye serving
  media locally (RTSP today, or the P2P LAN path later) matches how the camera already behaves.
- **Open, now testable:** does the session still form if `124.243.181.72:9100` is blocked? The camera
  reached that rendezvous even for a LAN-local session, so blocking it may stop new sessions from
  brokering (te-99x.8). That is the next capture: block the IP and retry.

## Method (for reproducing)

Pi 3 B+ as a hostapd AP (`tianeye-cap`, WPA2-CCMP, 2.4GHz ch6, country set so the AP runs at full
power — an unset country silently breaks client association), `eth0` uplinked to the main LAN with
NAT/forwarding, DHCP via dnsmasq. Camera + phone provisioned onto the AP; `tcpdump -i wlan0 host
<camera>` on the Pi. Full steps and tooling in [capturing.md](capturing.md). ARP-spoofing from the
Mac was tried first and failed (the TP-Link/Wi-Fi did not let it redirect the camera) — the Pi-AP
method is what worked.

## Still open (need more captures)

1. **Cloud-blocked behaviour** — block `124.243.181.72` and see if the camera still serves a LAN
   session or loops (te-99x.8).
2. **A real firmware download** — only a genuine upgrade reveals the firmware host/URL (te-cyj).
3. **Decrypting the P2P UDP** — the payloads are RC5; feed them through the documented scheme to
   confirm the frame format against the static analysis (te-99x.17, te-qgj).
4. **Whether a remote (off-LAN) viewer forces relay/cloud media** — this capture had both devices on
   one subnet, the best case for LAN-direct.
