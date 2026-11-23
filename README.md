# PS5 Dump Forge — Synology DSM 套件（SPK）

> **群晖专用 / Synology-only** — DS918+（apollolake，Intel Celeron J3455）在 DSM 7.2+ 上可直接安装的 PS5 游戏转 **zstd 资源包** 套件。全程图形界面，不需要 SSH、Docker 或 Container Manager。装完后在 DSM 主菜单点图标，打开 `http://<NAS-IP>:8095` 即可使用。

中文用户请直接读 **[完整安装说明](INSTALL.zh-CN.md)**：包含图形界面安装、共享文件夹 ACL 授权、zstd 入口位置、自动更新与回滚的完整步骤。

---

## 目录

- [这个仓库提供什么](#这个仓库提供什么)
- [来源与出处](#来源与出处)
- [主动更新：上游变，它跟着变](#主动更新上游变它跟着变)
- [校验值](#校验值)
- [兼容性与权限](#兼容性与权限)
- [zstd 在哪里](#zstd-在哪里)
- [免责声明与法律声明](#免责声明与法律声明)

---

## 这个仓库提供什么

| 文件 | 用途 |
| --- | --- |
| `PS5DumpForge-0.5.7-0002-apollolake.spk` | 可安装套件。DSM → 套件中心 → 手动安装 → 浏览选中即可 |
| `INSTALL.zh-CN.md` | 中文完整教程（安装 / 授权 / 使用 / 更新 / 回滚） |
| `README.md` | 本页：来源、更新机制、校验值、免责声明 |

套件内已打包：

- **内核**：`ps5-dump-forge` **v0.5.7**，Linux x86-64 headless CLI，**Web UI 完整内嵌**（不是占位页）。
- **三个内置浏览根目录**：`/volume3/download/ps5`、`/volume3/docker/motrix/downloads`、`/volume2/unreal/游戏/ps5`（安装/升级时自动补齐，可再增删）。
- **主动更新引擎**：`target/bin/forge-update.sh`，随套件启动自动检查并跟随上游最新版本。
- **加目录工具**：`target/bin/forge-add-root.sh`，幂等、支持中文路径。

---

## 来源与出处

这是一个**非官方重新打包**，所有上游功劳归原作者。链条如下：

```
quer3q/ps5-dump-forge  (上游源码, tag v0.5.7)
        │  上游官方 Linux 发布不带可用的 headless Web UI：
        │  只提供 GTK/WebKitGTK GUI，CLI 内嵌页面是 "web UI not built" 占位页
        ▼
Macondo1224/ps5-dump-forge-fpk-build  (社区自动构建, release v0.5.7-22)
        │  补上缺失的 npm run build:http，再 cargo build，
        │  并打一个网关路径补丁（根路径直连 8095 时为空操作，不影响本套件）
        ▼
本仓库  (Synology SPK 封装)
        INFO / 生命周期脚本 / 图标 / 内置更新引擎 / 三目录默认配置
```

| 组件 | 出处 | 版本 |
| --- | --- | --- |
| 应用本体 | [quer3q/ps5-dump-forge](https://github.com/quer3q/ps5-dump-forge) | tag **v0.5.7** |
| 带完整 Web UI 的 headless 二进制 | [Macondo1224/ps5-dump-forge-fpk-build](https://github.com/Macondo1224/ps5-dump-forge-fpk-build) | release **v0.5.7-22**（资产 `ps5-dump-forge-x86_64.0.5.7-22`） |
| Web UI 构建步骤与网关补丁 | 同上仓库 `ci/patch-gateway.mjs` | 随 v0.5.7-22 |
| Synology SPK 封装、更新引擎、默认配置 | **本仓库** | `0.5.7-0002` |

内核二进制**逐字节原样嵌入**，未做任何修改；其 SHA-256 与上游 Release 公布的 `digest` 完全一致（见下方校验值）。上游 v0.5.7 的变更见 [官方 Release Notes](https://github.com/quer3q/ps5-dump-forge/releases/tag/v0.5.7)。

> 说明：上游 v0.5.7 新增 CLI `edit` 子命令，修复 `.ffpfsc` 指针布局误判与 AMPR 索引被误判损坏两个问题；**没有** ampr-emu-ng 版本升级，也没有 zstd 流程变更。本页不夸大任何版本特性。

---

## 主动更新：上游变，它跟着变

套件**自带更新引擎**，不需要你手动建任务计划或上传脚本。更新链条是通的：

```
本套件的 forge-update.sh
   └─ 每次套件启动时后台检查
        └─ GET Macondo1224/ps5-dump-forge-fpk-build 的 releases/latest
             └─ 该构建仓库每 6 小时跟踪 quer3q/ps5-dump-forge 的上游 tag
                  └─ 上游发新版 → 构建仓库自动产出新资产 → 本套件下次启动即发现并更新
```

**触发时机**：套件每次启动（含 NAS 重启自启、套件中心点启动/重启）后约 20 秒，后台检查一次：

1. 本地哈希 == 远端最新 Release 的 digest → 立即退出，不下载、不打扰；
2. 有新版本 → 下载 → **四重校验** → 备份旧内核 → 替换 → `synopkg restart` 生效；
3. 网络不通 / GitHub 限流 → 记一条日志安静退出，**不影响套件正常启动**；
4. 更新完会重启一次，重启后哈希已一致，不会再检查第二遍；脚本还有 30 分钟失效的锁防止并发/递归。

**四重校验**（防止装坏或装错）：digest 哈希一致、ELF 头 `7f454c46`、含 `serve` 子命令、**拒收**含 `web UI not built` 的占位页版本。

**手动触发 / 关闭**：

```sh
# 立刻更新（控制面板 → 任务计划 → 用户定义的脚本，用户选 root）
/bin/sh /var/packages/PS5DumpForge/target/bin/forge-update.sh
/bin/sh /var/packages/PS5DumpForge/target/bin/forge-update.sh --check   # 只检查
/bin/sh /var/packages/PS5DumpForge/target/bin/forge-update.sh --force   # 强制重装

# 关闭自动更新
echo 0 > /var/packages/PS5DumpForge/var/auto-update.conf
# 恢复：内容改回 1 或删除该文件
```

**日志**：优先 `download/ps5/.forge/update.log`（File Station 可见），否则 `var/update.log`。
**回滚**：旧内核备份在 `target/bin/ps5-dump-forge.prev`，覆盖回去并重启即可。

> 边界要说清：自动更新只覆盖**内核二进制**（实际在迭代的部分）。**SPK 外壳**（INFO、生命周期脚本、图标、默认目录配置）不会自动更新——手动安装的包没有"套件来源"，DSM 不会提示外壳更新。外壳需要更新时，到本仓库下载新 SPK 手动覆盖安装。

---

## 校验值

```
SPK SHA-256    : 018034f1bdf92b8283c4df55f0fa886c856940f314426e18aa8b5b98c51f0bc4
Kernel SHA-256 : f50785a0426f2e7a6b2b90a99c9e7e09d1dde4c2b70661ebef242ee36da8bf16
Kernel size    : 12,192,696 bytes
Release asset  : ps5-dump-forge-x86_64.0.5.7-22
```

内核 SHA-256 与 [v0.5.7-22 Release](https://github.com/Macondo1224/ps5-dump-forge-fpk-build/releases/tag/v0.5.7-22) 的 `digest` 字段一致；SPK 由本仓库打包生成。

---

## 兼容性与权限

- **机型**：DS918+（`apollolake`）。其他 x86-64 群晖机型可能可用但未测试，`INFO` 声明 `arch="apollolake"`。
- **DSM**：`7.2-64570` 或更高；针对目标环境 **DSM 7.4-90075** 做过静态核对，**未在真机实机验证安装/启动**。
- **架构**：x86-64（Celeron J3455）。
- **运行身份**：未签名套件，DSM 7 以内部用户 `PS5DumpForge` 运行（`run-as: package`）。**必须**在 控制面板 → 共享文件夹 → 权限 里给该用户授予目标目录读写（ACL），否则报 `Permission denied`。用图形界面授权而非 `chmod`——新文件才会继承 ACL。

---

## zstd 在哪里

界面里没有名为"zstd 转换"的菜单。正确路径：

**Workshop（工坊）→ ② 要做什么：选 "Game Image"（游戏镜像）→ ③ 选项 → Image：勾 "zstd resource pack (minimal, for ShadowMount+)"**，勾上后才展开压缩级别（1–22）。

> 注意：默认任务类型是 `pkg`（标"推荐"），该分支**不显示** zstd 选项；必须先切到游戏镜像分支。非 AMPR 游戏在该选项上也可见——zstd 包由 ampr-emu-ng 运行时在主机侧解压，不挑游戏。

---

## 免责声明与法律声明

**软件性质**：本仓库是**非官方第三方打包**，与 Synology Inc.、上游 `quer3q`、构建方 `Macondo1224` 均无隶属或背书关系。DSM 安装时会提示"未经 Synology 验证"，属正常现象。

**无担保**：按"现状"（AS IS）提供，不作任何明示或暗示担保，包括但不限于适销性、特定用途适用性与不侵权。作者/打包者对任何直接、间接、附带或后果性损失（数据丢失、设备损坏、游戏损坏等）不承担责任。

**使用风险**：内核来自社区自动构建，虽经校验，仍可能与你的 DSM 版本或环境产生未知兼容问题。**建议先备份重要数据**，并保留旧版 SPK 与 `.prev` 内核以便回滚。自动更新会在每次启动时联网检查并可能自动替换内核与重启——如不希望如此，请按上文关闭自动更新。

**安全**：内置 Web 服务**没有任何登录认证**（官方 CLI 原文：`No protections: anyone on the network can use it`）。仅限可信内网使用。**不要**把 8095 端口做映射、反向代理、QuickConnect 或任何公网暴露。

**合法性**：仅用于你依法有权处理的游戏副本。请勿用于侵犯版权、规避保护或任何违法用途。使用者须自行确认其所在司法管辖区的合规性。

**许可与归属**：应用本体及其依赖遵循各自上游许可证（见上游仓库）；本仓库的封装脚本按同样精神自由使用与修改，但请保留来源署名。

---

<sub>打包版本 `0.5.7-0002` · 内核 v0.5.7 · 群晖 DS918+ 专用 · 来源与免责声明如上</sub>
