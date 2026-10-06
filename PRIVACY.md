# Privacy Policy

> Version: 1.0 / Effective date: 2026-10-02
> Scope: the Yutide desktop application. **This document ships with the installer and can be read offline.**

## In one sentence
**Nothing leaves your device by default.** Unless you configure a remote address, the app does not send
your data to any server; all logs, counters and diagnostics stay **on your machine**.

## 1. What we collect and where it goes
| Data | Sent off-device? | Notes |
|---|---|---|
| Ad impressions / clicks | **Local only** | Used for local statistics and the "promotion views remaining" counter, stored in the local `config.json`; **never uploaded**, and no third-party tracking scripts |
| Crash / error logs | **Local only** | Written to the local log directory (rotation + panic dump); diagnostic-bundle export is **off by default**; this version has **no upload implementation at all** |
| UI language / region preference | **Local only** | Stored in `config.json` |
| Data produced by the network tools (capture / serial / TCP / UDP / SSH / FTP / ADB, etc.) | **Never uploaded** | These features communicate only with **the peer you specify**, on your instruction; the app runs no relay server |

## 2. Advertising and third parties
* Creatives come from a **remote manifest** verified with an **Ed25519 signature**; **if no advertising
  public key is configured, the app makes no remote request at all**, and the promotion slot falls back to
  **built-in first-party creatives** (never blank, never an error).
* **No personalised targeting**: no behavioural profiling, no tracking cookies, **no third-party advertising SDKs**.
* Our current GDPR/CCPA position: because we do **no personalised targeting, no cross-site tracking and no
  off-device transfer of personal data**, the app runs on a **data-minimisation** basis; if we ever introduce
  any off-device capability (for example a crash-reporting service), it **must be off by default, enabled only
  with explicit user consent**, and this policy will be updated at the same time.

## 3. Logs and diagnostic bundles
* Logs are written by default under `logs/` in the app data directory (Windows:
  `%APPDATA%\super-engine\logs`; Linux: `~/.local/share/super-engine/logs`) and rotate by size.
* "Allow creating a diagnostic bundle" is **off by default**; while it is off, **nothing is sent off-device and
  no local bundle is created**. When enabled, a text bundle (version, platform, recent logs) is created in the
  **local** `diagnostics/` folder and **you must send it manually** — the app never sends it itself.
* A bundle may contain runtime information such as peer addresses; **please check it yourself before sending**.

## 4. Your control
* Turning off "Allow creating a diagnostic bundle" ⇒ no diagnostic files are created;
* Deleting the app data directory ⇒ clears local logs, counters and preferences;
* After uninstalling the app, the only files left behind are those you copied out yourself.

## 5. Third-party components and licensing
Third-party components and their licences are listed in the `THIRD_PARTY_NOTICES.md` shipped with the app
(reachable from the About page). The media component FFmpeg is used under **LGPL**.

Licensing, identical to the `LICENSE` shipped with the app: Free for personal, learning, research, evaluation and non-profit use. Using the protocol-parsing features in a company or other organisation requires a commercial licence; every other feature is unrestricted.

## 6. Contact
* Support email: hesen789a@outlook.com

---

# 隐私政策（Privacy Policy）

> 版本：1.0 ／ 生效日期：2026-10-02
> 适用范围：Yutide（雨霆）桌面应用。**本页随安装包分发，离线可读。**

## 一句话
**默认不外发。** 应用在未配置任何远程地址时，不会主动向任何服务器发送你的数据；
所有日志、计数与诊断信息都留在**本机**。

## 1. 我们收集什么、发到哪里
| 数据 | 是否外发 | 说明 |
|---|---|---|
| 广告曝光 / 点击计数 | **仅本地** | 用于本地统计与"剩余免广告次数"，保存在本机 `config.json`；**不上传**、不接第三方追踪脚本 |
| 崩溃 / 错误日志 | **仅本地** | 写入本机日志目录（滚动 + panic 落盘）；**默认关闭**诊断包导出；本版本**没有任何上传实现** |
| 界面语言 / 地区偏好 | **仅本地** | 保存在 `config.json` |
| 网络工具（抓包/串口/TCP/UDP/SSH/FTP/ADB 等）产生的数据 | **不上传** | 这些功能只在你的指令下与**你指定的对端**通信；应用不设中转服务器 |

## 2. 广告与第三方
* 广告素材由**远程清单**提供，清单使用 Ed25519 签名校验；**若未配置广告公钥，应用不会发起任何远程请求**，
  广告位回退到内置自营素材（不会空白、也不会报错）。
* **不做个性化定向**：不采集行为画像、不使用追踪 Cookie、不接入第三方广告 SDK。
* 我们当前的 GDPR/CCPA 立场：由于**不进行个性化定向、不做跨站追踪、也不外发个人数据**，
  应用按"最小化处理"原则运行；若将来引入任何外发能力（例如错误上报服务），
  **必须默认关闭、用户显式同意后才启用**，并同步更新本政策。

## 3. 日志与诊断包
* 日志默认写在应用数据目录的 `logs/`（Windows：`%APPDATA%\super-engine\logs`；
  Linux：`~/.local/share/super-engine/logs`），按大小滚动。
* 设置里的"允许生成诊断包"**默认关闭**；关闭时**既不外发、也不生成本地诊断包**。
  开启后仅在本机 `diagnostics/` 生成一个文本包（含版本、平台、最近日志），
  **需要你手动发送**给我们 —— 应用自己不会发送。
* 诊断包可能包含对端地址等运行信息，**发送前请自行检查**。

## 4. 你的控制
* 关闭"允许生成诊断包"⇒ 不再生成任何诊断文件；
* 删除应用数据目录 ⇒ 清空本地日志、计数与偏好；
* 卸载应用后会保留的只有你手动复制出去的文件。

## 5. 第三方组件许可与授权口径
第三方组件的许可见随包分发的 `THIRD_PARTY_NOTICES.md`（关于页有入口）。
其中媒体组件 FFmpeg 以 **LGPL** 方式使用。

授权口径（与随包分发的 `LICENSE` 一致）：个人、学习、研究、评估与非营利使用免费；在公司或其他组织中商业性使用「协议解析功能」需取得商业授权，其余功能不受限制。

## 6. 联系方式
* 支持邮箱：hesen789a@outlook.com
