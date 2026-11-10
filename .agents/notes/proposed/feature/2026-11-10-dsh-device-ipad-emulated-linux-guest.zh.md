# Agent Note: 基于模拟 Linux 客户机的 iPad DSH 设备

Status: proposed

[English](2026-11-10-dsh-device-ipad-emulated-linux-guest.md) | 中文

## Problem

DSH 作为 Node.js 进程运行在桌面操作系统上，其图形界面是 `dsh web` 通过回环 URL 提供给浏览器的 Web UI。目前没有任何配置能让 DSH 成为便携设备的全部使用体验：用户无法在 iPad 上打开一个 app，不经浏览器和地址栏就看到 DSH UI，说出“做一个计算器”，然后得到一个能用且保留在设备上的计算器。

iPadOS 从四个方面阻止直接移植：

- iOS app 不能 `fork` 或 `exec`。DSH 工具依赖真实子进程：`subprocess-local` 会 spawn 分离的 POSIX 进程组，bash、ripgrep、Python、git 和语言服务器工具都经由它运行。
- iOS 不向第三方 app 提供 hypervisor API，Secure Boot 也阻止在 iPhone 或 iPad 硬件上引导自定义内核。
- 一个 app 打开的回环 TCP 端口可被同一设备上的其他 app 访问，因此在 `127.0.0.1` 上提供 Web UI 及其 `/api` 路由，会暴露 [`client/connection`](../../../../packages/client/connection/README.md) 仅允许回环调用方访问的配置面。
- 模型编写的 UI 只能以[动态包](../../implemented/feature/2026-07-08-self-referential-cordis-toolset.md)的形式存在：保存在进程内存中，只能由定义它的会话控制，DSH 重启后消失，并在 DSH 页面内运行，外层只有一个被该工具集明确说明不构成安全边界的守卫。

## Proposal

把 DSH 作为一个 iOS app 发布到 iPad 上，该 app 包含三部分：绘制全部 UI 的原生外壳、一个 arm64 模拟器引擎，以及运行 DSH Host 和全部工具的 Linux 客户机。客户机没有显示设备。原生外壳在 `WKWebView` 中渲染现有 Web UI，并通过 virtio-vsock 与客户机交换消息，因此设备上不打开任何 TCP 端口，用户也看不到任何 URL。

```text
+------------------------ iOS app (one process) ------------------------+
|                                                                       |
|  Native shell (Swift)                                                 |
|    SwiftUI: home screen of installed apps, settings, lifecycle        |
|    WKWebView: DSH Web UI (bundled frontend, bridge carrier)           |
|    WKWebView per installed app: isolated, no DSH API                  |
|          ^                                                            |
|          | messages over virtio-vsock                                 |
|          v                                                            |
|  Emulator engine (C library, fork of Henge's engine)                  |
|    arm64 -> arm64 JIT, software MMU, PSCI, virtio devices, slirp      |
|      +--------------------- Linux guest (no display) -------------+   |
|      | init (PID 1) -> node + DSH Host (device bundle)            |   |
|      |   tools: bash, python3, git, rg, bwrap, language servers   |   |
|      | /      squashfs system image (read-only)                   |   |
|      | /data  ext4 data image: DSH_HOME, workspaces, apps         |   |
|      +------------------------------------------------------------+   |
+-----------------------------------------------------------------------+
```

### 仓库与许可证划分

引擎 fork 基于 [Henge](https://github.com/Leviidev/Henge) 的提交 `8bde8134e012966c5968c3e71baedcc1985ace59`，其许可证为 GPL-2.0-or-later。因此原生外壳、引擎 fork、客户机内核配置和客户机镜像构建都放在单独的 GPL 仓库中。本仓库只接收 DSH 侧的改动：桥接载体、Host 套接字监听、`device` 组合包和已安装 app。DSH 在客户机中作为独立程序运行，通过 virtio-vsock 与原生外壳通信，不链接任何 GPL 代码。

### 模拟器引擎

Henge 在没有 hypervisor 的情况下于 iPhone 和 iPad 上运行 macOS：一个 arm64 到 arm64 的翻译器原样复制大多数客户机指令，改写 load、store、分支和系统寄存器访问，并实现了软件 MMU 和 QEMU `vmapple` 机型的设备模型。引擎被构建为 C 库，Swift app 通过 `HengeBridge.h` 调用它。它的翻译器已经能运行 V8：Chrome、VS Code 和 Discord 都在 Henge 的 macOS 客户机中运行，客户机代码页变化时调度器会使已翻译的块失效。

fork 保留以下部分：

- `core/`：翻译器、软件 MMU（支持 4 KB、16 KB 和 64 KB 粒度，`core/mmu.c:9`）、异常处理，以及基于 HVC 的 PSCI 1.0，包含 `CPU_ON`、`CPU_OFF`、`SYSTEM_OFF` 和 `SYSTEM_RESET`（`core/exception.c:131`）。
- `devices/`：`gicv3`、`pl011`、`pl031`、`timer`、`pcie`、`virtio`、`virtio_blk` 和 `virtio_net`。
- `host/`：`jit_region`、`disk`（virtio-blk 的 flush 映射为宿主 `fsync`，`host/disk.c:79`）、`net_slirp` 和 `dns_proxy`。

fork 删除 macOS 专用部分：`vmapple`、半虚拟化 GPU（`pvg_*`、`metallib`）、`aes`、`bdif`、`xhci`、`virtio_usb`、`virtio_snd`、`virtio_input`、`third_party/` 下的恢复与安装栈和 `host/install.c`，以及 AVPBooter 启动 ROM 路径。

fork 新增四部分：

- 直接引导内核：加载 arm64 `Image`，为下文的机型生成扁平设备树，并在 `x0` 存放设备树地址后进入 EL1。
- virtio-rng，由 iOS 系统随机源提供数据。缺少它时，Node.js 和 TLS 在启动早期的 `getrandom` 调用可能阻塞。
- virtio-vsock，作为 UI 桥接以及宿主到客户机控制消息的传输通道。
- virtio-console，用于内核与 init 的诊断输出。

### 客户机机型

机型定义固定不变，使同一个客户机镜像既能在 iPad 上的引擎 fork 中启动，也能在为 macOS 构建的同一引擎（`hengevm`）中启动，还能在 Linux 开发主机上通过 `qemu-system-aarch64 -M virt,gic-version=3` 配合 virtio-pci 启动。

| 组件 | 设备 | 用途 |
| --- | --- | --- |
| CPU | arm64，EL0 与 EL1，基于 HVC 的 PSCI | Linux 多核启动、关机、重启 |
| 中断与时间 | GICv3、ARM 通用定时器、PL031 RTC | 调度，以及 TLS 证书校验所需的墙上时钟 |
| 控制台 | PL011、virtio-console | 早期启动日志、init 诊断 |
| 存储 | virtio-blk 系统盘与数据盘 | 只读系统镜像、可写用户数据 |
| 网络 | 经由 libslirp 的 virtio-net | 无需客户机 Wi-Fi 驱动即可向模型提供方发出 HTTPS 请求 |
| 熵源 | virtio-rng | 启动时为客户机随机数池提供种子 |
| 宿主通道 | virtio-vsock | UI 桥接与生命周期控制消息 |

### 客户机内核

客户机内核采用当前的 arm64 Linux LTS 版本，由一个 Kconfig 片段构建，不使用可加载模块。该片段只启用三组功能。

DSH 直接使用的功能，以及使用它们的代码：

| 功能 | 本仓库中的使用方 |
| --- | --- |
| 进程组与信号 | `subprocess-local/src/spawn.ts` 中用 `process.kill(-pid, sig)` 终止进程树 |
| 完整的 procfs，包括 `/proc/<pid>/task/<tid>/syscall`、`/proc/<pid>/mem` 和 `/proc/<pid>/fdinfo` | `subprocess-local/src/process-inspector.ts` 中的终端就绪判断与清理 |
| Unix98 伪终端与 devpts | 经由 `node-pty` 的持久终端会话 |
| user 与 mount 命名空间 | `sandbox-local` 的 bwrap 方案（`--ro-bind / /`、`--proc`、`--dev`、`--tmpfs`、`--bind`） |
| Landlock（ABI 1 至 5）与 `PR_SET_NO_NEW_PRIVS` | landlock-run 后备方案；ABI 5 需要 Linux 6.10 或更高版本 |
| inotify | `settings/settings` 和 `skill/skill-filesystem` 中的文件监视 |
| 共享文件映射 | `session-persistence-sqlite` 中 SQLite WAL 的 `-shm` 文件 |
| 线程、futex、可执行匿名映射 | V8，以及 `workflow-worker-thread` 和 `code-runtime-worker-thread` 中的 `worker_threads` |
| 出站 TCP 与 UDP DNS、epoll、eventfd | 发往模型提供方的 `fetch` 和 libuv |

在设备组合中，DSH 自身不打开任何监听 TCP 套接字，不使用 cgroups、seccomp 或网络命名空间，也不需要 io_uring，因为 libuv 会回退到其线程池。

DSH 运行的工具所期望的功能：带监听套接字的回环网络（测试服务器和部分语言服务器）、回环上的 IPv6（`localhost` 可能解析为 `::1`）、PID、IPC 和 UTS 命名空间以及 seccomp（非特权 bwrap 变体和工作区内的工具）、SysV IPC 与 POSIX 共享内存，以及 user 命名空间映射所需的 uid 与 gid 模型。

机型与存储布局所需的功能：GICv3、通用定时器、PL011、PL031、通用 PCIe 主机、带 blk、net、console、rng 和 vsock 驱动的 virtio-pci、ext4、squashfs、overlayfs、tmpfs、devtmpfs，以及共享文件夹所需的 FUSE。

其余功能全部关闭，包括 USB、声音、DRM 与帧缓冲、输入与 HID、蓝牙、无线、NVMe、SCSI、MMC、ACPI 与 UEFI、KVM、kexec、挂起与休眠、CPU 与内存热插拔、NUMA、交换分区、eBPF、netfilter、网桥、隧道、IPsec、SELinux、AppArmor、IMA、审计，以及所有其他文件系统。发布构建还会关闭 ftrace、perf、kprobes 和 debugfs。

### 客户机存储

| 路径 | 后备存储 | 内容 |
| --- | --- | --- |
| `/` | `system.img`，squashfs，只读 | Debian slim arm64、官方 Node.js linux-arm64 构建、DSH、bash、python3、git、ripgrep、bubblewrap |
| `/data` | `data.img`，ext4 | `DSH_HOME`、SQLite 会话存储、工作区、已安装 app、包缓存，以及存放用户自行安装软件包的 overlayfs 上层目录 |
| `/tmp` | tmpfs | 临时文件 |
| `/mnt/shared` | 用 davfs2 挂载的引擎 WebDAV 共享 | 与 iPadOS“文件”app 交换文件 |

app 更新替换 `system.img`，不触碰 `data.img`。工作区和 `DSH_HOME` 必须留在 ext4 上，因为 inotify 和 SQLite WAL 的共享映射在 WebDAV 挂载上无法工作；共享文件夹只用于导入导出文件。

iOS 可以随时终止被挂起的 app。客户机已经 flush 的数据会通过 `fsync` 写入宿主文件，在终止后保留；仍在客户机页缓存中的数据则会丢失。客户机以较短的 ext4 提交间隔挂载 `/data`，会话存储保持默认的 WAL 模式，原生外壳在 app 进入后台时通过 vsock 请求客户机执行 `sync`。

### 客户机 init

DSH 不作为 PID 1 运行。一个用 C 或 Rust 编写的小型 init 执行以下步骤：

1. 挂载 procfs、sysfs、devtmpfs、devpts、tmpfs、`/data` 和 overlay。
2. 配置静态 slirp 网络：地址 `10.0.2.15`，网关 `10.0.2.2`，DNS `10.0.2.3`。
3. 根据 PL031 RTC 设置系统时钟。
4. 以 `device` 组合包启动 DSH Host，并在其退出时重新启动。
5. 回收成为孤儿进程的工具进程。
6. 处理 vsock 控制消息：进入后台时执行 `sync`，恢复时重新同步时钟，以及关机。

### DSH UI 传输

原生外壳通过自定义 URL scheme 处理器在 `WKWebView` 中加载随 app 打包的 Web UI 前端。DSH 侧需要两处改动把它连接到客户机中的 Host：

- Host web 服务器目前只监听 `port` 和 `host`（`packages/host/webserver/src/index.ts:218`）。一个经过校验的 `Config` 字段增加 Unix 套接字监听，客户机中的转发进程把该套接字连接到一个 vsock 端口。scheme 处理器通过 vsock 转发静态资源、`/plugins` 包和一元 `/api` 调用的 HTTP 请求。
- `client/connection` 已经在同一个双流抽象之下区分浏览器载体（HTTP POST 上行、WebSocket 下行）和进程内载体。第三种载体通过 `WKScriptMessageHandler` 消息承载两条流，因为自定义 URL scheme 无法处理 WebSocket 升级。

`/api` 路由把特权方法（`host.pickDirectory`、`host.openPath`、设置与凭据面，以及 agent preset 编写）限制为回环调用方。桥接载体需要 Host 给出明确的信任分类：从 vsock 监听到达的请求来自同一设备上的原生外壳，因此获得回环信任级别，其他监听均不授予该级别。

### 已安装 app

现有动态包路径保持不变，用于修改 DSH UI 本身：它在 DSH 页面内运行，需要审批，并由[动态插件运行体系提案](../architecture/2026-08-08-cordis-web-dynamic-packages.md)管辖。

app 是另一种独立类型：

- 模型把一个自包含的 web app（HTML、JavaScript、CSS，以及包含名称和图标的 manifest）写入会话工作区，然后调用新的 `app_install` 工具。该工具校验 manifest，把文件复制到 `/data/apps/<app-id>/`，并把安装记录写入会话日志，因此每个已安装 app 都能从日志重建。
- 原生外壳通过一个 Host API 方法在主屏幕上列出已安装 app，并在各自独立的 `WKWebView` 中以每个 app 专属的数据存储打开它们。
- app 无法访问 DSH 客户端运行时或 API。内容安全策略默认阻止网络访问。需要运行工具的 app（例如 Python 计算）通过一个范围很窄的消息处理器发出请求，Host 把它转为工具调用，并经过常规权限流程。
- `app_uninstall` 删除文件并记录这次删除。

### device 组合包

`packages/bundle/` 中的 `device` 组合包是叠加在 `web-app` 之上的补丁层。它启用 Unix 套接字监听和桥接载体，保留 `subprocess-local` 与 `sandbox-local`，加入 app 工具，并关闭在设备上没有作用的插件：`pwsh-*` 提供方、`sandbox-windows-acl`、seatbelt 配置、`directory-picker-native`，以及在用户未选择远程执行时的 e2b 提供方。

### 阶段

1. 测量：在运行原版 Henge 和 macOS 客户机的 iPad 上执行 `npx @deepseek-ai/dsh web`，并对 `git status`、`rg` 和 `python3 -c pass` 各计时 100 次。
2. 客户机：构建内核片段、init 和两个磁盘镜像，在 Linux 上的 QEMU `virt` 中以 `device` 组合包和 Unix 套接字监听启动它们。
3. 引擎：为 fork 加入直接引导内核、virtio-rng、virtio-vsock 和 virtio-console，并在 macOS 上的 `hengevm` 中启动同一个客户机。
4. 外壳：构建带 Web UI 视图、桥接载体和生命周期消息的 iOS app。
5. app：加入 `app_install`、`app_uninstall`、app 列表 API、主屏幕和隔离的 app 视图。

## Alternatives considered

**在客户机控制台上运行终端 UI。** DSH 只发布 `base`、`headless` 和 `web-app` 三个组合包；终端 UI 需要新增一个组合包。Linux 帧缓冲控制台无法渲染韩文或其他 CJK 文本，通过输入法输入文本也需要客户机侧支持。Web UI 已经实现了对话、审批、计划和工具渲染。

**使用回环 URL 的浏览器。** 它不需要改动 DSH，但会显示浏览器和地址栏，而且在 iOS 上其他 app 能访问该回环端口和特权 `/api` 方法。

**Darwin（XNU）客户机。** Node.js 支持 darwin-arm64，因此无需移植 V8 和 libuv。但开源 Darwin 用户态不完整，面向虚拟或第三方硬件的 IOKit 驱动是闭源的，而在 Android 硬件上引导 XNU 的项目（ios-sweet、HackiOS、UniXNU）运行的是 Apple 固件中的用户态，本项目无法再分发。

**DSH 专用内核或 unikernel。** 单地址空间内核没有 `fork` 或 `exec`，因此 bash、Python 和 git 无法运行，而且需要移植 V8 和 libuv。之后仍可以通过 subprocess 与文件系统的 Service Provider 拆分为 DSH 专用内核和单独的 Linux 工具虚拟机，方式与 `subprocess-e2b` 和 `fs-e2b` 把工具放进远程沙箱相同，代价是两个被模拟的内核以及 V8 移植。

**WASI subprocess 提供方。** WASI 不能创建进程，因此 shell 脚本无法启动其他程序，git、原生编译器和大多数语言服务器也没有可用的 WASI 构建。

**仅使用远程执行。** e2b 提供方已经能在远程沙箱中运行工具。只使用它们会让每次工具调用都依赖网络和托管服务，设备无法独立工作。`device` 组合包保留它们作为可选项。

**客户机侧图形渲染。** 在客户机内渲染 UI 需要 GPU 设备，并且要在模拟下运行浏览器引擎；Henge 的半虚拟化 GPU 路径专用于 macOS 客户机。

**用 QEMU 代替 Henge 引擎。** Husk 在 iOS 上用 QEMU 运行 LineageOS 客户机，QEMU 也已经提供 `virt` 机型、直接引导内核、virtio-vsock、virtio-rng 和 virtio-console。它的 TCG 翻译器面向任意指令集，而 Henge 的翻译器原样复制大多数同指令集指令，预计运行更快。第 3 阶段用第 1 阶段的基准测试验证这一点；如果测得的差距不足以支撑对引擎的四项新增，就改用 QEMU。

**把动态包持久化为 app。** 动态包的浏览器侧在 DSH 页面内运行，外层守卫并不构成安全边界。把这种权限级别的模型编写代码跨重启持久化，会扩大一次审批所授予的范围，因此 app 改为在隔离视图中运行。

## Acceptance criteria

- 在搭载 A14 或 M1、运行 iPadOS 17 或更高版本的 iPad 上，打开 app 即显示 DSH UI，不出现浏览器和可见 URL，设备上也没有监听中的 TCP 套接字。
- 提示“做一个计算器”会产生一个已安装 app，它出现在主屏幕上，在 app 被终止并重新启动后依然存在，并在无法访问 DSH API 的隔离视图中运行。
- bash、python3、git 和 ripgrep 工具调用在客户机中成功执行，沙箱报告 bwrap 或 Landlock 后备方案处于生效状态。
- app 进入后台前写入的会话数据在 iOS 终止 app 后仍然保留。
- app 从超过一小时的挂起中恢复后，模型请求能够成功。
- 记录第 1 阶段的延迟测量结果，以及在引擎 fork 上的同样测量结果。

## Risks

- **许可证。** 原生外壳和引擎采用 GPL-2.0-or-later，无法通过 App Store 分发。在客户机中把 MIT 许可的 DSH 作为独立程序运行，通常不被视为构成合并作品，但分发前需要法律审查。
- **分发。** 用户必须侧载 app，并通过 StikDebug、TrollStore 或类似工具启用 JIT，而 JIT 的可用性会随 iOS 版本变化。
- **性能。** 所有客户机代码都在翻译下运行。agent（智能体）会启动大量短生命周期进程，每个进程都要为新代码页付出翻译开销；在第 1 阶段之前无法知道这项开销有多大。
- **上游成熟度。** Henge 0.1 发布于 2026-11-06；其 README 称 iPad 支持尚未在真实 iPad 上运行过，而 0.2 发布说明报告了贡献者在 iPad 上的测试。
- **内存。** iOS 限制 app 内存；4 至 8 GB 的客户机内存需要 increased-memory-limit 授权。
- **生成代码。** 已安装 app 是保留在设备上的模型编写代码。隔离依赖于每个 app 独立的 web 视图、内容安全策略，以及工具请求的权限流程。
- **时钟。** iOS 挂起 app 期间客户机时钟停止；如果错过恢复消息，TLS 证书校验会一直失败，直到下一次重新同步。
