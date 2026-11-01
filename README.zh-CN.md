# nxnative 教程 —— 在 iPhone 上运行 Switch 游戏

[English](README.md) | [中文](README.zh-CN.md)

[nxnative](https://github.com/seedds/nxnative) 的实操配套教程。nxnative 让 Nintendo Switch 游戏**原生**运行在苹果平台上：游戏自己的代码在 Mac/iPhone 上执行，由一个宿主环境应答 Horizon 的系统调用，并在 Metal 上实现 Switch 的图形 API（NVN、Vulkan、NVIDIA 桌面版 OpenGL）。

本教程带你把一款自己拥有的游戏从 NSP 文件变成 iPhone 上可玩的应用 —— 包含一次真实安装（马里奥赛车8豪华版 4.0.1 → iPhone 17 Pro，iOS 27.2）踩过的所有坑和对应解法。

> **只用自己的游戏。**你需要从自己主机 dump 出来的 NSP 文件和 `prod.keys`。本教程不分发游戏、密钥或固件。不要分享你的 `prod.keys` —— 构建过程只读取它，永不打印其内容。

## 哪些游戏能玩？

- **nxnative 的[游戏表](https://github.com/seedds/nxnative#readme)列出所有已验证的游戏** —— 90+ 款，从《旷野之息》《王国之泪》到马里奥赛车8豪华版、宝可梦、密特罗德 生存恐惧、空洞骑士 丝之歌、哈迪斯 II。每款都有测试过的 profile 和一份说明它需要什么的笔记。
- **不在表里的游戏**也能构建：脚本会调查它的二进制（生成 `<title ID>_survey/survey.md`）、猜测一个临时 profile，产出一个尽力而为的 IPA —— **可能开不了机**；调通一个新游戏是真正的工作量（见 nxnative 的 `docs/agents`）。
- 同样的流程**纯 Mac 也能走**：`--platform mac-app` 打包出双击即玩的 `.app`，完全不需要 iPhone。

## 前置条件

| 工具 | 说明 |
|---|---|
| 克隆的 [nxnative](https://github.com/seedds/nxnative) | 所有命令在仓库根目录执行 |
| Apple 芯片 Mac、Xcode | Metal 编译器必须匹配 nxnative 的工具链锁定版本（见其 `docs/development.md`） |
| Python 3.12+ | 需 `lz4`、`capstone`、`fonttools`（3.14 之前还需 `backports.zstd`）；文件是 NSZ/XCZ 才需要 `pip install nsz` |
| xcodegen、.NET SDK 10、zstd | 在 `PATH` 上 |
| 某一款游戏的 NSP | 本体、可选的更新和 DLC，放在同一个文件夹 |
| `prod.keys` | 来自你自己的主机 |
| [Sideloadly](https://sideloadly.io) | 用 Apple ID 给 IPA 签名并安装 |
| Apple ID | 免费账号即可：3 个应用、每 7 天重签；付费账号一年一签 |
| iPhone + USB 线 | 已开启开发者模式（一次性，见排查表第 3 条） |

**磁盘布局提示**：NSP 和成品 IPA 放哪都行，但构建工作目录（`<title ID>_build/`）要放在内置 APFS 盘上。exFAT 外置盘会让着色器阶段慢约 10 倍（每个着色器一次 `metal` 调用，几万个小文件）。

## 30 秒版本

```bash
# 1. 构建 —— 小 IPA + 分离的游戏镜像（超过 ~4 GB 的游戏必须如此）
python3 scripts/build_ipa_from_nsp.py /path/to/game-nsps -k /path/to/prod.keys \
  --romfs separate -o /path/to/Game.ipa
#    （构建结束时会把确切的 push_romfs.sh 命令打印出来）

# 2. 用 Sideloadly 安装 Game.ipa（你的 Apple ID）

# 3. 把游戏文件推进应用的沙盒
xcrun devicectl device info apps --device <UDID>          # 查真实 bundle ID
bash scripts/push_romfs.sh <UDID> <bundle ID> /path/to/<titleID>_build/extract/<version>/RomFS.img
```

首次构建 2 分钟（《密特罗德 生存恐惧》）到 10 分钟（《旷野之息》）；输入不变的重复构建约 20 秒。手机上按游戏体积的两倍左右留空间：IPA（约 1 GB）加游戏镜像（马里奥赛车8豪华版：11.3 GB）。

## 第 1 步 —— 构建 IPA

```bash
python3 scripts/build_ipa_from_nsp.py /path/to/game-nsps \
  -k /path/to/prod.keys \
  --romfs separate \
  -o /path/to/Game.ipa \
  --progress lines
```

过程：NSP 按标题 ID 识别并匹配游戏的 profile，解包、在 Apple 内核要求处重写代码模块、把着色器翻译成 Metal，最后用 Xcode 打包。所有中间文件在 NSP 文件夹的 `<title ID>_build/` 下；着色器缓存让重复构建很快，且中断可续。

常用选项：

| 选项 | 作用 |
|---|---|
| `--romfs separate` | 游戏文件不进 IPA（见下）—— iPhone 安装一律用它 |
| `-p v327680` | 打包指定版本（默认：文件夹里最新的更新） |
| `-l zh-Hans` | 游戏语言；之后可免重构建修改：应用 `Documents/env.txt` 里的 `DM_LANG` |
| `--app-name`、`--bundle-id` | 主屏名称和 bundle ID |
| `--allow-other-metal` | 本机 Xcode 的 Metal 编译器与测试过的不同时继续（先读 nxnative 对 issue #242 的说明） |
| `--platform mac-app` | 构建双击即玩的 Mac `.app` 而非 IPA |
| `--survey-only` | 只分析一款未知游戏 |

**为什么用 `--romfs separate`：**iOS 安装器永远见不到超过 ~4 GB 的档案（App Store 的上限），大游戏一体打包的 IPA 是 zip64 且内含单个 10+ GB 文件 —— 手机必然报 `PackageExtractionFailed`。分离后得到约 1 GB 的 IPA 加原始游戏镜像，第 3 步直接把镜像拷进应用沙盒。

## 第 2 步 —— 用 Sideloadly 安装

打开 Sideloadly，拖入 IPA，用你的 Apple ID 签名安装。两点须知：

- **登录失败**报 `SSLError(SSLEOFError)`（gsa.apple.com）：Sideloadly 自带老版 OpenSSL，其 TLS 握手在部分网络上会被重置（联通直连必然如此；系统 `curl` 仍能通，所以"curl 没问题"说明不了什么）。走代理即可 —— 开系统代理，Sideloadly 会自动使用。`Fake2-Anisette download failed` 警告无害。
- **iOS 26+/27 上 Sideloadly 会在 bundle ID 后追加你的团队 ID**：`org.nxnative.<titleID>` 变成 `org.nxnative.<titleID>.<TEAMID>`。第 3 步需要的是后者。

## 第 3 步 —— 推送游戏文件

应用带着构建时镜像的身份（`romfs.id`），只接受身份匹配的 `Documents/romfs.img` —— 所以先装应用，再推一次：

```bash
# 查真实（带团队后缀）的 bundle ID
xcrun devicectl device info apps --device <UDID> | grep nxnative

# 推送（手机解锁、开发者模式已开）
bash scripts/push_romfs.sh <UDID> <bundle ID> \
  "/path/to/<titleID>_build/extract/<version>/RomFS.img"
```

`push_romfs.sh` 把镜像作为单个文件拷入应用数据容器。中断的拷贝永远不会被使用（应用校验镜像身份），失败后重跑即可。

## 日常使用

- **每 7 天**（免费 Apple ID）：签名过期 —— 用 Sideloadly 重装同一个 IPA。游戏文件不动，无需重推。
- **只在游戏或版本变化时重推** —— 同一版本的重新构建镜像身份不变。
- 支持与 iPhone 配对的手柄游玩。

## 排查表

| 症状 | 原因 → 解法 |
|---|---|
| Sideloadly 登录 `SSLError(SSLEOFError)` | 老 OpenSSL 指纹被在线重置 → 开系统代理 |
| 手机报 `PackageExtractionFailed` | IPA 是 zip64 / 超 4 GB → 用 `--romfs separate` 重建 |
| 手机上没有*开发者模式*开关 | 与 Xcode 配对一次前是隐藏的（Window → Devices and Simulators），然后 设置 → 隐私与安全性 → 开发者模式 → 开 → 重启 |
| 构建停在 Metal 编译器 `[RED]` 行 | 编译器 ≠ 测试版本 → `--allow-other-metal`（或装匹配的 Xcode） |
| `The Xcode build system has crashed` 反复出现 | 构建目录跨盘移动，绝对路径失效 → `rm -rf <titleID>_build/ipa/build/ios/pkg/dd` 后重建 |
| 着色器阶段极慢（数小时） | 工作目录在 exFAT 上 → 把 `<titleID>_build/` 挪到内置 APFS（带上 `shader/`；续跑会恢复已完成的着色器） |
| `devicectl` 设备 `unavailable` | 手机锁屏 → 解锁 |
| `push_romfs.sh`：找不到应用 | bundle ID 不对 → 用 `device info apps` 查；Sideloadly 加了你的团队 ID |
| 用 devicectl 装 Sideloadly 的大 IPA：文件不存在 | Sideloadly 退出时清掉签名缓存 —— 不是真路径；改用小号分离 IPA 安装 |

## 本文件夹内的文件

- `README.md` / `README.zh-CN.md` —— 本教程（英文 / 中文）
- `SKILL.md` —— 同一流程的 Claude Code skill（见下）
- `scripts/build_ipa_from_nsp.py`、`scripts/push_romfs.sh` —— 本教程驱动的两个 nxnative 脚本，复制到此处**仅供阅读**（MIT 许可，见 `scripts/LICENSE`）。它们按 nxnative 目录树解析相对路径，请从 [nxnative](https://github.com/seedds/nxnative) 的克隆中运行，不要在本文件夹里跑。
- `LICENSE` —— 本仓库的 MIT 许可证。

## 作为 Claude Code skill 使用

`SKILL.md` 把本教程变成一个 agent skill：Claude 会执行构建、查询设备和 bundle ID、驱动推送。

```bash
git clone https://github.com/leezhong/nxnative-tutorial
ln -s "$(pwd)/nxnative-tutorial" ~/.claude/skills/nxnative-tutorial
```

之后在任何会话里说："把马里奥赛车编译装到手机" / "build and install this Switch game on my iPhone"。

---

*2026-11-01 端到端验证：马里奥赛车8豪华版 4.0.1（本体 1.0.0 US + 更新 + DLC）→ 1004 MB IPA + 11.3 GB 镜像 → iPhone 17 Pro，iOS 27.2。*
