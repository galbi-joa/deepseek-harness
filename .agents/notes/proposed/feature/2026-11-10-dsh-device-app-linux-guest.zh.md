# Agent Note: 围绕 Linux 客户机的 iPad 与 Mac DSH 设备 app

Status: proposed

[English](2026-11-10-dsh-device-app-linux-guest.md) | 中文

## Problem

DSH 作为 Node.js 进程运行在桌面操作系统上，其图形界面是 `dsh web` 通过回环 URL 提供给浏览器的 Web UI。目前没有任何配置能让 DSH 成为 iPad 或 Mac 上某个 app 的全部使用体验：用户无法打开一个 app，不经浏览器和地址栏就看到 DSH UI，说出“做一个计算器”，然后得到一个能用且保留在设备上的计算器。

有四项限制阻碍这一点：

- iOS app 不能 `fork` 或 `exec`。DSH 工具依赖真实子进程：`subprocess-local` 会 spawn 分离的 POSIX 进程组，bash、ripgrep、Python、git 和语言服务器工具都经由它运行。
- iOS 不向第三方 app 提供 hypervisor API，Secure Boot 也阻止在 iPhone 或 iPad 硬件上引导自定义内核。
- 一个 app 打开的回环 TCP 端口可被同一设备上的其他 app 访问，因此在 `127.0.0.1` 上提供 Web UI 及其 `/api` 路由，会暴露 [`client/connection`](../../../../packages/client/connection/README.md) 仅允许回环调用方访问的配置面。
- 模型编写的 UI 只能以[动态包](../../implemented/feature/2026-07-08-self-referential-cordis-toolset.md)的形式存在：保存在进程内存中，只能由定义它的会话控制，DSH 重启后消失，并在 DSH 页面内运行，外层只有一个被该工具集明确说明不构成安全边界的守卫。

macOS 允许子进程和硬件虚拟化，但直接在 macOS 上运行 DSH 时，工具面对的操作系统与 iPad 不同，隔离也只有 Seatbelt 的写入限制。

## Proposal

构建一个面向 iOS、iPadOS 和 macOS 的 SwiftUI app。app 以原生方式绘制全部 UI，并在 `WKWebView` 中渲染现有 Web UI。DSH 与全部工具运行在没有显示设备的 Linux 客户机中；同一个客户机镜像运行在两种执行后端上。在 iOS 和 iPadOS 上，由本项目编写的 arm64 到 arm64 JIT 引擎 `dsh-vm` 运行客户机。在 macOS 上，由 Virtualization.framework 以硬件虚拟化运行客户机，可选的原生模式则把 DSH 作为本地进程运行而不使用虚拟机。UI 与客户机通过 virtio-vsock 交换消息，因此不打开任何 TCP 端口，用户也看不到任何 URL。app 在 App Store 之外分发。

```text
DSH app (one SwiftUI codebase for iOS, iPadOS, and macOS)
|
+-- Native shell
|     SwiftUI: home screen of installed apps, settings, lifecycle
|     WKWebView: DSH Web UI (bundled frontend, bridge carrier)
|     WKWebView per installed app: isolated, no DSH API
|
+-- Bridge: vsock to the guest, or a Unix socket in macOS native mode
|
+-- Execution backend
      iOS and iPadOS:  dsh-vm, an arm64-to-arm64 JIT engine  --+
      macOS (default): Virtualization.framework              --+--> Linux guest
      macOS (option):  DSH as a local process, no VM

Linux guest (no display, same image on both backends)
  init (PID 1) -> node + DSH Host (device bundle)
  tools: bash, python3, git, rg, bwrap, language servers
  /      squashfs system image (read-only)
  /data  ext4 data image: DSH_HOME, workspaces, installed apps
```

### 仓库布局

所有部分都为本项目编写并采用 MIT 许可证，因此都放在本仓库中：

| 路径 | 内容 |
| --- | --- |
| `native/dsh-vm/` | 用 Rust 编写的 iOS 执行引擎，附带用于测试的 macOS 命令行构建 |
| `native/dsh-guest/` | 客户机内核 Kconfig 片段、init，以及构建 `system.img` 的脚本 |
| `apps/device/` | 面向 iOS、iPadOS 和 macOS 的 SwiftUI app |
| `packages/` | DSH 侧改动：桥接载体、Host 套接字监听、`device` 组合包和已安装 app |

### 硬件访问

客户机没有显示、输入、GPU 或 USB 控制器设备。硬件归宿主操作系统所有，客户机只接收文件、套接字或直通的 USB 设备。

| 能力 | iOS 与 iPadOS | macOS |
| --- | --- | --- |
| 触摸、触控板、键盘、输入法 | 由 `WKWebView` 处理 | 由 `WKWebView` 处理 |
| UI 绘制与已安装 app 的图形 | WebKit 使用宿主 GPU | WebKit 使用宿主 GPU |
| 已安装 app 使用摄像头和麦克风 | `WKWebView` 媒体采集权限 | `WKWebView` 媒体采集权限 |
| USB 存储设备上的文件 | 由“文件”app 挂载，经 9p 共享给客户机 | 挂载在 `/Volumes` 下，经 virtiofs 共享；原生模式下直接读取 |
| 其他 USB 设备 | 不支持：iPadOS 要求 DriverKit 驱动以及按厂商 ID 授予的授权 | macOS 27 或更高版本上通过 `VZUSBPassthroughDeviceConfiguration` 直通；原生模式下直接访问 |
| 工具使用 GPU 计算 | 不支持 | 仅限原生模式（Metal） |

### dsh-vm 引擎

`dsh-vm` 在没有 hypervisor API 的 iOS 和 iPadOS 上运行客户机。它的实现只依据公开规范：Arm 架构参考手册（A64、VMSAv8-64、通用定时器）、GICv3 架构规范、virtio 1.2 规范、PSCI 1.0、Linux arm64 引导协议，以及设备树规范。QEMU 只作为差分测试的参照，从不链接或复制。[Henge](https://github.com/Leviidev/Henge) 证明了 iOS 上的同指令集翻译能够运行执行 V8 的客户机；本项目不使用它的源码。

由于本项目同时构建客户机内核，引擎只支持客户机需要的部分：

- CPU：EL0 与 EL1。ID 寄存器不声明指针认证、BTI、MTE、SVE 或 SME，因此客户机内核和 glibc 永远不会发出这些指令。
- 翻译：数据处理指令原样复制；load、store、原子操作和独占访问经过软件 MMU；分支和 PC 相对指令被改写；系统寄存器和异常调用辅助函数；写入已翻译的代码页会使其对应的块失效。
- MMU：VMSAv8-64 第 1 阶段，只有一种 16 KB 粒度，虚拟地址 48 位。
- 首个版本只有一个 vCPU。翻译器改写的指令全部通过差分测试后再支持 SMP。
- 设备：GICv3、通用定时器、PL011、PL031，以及带 blk、net、rng、vsock、console 和 9p 设备的 virtio-mmio 传输。
- 引导：加载内核 `Image`，生成扁平设备树，并在 `x0` 存放其地址后进入 EL1。基于 HVC 的 PSCI 提供 `SYSTEM_OFF`、`SYSTEM_RESET`，以及之后的 `CPU_ON`。
- 宿主服务：客户机内存使用一个映射；进程启用 JIT 后，JIT 区域使用分离的可写映射和可执行映射；磁盘文件的 virtio-blk flush 映射为 `fsync`；基于 lwIP（BSD-3-Clause）的用户态 TCP/IP 栈通过宿主操作系统打开真实套接字；vsock 端点交给 Swift 外壳；共享文件夹使用 9P2000.L 服务器。

实现语言选用 Rust，因为 virtio 队列处理需要解析由客户机控制的描述符；unsafe 代码只限于代码生成和内存映射模块。

### macOS 后端

macOS app 默认用 Virtualization.framework（`VZLinuxBootLoader`）启动同一个内核和镜像：两块磁盘使用 `VZVirtioBlockDeviceConfiguration`，网络使用 `VZVirtioNetworkDeviceConfiguration` 加 `VZNATNetworkDeviceAttachment`，另有 `VZVirtioEntropyDeviceConfiguration`、用于桥接的 `VZVirtioSocketDeviceConfiguration`、`VZVirtioConsoleDeviceConfiguration`，以及用于共享文件夹的 `VZVirtioFileSystemDeviceConfiguration`。在 macOS 27 及更高版本上，`VZXHCIControllerConfiguration` 配合 `VZUSBPassthroughDeviceConfiguration` 可把宿主 USB 设备直通给客户机。app 带有 `com.apple.security.virtualization` 授权。

原生模式是一项设置。app 以同一个 `device` 组合包把 DSH Host 作为本地进程启动；`sandbox-local` 选择其 macOS Seatbelt 方案，工具可直接访问 USB 设备、串口和 Metal。

### 客户机机型

内核启用两种后端所呈现设备的驱动，因此同一个客户机镜像既能在 `dsh-vm` 下启动，也能在 Virtualization.framework 下启动，还能在 Linux 开发主机上通过 `qemu-system-aarch64 -M virt,gic-version=3` 启动。

| 组件 | dsh-vm（iOS 与 iPadOS） | Virtualization.framework（macOS） |
| --- | --- | --- |
| CPU | arm64 EL0 与 EL1，初期一个 vCPU，基于 HVC 的 PSCI | 硬件 vCPU |
| 中断与时间 | GICv3、通用定时器、PL031 RTC | 由平台提供；设备清单在第 2 阶段确认 |
| 控制台 | PL011、virtio-console | virtio-console |
| 存储 | 经 virtio-mmio 的 virtio-blk | 经 virtio-pci 的 virtio-blk |
| 网络 | 连接引擎 lwIP 栈的 virtio-net | 带 NAT 的 virtio-net |
| 熵源 | 由 iOS 系统随机源提供数据的 virtio-rng | virtio-rng |
| 宿主通道 | virtio-vsock | virtio-vsock |
| 共享文件夹 | 9p | virtiofs |
| USB | 无 | macOS 27 或更高版本上的 USB 直通 |

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

后端与存储布局所需的功能：GICv3、通用定时器、PL011、PL031、通用 PCIe 主机、带 blk、net、console、rng 和 vsock 驱动的 virtio-mmio 与 virtio-pci、9p 与 virtiofs、带 DHCP 的内核 IP 自动配置、ext4、squashfs、overlayfs、tmpfs 和 devtmpfs。只有第 2 阶段证明 Virtualization.framework 平台需要 ACPI 时才保留 ACPI。

其余功能全部关闭，包括 USB、声音、DRM 与帧缓冲、输入与 HID、蓝牙、无线、NVMe、SCSI、MMC、UEFI、KVM、kexec、挂起与休眠、CPU 与内存热插拔、NUMA、交换分区、eBPF、netfilter、网桥、隧道、IPsec、SELinux、AppArmor、IMA、审计，以及所有其他文件系统。发布构建还会关闭 ftrace、perf、kprobes 和 debugfs。在 macOS 上直通的宿主 USB 设备需要该设备对应的 USB 主机驱动和类驱动，片段会在该功能发布时加入它们。

### 客户机存储

| 路径 | 后备存储 | 内容 |
| --- | --- | --- |
| `/` | `system.img`，squashfs，只读，随 app 一起发布 | Debian slim arm64、官方 Node.js linux-arm64 构建、DSH、bash、python3、git、ripgrep、bubblewrap |
| `/data` | `data.img`，ext4，首次启动时创建 | `DSH_HOME`、SQLite 会话存储、工作区、已安装 app、包缓存，以及存放用户自行安装软件包的 overlayfs 上层目录 |
| `/tmp` | tmpfs | 临时文件 |
| `/mnt/shared` | iOS 与 iPadOS 上为 9p，macOS 上为 virtiofs | 用户从“文件”app、Mac 文件夹或 USB 存储设备共享的文件 |

app 更新替换 `system.img`，不触碰 `data.img`。工作区和 `DSH_HOME` 必须留在 ext4 上，因为共享挂载不会通过 inotify 报告宿主侧的变化，也不能可靠支持 SQLite WAL 使用的共享映射；共享文件夹只用于导入导出文件。

iOS 可以随时终止被挂起的 app。客户机已经 flush 的数据会通过 `fsync` 写入宿主文件，在终止后保留；仍在客户机页缓存中的数据则会丢失。客户机以较短的 ext4 提交间隔挂载 `/data`，会话存储保持默认的 WAL 模式，原生外壳在 app 进入后台时通过 vsock 请求客户机执行 `sync`。

### 客户机 init

DSH 不作为 PID 1 运行。一个用 Rust 编写的小型 init 执行以下步骤：

1. 挂载 procfs、sysfs、devtmpfs、devpts、tmpfs、`/data`、overlay 和共享文件夹。
2. 配置网络：使用 `dsh-vm` 网络栈预期的静态地址，或使用内核 IP 自动配置从 Virtualization.framework NAT 获得的地址。
3. 根据 RTC 设置系统时钟。
4. 以 `device` 组合包启动 DSH Host，并在其退出时重新启动。
5. 回收成为孤儿进程的工具进程。
6. 处理 vsock 控制消息：iOS 把 app 移入后台时执行 `sync`，恢复或宿主唤醒时重新同步时钟，以及关机。

### DSH UI 传输

原生外壳通过自定义 URL scheme 处理器在 `WKWebView` 中加载随 app 打包的 Web UI 前端。DSH 侧需要两处改动把它连接到 Host：

- Host web 服务器目前只监听 `port` 和 `host`（`packages/host/webserver/src/index.ts:218`）。一个经过校验的 `Config` 字段增加 Unix 套接字监听。在客户机中，转发进程把该套接字连接到一个 vsock 端口；在 macOS 原生模式下，外壳直接连接该套接字。scheme 处理器转发静态资源、`/plugins` 包和一元 `/api` 调用的 HTTP 请求。
- `client/connection` 已经在同一个双流抽象之下区分浏览器载体（HTTP POST 上行、WebSocket 下行）和进程内载体。第三种载体通过 `WKScriptMessageHandler` 消息承载两条流，因为自定义 URL scheme 无法处理 WebSocket 升级。

`/api` 路由把特权方法（`host.pickDirectory`、`host.openPath`、设置与凭据面，以及 agent preset 编写）限制为回环调用方。桥接载体需要 Host 给出明确的信任分类：从 Unix 套接字监听到达的请求来自原生外壳，因此获得回环信任级别，其他监听均不授予该级别。在 macOS 原生模式下，该套接字位于 app 容器中，权限为 `0600`；同一用户的其他进程仍然可以连接它，这与当前回环监听的暴露范围相同。

### 已安装 app

现有动态包路径保持不变，用于修改 DSH UI 本身：它在 DSH 页面内运行，需要审批，并由[动态插件运行体系提案](../architecture/2026-08-08-cordis-web-dynamic-packages.md)管辖。

app 是另一种独立类型：

- 模型把一个自包含的 web app（HTML、JavaScript、CSS，以及包含名称和图标的 manifest）写入会话工作区，然后调用新的 `app_install` 工具。该工具校验 manifest，把文件复制到 `/data/apps/<app-id>/`，并把安装记录写入会话日志，因此每个已安装 app 都能从日志重建。
- 原生外壳通过一个 Host API 方法在主屏幕上列出已安装 app，并在各自独立的 `WKWebView` 中以每个 app 专属的数据存储打开它们。
- app 无法访问 DSH 客户端运行时或 API。内容安全策略默认阻止网络访问。需要运行工具的 app（例如 Python 计算）通过一个范围很窄的消息处理器发出请求，Host 把它转为工具调用，并经过常规权限流程。
- `app_uninstall` 删除文件并记录这次删除。

### device 组合包

`packages/bundle/` 中的 `device` 组合包是叠加在 `web-app` 之上的补丁层。它启用 Unix 套接字监听和桥接载体，保留 `subprocess-local` 与 `sandbox-local`，加入 app 工具，并关闭在 app 中没有作用的插件：`pwsh-*` 提供方、`sandbox-windows-acl`、`directory-picker-native`，以及在用户未选择远程执行时的 e2b 提供方。客户机和 macOS 原生模式使用同一个组合包；`sandbox-local` 在客户机中选择 bwrap 和 Landlock，在 macOS 上选择 Seatbelt。

### 分发

app 在 App Store 之外分发，使用付费 Apple Developer Program 会员资格：

- macOS：使用 Developer ID 签名并经过公证，用户直接下载。
- iOS 与 iPadOS：使用会员资格的描述文件签名（有效期一年）并侧载。JIT 仍需通过 StikDebug 或类似工具附加调试器，因为会员资格不会给 iOS app 授予 JIT 授权。会员资格提供用于客户机内存的 increased-memory-limit 授权。

### 阶段

1. 测量：在现有的 iOS 模拟器 app（原版 Henge 加 macOS 客户机）中运行 `npx @deepseek-ai/dsh web`，并对 `git status`、`rg` 和 `python3 -c pass` 各计时 100 次，以估算工具密集型工作的翻译开销。
2. 客户机：构建内核片段、init 和两个镜像，在 Linux 上的 QEMU `virt` 和 macOS 上的 Virtualization.framework 中以 `device` 组合包和 Unix 套接字监听启动它们。
3. macOS app：构建 SwiftUI 外壳、Web UI 视图、桥接载体、Virtualization.framework 后端、原生模式，以及 macOS 27 上的 USB 直通。
4. 已安装 app：加入 `app_install`、`app_uninstall`、app 列表 API、主屏幕和隔离的 app 视图。
5. 引擎：在 Apple Silicon 上把 `dsh-vm` 构建为 macOS 命令行程序，通过与 QEMU 的差分测试，并以一个 vCPU 启动客户机。
6. iOS app：嵌入 `dsh-vm`，检测 JIT 是否可用，加入生命周期消息，然后启用 SMP。

## Alternatives considered

**在客户机控制台上运行终端 UI。** DSH 只发布 `base`、`headless` 和 `web-app` 三个组合包；终端 UI 需要新增一个组合包。Linux 帧缓冲控制台无法渲染韩文或其他 CJK 文本，通过输入法输入文本也需要客户机侧支持。Web UI 已经实现了对话、审批、计划和工具渲染。

**使用回环 URL 的浏览器。** 它不需要改动 DSH，但会显示浏览器和地址栏，而且在 iOS 上其他 app 能访问该回环端口和特权 `/api` 方法。

**Darwin（XNU）客户机。** Node.js 支持 darwin-arm64，因此无需移植 V8 和 libuv。但开源 Darwin 用户态不完整，面向虚拟或第三方硬件的 IOKit 驱动是闭源的，而在 Android 硬件上引导 XNU 的项目（ios-sweet、HackiOS、UniXNU）运行的是 Apple 固件中的用户态，本项目无法再分发。

**DSH 专用内核或 unikernel。** 单地址空间内核没有 `fork` 或 `exec`，因此 bash、Python 和 git 无法运行，而且需要移植 V8 和 libuv。之后仍可以通过 subprocess 与文件系统的 Service Provider 拆分为 DSH 专用内核和单独的 Linux 工具虚拟机，方式与 `subprocess-e2b` 和 `fs-e2b` 把工具放进远程沙箱相同，代价是两个内核以及 V8 移植。

**在 app 内实现 Linux 系统调用层的用户态模拟。** 只翻译客户机用户态代码，并在 app 内实现 Linux 系统调用、进程创建、信号和伪终端，是 iSH 在 iOS 上采用的做法。这相当于编写一个兼容 Linux 的内核，而且存在已知的兼容性缺口；运行真实 Linux 内核只需模拟少量设备，bwrap、Landlock 和伪终端也能保持真实行为。

**WASI subprocess 提供方。** WASI 不能创建进程，因此 shell 脚本无法启动其他程序，git、原生编译器和大多数语言服务器也没有可用的 WASI 构建。

**仅使用远程执行。** e2b 提供方已经能在远程沙箱中运行工具。只使用它们会让每次工具调用都依赖网络和托管服务，app 无法独立工作。`device` 组合包保留它们作为可选项。

**fork Henge 的引擎。** Henge 的引擎采用 GPL-2.0-or-later，会迫使 app 放进单独的 GPL 仓库。它还包含约 7,700 行 macOS 专用设备代码（半虚拟化 GPU、USB 控制器、触控板、AES、`vmapple`），并处理指针认证指令、三种页面粒度和 Apple 系统寄存器，而本客户机都用不到，因为 Henge 必须运行未经修改的 macOS，而本项目可以自行选择客户机内核。

**在 iOS 上使用 QEMU。** Husk 在 iOS 上用 QEMU 运行 LineageOS 客户机，QEMU 也已经提供客户机需要的全部设备。但 QEMU 采用 GPL-2.0-or-later，其 TCG 翻译器面向任意指令集，而同指令集翻译可以原样复制大多数指令。QEMU 仍作为 `dsh-vm` 差分测试的参照。

**在 macOS 上也使用 dsh-vm。** macOS app 可以使用 Virtualization.framework，它以硬件速度运行客户机，并已提供 virtio 设备、共享文件夹和 USB 直通，因此只有在没有 hypervisor API 的平台上才需要该引擎。

**把原生模式作为 macOS 唯一后端。** 它让工具在所有 macOS 版本上都能直接访问硬件，但工具运行的操作系统会与 iPad 版本不同，隔离也从虚拟机降为 Seatbelt 的写入限制。原生模式作为一项设置保留。

**在客户机中模拟显示、输入、GPU 和 USB 控制器设备。** 自己绘制 UI 的客户机需要这些设备，Henge 的 macOS 客户机就是如此。本客户机不绘制任何内容；`WKWebView` 在宿主上处理输入和绘制，USB 设备只能通过共享文件夹或 Virtualization.framework 直通进入客户机。

**把动态包持久化为 app。** 动态包的浏览器侧在 DSH 页面内运行，外层守卫并不构成安全边界。把这种权限级别的模型编写代码跨重启持久化，会扩大一次审批所授予的范围，因此 app 改为在隔离视图中运行。

## Acceptance criteria

- 在 macOS 上，打开 app 会在 Virtualization.framework 下启动客户机并显示 DSH UI，不出现浏览器和可见 URL，也没有监听中的 TCP 套接字；原生模式运行同一个 UI，工具调用在 Seatbelt 下执行。
- 在 macOS 27 上，用户连接给客户机的 USB 设备出现在客户机的设备列表中。
- 在搭载 A14 或 M1、运行 iPadOS 17 或更高版本的 iPad 上，打开 app 会在 `dsh-vm` 下启动客户机，并在相同条件下显示同一个 UI。
- 提示“做一个计算器”会产生一个已安装 app，它出现在主屏幕上，在 app 被终止并重新启动后依然存在，并在无法访问 DSH API 的隔离视图中运行。
- bash、python3、git 和 ripgrep 工具调用在客户机中成功执行，沙箱报告 bwrap 或 Landlock 后备方案处于生效状态。
- app 进入后台前写入的会话数据在 iOS 终止 app 后仍然保留。
- app 从超过一小时的挂起中恢复后，模型请求能够成功。
- `dsh-vm` 对其改写的每一类指令都通过与 QEMU 的差分测试，并记录第 1 阶段的延迟测量结果以及在 `dsh-vm` 上的同样测量结果。

## Risks

- **引擎工作量。** 主要难点集中在翻译器和软件 MMU：异常与中断时序、代码写入后的失效处理，以及性能调优。从一个 vCPU 开始可以推迟跨 CPU 的 TLB 失效和竞争下的独占访问。
- **代码来源。** 本设计参考了 Henge 的文档和部分源码。实现者应只依据公开规范编写 `dsh-vm`，不查阅 Henge 或 QEMU 的源码，并在每个模块中记录所实现的规范章节。
- **iOS 分发。** 用户需要基于调试器的 JIT 工具，而 JIT 的可用性会随 iOS 版本变化。使用付费会员资格安装到他人设备时需要登记设备标识，每个会员年度每类设备上限为 100 台；用免费 Apple ID 自行签名的用户必须每七天重新签名一次。
- **iOS 上的性能。** 所有客户机代码都在翻译下运行。agent（智能体）会启动大量短生命周期进程，每个进程都要为新代码页付出翻译开销；在第 1 和第 5 阶段之前无法知道这项开销有多大。
- **内存。** 即使有 increased-memory-limit 授权，iOS 仍限制 app 内存，这决定了客户机内存的上限。
- **macOS 版本。** USB 直通需要 macOS 27；在更早的版本上，USB 存储设备只能通过共享文件夹进入客户机，其他 USB 设备只能通过原生模式使用。
- **原生模式的隔离。** Seatbelt 限制文件写入，但不限制网络访问和读取，因此原生模式下模型执行的命令能触及的范围比虚拟机更大。
- **网络栈。** 如果 lwIP 无法满足工具的需要，后备方案是 libslirp；它依赖 LGPL-2.1 许可的 glib，在静态链接的 iOS app 中很难满足其重新链接义务。
- **生成代码。** 已安装 app 是保留在设备上的模型编写代码。隔离依赖于每个 app 独立的 web 视图、内容安全策略，以及工具请求的权限流程。
- **时钟。** iOS 挂起 app 期间客户机时钟停止；如果错过恢复消息，TLS 证书校验会一直失败，直到下一次重新同步。
