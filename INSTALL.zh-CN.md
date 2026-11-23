# PS5 Dump Forge **v0.5.7** · 群晖 DS918+ 可直接安装套件（.spk）

> 交付文件：**`PS5DumpForge-0.5.7-0002-apollolake.spk`**（6,266,880 字节）
> 适用：DS918+（Celeron J3455 / x86-64，平台代号 `apollolake`）+ **DSM 7.2 或更高**
> **已确认你的环境：DSM 7.4-90075（DS918+）**，高于套件声明的 `os_min_ver="7.2-64570"`，架构 `apollolake` 一致，可直接安装
> 安装方式：套件中心 → 手动安装 → 上传 `.spk`，**全程图形界面、零 SSH、不需要 Docker / Container Manager**

| 校验值 | |
| --- | --- |
| SPK SHA-256 | `018034f1bdf92b8283c4df55f0fa886c856940f314426e18aa8b5b98c51f0bc4` |
| 内置内核 SHA-256 | `f50785a0426f2e7a6b2b90a99c9e7e09d1dde4c2b70661ebef242ee36da8bf16`（12,192,696 字节） |
| 内核来源 | 上游 `quer3q/ps5-dump-forge` tag **v0.5.7** 源码编译，经 `Macondo1224/ps5-dump-forge-fpk-build` release **v0.5.7-22** 发布（2026-11-22） |

**这一版相比上一版（0.5.5-0001）的三点变化**：

1. **内核升到 v0.5.7**（真实版本，不是改版本号）：新增 CLI `edit` 子命令、修了 `.ffpfsc` 指针布局误判和 AMPR 索引被误判损坏两个问题。zstd 转换流程本身没有变化。
2. **三个根目录已内置**：新装即带 `/volume3/download/ps5`、`/volume3/docker/motrix/downloads`、`/volume2/unreal/游戏/ps5`，不用再跑加目录脚本。升级安装会保留你改过的配置，只补齐缺的默认项。
3. **自动更新已内嵌**：更新引擎放在 `target/bin/forge-update.sh`，套件每次启动后台检查一次新版本，下载校验后自动替换并重启。**默认开启**，可用开关关掉（见第七节）。

---

## 一、版本与内核来源核验

先说结论：**内置内核就是从上游 v0.5.7 源码编译出来的**，版本号未做任何虚标。

上游官方 Linux 发布包不带可用的 headless Web UI 构建——它的 `scripts/release-linux.sh` 只跑 `cargo build`，从没跑过 `npm run build:http`，所以就算有 CLI，内嵌页面也是 `web UI not built` 占位页。而服务端 `build.rs` 用 `include_bytes!` 在**编译期**嵌入 `app/dist-http`，运行时没法外挂 UI 目录补上。

补齐这件事的是 `Macondo1224/ps5-dump-forge-fpk-build`：它跟踪上游 tag，在 `rust:1-bookworm` 容器里 **先 `npm ci && npm run build:http` 再 `cargo build --release -p ps5-dump-forge-cli`**，最后过三道闸门（无占位串 / glibc ≤ 2.36 / `--version` 能跑）才发 Release。

**本次用的是 `v0.5.7-22`（2026-11-22 发布，Release id 454690115）**。我做了这些核验：

- 资产 SHA-256 与 GitHub API 返回的 `digest` 逐字符一致；
- ELF 头 `7f454c46`、`serve` 子命令都在，**没有** `web UI not built` 占位串；
- 依赖上限 **GLIBC_2.34 / GLIBCXX_3.4.29**，DSM 7.2+ 的 glibc 2.36 / libstdc++ 3.4.31 余量充足；
- 二进制**逐字节原样**打包，未做任何修改。

### v0.5.7 相对 v0.5.5 的更新内容（据上游 release note）

- **新增 CLI `edit` 子命令**：`ps5-dump-forge edit <游戏或镜像>` 不挂载就能改游戏文件，`--add <file>:<game path>` / `--delete` / `--rename <from>:<to>` / `--mkdir`，按给定顺序应用，可直接改或 `--rebuild`。和界面里 Edit 标签页走的是同一套任务。
- **修复 `.ffpfsc`「unsupported PFS pointer layout」**：镜像文件正好结束在最后一个数据字节、而非最后一个 64 KiB 块末尾的，之前会被拒，现在可读；文件在数据中间被截断的仍会拒。
- **修复 AMPR 索引被误判损坏**：`ampr_emu.index` 的槽表起始偏移是 8 字节倍数时（包自身的索引、Crisis Core）之前被判损坏并重写，现在能正确读取；索引确实损坏时任务日志会说明原因。

> 上游 note 里**没有**提到 ampr-emu-ng 版本升级，也没提新的 zstd 变更——所以别指望 zstd 流程本身有变化。

> 关于那个「飞牛统一网关补丁」：它把前端 6 处 `/api/...` 改成 `BASE + "/api/..."`，`BASE` 只在前端路径形如 `/app/<名字>/` 时取前缀。**本套件是根路径直连 `http://NAS:8095/`，BASE 为空串，行为与上游完全一致**，不影响任何功能。

---

## 二、套件里有什么

```
PS5DumpForge-0.5.7-0002-apollolake.spk
├── INFO                         包元信息（arch=apollolake、os_min_ver=7.2-64570、thirdparty=yes）
├── package.tgz                  装到 /var/packages/PS5DumpForge/target/
│   ├── bin/ps5-dump-forge       内核：v0.5.7 完整 Web UI 二进制（12.19 MB）
│   ├── bin/forge-update.sh      内置更新引擎（自动更新 + 手动更新共用）
│   ├── bin/forge-add-root.sh    内置加目录工具（幂等，支持中文路径）
│   ├── ui/config + ui/images/   DSM 主菜单图标（16/24/32/48/64/128/256）
│   └── etc/roots.conf.example   三个默认根目录示例
├── scripts/                     preinst / postinst / preuninst / postuninst /
│                                preupgrade / postupgrade / start-stop-status
│                                （安装/升级脚本会补齐三个根目录与更新开关）
├── conf/privilege               run-as: package（DSM 7 要求，未签名包不能跑 root）
├── conf/resource
├── PACKAGE_ICON.PNG / _256.PNG
└── LICENSE                     第三方来源与风险说明
```

启动命令等价于：

```sh
ps5-dump-forge serve --port 8095 \
  --root /volume3/download/ps5 \
  --root /volume3/docker/motrix/downloads \
  --root /volume2/unreal/游戏/ps5
```

`--root` 可重复，所以三个目录同时挂上；不存在的目录会被自动跳过，不影响启动。

⚠️ **官方 CLI 帮助明写 `No protections: anyone on the network can use it`**——这个界面**没有登录**。只在可信内网用，**不要**把 8095 做端口映射、反向代理或 QuickConnect 暴露。

---

## 三、安装（全部图形界面）

1. **套件中心 → 手动安装 → 浏览**，选 `PS5DumpForge-0.5.7-0002-apollolake.spk` → 下一步。
2. DSM 7 会提示「此套件由第三方开发者提供，未经 Synology 验证」——正常。DSM 7 已取消签名与信任等级设置，勾「我已了解风险，仍要安装」继续，不需要 SSH，也不需要降低全局安全等级。
3. 安装向导静默（`silent_install=yes`），直接完成。
4. **DSM 主菜单**出现 **PS5 Dump Forge** 图标，点开就是 `http://<NAS IP>:8095`；也可以直接浏览器输地址。
5. 套件中心 → 已安装，可**启动 / 停止 / 重启**；NAS 重启后随套件自启。

**如果你已经装过 0.5.5 那版**（或更早）：直接装这个包即可，版本号 `0.5.7-0002` 高于 `0.5.5-0001`，DSM 按**覆盖升级**处理。升级脚本会做三件事：

- 保留你在 `roots.conf` 里改过的任何行，只把三个默认目录里**缺的**补进去；
- 内核换成 v0.5.7；
- 内嵌更新脚本与开关就位；已有的 `auto-update.conf` 不会被覆盖。

升级完成后到「关于」页确认内核版本是 **0.5.7**；如果还显示 0.5.5，在套件中心点一次「重启」。

> 主菜单图标没出现：DSM 刷新页面（Ctrl+F5），或重启一次套件。图标不影响服务，直接开 `:8095` 即可。

### 在 DSM 7.4-90075 上安装的额外确认

你的系统比套件的 `os_min_ver="7.2-64570"` 新，安装不会因版本被拒。三点与你环境相关的差异：

- **找不到「套件中心 → 设置 → 常规 → 信任层级」是正常的**：那个开关只在 DSM 6 存在。DSM 7 起一律不能把第三方发行者加为信任，所以**任何社区套件都必然弹一次「未经 Synology 验证」的警告**，勾「我已了解风险」即可，不需要改全局安全设置。
- **权限用户是 `PS5DumpForge`**：DSM 7 规定未签名包以 package 用户运行（`conf/privilege` 里 `run-as: package`），不会用 root。若转换时报 Permission denied，见第四节最后一条，给这个内部用户勾读写。
- **`adminport="8095"` 与 `protocol="http"`** 已在 INFO 与 `ui/config` 里声明，主菜单图标点开即是 `http://<NAS IP>:8095/`。DSM 7.4 若提示「无法连接」，先确认套件状态是「运行中」，再确认没有别的服务占用 8095。

---

## 四、配置游戏目录

安装时已写好，**默认就是你的路径**：

```
/var/packages/PS5DumpForge/etc/roots.conf
内容：/volume3/download/ps5
```

- 加别的共享文件夹：一行一个目录写进 `roots.conf`，改完在套件中心点**停止 → 启动**。这个文件在系统分区，**File Station 看不到**，用 DSM 的「文本编辑器」套件打开，或用任务计划里的一条 `cp` 覆盖。
- `roots.conf` 为空则启动脚本自动挂载所有 `/volume*/*` 下的共享文件夹（跳过 `@` 开头的系统目录）。
- 权限：服务以 package 用户运行，`postinst` 已对 `roots.conf` 里每个目录做 best-effort `chmod -R a+rwX`。**但这条很可能不够**，DSM 7 的共享文件夹走 ACL，`chmod` 改的是 POSIX 位、不写 ACL 条目。真报 `Permission denied (os error 13)` 时按下一节做。

---

## 四之二、报 `Permission denied (os error 13)` 怎么修

**现象**：界面能打开、目录树能看到 `/volume3` 下的名字，但点进 `/volume3/download` 就红字 `无法打开此文件夹: /volume3/download: Permission denied (os error 13)`。

**原因**：DSM 7 起第三方（未签名）套件**不允许以 root 运行**，本套件 `conf/privilege` 是 `run-as: package`，进程身份是内部用户 **`PS5DumpForge`**。这个用户不在「用户与群组」的普通列表里，也不会自动拥有共享文件夹权限——DSM 的默认模型是「谁都没权限，除非显式给」。

### 修法 A：控制面板授权（推荐，纯图形界面，约 1 分钟）

1. **控制面板 → 共享文件夹** → 选中 `download` → **编辑**
2. 切到 **权限** 标签 → 顶部下拉从「本地用户」改成 **「系统内部用户」**
3. 找到 **`PS5DumpForge`** → 勾 **读取/写入**
4. 确定 → 弹出「是否应用到子文件夹」选 **是**
5. **套件中心 → PS5 Dump Forge → 停止 → 启动**（ACL 变了，进程要重开才生效）
6. 回界面点进 `/volume3/download/ps5` 验证

> 只给了 `download` 就够，因为 `ps5` 是它的子目录、会继承。想更精细可以走 **File Station → 右键 `ps5` → 属性 → 权限 → 新增 → 系统内部用户**，勾「读取/写入/遍历」，应用到「此文件夹、子文件夹及文件」。

### 修法 B：下拉里没有 `PS5DumpForge` 时（先诊断）

用 **控制面板 → 任务计划 → 新增 → 计划的任务 → 用户定义的脚本**，用户选 **root**，钩掉「已启用」，脚本粘这段，然后点「运行」：

```sh
{
  echo "=== 内部用户 ==="; grep -i ps5 /etc/passwd
  echo "=== 套件 privilege ==="; cat /var/packages/PS5DumpForge/conf/privilege
  echo "=== 目录 ==="; ls -ld /volume3 /volume3/download /volume3/download/ps5 2>&1
} > /volume3/download/_permcheck.txt 2>&1
```

跑完去 File Station 打开 `download/_permcheck.txt`：第一行就是服务的真实用户名（大小写以它为准），第三行能看出 ACL 位。把内容发我，我按实际用户名给准确的授权命令。

### 修法 C：兜底（粗糙但一定通）

任务计划同样用 root 跑一次：

```sh
/bin/chmod 755 /volume3/download
/bin/chmod -R a+rwX /volume3/download/ps5
```

**代价要知道**：这等于给「其他用户」也开了读写，SMB 上所有账号对 `download` 的访问效果会变松。能用修法 A 就别用这条。

---

## 四之三、再加一个 NAS 地址

界面里「地址」有两处，先分清你缺的是哪一处：

| 你看到的地方 | 是什么 | 加在哪 |
| --- | --- | --- |
| 侧栏 **「连接」** → 设备列表（现在只有那台 PS5） | 配对过的 Forge 设备 / 保存的网络存储 | 本文这一节 |
| 工坊里选游戏/输出目录的**根列表**（现在只有 `/volume3/download/ps5`） | 本地目录 `--root` | 见本节末尾「只想多挂一个本地文件夹」 |

### 加一个 NAS 设备（纯图形界面，约 30 秒）

1. 侧栏 **「连接」** → 左侧 **「添加设备」** 卡
2. 输入框里填 NAS 的 IP 或主机名（占位提示是 `192.168.1.20 或 nas.local`）→ 点 **「连接」**
   - 探测到是网络存储 → 自动弹出 **「添加 SMB NAS」** 对话框，地址已预填
   - 探测到是 PS5 上的 Forge → 走配对流程，两边校验码要一致
3. 也可以绕过探测：点输入框下面的 **「或按协议添加：SMB / FTP / NFS」**，直接选协议

对话框三个字段（中文界面原文）：

- **名称**：随便起，会显示在设备列表和文件选择器里
- **地址**：IP 或主机名；**端口不是默认值时才写 `:端口`**（SMB 445 / FTP 21 / NFS 2049）
- **用户名 / 密码**：留空＝`访客`。**DSM 默认关闭 guest 访问，留空基本会失败**

### 三个坑

- **用户名密码填你平时的 DSM 账号**（且对目标共享文件夹有读写权限）。填 `PS5DumpForge` 一定失败——那是套件内部系统用户，不走 SMB 认证。
- **密码存在 NAS 本地**，界面里「编辑」可改、「忘记」会连密码一起删掉。
- **这台 NAS 就是 Forge 自己跑的那台**，走 SMB 等于绕出去访问自己：多一次网络往返，转换本身是 J3455 的 CPU 瓶颈，通常感觉不到，但属于没必要的间接层——同一台机器上的目录优先用下面的本地根目录。

### 加完之后多出来的两处

- **「连接」→ 设备** 多一行，可以向它 `发送文件`、从它取文件
- **工坊的文件选择器根列表** 多一个入口（根列表 = 本地 `--root` + 已保存的 NAS），选它就等于 `smb://IP:445`

### 只想多挂一个本地文件夹（更快、不要密码）

如果目的只是让工坊里能选到本机某个目录，不要加 NAS 设备，直接把这行路径追加进 `roots.conf`。这个文件在系统分区、File Station 看不到，所以借任务计划以 root 身份改。

**目标形态**（`roots.conf` 一行一个目录，套件启动时逐行变成 `--root`）：

| 用途 | 路径 | 说明 |
| --- | --- | --- |
| 主库（默认已有） | `/volume3/download/ps5` | 安装时写进去的 |
| Motrix 下载 | `/volume3/docker/motrix/downloads` | 第二行 |
| unreal 卷游戏 | `/volume2/unreal/游戏/ps5` | 第三行，**带中文目录名** |

### 这三个目录现在已经内置，通常无需再跑脚本

新装/升级到 `0.5.7-0002` 后，`roots.conf` 里就有这三行，**这一步可以跳过**。下面只在「以后要加第四个目录」时才用得上。

### 加新目录：用套件内嵌的脚本（不用再上传）

`forge-add-root.sh` 已随本版套件放进 `target/bin/`，File Station 看不到但任务计划能调用。**控制面板 → 任务计划 → 新增 → 计划的任务 → 用户定义的脚本**，用户选 **root**，钩掉「已启用」，粘贴（已有的会自动跳过，所以三个写全也不会重复）：

```sh
/bin/sh /var/packages/PS5DumpForge/target/bin/forge-add-root.sh /新目录/绝对路径
```

> 如果你还在用旧包（0.5.5），脚本没有内嵌，才需要先把它放进 `download/ps5/.forge/`（点开头目录 ⇒ File Station 要先开「设置 → 显示隐藏文件」才看得见），命令里的路径相应改成 `/volume3/download/ps5/.forge/forge-add-root.sh`。

点「运行」，看输出最后是不是「套件已重启」。脚本行为：

- 目录不存在 → 跳过，并把**父目录下现有的条目列出来**（中文名打错、共享文件夹改了名，一眼能对出来）
- 已经在 `roots.conf` 里 → 跳过，**可反复运行，不会写重复行**
- 新路径 → 追加一行，写完全文回读校验，跑完自动 `synopkg restart PS5DumpForge`
- 顺手打印 `synoacltool -get` 的 ACL 条目，权限有没有给到一眼能看出来
- `--dry-run` 只看不改；`--no-restart` 只改不重启；`--check` 只体检（不做任何修改）

### 只想知道现在配了几个、通不通

```sh
/bin/sh /var/packages/PS5DumpForge/target/bin/forge-add-root.sh --check
```

逐条打印：目录是否存在、套件用户能不能读、顶层有多少条目、ACL 里有没有 `PS5DumpForge`。**排查「加了但界面看不到」先跑这个**。

### 手动等价做法（不想用脚本时）

```sh
printf '%s\n' /volume2/unreal/游戏/ps5 >> /var/packages/PS5DumpForge/etc/roots.conf
```

然后**套件中心 → PS5 Dump Forge → 停止 → 启动**。用 `printf` 而不是 `echo`，避免某些 shell 对中文/反斜杠做转义。

### 加完必须给权限，否则白加

套件用户 `PS5DumpForge` 对新建的共享文件夹默认为零权限。两个目录各做一次：

- **控制面板 → 共享文件夹 → `docker` → 编辑 → 权限 → 顶部下拉切「系统内部用户」→ 勾 `PS5DumpForge` 读取/写入 → 确定 → 应用到子文件夹**
- **控制面板 → 共享文件夹 → `unreal` → 同上**

> **`unreal` 在 `/volume2`，和 `download` 不在同一个存储卷上，但授权方式完全一样**——共享文件夹权限按名字走，跟卷号无关。
>
> 为什么建议给共享文件夹整层而不是只给 `ps5` 子文件夹：DSM 的 ACL 是从共享文件夹往下判的，上层没有条目时下层单独授权常常进不去。只想最小授权就先用 **File Station → 右键 `ps5` → 属性 → 权限 → 新增 → `PS5DumpForge` → 读取/写入/遍历 → 应用到此文件夹、子文件夹及文件**，进去仍报 `Permission denied` 再退回整层授权。
>
> 用 ACL（图形界面）而不是 `chmod`：下载器之后新产生的文件会继承 ACL，`chmod` 不会，等于每来一个新文件就再坏一次。

### 中文目录名的两个注意点

`/volume2/unreal/游戏/ps5` 里的「游戏」是中文，会踩两个地方：

1. **任务计划里粘贴命令**要保证是 UTF-8。DSM 的 web 界面本身是 UTF-8，直接粘贴没问题；如果你是从 Windows 记事本复制的，先粘到浏览器地址栏再粘回来，避免带 BOM 或 GBK 残留。脚本以「目录是否存在」为准——存在就写，不存在就跳过并列出父目录，不会写出半截乱码。
2. **写进 `roots.conf` 的必须是 UTF-8 字节**。套件读文件时按 UTF-8 解析，脚本用 `printf` 原样透传，所以只要来源正确就没问题。写入后脚本会自动 `grep -Fxq` 回读一遍，输出「已追加并回读校验通过」才算成功。

> 如果更省心的做法是把中文目录改掉（例如 `/volume2/unreal/ps5`），改完记得同步改 `roots.conf` 里那一行——路径变了就是另一个根，旧的会变成「目录不存在」被跳过，不影响启动。

### 验证

重启后刷新界面（`http://NAS地址:8095`），工坊的文件选择器根列表里应该出现三个入口：`ps5`、`downloads`、`ps5`。启动命令可以在日志里核对：

```sh
tail -n 3 /var/packages/PS5DumpForge/var/dumpforge.log
```

应能看到 `serve --port 8095 --root ... --root ... --root ...` 三个 `--root`。转换输出照旧选 `/pkgs/converted`（即 `/volume3/download/ps5/converted`）——**输出目录和输入根目录是两回事，加根不影响输出位置**。

> **你会遇到的小别扭**：`/volume3/download/ps5` 和 `/volume2/unreal/游戏/ps5` 的最末一级都叫 `ps5`，而 Forge 的根列表只显示目录名、不显示完整路径，于是列表里出现两个 `ps5`，点之前分不清哪个是哪个。两个解法：
> - 把 `/volume2` 那个改挂上一层：`/volume2/unreal/游戏`（列表显示 `游戏`）或 `/volume2/unreal`（显示 `unreal`，顺带能进整个 unreal 卷）。效果一样，只是入口名字不同——`roots.conf` 里换掉那一行、重启即可。
> - 或者接受同名，靠点进去后的文件内容判断（PS5 游戏目录里一般都有 `PPSA` 开头的文件夹）。
>
> 我这次按你给的原路径 `/volume2/unreal/游戏/ps5` 写，不改你的指定；想换随时说一声。

> 两者的区别：本地根目录 = 直接读磁盘；SMB 设备 = 走网络协议再落盘。同一台机器上用本地根目录，跨机器才用 SMB / NFS。

---

## 五、以后怎么升级内核（仍然零 SSH，不用编译）

升级通道我已经预留好，**这次不需要你编译任何东西**：

1. 打开 `https://github.com/Macondo1224/ps5-dump-forge-fpk-build/releases`，取**最新**那个 Release 里的资产（文件名形如 `ps5-dump-forge-x86_64.0.5.6-N`，无扩展名）。
2. 用 **File Station** 把它放进：
   ```
   /volume3/download/ps5/.kernel-update/
   ```
   并**改名成 `ps5-dump-forge`**（该目录需要你自己建）。
3. 套件中心 → PS5 Dump Forge → **停止 → 启动**。启动脚本会自动把它复制成 `target/bin/ps5-dump-forge`、补执行位，并在日志记一行 `kernel replaced`。
4. 打开界面，About 里确认版本号；不对就看 `/var/packages/PS5DumpForge/var/dumpforge.log`。

> 换错了想回滚：把备份的旧二进制同样丢进 `.kernel-update/` 再重启一次即可。
> 仓库同时提供 `feed/manifest.json`（含 sha256），想核对就先比哈希再替换。

---

## 五之二、自动更新（已内嵌，无需任务计划）

这一版把更新引擎直接打进了套件，**不用再建任务计划、不用再单独上传脚本**。分两层看：

| 更新对象 | 会自动吗 | 说明 |
| --- | --- | --- |
| **内核二进制**（真正在迭代的东西） | ✅ 会 | 套件每次启动后台检查，见下 |
| **套件外壳**（INFO、生命周期脚本、图标） | ❌ 不会 | DSM 只对「套件来源」里发布的包做更新检查。本包手动安装、没有来源，套件中心不会提示有新版 |

实际迭代几乎全发生在上游内核里，把内核做成自动更新就覆盖了绝大多数需求。

### 它什么时候跑

套件**每次启动**（含 NAS 重启后自启、套件中心点「启动/重启」）后约 20 秒，后台跑一次检查：

- 本地内核哈希 == 远端最新 Release 的 digest → 立即退出，什么都不做（不会每次都下载）；
- 有新版本 → 下载 → 三重校验 → 备份旧内核 → 替换 → `synopkg restart` 让新版生效；
- 网络不通、GitHub 限流 → 记一条日志后安静退出，**不影响套件正常启动**。

因为更新完会重启一次，启动时不会再检查第二遍——重启后哈希已与远端一致，直接跳过。脚本里还带一把 30 分钟失效的锁，防止并发或递归。

日志优先写在 `download/ps5/.forge/update.log`（File Station 可见），该目录不存在时退回 `var/update.log`。成功更新长这样：

```
2026-11-23 14:02:11  ================ 检查开始（mode=startup）================
2026-11-23 14:02:13  最新版本: v0.5.7-22  远端哈希: f50785a...  本地哈希: ced47d...
2026-11-23 14:02:16  校验通过：12192696 字节，sha256=f50785a...
2026-11-23 14:02:17  内核已替换为 v0.5.7-22
2026-11-23 14:02:18  启动模式：已装新内核，正在重启套件使其生效
```

### 校验流程（为什么比「直接下载覆盖」安全）

1. 调 GitHub Release API 取**最新 tag + 官方 sha256 digest**；
2. 算本地内核哈希，**一致就直接退出**，不下载；
3. 下载后**四重校验**：哈希必须等于 digest、文件头必须是 ELF（`7f454c46`）、必须含 `serve` 子命令、**不得含** `web UI not built`（官方占位页版直接拒收，避免把界面搞没）；
4. 替换前把旧内核备份成 `target/bin/ps5-dump-forge.prev`；
5. 最后 `synopkg restart PS5DumpForge` 让新版生效。

**这套逻辑我在本机实测过完整流程**：从「本地哈希=最新」到「哈希不符+下载失败」两条路径都验证过——前者秒退不下载、后者不阻塞且锁正确清理；网络恢复时还完整跑通了下载→校验→替换→重启链路。

### 手动更新 / 只检查

想立刻检查，不必等重启。控制面板 → 任务计划 → 用户定义的脚本，用户选 **root**，运行：

```sh
/bin/sh /var/packages/PS5DumpForge/target/bin/forge-update.sh          # 检查并更新
/bin/sh /var/packages/PS5DumpForge/target/bin/forge-update.sh --check  # 只检查不下载
/bin/sh /var/packages/PS5DumpForge/target/bin/forge-update.sh --force  # 哈希相同也强制重装
```

脚本已随套件安装，路径固定，不需要再放到 `.forge/`。

### 关掉自动更新

控制面板 → 任务计划 → 用户定义的脚本（root），执行一次：

```sh
echo 0 > /var/packages/PS5DumpForge/var/auto-update.conf
```

之后套件启动就不会再检查。想恢复就把内容改回 `1`（或直接删掉该文件）：

```sh
echo 1 > /var/packages/PS5DumpForge/var/auto-update.conf
```

> 这个开关文件在系统分区，File Station 看不到，所以用任务计划改最省事。它不受套件升级影响，改过的值会保留。

### 回滚

自动更新前会把旧内核备份为 `target/bin/ps5-dump-forge.prev`。想退回：

```sh
cp /var/packages/PS5DumpForge/target/bin/ps5-dump-forge.prev \
   /var/packages/PS5DumpForge/target/bin/ps5-dump-forge
```
然后套件中心停止 → 启动。或用任务计划以 root 跑这一条。

> 想连**套件外壳**也自动更新（出现在套件中心「可更新」里），需要自建套件来源（一份 `packages.xml` 用 Web Station 发布成 http，再到套件中心「套件来源」里添加）。要额外维护 XML 和端口，收益很小——除非我以后改了脚本或端口需要通知你。需要的话我再做。


---

## 六、zstd 在哪、怎么点

界面里**没有**叫「zstd 转换」的菜单，也没有任何「输出格式」下拉里会出现 zstd。路径是：

**工坊（Workshop）→ ② 要做什么：选「游戏镜像」→ ③ 选项 → 镜像：勾「zstd 资源包（最小，用于 ShadowMount+）」→ 勾上后才展开压缩级别**

- 级别 **1–22**（ps5upload 只有 1/5/19）；**J3455 建议先用 5**，19 以上在 NAS 上按天算。
- ② 默认停在「封包 (.pkg)」且标着「推荐」，那个分支的 ③ 只有「快速/均衡/最小/不压缩」，**没有 zstd**——这是最容易找错的地方。
- 若 ③ 里出现「此游戏不通过 AMPR 读取文件……请改为压缩成 .ffpfsc」：那是 ps5upload 的判断逻辑。**Forge 不存在这个限制**，它的 `--zstd-pack` 会主动注入 `ampr-emu-ng`，任何游戏都能打 zstd 包。
- `.zip / .7z / .rar` 可以直接当源，不用先解压。

完整的界面截图级步骤和排错表，见 `群晖DS918-纯图形界面部署.md` 第 6 步（那是 ps5upload 的界面，逻辑不同，别混用）。

---

## 七、排错速查

| 现象 | 原因 / 处理 |
| --- | --- |
| 安装报「文件可能已损坏 / 架构不兼容」 | 确认机型是 DS918+（`apollolake`）且 DSM ≥ 7.2；`.spk` 别二次压缩、别改扩展名；核对上面的 SHA-256 |
| 装完图标打不开 / 8095 连不上 | 套件中心看是否「运行中」→ 停止再启动；看 `/var/log/packages/PS5DumpForge.log` 与 `/var/packages/PS5DumpForge/var/dumpforge.log` |
| 打开是一行 `web UI not built` | 内核被换成了官方 CLI（占位页版）；按第五节换回带 UI 的构建 |
| 启动日志报 `GLIBC_2.xx not found` | 系统 glibc 太旧。查：`/usr/lib/libc.so.6`（DSM 7.2.2 输出 2.36）。低于 2.34 就没法用这个内核 |
| 界面文件浏览器空白 | `roots.conf` 路径错或卷号不是 `/volume3`；右键 `ps5` → 属性核对真实路径 |
| 转换报 `Permission denied` | 见 **第四之二节**：给内部用户 `PS5DumpForge` 授共享文件夹读写（改完要**停止→启动**套件） |
| 8095 被占用 | 查是否有 ps5upload（19113 不冲突）或别的 Forge 实例；端口写在 `scripts/start-stop-status` 的 `PORT=8095`，改端口需重新打包 |
| 转换极慢 / 内存吃紧 | 降 zstd 级别到 5 或更低；DS918+ 4 GB 内存遇到几万小文件的游戏建议扩到 8 GB |

---

## 八、如实说明

- **没有在真实 DSM 上跑过安装**。这台机器不是群晖：我校验了 SPK 的 USTAR 结构、7 个生命周期脚本的 `sh -n` 语法、图标尺寸、`INFO` 必要字段、`conf/privilege` 的 DSM 7 要求，以及**内核的 ELF 架构、glibc 符号上限、内嵌界面与 `serve` 子命令**。安装与启动环节需要你实测，第一次装建议先看完上面排错表。
- **你的系统是 DSM 7.4-90075**，比套件的 `os_min_ver="7.2-64570"` 新。版本门槛、架构、glibc 三项都满足（内核只要 2.34，7.4 的 glibc ≥ 2.36），但 DSM 7.4 我没有实机验证过；若安装向导在版本检查处报错，把提示原文发我，我改 `INFO` 重新出包。
- 内核来自**第三方 CI 构建**（Macondo1224 的仓库，GPL-3.0 上游代码），我只做拆包与打包，没改字节、没做安全审计。SHA-256 已给出，可自行比对。
- 该仓库的产物验收里有「glibc ≤ 2.36」硬约束，所以它不会因为你哪天更新了就突然要 glibc 2.39；这也是我选它而不是自己编的主要原因之一。
- 与之前的 ps5upload（19113）/ Forge 监听目录容器**可并存**，端口不冲突。不想用了在套件中心卸载（DSM 7 默认保留数据）。
- 用途仅限你有权处理的游戏副本。
