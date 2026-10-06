# Changelog

All notable changes to Yutide are documented here.
This project follows semantic versioning.

## Unreleased

### Added
* Media rules now load from a **rules folder** instead of a compiled-in list: every `*.json` in the
  built-in rules directory and in the user rules directory (`<app-data>/super-engine/media-rules/`)
  is loaded, with a reload action so changes apply without restarting. Built-in rules remain
  **protocol-level only**; site-specific adapters are user-supplied.
* Browser resource panel: **copy resource link** / copy all / copy as `curl`. The app never
  auto-downloads these links — it only exposes the URL the browser already fetched.
* Media rules import/export.

### Changed
* **Product renamed to `Yutide`** (was `Tidekit`): `productName`, `identifier`
  (`com.yutide.desktop`), `mainBinaryName`, window title, publisher/copyright, the brand values in
  `src/brand.ts`, the icon sources (`brand/yutide-*.svg`) and every hardcoded name in the build
  scripts. The identifier change is done **before** the first public tag on purpose: renaming it
  after a release would break the update path. **User data is not affected** — it lives under the
  literal path `%APPDATA%\super-engine` (`engine_xfer::app_data_dir("super-engine")`), not under the
  identifier. Board/diagnostic scripts used to find the app window by its **title**; they now match
  the X11 **WM_CLASS** instead, so a future rename cannot silently break them again.

### Fixed
* **Linux packages no longer ship the local site profile `profile.json`.** The Linux resource glob
  was `resources/media-rules/*.json`, which also matched that file — a local, git-ignored file that
  holds real site hosts — and installed it to `/usr/lib/<product>/media-rules/`. The glob is now
  `*-rule.json`, matching the base config, the docs and the existing unit-test contract
  (`test_embedded_rule_file_names_are_packaged_by_glob`). A new unit test
  (`test_platform_configs_pack_only_rule_json`) parses every `tauri*.conf.json` and fails if a
  `media-rules` glob is broader than `*-rule.json`.
  *How this was proven:* the package was rebuilt **with the stale staged copy still on disk** and came
  out clean — the bundler reads the **source glob**, so deleting the staged file would only have
  masked the bug.
* The portable Linux tarball builder silently omitted `PRIVACY.md` (it copied the legal documents
  from a directory that does not contain it). It now searches both locations and fails loudly when
  one is missing.

### Removed
* **The bundled `ffmpeg.exe` / `ffprobe.exe` sidecars** (about 139 MB each) are gone from the
  Windows installer: `bundle.externalBin` was removed and the sidecar sources were deleted. All
  media work already runs in-process through the vendored LGPL libraries
  (`commands/media/engine_libav`: remux/merge, transcode, streaming, capture, probing), so these
  were dead weight: the NSIS installer drops from **140.43 MB to 61.95 MB**
  (`Yutide_1.0.0_x64-setup.exe`, 64,959,995 bytes, sha256 `63f5dfe0…`). The seven
  `avcodec`/`avfilter`/`avformat`/`avdevice`/`avutil`/`swscale`/`swresample` DLLs are **kept**.

## 1.0.0

First public release.

### Protocol engine
* 58 built-in protocol definitions across industrial, telecom, IoT and standard Internet families.
* **Protocol families** with automatic per-frame dispatch: select a family once, and every message
  type of that protocol decodes — each member matched by discriminator first, then by clean-parse
  scoring, and never dropping a frame.
* Framing: fixed length, length-field (explicit `includes` semantics for the length field, header and
  trailer) and terminator-delimited, with a protocol-level `max_frame` guard and half-frame waiting.
* VarInt7 length encoding (MQTT "remaining length"), field-level and framing-level.
* Stream resynchronisation after garbage; `FrameTooLarge` reporting instead of misreporting corrupt
  input as a protocol-definition error.
* **Fixture-verified rule gate:** any rule declaring framing or discriminator must prove itself
  against its fixture (exactly one whole frame, no residue; discriminators must be self-satisfied and
  mutually exclusive within a family).

### Network tools
* TCP client/server, UDP, MQTT and serial sessions with live protocol decoding.
* Packet capture (Npcap on Windows, `AF_PACKET` on Linux/WSL).
* Protocol parser with per-family selection.

### Remote
* SSH, Telnet, ADB and FTP/SFTP sessions with file transfer, including correct cancellation of
  in-flight transfers.

### Downloads
* HTTP, BitTorrent and magnet downloads with resume.
* Bounded automatic retries (no unbounded loops), user-initiated cancellation at any time, and
  **plain-language failure reasons** shown inline with full details on expand.

### Device link
* Sharing, groups and messaging between devices, with large-file and many-small-file transfer paths.

### Media and browser
* Local media playback and media centre.
* Built-in browser with resource detection.

### Recording
* Screen recording; audio capture via WASAPI loopback on Windows.

### Privacy and compliance
* No telemetry by default; no third-party advertising SDKs.
* Diagnostics are **off by default**, and when off they neither upload nor write anything.
* `LICENSE` (proprietary, personal use free / commercial licence for the protocol-parsing features),
  `PRIVACY.md` and `THIRD_PARTY_NOTICES.md` (including the provenance of protocol definitions) ship
  with the application and are openable offline from the About page.

### Platform
* Windows and WSL2 Ubuntu from one codebase.
* Documented WSL boundaries: capture sees only the WSL virtual NIC, serial requires `usbipd-win`,
  recording audio is unavailable, and hardware decoding depends on `/dev/dri`.
