# Andowl Q-S2000

![Andowl Q-S2000](images/s2000.jpeg)

The **Andowl Q-S2000** (sold in South Africa as the *Full HD 4K Wireless Smart Camera – Waterproof Outdoor CCTV*) is a rebrand of a Chinese Fullhan-based PTZ camera line. This repo documents what I found on one unit I own.

> Everything here comes from a single unit on my personal bench and network. Other Q-S2000 / Andowl / rebrand units may use different silicon. Testing was done on hardware I own - if you use any of this, do the same.

## The short version

- **SoC:** Fullhan FH8626V100 (ARMv6, 600 MHz), running **Linux 4.9.129**
- **Flash / RAM:** 8 MB SPI NOR, 512 Mbit DDR
- **WiFi:** USB Realtek RTL8188FU dongle
- **Video:** 1080p max (the "4K" in the marketing isn't real on my unit), H.264, 3.6 mm wide + 12 mm telephoto lenses
- **Console:** UART `ttyS0` @ 115200
- **Firmware:** `YGT.AJL33PQ0806-v220831.1043` (companion app: CareCamPro 2.8.23)

## ⚠️ Security - read this first

- **TCP 1300 runs a maintenance service with no authentication at all.** Anyone on the same network can send a command and have it run as root, with the output sent back to them. This is a full compromise, not a theoretical one.
- **TCP 6688 serves a live snapshot with no login:** `http://<cam>:6688/snapshot.jpg`.
- **Mitigation:** put the camera on an isolated network or VLAN, or block inbound LAN access to ports 1300 and 6688, until/unless the vendor fixes this.

## Using the streams

| Stream | URL | Result |
|---|---|---|
| Main | `rtsp://<cam>:8554/profile0` | H.264 1920×1080 @ 16.67 fps + G.711 audio |
| Sub | `rtsp://<cam>:8554/profile1` | H.264 640×360 @ 16.67 fps + same audio |
| Still frame | `http://<cam>:6688/snapshot.jpg` | 640×368 JPEG, no authentication |

```bash
ffprobe -rtsp_transport tcp "rtsp://admin:admin123456@<cam>:8554/profile0"
```

Always force `-rtsp_transport tcp` - UDP gives artefacts over Wi-Fi. VLC handles the digest challenge on its own (*Media → Open Network Stream*).

## Credentials found

| Credential | Where it works | Status |
|---|---|---|
| `admin` / `admin123456` | RTSP, TCP 8554 (Digest only) | **Verified working** |
| *(none)* | TCP 1300 maintenance service | **Verified: no password, no IP check** |
| *(none)* | TCP 6688 `/snapshot.jpg` | **Verified: no login** |
| `!qwert12345` | app diagnostic-log zip (ZipCrypto) | code-derived, not yet reproven on a fresh export |

**Change the RTSP password if your firmware allows it, and isolate the camera on the network regardless** - the 1300 and 6688 issues above aren't fixed by a password change.

## Full technical details

For the complete hardware table, port map, the root-shell proof of concept, the ONVIF findings, log layout, and gotchas from the reverse-engineering process, see the **[detailed README](docs/README-DETAILED.md)**.

## Status

Work in progress. More info will be added over time.
