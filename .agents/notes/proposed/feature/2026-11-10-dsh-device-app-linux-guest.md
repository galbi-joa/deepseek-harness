# Agent Note: DSH Device App for iPad and Mac Around a Linux Guest

Status: proposed

English | [中文](2026-11-10-dsh-device-app-linux-guest.zh.md)

## Problem

DSH runs as a Node.js process on a desktop operating system, and its graphical surface is the Web UI that `dsh web` serves to a browser at a loopback URL. No configuration makes DSH the whole experience of an app on an iPad or a Mac: the user cannot open one app, see the DSH UI without a browser or address bar, ask for "a calculator", and receive a working calculator that stays on the device.

Four constraints block this:

- An iOS app cannot `fork` or `exec`. DSH tools depend on real child processes: `subprocess-local` spawns detached POSIX process groups, and the bash, ripgrep, Python, git, and language-server tools all run through it.
- iOS gives third-party apps no hypervisor API, and Secure Boot prevents booting a custom kernel on iPhone or iPad hardware.
- A loopback TCP port opened by one app is reachable by other apps on the same device, so serving the Web UI and its `/api` route on `127.0.0.1` exposes the configuration plane that [`client/connection`](../../../../packages/client/connection/README.md) restricts to loopback callers.
- Model-written UI exists only as [dynamic packages](../../implemented/feature/2026-07-08-self-referential-cordis-toolset.md): held in process memory, controllable only from the defining session, gone after a DSH restart, and executed inside the DSH page behind a guard that the toolset documents as not a security boundary.

macOS allows child processes and hardware virtualization, but DSH running directly on macOS gives tools a different operating system from the iPad and only Seatbelt write restrictions as isolation.

## Proposal

Build one SwiftUI app for iOS, iPadOS, and macOS. The app draws all UI natively and renders the existing Web UI in a `WKWebView`. DSH and every tool run in a display-less Linux guest; the same guest image runs on two execution backends. On iOS and iPadOS, `dsh-vm`, an arm64-to-arm64 JIT engine written for this project, runs the guest. On macOS, Virtualization.framework runs the guest with hardware virtualization, and an optional native mode runs DSH as a local process without a VM. The UI and the guest exchange messages over virtio-vsock, so no TCP port opens and the user never sees a URL. The app is distributed outside the App Store.

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

### Repository layout

All parts are written for this project and licensed MIT, so they live in this repository:

| Path | Contents |
| --- | --- |
| `native/dsh-vm/` | The iOS execution engine in Rust, with a macOS command-line build used for tests |
| `native/dsh-guest/` | The guest kernel Kconfig fragment, init, and the scripts that build `system.img` |
| `apps/device/` | The SwiftUI app for iOS, iPadOS, and macOS |
| `packages/` | DSH-side changes: the bridge carrier, the Host socket listener, the `device` bundle, and installed apps |

### Hardware access

The guest has no display, input, GPU, or USB controller device. The host operating system owns the hardware, and the guest receives files, sockets, or a passed-through USB device.

| Capability | iOS and iPadOS | macOS |
| --- | --- | --- |
| Touch, trackpad, keyboard, input methods | Handled by `WKWebView` | Handled by `WKWebView` |
| UI drawing and graphics in installed apps | WebKit on the host GPU | WebKit on the host GPU |
| Camera and microphone in installed apps | `WKWebView` media capture permission | `WKWebView` media capture permission |
| Files on USB storage | Mounted by the Files app, shared to the guest over 9p | Mounted under `/Volumes`, shared over virtiofs; read directly in native mode |
| Other USB devices | Not supported: iPadOS requires a DriverKit driver and an entitlement granted per vendor ID | Passed through with `VZUSBPassthroughDeviceConfiguration` on macOS 27 or later; accessed directly in native mode |
| GPU compute for tools | Not supported | Native mode only (Metal) |

### dsh-vm engine

`dsh-vm` runs the guest on iOS and iPadOS, where no hypervisor API exists. Its implementation follows public specifications only: the Arm Architecture Reference Manual (A64, VMSAv8-64, generic timer), the GICv3 architecture specification, the virtio 1.2 specification, PSCI 1.0, the Linux arm64 boot protocol, and the devicetree specification. QEMU serves as a differential-test oracle and is never linked or copied. [Henge](https://github.com/Leviidev/Henge) shows that same-ISA translation on iOS can run guests that execute V8; its source is not used.

The engine supports only what the guest needs, because this project also builds the guest kernel:

- CPU: EL0 and EL1. The ID registers advertise no pointer authentication, BTI, MTE, SVE, or SME, so the guest kernel and glibc never emit those instructions.
- Translation: data-processing instructions are copied unchanged; loads, stores, atomics, and exclusives go through the software MMU; branches and PC-relative instructions are rewritten; system registers and exceptions call helpers; a write to a translated code page invalidates its blocks.
- MMU: VMSAv8-64 stage 1 with one 16 KB granule and 48-bit virtual addresses.
- One vCPU in the first release. SMP follows once differential tests pass for the instructions the translator rewrites.
- Devices: GICv3, the generic timer, PL011, PL031, and the virtio-mmio transport with blk, net, rng, vsock, console, and 9p devices.
- Boot: load the kernel `Image`, generate a flattened device tree, and enter EL1 with `x0` holding its address. PSCI over HVC provides `SYSTEM_OFF`, `SYSTEM_RESET`, and later `CPU_ON`.
- Host services: one mapping for guest RAM; a JIT region using separate writable and executable mappings once JIT is enabled for the process; disk files whose virtio-blk flush maps to `fsync`; a user-mode TCP/IP stack built on lwIP (BSD-3-Clause) that opens real sockets through the host operating system; vsock endpoints delivered to the Swift shell; and a 9P2000.L server for the shared folder.

Rust is the implementation language because virtio queue processing parses descriptors the guest controls; unsafe code is confined to the code-emission and memory-mapping modules.

### macOS backend

By default the macOS app boots the same kernel and images with Virtualization.framework (`VZLinuxBootLoader`): `VZVirtioBlockDeviceConfiguration` for both disks, `VZVirtioNetworkDeviceConfiguration` with `VZNATNetworkDeviceAttachment`, `VZVirtioEntropyDeviceConfiguration`, `VZVirtioSocketDeviceConfiguration` for the bridge, `VZVirtioConsoleDeviceConfiguration`, and `VZVirtioFileSystemDeviceConfiguration` for shared folders. On macOS 27 and later, a `VZXHCIControllerConfiguration` with `VZUSBPassthroughDeviceConfiguration` passes a host USB device to the guest. The app carries the `com.apple.security.virtualization` entitlement.

Native mode is a setting. The app starts the DSH Host as a local process with the same `device` bundle; `sandbox-local` selects its macOS Seatbelt runner, and tools reach USB devices, serial ports, and Metal directly.

### Guest machine

The kernel enables the drivers both backends present, so one guest image boots under `dsh-vm`, under Virtualization.framework, and under `qemu-system-aarch64 -M virt,gic-version=3` on Linux development hosts.

| Component | dsh-vm (iOS and iPadOS) | Virtualization.framework (macOS) |
| --- | --- | --- |
| CPU | arm64 EL0 and EL1, one vCPU at first, PSCI over HVC | Hardware vCPUs |
| Interrupts and time | GICv3, generic timer, PL031 RTC | Provided by the platform; the device list is confirmed in phase 2 |
| Console | PL011, virtio-console | virtio-console |
| Storage | virtio-blk over virtio-mmio | virtio-blk over virtio-pci |
| Network | virtio-net to the engine's lwIP stack | virtio-net with NAT |
| Entropy | virtio-rng backed by the iOS system random source | virtio-rng |
| Host channel | virtio-vsock | virtio-vsock |
| Shared folder | 9p | virtiofs |
| USB | None | USB passthrough on macOS 27 or later |

### Guest kernel

The guest kernel is a current Linux LTS release for arm64, built from a Kconfig fragment with no loadable modules. The fragment enables only three groups of features.

Features DSH uses directly, with the code that uses them:

| Feature | User in this repository |
| --- | --- |
| Process groups and signals | Process-tree teardown with `process.kill(-pid, sig)` in `subprocess-local/src/spawn.ts` |
| Complete procfs, including `/proc/<pid>/task/<tid>/syscall`, `/proc/<pid>/mem`, and `/proc/<pid>/fdinfo` | Terminal readiness and teardown in `subprocess-local/src/process-inspector.ts` |
| Unix98 pseudo-terminals and devpts | Persistent terminal sessions through `node-pty` |
| User and mount namespaces | The bwrap rung of `sandbox-local` (`--ro-bind / /`, `--proc`, `--dev`, `--tmpfs`, `--bind`) |
| Landlock (ABI 1 to 5) and `PR_SET_NO_NEW_PRIVS` | The landlock-run fallback rung; ABI 5 requires Linux 6.10 or later |
| inotify | File watching in `settings/settings` and `skill/skill-filesystem` |
| Shared file mappings | SQLite WAL `-shm` files in `session-persistence-sqlite` |
| Threads, futex, executable anonymous mappings | V8 and `worker_threads` in `workflow-worker-thread` and `code-runtime-worker-thread` |
| Outbound TCP and UDP DNS, epoll, eventfd | `fetch` to the model provider and libuv |

DSH itself opens no listening TCP socket in the device composition, uses no cgroups, no seccomp, and no network namespace, and does not require io_uring because libuv falls back to its thread pool.

Features that tools run by DSH expect: loopback networking with listening sockets (test servers and some language servers), IPv6 on loopback (`localhost` can resolve to `::1`), PID, IPC, and UTS namespaces and seccomp (unprivileged bwrap variants and tools inside workspaces), SysV IPC and POSIX shared memory, and the uid and gid model that user-namespace mapping requires.

Features required by the backends and the storage layout: GICv3, the generic timer, PL011, PL031, the generic PCIe host, virtio-mmio and virtio-pci with blk, net, console, rng, and vsock drivers, 9p and virtiofs, kernel IP autoconfiguration with DHCP, ext4, squashfs, overlayfs, tmpfs, and devtmpfs. ACPI stays enabled only if phase 2 shows that the Virtualization.framework platform requires it.

Everything else stays disabled, including USB, sound, DRM and framebuffer, input and HID, Bluetooth, wireless, NVMe, SCSI, MMC, UEFI, KVM, kexec, suspend and hibernation, CPU and memory hotplug, NUMA, swap, eBPF, netfilter, bridging, tunnels, IPsec, SELinux, AppArmor, IMA, audit, and all other filesystems. Release builds also disable ftrace, perf, kprobes, and debugfs. A host USB device passed through on macOS requires the USB host and class drivers for that device, which the fragment adds when that feature ships.

### Guest storage

| Path | Backing | Contents |
| --- | --- | --- |
| `/` | `system.img`, squashfs, read-only, shipped inside the app | Debian slim arm64, the official Node.js linux-arm64 build, DSH, bash, python3, git, ripgrep, bubblewrap |
| `/data` | `data.img`, ext4, created at first launch | `DSH_HOME`, the SQLite session store, workspaces, installed apps, package caches, the overlayfs upper directory for packages the user installs |
| `/tmp` | tmpfs | Temporary files |
| `/mnt/shared` | 9p on iOS and iPadOS, virtiofs on macOS | Files the user shares from the Files app, Mac folders, or USB storage |

An app update replaces `system.img` and leaves `data.img` untouched. Workspaces and `DSH_HOME` stay on ext4 because the shared mounts do not report host-side changes through inotify and do not reliably support the shared mappings that SQLite WAL uses; the shared folder is for moving files in and out.

iOS can terminate a suspended app at any time. Data the guest has flushed reaches the host file through `fsync` and survives termination; data still in the guest page cache does not. The guest mounts `/data` with a short ext4 commit interval, the session store keeps its WAL default, and the native shell asks the guest to `sync` over vsock when the app moves to the background.

### Guest init

DSH does not run as PID 1. A small init written in Rust performs these steps:

1. Mount procfs, sysfs, devtmpfs, devpts, tmpfs, `/data`, the overlay, and the shared folder.
2. Configure the network: static addresses that the `dsh-vm` network stack expects, or the address that kernel IP autoconfiguration obtains from the Virtualization.framework NAT.
3. Set the system clock from the RTC.
4. Start the DSH Host with the `device` bundle and restart it when it exits.
5. Reap orphaned tool processes.
6. Handle vsock control messages: `sync` when iOS moves the app to the background, clock resynchronization on resume or host wake, and power off.

### DSH UI transport

The native shell loads the bundled Web UI frontend in a `WKWebView` through a custom URL scheme handler. Two DSH-side changes connect it to the Host:

- The Host web server listens only on `port` and `host` today (`packages/host/webserver/src/index.ts:218`). A validated `Config` field adds a Unix socket listener. In the guest, a relay connects that socket to a vsock port; in macOS native mode, the shell connects to the socket directly. The scheme handler forwards HTTP requests for static assets, `/plugins` bundles, and unary `/api` calls.
- `client/connection` already separates its browser carrier (HTTP POST up, WebSocket down) from an in-process carrier behind one two-stream abstraction. A third carrier carries both streams over `WKScriptMessageHandler` messages, because a custom URL scheme cannot serve WebSocket upgrades.

The `/api` route restricts privileged methods (`host.pickDirectory`, `host.openPath`, the settings and credentials plane, and agent-preset authoring) to loopback callers. The bridge carrier needs an explicit trust classification in the Host: requests on the Unix socket listener come from the native shell, which grants them the loopback trust level, and no other listener grants it. In macOS native mode the socket lives in the app container with mode `0600`; other processes of the same user can still connect to it, which matches the exposure of the loopback listener today.

### Installed apps

The existing dynamic-package path stays as it is for changes to the DSH UI itself: it runs inside the DSH page, requires approval, and is governed by the [dynamic plugin runtime proposal](../architecture/2026-08-08-cordis-web-dynamic-packages.md).

Apps are a separate kind:

- The model writes a self-contained web app (HTML, JavaScript, CSS, and a manifest with name and icon) to a session workspace and calls a new `app_install` tool. The tool validates the manifest, copies the files to `/data/apps/<app-id>/`, and records the install in the session log, so every installed app is reconstructable from the log.
- The native shell lists installed apps on its home screen through a Host API method and opens each app in its own `WKWebView` with a per-app data store.
- An app has no access to the DSH client runtime or API. A content security policy blocks network access by default. An app that needs a tool run, such as a Python computation, sends a request through a narrow message handler, and the Host turns it into a tool call subject to the normal permission flow.
- `app_uninstall` removes the files and records the removal.

### The device bundle

A `device` bundle in `packages/bundle/` is a patch layer over `web-app`. It enables the Unix socket listener and the bridge carrier, keeps `subprocess-local` and `sandbox-local`, adds the app tools, and disables plugins that have no function in the app: the `pwsh-*` providers, `sandbox-windows-acl`, `directory-picker-native`, and the e2b providers unless the user selects remote execution. The same bundle serves the guest and macOS native mode; `sandbox-local` selects bwrap and Landlock in the guest and Seatbelt on macOS.

### Distribution

The app is distributed outside the App Store under a paid Apple Developer Program membership:

- macOS: signed with Developer ID, notarized, and downloaded directly.
- iOS and iPadOS: signed with the membership's provisioning profile, which stays valid for one year, and sideloaded. JIT still requires attaching a debugger through StikDebug or a similar tool, because the membership grants no JIT entitlement to iOS apps. The membership provides the increased-memory-limit entitlement for guest RAM.

### Phases

1. Measure: run `npx @deepseek-ai/dsh web` inside an existing iOS emulator app (stock Henge with a macOS guest) and time 100 runs each of `git status`, `rg`, and `python3 -c pass` to estimate translation cost for tool-heavy work.
2. Guest: build the kernel fragment, init, and both images, and boot them under QEMU `virt` on Linux and under Virtualization.framework on macOS with the `device` bundle and the Unix socket listener.
3. macOS app: build the SwiftUI shell, the Web UI view, the bridge carrier, the Virtualization.framework backend, native mode, and USB passthrough on macOS 27.
4. Installed apps: add `app_install`, `app_uninstall`, the app list API, the home screen, and isolated app views.
5. Engine: build `dsh-vm` as a macOS command-line program on Apple Silicon, pass differential tests against QEMU, and boot the guest with one vCPU.
6. iOS app: embed `dsh-vm`, detect JIT availability, add lifecycle messages, and then enable SMP.

## Alternatives considered

**Terminal UI on the guest console.** DSH ships only the `base`, `headless`, and `web-app` bundles; a terminal UI would be a new bundle. The Linux framebuffer console cannot render Hangul or other CJK text, and text input through an input method would need guest-side support. The Web UI already implements conversation, approvals, plans, and tool rendering.

**Browser with a loopback URL.** It requires no DSH changes, but it shows a browser and address bar, and on iOS other apps can reach the loopback port and the privileged `/api` methods.

**Darwin (XNU) guest.** Node.js supports darwin-arm64, so V8 and libuv would need no port. The open-source Darwin userland is incomplete, IOKit drivers for virtual or third-party hardware are closed, and the projects that boot XNU on Android hardware (ios-sweet, HackiOS, UniXNU) run Apple's firmware userland, which this project cannot redistribute.

**A DSH-specific kernel or unikernel.** A single-address-space kernel has no `fork` or `exec`, so bash, Python, and git cannot run, and it requires porting V8 and libuv. A later split into a DSH-specific kernel and a separate Linux tool VM remains possible through subprocess and filesystem Service Providers, in the same way `subprocess-e2b` and `fs-e2b` place tools in a remote sandbox, at the cost of two kernels and the V8 port.

**User-mode emulation with an in-app Linux system call layer.** Translating only guest user code and implementing Linux system calls, process creation, signals, and pseudo-terminals inside the app is the approach iSH takes on iOS. It amounts to writing a Linux-compatible kernel with known compatibility gaps; running the real Linux kernel requires emulating only a few devices and gives bwrap, Landlock, and pseudo-terminals their real behavior.

**WASI subprocess provider.** WASI has no process creation, so shell scripts cannot launch other programs, and git, native compilers, and most language servers have no usable WASI builds.

**Remote execution only.** The e2b providers already run tools in a remote sandbox. Using them exclusively makes every tool call depend on network access and a hosted service, so the app does not work on its own. The `device` bundle keeps them as an option.

**Forking Henge's engine.** Henge's engine is GPL-2.0-or-later, which would force the app into a separate GPL repository. It also carries about 7,700 lines of macOS-specific devices (paravirtual GPU, USB controllers, trackpad, AES, `vmapple`) and handles pointer-authentication instructions, three page granules, and Apple system registers that this guest never uses, because Henge must run an unmodified macOS while this project chooses its guest kernel.

**QEMU on iOS.** Husk runs a LineageOS guest under QEMU on iOS, and QEMU already provides every device the guest needs. QEMU is GPL-2.0-or-later, and its TCG translator is generic across instruction sets, while same-ISA translation copies most instructions unchanged. QEMU remains the differential-test oracle for `dsh-vm`.

**dsh-vm on macOS as well.** macOS apps can use Virtualization.framework, which runs the guest at hardware speed and already provides virtio devices, shared folders, and USB passthrough, so the engine is needed only where no hypervisor API exists.

**Native mode as the only macOS backend.** It gives tools direct hardware access on every macOS version, but tools then run on a different operating system from the iPad version, and isolation drops from a VM to Seatbelt write restrictions. Native mode stays available as a setting.

**Emulating display, input, GPU, and USB controller devices in the guest.** A guest that draws its own UI needs these devices, as Henge's macOS guest does. This guest draws nothing; `WKWebView` handles input and drawing on the host, and USB devices reach the guest only through shared folders or Virtualization.framework passthrough.

**Persisting dynamic packages as apps.** A dynamic package's browser half runs inside the DSH page behind a guard that is not a security boundary. Persisting model-written code at that privilege level across restarts widens what an approval grants, so apps run in isolated views instead.

## Acceptance criteria

- On macOS, opening the app boots the guest under Virtualization.framework and shows the DSH UI with no browser, no visible URL, and no TCP listening socket; native mode runs the same UI with tool calls under Seatbelt.
- On macOS 27, a USB device the user attaches to the guest appears in the guest's device list.
- On an A14 or M1 iPad with iPadOS 17 or later, opening the app boots the guest under `dsh-vm` and shows the same UI under the same conditions.
- The prompt "build a calculator" produces an installed app that appears on the home screen, survives app termination and relaunch, and runs in an isolated view without access to the DSH API.
- bash, python3, git, and ripgrep tool calls succeed in the guest, and the sandbox reports bwrap or the Landlock fallback as active.
- Session data written before the app moves to the background survives iOS terminating the app.
- Model requests succeed after the app resumes from a suspension longer than one hour.
- `dsh-vm` passes differential tests against QEMU for every instruction class it rewrites, and the phase 1 latency measurements and the same measurements on `dsh-vm` are recorded.

## Risks

- **Engine effort.** The translator and software MMU hold most of the difficulty: exception and interrupt timing, invalidation after code writes, and performance tuning. Starting with one vCPU defers cross-CPU TLB invalidation and contended exclusives.
- **Provenance.** This design was informed by reading Henge's documentation and parts of its source. Implementers write `dsh-vm` from the public specifications without consulting Henge or QEMU source, and each module records the specification sections it implements.
- **iOS distribution.** Users need a debugger-based JIT tool, and JIT availability changes with iOS releases. Installing on other people's devices with the paid membership requires registering their device identifiers, limited to 100 devices per device family per membership year; users who sign the app themselves with a free Apple ID must re-sign every seven days.
- **Performance on iOS.** All guest code runs under translation. Agents start many short-lived processes, each of which pays translation cost for new code pages; the size of this cost is unknown until phases 1 and 5.
- **Memory.** iOS limits app memory even with the increased-memory-limit entitlement, which bounds guest RAM.
- **macOS version.** USB passthrough requires macOS 27; on earlier versions, USB storage reaches the guest only through shared folders, and other USB devices only through native mode.
- **Native mode isolation.** Seatbelt restricts file writes but not network access or reads, so native mode gives model-run commands more reach than the VM.
- **Network stack.** If lwIP does not cover what tools need, libslirp is the fallback; it depends on glib under LGPL-2.1, whose relinking obligations are hard to satisfy in a statically linked iOS app.
- **Generated code.** Installed apps are model-written code that persists on the device. Isolation depends on the per-app web view, the content security policy, and the permission flow for tool requests.
- **Clock.** The guest clock stops while iOS suspends the app; a missed resume message leaves TLS certificate validation failing until the next resynchronization.
