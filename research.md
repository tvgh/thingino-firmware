# Thingino Firmware - Deep Research Report

## 1. Project Overview

**Thingino** (_/thin-jee-no/_) is an open-source firmware distribution for IP cameras powered by **Ingenic SoC** processors. It is built on top of [Buildroot](https://buildroot.org/) using the **BR2_EXTERNAL** (external tree) mechanism, which means Thingino layers its custom packages, configurations, and overlays on top of
a standard Buildroot build system without modifying Buildroot's own source code.

- **Repository:** `https://github.com/themactep/thingino-firmware`
- **License:** MIT
- **Primary target:** MIPS32-based Ingenic T-series and A-series SoCs (XBurst1 / XBurst2 architectures)
- **Branches:**
  - `stable` — Tested, reliable builds for end-users
  - `master` — Active development with experimental features (Matroska, Opus, improved Prudynt)

---

## 2. Repository Structure

```
thingino-firmware/
├── board/                  # Board-specific files (currently minimal)
├── board.mk                # Camera/board selection logic
├── buildroot/              # Buildroot submodule (git submodule)
├── configs/                # Camera defconfigs, fragments, GitHub CI configs
│   ├── cameras/            # Per-camera configuration directories
│   ├── cameras-exp/        # Experimental camera configs
│   ├── fragments/          # Reusable config fragments (SoC, toolchain, system)
│   ├── github/             # CI-specific configs (toolchain/cache builds)
│   └── common.uenv.txt     # Shared U-Boot environment defaults
├── docs/                   # Documentation (hardware, OTA, firmware structure, etc.)
├── linux/                  # Kernel extension configs & patches
├── overlay/                # Root filesystem overlay (init, /etc, base dirs)
├── package/                # 130+ custom Buildroot packages
├── scripts/                # Build, deployment, and utility scripts
├── user/                   # User-local overrides (fragments, overlay, uenv)
├── Config.in               # Master Kconfig menu
├── Config.soc.in           # SoC model selection and ISP/AVPU/IPU config
├── Config.sensor.in        # Image sensor configuration
├── Config.in.host          # Host tool configuration
├── Makefile                 # Top-level build orchestrator
├── Makefile.docker          # Docker build environment management
├── thingino.mk              # SoC → kernel → sensor → ISP → U-Boot variable resolution
├── board.mk                # Interactive camera selection and defconfig resolution
├── external.desc            # BR2_EXTERNAL tree descriptor (name: THINGINO)
├── external.mk              # Includes all package/*.mk files
├── Dockerfile               # Reproducible build container (Debian Trixie)
└── docker-build.sh          # Docker image build wrapper
```

---

## 3. Supported Hardware

### 3.1 Ingenic SoC Families

Thingino supports an extensive range of Ingenic SoCs organized into families:

| SoC Family | Models | RAM Range | Architecture | Kernel |
|:-----------|:-------|:----------|:-------------|:-------|
| **T10** | T10L, T10N, T10A | 64 MB | XBurst1 | 3.10.14 |
| **T20** | T20L, T20N, T20X, T20Z | 64–128 MB | XBurst1 | 3.10.14 |
| **T21** | T21L, T21N, T21X, T21Z, T21ZL | 64–128 MB | XBurst1 | 3.10.14 |
| **T23** | T23N, T23ZN, T23DL, T23DN | 32–64 MB | XBurst1 | 3.10.14 / 4.4.94 |
| **T30** | T30L, T30N, T30X, T30A | 64–128 MB | XBurst1 | 3.10.14 |
| **T31** | T31L, T31LC, T31N, T31X, T31A, T31AL, T31ZL, T31ZX | 64–128 MB | XBurst1 | 3.10.14 / 4.4.94 |
| **C100** | C100 | 128 MB | XBurst1 | 3.10.14 / 4.4.94 |
| **T32** | T32LQ, T32NQ, T32XQ, T32A, T32ZL, T32ZN, T32ZX, T32ZR, T32ZMC | Various | XBurst1 | 4.4.94 |
| **T40** | T40N, T40NN, T40XP, T40A | 128–256 MB | XBurst2 | 4.4.94 |
| **T41** | T41LQ, T41NQ, T41ZL, T41ZN, T41ZX, T41A | 64–512 MB | XBurst2 | 3.10.14 / 4.4.94 |
| **A1** | A1N, A1NT, A1X, A1L, A1A | 128–512 MB | XBurst2 | 4.4.94 |

### 3.2 Device Types

Beyond standard IP cameras, the build system supports:
- **IP Camera** (default) — with optional Pan/Tilt/Zoom motors
- **Webcam** — USB-connected camera mode
- **NVR** — Network Video Recorder
- **SBC** — Single Board Computer
- **Accessories:** Doorbell, Siren, Floodlight, Laser Pointer, Alarm

### 3.3 Image Sensors

The system supports up to **4 image sensors** per device, configured via `Config.sensor.in`. Sensors are specified by name (e.g., `gc2053`, `sc2335`, `imx327`) and each can have custom module parameters.

### 3.4 WiFi Drivers

The `package/wifi/` directory contains drivers for dozens of WiFi chipsets, including:
- Realtek (RTL8188EU, RTL8189FTV, RTL8733BU, etc.)
- ATBM (ATBM6031, ATBM6032, etc.)
- Broadcom/Cypress (CYW43XXX)
- Mediatek, AIC, SSV, iComm, and many more

---

## 4. Build System Architecture

### 4.1 Build Flow

```
make [CAMERA=<name>]
  └── board.mk           ← Interactive camera selection (fzf/whiptail/dialog)
  └── thingino.mk         ← Resolve SoC, kernel, sensor, ISP, U-Boot variables
  └── defconfig            ← Assemble .config from fragments + camera defconfig
  └── build_fast           ← Parallel Buildroot compilation
  └── pack                 ← Assemble firmware binary partitions
```

**Key build targets:**
| Target | Description |
|:-------|:------------|
| `make` / `make all` | Full build (defconfig → parallel build → pack) |
| `make dev` | Serial build for debugging compilation issues |
| `make cleanbuild` | Distclean + fresh parallel build |
| `make release` | Build without local fragments |
| `make pack` | Assemble firmware image from compiled components |
| `make rebuild-<pkg>` | Clean rebuild of a single package |
| `make menuconfig` | Buildroot interactive configuration |
| `make update` | Git pull + submodule update |
| `make run <bin>` | Run a target binary via QEMU emulation |

### 4.2 Configuration System ("Fragments")

Thingino uses a **composition-based configuration** system. Rather than having a monolithic defconfig per camera, each camera's `_defconfig` file references reusable **fragments** via a special `# FRAG:` comment line:

```
# FRAG: soc-xburst1 toolchain-musl system kernel rootfs uboot
BR2_SOC_INGENIC_T31X=y
BR2_SENSOR_1_NAME="gc2053"
BR2_PACKAGE_WIFI_ATBM6031=y
FLASH_SIZE_MB=16
```

During `make defconfig`, the build system:
1. Parses the `# FRAG:` line from the camera defconfig
2. Concatenates each referenced `.fragment` file from `configs/fragments/`
3. Appends the camera-specific config
4. Optionally appends `user/local.fragment` (for local overrides)
5. Runs Buildroot's `olddefconfig` to resolve all dependencies

**Fragment categories include:**
- `soc-xburst1` / `soc-xburst2` — CPU architecture settings
- `toolchain-musl` / `toolchain-uclibc` / `toolchain-glibc` — C library selection
- `system` — Core system packages (Prudynt, Dropbear, WebUI, etc.)
- `kernel` — Kernel build options
- `rootfs` — Root filesystem options (SquashFS)
- `uboot` — Bootloader configuration

### 4.3 U-Boot Environment Layering

The U-Boot environment (`uenv.txt`) is assembled from three layers:
1. `configs/common.uenv.txt` — Universal defaults (e.g., `enable_updates=true`)
2. `configs/cameras/<name>/<name>.uenv.txt` — Camera-specific GPIO mappings, LED pins, WiFi settings
3. `user/local.uenv.txt` — User-local overrides

These are sorted and deduplicated, then compiled into a binary environment image via `mkenvimage`.

---

## 5. Firmware Image Structure

### 5.1 Partition Layout

The firmware is assembled into an 8 MB (default) SPI NOR/NAND flash image with the following partition layout:

| Partition | Default Size | Offset | Format | Contents |
|:----------|:-------------|:-------|:-------|:---------|
| **U-Boot** | 256 KB | 0x00000 | Raw binary | Bootloader (LZO compressed with SPL) |
| **U-Boot Env** | 32 KB | 0x40000 | mkenvimage | Boot environment variables |
| **Config** | 224 KB | 0x48000 | JFFS2 | Persistent user configuration |
| **Kernel** | Dynamic | 0x80000 | uImage | Linux kernel |
| **RootFS** | Dynamic | Variable | SquashFS | Read-only root filesystem |
| **Extras** | Remaining | Variable | JFFS2 | `/opt` contents (writable) |

All partitions are 32 KB aligned (configurable via `BR2_THINGINO_ALIGN_SIZE_KB`).

### 5.2 Image Assembly (`make pack`)

Two firmware images are produced:
- **Full Image** (`thingino-<camera>.bin`): Contains all 6 partitions — used for initial flashing
- **Update Image** (`thingino-<camera>-update.bin`): Excludes U-Boot and env — used for OTA updates

The full image starts as an 8 MB slab filled with `0xFF` (erased flash state), then each partition binary is `dd`'d into its calculated offset. SHA256 checksums are generated for both images.

### 5.3 Root Filesystem Architecture

The root filesystem uses a sophisticated overlay approach:
1. **SquashFS** — Read-only compressed root filesystem
2. **JFFS2 overlay** — Writable layer mounted via `overlayfs`
3. **`overlay/init`** — A pre-init script that mounts `/proc`, determines the rootfs type, creates the overlay mount, and performs `pivot_root`

This allows the firmware to maintain a factory-resettable read-only base while persisting user changes.

---

## 6. Key Packages

### 6.1 Core System

| Package | Purpose |
|:--------|:--------|
| `thingino-core` | Master init — generates `/etc/thingino.json` by merging common, camera-specific, and user JSON configs using `jct` |
| `thingino-system` | System services, init scripts, base configuration |
| `thingino-sysupgrade` | OTA system upgrade mechanism |
| `thingino-uboot` | Custom U-Boot bootloader build (supports Legacy and Kconfig) |
| `thingino-kopt` | Kernel module options and configuration |
| `thingino-jct` | JSON Configuration Tool — used for layered config merging |

### 6.2 Media & Streaming

| Package | Purpose |
|:--------|:--------|
| `prudynt-t` | **Primary video streamer** — H.264/H.265, AAC/Opus/MP3/FLAC, WebRTC (via libdatachannel), RTSP, motion detection with pre-buffering |
| `raptor-ipc` | Alternative IPC streamer |
| `go2rtc` / `go2rtc-mini` | Go-based streaming proxy |
| `thingino-ffmpeg` | Custom FFmpeg build for the platform |
| `thingino-live555` | RTSP library |
| `thingino-streamer` | Streamer management utilities |
| `thingino-v4l2loopback` | V4L2 loopback device for webcam mode |
| `thingino-fonts` | OSD font rendering |
| `faac` / `libhelix-aac` / `libhelix-mp3` / `libflac` | Audio codecs |
| `libjuice` / `libdatachannel` / `usrsctp` | WebRTC stack |

### 6.3 Hardware Abstraction

| Package | Purpose |
|:--------|:--------|
| `ingenic-sdk` | Core Ingenic ISP/AVPU kernel modules and sensor drivers — adapts to SoC family and kernel version |
| `ingenic-lib` | Ingenic userspace libraries |
| `ingenic-musl` | musl compatibility shims for Ingenic SDK |
| `ingenic-pwm` | PWM controller driver |
| `ingenic-audiodaemon` | Audio subsystem daemon |
| `ingenic-diag-tools` | Hardware diagnostic utilities |
| `ingenic-libimp-control` | IMP (Ingenic Media Platform) control library |

### 6.4 Connectivity & Management

| Package | Purpose |
|:--------|:--------|
| `thingino-webui` | Browser-based management interface (CGI-based with Auburn shell) |
| `thingino-uhttpd` / `thingino-webserver` | HTTP server |
| `thingino-httpd-ssl` | HTTPS support |
| `thingino-onvif` | ONVIF protocol support for interoperability |
| `thingino-mosquitto` | MQTT broker/client |
| `thingino-libwebsockets` | WebSocket support |
| `telegrambot` | Telegram bot integration for alerts |
| `thingino-vpn` / `thingino-tailscale` | VPN connectivity |
| `thingino-esphome` | ESPHome integration |
| `thingino-prusa-connect` | Prusa 3D printer camera integration |
| `thingino-provision` | Device provisioning |
| `lightnvr` | Lightweight NVR functionality |

### 6.5 Hardware Feature Packages

| Package | Purpose |
|:--------|:--------|
| `thingino-motors` | Pan/Tilt/Zoom motor control (GPIO, PWM/TCU, SPI ms419xx) |
| `thingino-button` | Physical button GPIO handler |
| `thingino-gpio` | GPIO management utilities |
| `thingino-ledd` | LED daemon |
| `thingino-daynightd` | Day/night mode switching (IR cut filter control) |
| `thingino-dusk2dawn` | Astronomical dusk/dawn calculations |
| `thingino-mmc` | SD/MMC card support |
| `thingino-ethernet` | Ethernet networking |
| `thingino-sounds` | Audio playback support |
| `thingino-bluetooth` / `thingino-bluez` / `thingino-nimble` / `thingino-libble` | Bluetooth stack options |
| `exfat-nofuse` | exFAT filesystem (kernel module, no FUSE) |
| `wyze-accessory` | Wyze camera accessory support |

### 6.6 Development & Debugging

| Package | Purpose |
|:--------|:--------|
| `thingino-developer` | Development tools meta-package |
| `thingino-devscripts` | Development helper scripts |
| `thingino-rt-tests` | Real-time kernel testing |
| `thingino-diag` | System diagnostics |
| `gadget-serial` | USB serial gadget for debugging |
| `nino` | Utility tool |

---

## 7. ISP (Image Signal Processor) Configuration

The ISP configuration is one of the most complex aspects of the system. It is resolved in `thingino.mk` from Kconfig variables and exported for use by the `ingenic-sdk` package.

### 7.1 Clock Configuration

Multiple clock domains are configurable:
- **AVPU Clock** (Video Processing Unit): 400–700 MHz, source selectable (APLL/MPLL/VPLL/Internal)
- **ISP Clock**: 90–350 MHz, source selectable
- **ISP Scaler Clock**: 400–700 MHz (T23/T41 only)
- **ISP AXI Bus Clock**: 400–700 MHz (T23/T41 only)
- **IPU Clock** (A1 only): 400–650 MHz

### 7.2 Memory Configuration

- **rmem** — ISP reserved RAM (10–64 MB depending on SoC RAM)
- **ispmem** — ISP-specific memory (T10/T20 only, default 8 MB)
- **nmem** — Neural network reserved memory (T40/T41 only)
- **ISP Memory Optimization** — 4 levels (disabled, bypass Lynne+BGM, bypass Lynne+BGM+Ass, normal)

### 7.3 Advanced ISP Features

- Direct/Semi-Direct/Non-Direct modes (T23/T41)
- Day/Night switch frame dropping
- Pre-dequeue timing
- IVDC memory line and threshold configuration
- ISP output cropping and scaling (T30)
- Multi-sensor MIPI switching (T23)
- Configurable debug print levels (0–3)

---

## 8. Ingenic SDK Integration

The `ingenic-sdk` package is the bridge between Thingino and the proprietary Ingenic hardware:

- **SDK Versions** are matched per SoC family (e.g., T31 uses SDK 1.1.6, T41 uses 1.2.0, A1 uses 1.6.2)
- **GCC Compatibility** — SDK libraries were compiled with specific GCC versions (4.7.2, 5.4.0, or 7.2.0), and the build system ensures the toolchain matches
- **C Library** — Supports musl, glibc, and uClibc, with appropriate shims
- **Sensor Drivers** — Binary `.bin` sensor drivers are selected based on the `SENSOR_*_MODEL` variables

---

## 9. Toolchain

Thingino uses **external cross-compilation toolchains** provided per SoC family:

- Older SoCs (T10–T30): GCC 4.7.2 or 5.4.0 (XBurst1)
- Newer SoCs (T40/T41/A1): GCC 7.2.0 (XBurst2)
- C libraries: musl (default for most), uClibc, or glibc
- Architecture: MIPS32 (little-endian, soft-float typically)

---

## 10. Deployment & OTA

### 10.1 Flashing Methods

| Method | Make Target | Description |
|:-------|:------------|:------------|
| OTA Update | `make update_ota IP=x.x.x.x` | Flash kernel+rootfs over network |
| OTA Upgrade | `make upgrade_ota IP=x.x.x.x` | Flash full image over network |
| OTA Boot Update | `make upboot_ota IP=x.x.x.x` | Flash bootloader over network |
| TFTP Upload | `make upload_tftp` | Upload to TFTP server |
| TFTP Server | `make tftpd-start` | Start local TFTP server |

### 10.2 OTA Process (`scripts/fw_ota.sh`)

1. SCP the firmware binary to the camera
2. Verify `IMAGE_ID` compatibility
3. Validate SHA256 checksum
4. Invoke `sysupgrade` on the device
5. The device handles flash writing and reboots

### 10.3 QEMU Support

`make run <binary>` allows running target binaries under QEMU user-mode emulation using the build sysroot, enabling testing without physical hardware.

---

## 11. Docker Build Environment

A fully containerized build environment is provided:

- **Base:** Debian Trixie (Testing)
- **Dockerfile** installs all build dependencies: cross-compilers, Go, Python, host utilities
- **`Makefile.docker`** provides targets:
  - `docker-make` — Run build inside container
  - `docker-shell` — Interactive shell inside container
  - `docker-menuconfig` — Run Buildroot menuconfig via X11/TTY
- **Non-root builder** user inside the container for safety

---

## 12. CI/CD (GitHub Actions)

The `.github/` directory contains workflows for:
- **Toolchain builds** (`toolchain.yaml`, `toolchain-x86_64.yaml`) — Pre-build cross-compilation toolchains
- **Firmware builds** (`firmware.yaml`, `firmware-stable.yml`) — Automated builds for supported cameras
- Camera configs in `configs/github/` are specifically structured for CI builds
- Build caches can be downloaded via `make download-cache`

---

## 13. User Customization

Users can override any aspect of the build without modifying tracked files:

| File | Purpose |
|:-----|:--------|
| `user/local.fragment` | Additional Buildroot config options |
| `user/local.uenv.txt` | Additional U-Boot environment variables |
| `user/local.thingino.json` | Camera-specific JSON config overrides |
| `user/overlay/` | Additional files to merge into the root filesystem |
| `local.mk` | Buildroot local.mk for package source overrides |

All of these are excluded from `release` builds.

---

## 14. Key Design Decisions & Architectural Patterns

1. **Composition over Monolith** — Camera configs are tiny files that reference shared fragments, making it trivial to add new camera support.

2. **Layered Configuration** — JSON configs (`thingino.json`) are merged at build time using `jct`, allowing common → camera-specific → user-local override chains.

3. **Read-Only Root + Writable Overlay** — SquashFS base with JFFS2 overlay via `overlayfs`, enabling factory reset while persisting user changes.

4. **Hardware Abstraction via Make Variables** — `SOC_FAMILY`, `SOC_MODEL`, `KERNEL_VERSION`, `SENSOR_*_MODEL` propagate through the entire build system, allowing packages to adapt their behavior.

5. **Self-Contained External Tree** — The entire firmware customization exists outside of Buildroot, making Buildroot updates straightforward (it's a git submodule: `chmod -R a-w` is applied after update to prevent accidental modification).

6. **Smart Configuration Caching** — The `check-config` / `force-config` system tracks input file modification times to avoid unnecessary configuration regeneration.

7. **Multiple Flash Support** — SPI NOR and SPI NAND are both supported, with U-Boot board names automatically adjusted.

---

## 15. Notable Technical Details

- **Flash size handling:** Default 8 MB, with 32 KB alignment. The build system validates that the assembled firmware fits within the flash and warns about oversized builds.
- **Extras partition:** `/opt` content is extracted from the rootfs into a separate JFFS2 partition, keeping the SquashFS rootfs smaller and allowing writable opt space.
- **Kernel sources:** Hosted at `github.com/gtxaspec/thingino-linux` with branches per SoC family. The kernel hash is resolved at build time via `git ls-remote`.
- **U-Boot sources:** Hosted at `github.com/gtxaspec/ingenic-u-boot-{xburst1,xburst2}` with branches per SoC.
- **Interactive camera selection:** `board.mk` supports `fzf`, `whiptail`, and `dialog` for terminal-based device selection, with a memo file (`/tmp/thingino-board.*`) to remember the last selection.
- **Dependency checking:** `scripts/dep_check.sh` supports Debian, RHEL, Arch, Alpine, and OpenSUSE, ensuring all host build dependencies are present.

---

*Report generated on 2026-03-05 by deep analysis of the thingino-firmware repository (master branch).*
