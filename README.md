# Yutide

A professional network toolbox for engineers and operators — and a clean everyday companion for transferring, downloading and diagnosing, on Windows and WSL2 Ubuntu.

---

## What it is

**For engineers (the core):** protocol decoding with a 58-protocol engine, protocol **families** with automatic per-frame dispatch, stream framing (fixed / length-prefixed / terminator), packet capture, serial/MQTT, SSH/Telnet/ADB/FTP/SFTP, and a high-speed cross-device transfer engine.

**Also for everyday use:** move files between devices, download over HTTP/BT/magnet with resume, and run a one-click network checkup — all inside the same app.

**Privacy is a design constraint, not a slogan:** no telemetry by default, no third-party advertising SDKs, and no outbound requests unless you configure them.

---

## Highlights

### 1. A protocol engine that cannot lie about its rules
* **58 built-in protocol definitions** across industrial, telecom, IoT and standard Internet families.
* **Protocol families:** pick one family (e.g. `szl4`) and every message type of that protocol is decoded — frames are dispatched per member, using each member's own framing rule, with deterministic discriminator matching first.
* **Framing modes:** fixed length, length-field (with explicit "does the length include itself / the header / a trailer" semantics) and terminator-delimited, plus a protocol-level `max_frame` guard.
* **A fixture-verified gate:** every rule declaring framing must decode its fixture as *exactly one whole frame, with no residue*. Rules that cannot prove themselves are not shipped.
* **Never silently drops data:** stream resynchronises after garbage, half-frames wait for more bytes, and unrecognised frames are still handed to you verbatim.
* **Rules are files, not hard-coded logic** — built-in rules ship as JSON and user rules load from your own rules folder.

### 2. Transfer engine built for the awkward cases
Large files transferred directly, huge numbers of small files aggregated, and interrupted transfers resumed.

### 3. One tool for host and WSL
The same app runs on Windows and inside WSL2 Ubuntu, so host-side and WSL-side debugging share one toolbox.

---

## Features

| Area | What you get |
|---|---|
| **Network tools** | TCP client/server, UDP, MQTT, serial; live protocol decoding; packet capture |
| **Remote** | SSH, Telnet, ADB, FTP/SFTP with file transfer |
| **Downloads** | HTTP, BitTorrent and magnet links; resume; bounded retries; **plain-language failure reasons** |
| **Device link** | Share, groups and messaging between devices |
| **Media** | Local playback and a media centre |
| **Browser** | Built-in browser with resource detection (copy a resource link to use it in any other tool) |
| **Recording** | Screen recording with audio (Windows) |

---

## Platform support

| Capability | Windows | WSL2 Ubuntu |
|---|---|---|
| Core toolbox, transfer, downloads, protocol decoding | ✅ | ✅ |
| Graphical interface | ✅ native | ✅ requires **WSLg** (Windows 11) |
| Packet capture | ✅ via Npcap (captures host/LAN traffic) | ⚠️ via `AF_PACKET`, but **only the WSL virtual NIC is visible** — not host or LAN traffic |
| Serial ports | ✅ COM ports | ⚠️ requires `usbipd-win` to attach the device |
| Screen recording audio | ✅ WASAPI loopback | ❌ not available |
| Hardware video decoding | ✅ | ⚠️ depends on `/dev/dri`; falls back to software |
| Administrator rights | needed for capture | root/`cap_net_raw` for `AF_PACKET` |

We document the WSL boundaries deliberately: the Linux build is intended as a **debugging / server-side** toolbox rather than a 1:1 copy of the Windows experience.

---

## Install

* **Windows** — run `Yutide_<version>_x64-setup.exe` from the Releases page.
* **WSL2 Ubuntu** — install the `.deb`, or unpack the `tar.gz`; a one-line install script is provided.
* **Package managers** — `winget` submission planned.

Verify your download against the published `SHA256SUMS`.

---

## Privacy

* **No telemetry by default.**
* **No third-party advertising SDKs.** The in-app promotion slots are first-party content.
* **Diagnostics are off by default**, and when off they neither upload nor write anything.
* Media downloads and protocol decoding never send your data anywhere; remote requests (ad manifest, update check) only happen **if you configure them**.

See [PRIVACY.md](PRIVACY.md) for the full statement, and [THIRD_PARTY_NOTICES.md](../THIRD_PARTY_NOTICES.md) for bundled third-party components and the provenance of the protocol definitions.

---

## Responsible use

Yutide is a general-purpose networking tool.

* **Download only what you are entitled to download**, and comply with the terms of the sites you use and the laws that apply to you.
* The app **does not circumvent DRM or technical protection measures, and does not bypass paid access**.
* **Built-in media rules are protocol-level only** (HLS/DASH/direct media). Site-specific adapters are **user-supplied**, at your own responsibility.
* Copying a detected resource link simply exposes a URL your browser already fetched — using it is your decision.

---

## Roadmap

The cross-device experience — including a mobile companion — is in planning. There is no date to announce; follow this repository to hear when it lands.

---

## License

Proprietary — not open source. Free for personal, learning, research, evaluation and non-profit use. Using the protocol-parsing features in a company or other organisation requires a commercial licence; every other feature is unrestricted. See [LICENSE](../LICENSE).

---

## 中文简介

**雨霆（Yutide）** 是一个面向工程与运维的**专业网络工具箱**，也能日常使用：

* **协议引擎**：内置 **58 份协议定义**；按**协议族**一次选择即可解析该协议的全部报文（逐成员按各自的 framing 规则分帧、判别字段优先、分派失败也不丢帧）；支持定长 / 长度前缀 / 结束符三种分帧与协议级最大帧长；**规则以 JSON 文件形式提供**，用户可放自己的规则文件夹；每条规则都必须通过"夹具恰好解出一整帧"的校验才能发布。
* **日常也能用**：跨设备传文件（大文件直传 + 海量小文件聚合 + 断点续传）、下载（HTTP / BT / 磁力，失败会显示人话原因）、一键网络体检。
* **隐私优先**：默认无遥测、不接第三方广告 SDK、诊断默认关闭且关闭时零外发；只有你显式配置后才会有远程请求。
* **平台**：Windows 与 WSL2 Ubuntu 同一套工具；⚠ WSL 下**抓包只能看到 WSL 自身网卡**、串口需 `usbipd-win` 转发、**无录屏音频**、无 `/dev/dri` 时软解 —— 我们把这些边界都写清楚，WSL 版定位为**调试/服务器向**。
* **合规**：不做 DRM 绕过、不绕付费墙；内置媒体规则只含协议层，站点适配由用户自建；请只下载你有权下载的内容。

**跨设备（含手机）体验正在规划中**，暂无时间表。
