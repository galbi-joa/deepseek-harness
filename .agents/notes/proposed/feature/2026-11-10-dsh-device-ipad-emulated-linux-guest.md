# Agent Note: DSH Device on iPad Through an Emulated Linux Guest

Status: proposed

English | [中文](2026-11-10-dsh-device-ipad-emulated-linux-guest.zh.md)

## Problem

DSH runs as a Node.js process on a desktop operating system, and its graphical surface is the Web UI that `dsh web` serves to a browser at a loopback URL. No configuration makes DSH the whole experience of a portable device: the user cannot open one app on an iPad, see the DSH UI without a browser or address bar, ask for "a calculator", and receive a working calculator that stays on the device.

iPadOS blocks the direct port in four ways:

- An iOS app cannot `fork` or `exec`. DSH tools depend on real child processes: `subprocess-local` spawns detached POSIX process groups, and the bash, ripgrep, Python, git, and language-server tools all run through it.
- iOS gives third-party apps no hypervisor API, and Secure Boot prevents booting a custom kernel on iPhone or iPad hardware.
- A loopback TCP port opened by one app is reachable by other apps on the same device, so serving the Web UI and its `/api` route on `127.0.0.1` exposes the configuration plane that [`client/connection`](../../../../packages/client/connection/README.md) restricts to loopback callers.
- Model-written UI exists only as [dynamic packages](../../implemented/feature/2026-07-08-self-referential-cordis-toolset.md): held in process memory, controllable only from the defining session, gone after a DSH restart, and executed inside the DSH page behind a guard that the toolset documents as not a security boundary.

## Proposal

Ship DSH on iPad as one iOS app that contains three parts: a native shell that draws all UI, an arm64 emulator engine, and a Linux guest that runs the DSH Host and every tool. The guest has no display device. The native shell renders the existing Web UI in a `WKWebView` and exchanges messages with the guest over virtio-vsock, so the device opens no TCP port and the user never sees a URL.

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

### Repository and license split

The engine fork derives from [Henge](https://github.com/Leviidev/Henge) at commit `8bde8134e012966c5968c3e71baedcc1985ace59`, which is GPL-2.0-or-later. The native shell, the engine fork, the guest kernel configuration, and the guest image build therefore live in a separate GPL repository. This repository receives only DSH-side changes: the bridge carrier, the Host socket listener, the `device` bundle, and installed apps. DSH runs in the guest as a separate program that communicates with the native shell over virtio-vsock and does not link against GPL code.

### Emulator engine

Henge runs macOS on iPhone and iPad without a hypervisor: an arm64-to-arm64 translator copies most guest instructions unchanged, rewrites loads, stores, branches, and system-register access, and implements a software MMU and the device model of QEMU's `vmapple` machine. The engine is built as a C library that the Swift app calls through `HengeBridge.h`. Its translator already runs V8: Chrome, VS Code, and Discord run inside Henge's macOS guests, and the dispatcher invalidates translated blocks when guest code pages change.

The fork keeps these parts:

- `core/`: the translator, the software MMU (4 KB, 16 KB, and 64 KB granules, `core/mmu.c:9`), exceptions, and PSCI 1.0 over HVC with `CPU_ON`, `CPU_OFF`, `SYSTEM_OFF`, and `SYSTEM_RESET` (`core/exception.c:131`).
- `devices/`: `gicv3`, `pl011`, `pl031`, `timer`, `pcie`, `virtio`, `virtio_blk`, and `virtio_net`.
- `host/`: `jit_region`, `disk` (virtio-blk flush maps to host `fsync`, `host/disk.c:79`), `net_slirp`, and `dns_proxy`.

The fork removes the macOS-specific parts: `vmapple`, the paravirtual GPU (`pvg_*`, `metallib`), `aes`, `bdif`, `xhci`, `virtio_usb`, `virtio_snd`, `virtio_input`, the restore and install stack under `third_party/` and `host/install.c`, and the AVPBooter boot ROM path.

The fork adds four parts:

- Direct kernel boot: load an arm64 `Image`, generate a flattened device tree for the machine below, and enter EL1 with `x0` holding the device-tree address.
- virtio-rng, backed by the iOS system random source. Without it, early `getrandom` calls in Node.js and TLS can block during boot.
- virtio-vsock, the transport for the UI bridge and for host-to-guest control messages.
- virtio-console for kernel and init diagnostics.

### Guest machine

The machine definition is fixed so that the same guest image boots under the engine fork on iPad, under the same engine built for macOS (`hengevm`), and under `qemu-system-aarch64 -M virt,gic-version=3` with virtio-pci on Linux development hosts.

| Component | Device | Purpose |
| --- | --- | --- |
| CPU | arm64, EL0 and EL1, PSCI over HVC | Linux SMP bring-up, power off, reboot |
| Interrupts and time | GICv3, ARM generic timer, PL031 RTC | Scheduling, wall-clock time for TLS certificate validation |
| Console | PL011, virtio-console | Early boot log, init diagnostics |
| Storage | virtio-blk system disk and data disk | Read-only system image, writable user data |
| Network | virtio-net through libslirp | Outbound HTTPS to the model provider without a guest Wi-Fi driver |
| Entropy | virtio-rng | Seeding the guest random pool at boot |
| Host channel | virtio-vsock | UI bridge and lifecycle control messages |

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

Features required by the machine and the storage layout: GICv3, the generic timer, PL011, PL031, the generic PCIe host, virtio-pci with blk, net, console, rng, and vsock drivers, ext4, squashfs, overlayfs, tmpfs, devtmpfs, and FUSE for the shared folder.

Everything else stays disabled, including USB, sound, DRM and framebuffer, input and HID, Bluetooth, wireless, NVMe, SCSI, MMC, ACPI and UEFI, KVM, kexec, suspend and hibernation, CPU and memory hotplug, NUMA, swap, eBPF, netfilter, bridging, tunnels, IPsec, SELinux, AppArmor, IMA, audit, and all other filesystems. Release builds also disable ftrace, perf, kprobes, and debugfs.

### Guest storage

| Path | Backing | Contents |
| --- | --- | --- |
| `/` | `system.img`, squashfs, read-only | Debian slim arm64, the official Node.js linux-arm64 build, DSH, bash, python3, git, ripgrep, bubblewrap |
| `/data` | `data.img`, ext4 | `DSH_HOME`, the SQLite session store, workspaces, installed apps, package caches, the overlayfs upper directory for packages the user installs |
| `/tmp` | tmpfs | Temporary files |
| `/mnt/shared` | The engine's WebDAV share mounted with davfs2 | File exchange with the iPadOS Files app |

An app update replaces `system.img` and leaves `data.img` untouched. Workspaces and `DSH_HOME` stay on ext4 because inotify and SQLite WAL shared mappings do not work on the WebDAV mount; the shared folder is only for moving files in and out.

iOS can terminate a suspended app at any time. Data the guest has flushed reaches the host file through `fsync` and survives termination; data still in the guest page cache does not. The guest mounts `/data` with a short ext4 commit interval, the session store keeps its WAL default, and the native shell asks the guest to `sync` over vsock when the app moves to the background.

### Guest init

DSH does not run as PID 1. A small init written in C or Rust performs these steps:

1. Mount procfs, sysfs, devtmpfs, devpts, tmpfs, `/data`, and the overlay.
2. Configure the static slirp network: address `10.0.2.15`, gateway `10.0.2.2`, DNS `10.0.2.3`.
3. Set the system clock from the PL031 RTC.
4. Start the DSH Host with the `device` bundle and restart it when it exits.
5. Reap orphaned tool processes.
6. Handle vsock control messages: `sync` on background, clock resynchronization on resume, and power off.

### DSH UI transport

The native shell loads the bundled Web UI frontend in a `WKWebView` through a custom URL scheme handler. Two DSH-side changes connect it to the guest Host:

- The Host web server listens only on `port` and `host` today (`packages/host/webserver/src/index.ts:218`). A validated `Config` field adds a Unix socket listener, and a guest relay connects that socket to a vsock port. The scheme handler forwards HTTP requests for static assets, `/plugins` bundles, and unary `/api` calls over vsock.
- `client/connection` already separates its browser carrier (HTTP POST up, WebSocket down) from an in-process carrier behind one two-stream abstraction. A third carrier carries both streams over `WKScriptMessageHandler` messages, because a custom URL scheme cannot serve WebSocket upgrades.

The `/api` route restricts privileged methods (`host.pickDirectory`, `host.openPath`, the settings and credentials plane, and agent-preset authoring) to loopback callers. The bridge carrier needs an explicit trust classification in the Host: requests that arrive on the vsock listener originate from the native shell on the same device, which grants them the loopback trust level, and no other listener grants it.

### Installed apps

The existing dynamic-package path stays as it is for changes to the DSH UI itself: it runs inside the DSH page, requires approval, and is governed by the [dynamic plugin runtime proposal](../architecture/2026-08-08-cordis-web-dynamic-packages.md).

Apps are a separate kind:

- The model writes a self-contained web app (HTML, JavaScript, CSS, and a manifest with name and icon) to a session workspace and calls a new `app_install` tool. The tool validates the manifest, copies the files to `/data/apps/<app-id>/`, and records the install in the session log, so every installed app is reconstructable from the log.
- The native shell lists installed apps on its home screen through a Host API method and opens each app in its own `WKWebView` with a per-app data store.
- An app has no access to the DSH client runtime or API. A content security policy blocks network access by default. An app that needs a tool run, such as a Python computation, sends a request through a narrow message handler, and the Host turns it into a tool call subject to the normal permission flow.
- `app_uninstall` removes the files and records the removal.

### The device bundle

A `device` bundle in `packages/bundle/` is a patch layer over `web-app`. It enables the Unix socket listener and the bridge carrier, keeps `subprocess-local` and `sandbox-local`, adds the app tools, and disables plugins that have no function on the device: the `pwsh-*` providers, `sandbox-windows-acl`, the seatbelt profile, `directory-picker-native`, and the e2b providers unless the user selects remote execution.

### Phases

1. Measure: on an iPad running stock Henge with a macOS guest, run `npx @deepseek-ai/dsh web` and time 100 runs each of `git status`, `rg`, and `python3 -c pass`.
2. Guest: build the kernel fragment, init, and both disk images, and boot them under QEMU `virt` on Linux with the `device` bundle and the Unix socket listener.
3. Engine: add direct kernel boot, virtio-rng, virtio-vsock, and virtio-console to the fork, and boot the same guest under `hengevm` on macOS.
4. Shell: build the iOS app with the Web UI view, the bridge carrier, and lifecycle messages.
5. Apps: add `app_install`, `app_uninstall`, the app list API, the home screen, and isolated app views.

## Alternatives considered

**Terminal UI on the guest console.** DSH ships only the `base`, `headless`, and `web-app` bundles; a terminal UI would be a new bundle. The Linux framebuffer console cannot render Hangul or other CJK text, and text input through an input method would need guest-side support. The Web UI already implements conversation, approvals, plans, and tool rendering.

**Browser with a loopback URL.** It requires no DSH changes, but it shows a browser and address bar, and on iOS other apps can reach the loopback port and the privileged `/api` methods.

**Darwin (XNU) guest.** Node.js supports darwin-arm64, so V8 and libuv would need no port. The open-source Darwin userland is incomplete, IOKit drivers for virtual or third-party hardware are closed, and the projects that boot XNU on Android hardware (ios-sweet, HackiOS, UniXNU) run Apple's firmware userland, which this project cannot redistribute.

**A DSH-specific kernel or unikernel.** A single-address-space kernel has no `fork` or `exec`, so bash, Python, and git cannot run, and it requires porting V8 and libuv. A later split into a DSH-specific kernel and a separate Linux tool VM remains possible through subprocess and filesystem Service Providers, in the same way `subprocess-e2b` and `fs-e2b` place tools in a remote sandbox, at the cost of two emulated kernels and the V8 port.

**WASI subprocess provider.** WASI has no process creation, so shell scripts cannot launch other programs, and git, native compilers, and most language servers have no usable WASI builds.

**Remote execution only.** The e2b providers already run tools in a remote sandbox. Using them exclusively makes every tool call depend on network access and a hosted service, so the device does not work on its own. The `device` bundle keeps them as an option.

**Guest-side graphics.** Rendering the UI inside the guest requires a GPU device and runs the browser engine under emulation; Henge's paravirtual GPU path is specific to macOS guests.

**QEMU instead of the Henge engine.** Husk runs a LineageOS guest under QEMU on iOS, and QEMU already provides the `virt` machine, direct kernel boot, virtio-vsock, virtio-rng, and virtio-console. Its TCG translator is generic across instruction sets, while Henge's translator copies most same-ISA instructions unchanged and is expected to run faster. Phase 3 confirms this with the phase 1 benchmark; QEMU is the fallback if the measured difference does not justify the four engine additions.

**Persisting dynamic packages as apps.** A dynamic package's browser half runs inside the DSH page behind a guard that is not a security boundary. Persisting model-written code at that privilege level across restarts widens what an approval grants, so apps run in isolated views instead.

## Acceptance criteria

- On an A14 or M1 iPad with iPadOS 17 or later, opening the app shows the DSH UI with no browser, no visible URL, and no TCP listening socket on the device.
- The prompt "build a calculator" produces an installed app that appears on the home screen, survives app termination and relaunch, and runs in an isolated view without access to the DSH API.
- bash, python3, git, and ripgrep tool calls succeed in the guest, and the sandbox reports bwrap or the Landlock fallback as active.
- Session data written before the app moves to the background survives iOS terminating the app.
- Model requests succeed after the app resumes from a suspension longer than one hour.
- The phase 1 latency measurements and the same measurements on the engine fork are recorded.

## Risks

- **License.** The native shell and engine are GPL-2.0-or-later and cannot be distributed through the App Store. Running MIT-licensed DSH as a separate program in the guest is not commonly treated as forming a combined work, but this needs legal review before distribution.
- **Distribution.** Users must sideload the app and enable JIT through StikDebug, TrollStore, or a similar tool, and JIT availability changes with iOS releases.
- **Performance.** All guest code runs under translation. Agents start many short-lived processes, each of which pays translation cost for new code pages; the size of this cost is unknown until phase 1.
- **Upstream maturity.** Henge 0.1 was released on 2026-11-06; its README states that iPad support had not run on a real iPad, while the 0.2 release notes report contributor testing on iPads.
- **Memory.** iOS limits app memory; guest RAM of 4 to 8 GB requires the increased-memory-limit entitlement.
- **Generated code.** Installed apps are model-written code that persists on the device. Isolation depends on the per-app web view, the content security policy, and the permission flow for tool requests.
- **Clock.** The guest clock stops while iOS suspends the app; a missed resume message leaves TLS certificate validation failing until the next resynchronization.
