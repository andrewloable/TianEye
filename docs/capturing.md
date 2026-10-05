# Capturing Phone ↔ Camera Traffic (macOS)

A playbook for capturing and decrypting the traffic between the Yoosee Android app, the camera, and
the cloud, on hardware we own. This is the practical side of the deferred capture work (`te-2dc`)
that unblocks most of the remaining unknowns: the camera's own cloud endpoints, the P2P handshake on
the wire, whether LAN mode is actually used, whether blocking the cloud breaks the camera, the
firmware download host, and whether the LAN-password check opens a session.

**Scope:** only on cameras and phones you own, on an isolated network. Nothing captured gets
committed — pcaps hold device IDs, passwords and LAN IPs. Redact to placeholder IPs (`192.0.2.x`)
in anything that lands in `docs/`. See the project CLAUDE.md.

## The key constraint

The phone↔camera P2P path is mostly **UDP + KCP (reliable-UDP)** and the frame bodies are
**RC5-encrypted** ([protocol-iotvideo.md](protocol-iotvideo.md), [protocol-gwell.md](protocol-gwell.md)).
So:

- A **TCP proxy / listener won't capture it** — you need a **packet capture** (tcpdump/Wireshark).
- You decrypt the frame bodies **offline**, using the documented RC5 scheme (round count still to be
  pinned, te-qgj.4). The envelope (endpoints, ports, sizes, timing) is readable immediately without
  any decryption — and that alone answers most of the open questions.
- The only TCP parts are the P2P **TCP relay fallback** and the camera's **TCP 50000**.

## Vantage point decides what you see

| Capture point | phone↔camera P2P | **camera→cloud** | Effort |
|---|---|---|---|
| On the phone (PCAPdroid) | yes | **no** — the camera's own traffic never reaches the phone | lowest |
| On the LAN (Mac or router inline) | yes | **yes** | medium |

The real unknowns (which servers the *camera* calls, firmware host, cloud-blocked behaviour) live in
the **camera→cloud** leg, which the phone cannot see. For those, capture on the LAN.

## macOS tools

| Tool | Install | Role |
|------|---------|------|
| **tcpdump** | built in | the actual capture; write a pcap |
| **Wireshark** + **tshark** | `brew install --cask wireshark` | inspect/filter the pcap (GUI) and script it (tshark CLI) |
| **bettercap** | `brew install bettercap` | ARP-spoof the camera so its traffic routes through the Mac (LAN capture without re-provisioning) |
| **nmap** | `brew install nmap` | port/host discovery ([discovery.md](discovery.md)) |
| **mitmproxy** | `brew install mitmproxy` | decrypt the **HTTPS** API leg *if* a MITM CA is in place (see TLS note) |
| **Frida** | `pip install frida-tools` | hook the app on a rooted phone: defeat TLS validation, or log the plaintext `downUrl` / handshake directly ([ota-firmware.md](ota-firmware.md)) |
| **socat** | `brew install socat` | optional: relay/poke the camera's TCP 50000 or the TCP relay path |

macOS **Internet Sharing** (System Settings → General → Sharing) is built in and is used by the
Mac-as-AP method below; no install.

## Method 1 — Mac in the middle via ARP-spoof (recommended; no re-provisioning)

The camera stays on your normal Wi-Fi (`192.168.0.x`). The Mac poses as the gateway so the camera's
packets flow through it.

```sh
# 0. enable IP forwarding so the camera keeps working while you capture
sudo sysctl -w net.inet.ip.forwarding=1

# 1. route the camera's traffic through this Mac (replace with the real camera IP + gateway)
sudo bettercap -eval "set arp.spoof.targets <CAMERA_IP>; arp.spoof on"

# 2. capture everything to/from the camera, full packets, to a file
sudo tcpdump -i en0 -s0 -w cam.pcap host <CAMERA_IP>
```

This sees **both** camera↔cloud and phone↔camera. To also capture the phone, add it to
`arp.spoof.targets` as a comma-separated list. Stop with Ctrl-C; turn forwarding back off when done.

## Method 2 — Mac as the Wi-Fi access point (cleanest, but re-provision)

Share the Mac's connection to a Wi-Fi SSID, re-provision the camera + phone onto it
([add-camera.md](add-camera.md)), then capture the bridge:

```sh
sudo tcpdump -i bridge100 -s0 -w cam.pcap
```

`bridge100` is the interface macOS Internet Sharing creates. This gives the least-noisy capture
(only your two devices) but requires putting the camera through setup again.

## Method 3 — on a router

If you run OpenWRT/pfSense, `tcpdump -i <lan> -w cam.pcap host <CAMERA_IP>` there, or use port
mirroring. Zero endpoint changes; the natural home for a long-running capture.

## Phone side — PCAPdroid (no root)

Install **PCAPdroid** from the Play Store / F-Droid. It uses Android's VPNService to capture the
Yoosee app's own packets (pick "com.yoosee" as the target app), and exports a pcap you pull to the
Mac. Easiest way to get the **phone↔camera** and **phone↔cloud** legs; it will **not** show the
camera's independent cloud traffic. Its built-in mitm addon can decrypt TLS only with the extra CA
trusted, which on SDK 36 still needs root or a repackaged app (TLS note below).

## Reading the capture

Useful Wireshark / tshark display filters, using the ports we already know:

```
host <CAMERA_IP>                         # everything to/from the camera
udp.port == 8899 || udp.port == 8900     # LAN search (legacy / IoTVideo) — direct-control.md
ip.src == <CAMERA_IP> && !(ip.dst == 192.168.0.0/24)   # the camera's NON-LAN (cloud) traffic
tcp.port == 554 || tcp.port == 5000      # RTSP / ONVIF
tcp.port == 50000                        # the camera's unknown TCP service
```

First pass, no decryption needed, answers:

- **Which hosts/IPs the camera itself contacts** (resolve them against [cloud-endpoints.md](cloud-endpoints.md); flag any not listed).
- **The P2P relay UDP ports** (currently unknown — this is how you find them).
- **Whether the session goes LAN-direct, hole-punched, or via a relay** (by looking at who the media flow is actually with).
- **The firmware check/download host** when an update runs ([ota-firmware.md](ota-firmware.md)).

## Decryption

- **P2P frames (RC5):** the frame layout and cipher are in
  [protocol-iotvideo.md](protocol-iotvideo.md) / [protocol-gwell.md](protocol-gwell.md). Extract the
  UDP payloads (`tshark -r cam.pcap -Y 'udp' -T fields -e data`) and run them through a standard
  RC5-32 implementation once the round count is confirmed (te-qgj.4). The legacy per-device key is
  computable from the device password ([protocol-gwell.md](protocol-gwell.md)); the IoTVideo session
  key derives from the access token, which you can read from the app's MMKV store or a Frida hook
  ([app-overview.md](app-overview.md)).
- **HTTPS API (TLS):** no cert pinning, but the app trusts only system CAs (SDK 36), so a
  user-installed proxy CA is rejected. To MITM-decrypt it you need **one of**: a rooted phone with
  the mitmproxy CA in the system store, **Frida** to disable TLS validation at runtime, or a
  repackaged APK that trusts user CAs ([crypto-and-auth.md](crypto-and-auth.md)). Then point the
  phone's proxy at `mitmproxy` on the Mac.

## What each capture closes out

| Capture shows | Closes / advances |
|---------------|-------------------|
| The camera's own cloud hosts/IPs/ports | `te-2dc`, [cloud-endpoints.md](cloud-endpoints.md) open questions |
| Camera still works with WAN blocked | `te-99x.8` — the one must-test item in [cloud-redirect.md](cloud-redirect.md) |
| DNS-redirect to a local server is accepted | `te-99x.9` |
| Upload behaviour with no cloud plan | `te-ab8`, [camera-uploads.md](camera-uploads.md) |
| Firmware check + download URL | `te-cyj`, `te-99x.1`, [ota-firmware.md](ota-firmware.md) |
| Whether the LAN-password check opens a session | `te-qgj.9`, [protocol-iotvideo.md](protocol-iotvideo.md) |
| Real frame bytes vs. the static layout | `te-99x.17` (validate the docs) |
