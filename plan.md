# Plan: Add defconfig for 360 AP6PCM03 (T31X, GC4653, ETH+ATBM6031)

## 1. Background 

The **360 AP6PCM03** is a camera based on the **T31X** SoC with a **GC4653** image sensor and **ATBM6031** WiFi, identical to the existing `360_ap1pa3_t31x_gc4653_atbm6031` configuration but with the **addition of an Ethernet port**. The user wants to be able to choose between WiFi and Ethernet as needed at runtime, and ensure ethernet does not block the boot process.

### 1.1 Reference Configurations

| Config | SoC | Sensor | WiFi | WiFi Bus | Ethernet | MMC1 Pin Group |
|:-------|:----|:-------|:-----|:---------|:---------|:---------------|
| `360_ap1pa3_t31x_gc4653_atbm6031` | T31X | GC4653 | ATBM6031 | **SDIO** | No | **PA** (Port A) |
| `zte_k540_t31x_sc4336_eth+atbm6032` | T31X | SC4336 | ATBM6032 | **USB** | Yes | None (no MMC1) |
| `sim_simp400t_t31x_gc4653_eth+ssv6155p` | T31X | GC4653 | SSV6155P | **USB** | Yes | None (no MMC1) |
| `feisda_hd620_t31x_jxq03_eth+rtl8188ftv` | T31X | JXQ03 | RTL8188FTV | **USB** | Yes | None (no MMC1) |
| `tptek_wor504jch_t31x_gc4653_eth+rtl8731` | T31X | GC4653 | RTL8731 | **USB** | Yes | None (no MMC1) |
| `vanhua_h53e_t31x_gc4023_eth_rtl8188ftv` | T31X | GC4023 | RTL8188FTV | **SDIO** | Yes | **PB** (Port B) |

---

## 2. CRITICAL: GMAC vs WiFi SDIO GPIO Pin Conflict Analysis

### 2.1 The Problem

On the T31X SoC, the **GMAC RMII** (Ethernet MAC) interface uses **Port A** pins (PA0–PA11). The ATBM6031 WiFi module connects via **SDIO** (confirmed in `package/wifi/wifi.mk:26`: `WIFI_ADD_DRIVER,BR2_PACKAGE_WIFI_ATBM6031,atbm6031,sdio`).

The base `360_ap1pa3` configuration uses **`MMC1_PA_4BIT`** — meaning the WiFi SDIO bus is routed through **Port A** pins. This **directly conflicts** with the GMAC RMII pins on the same port.

**Evidence from the codebase:**
- `package/thingino-kopt/thingino-kopt.mk:87` → `MMC1_PA_4BIT` enables `CONFIG_JZMMC_V12_MMC1_PA_4BIT`
- `package/thingino-kopt/thingino-kopt.mk:15-20` → `JZ_MAC_V13` enables `CONFIG_JZ_MAC`, `CONFIG_JZ_MAC_RMII` (RMII uses Port A)
- **Every existing T31X+ETH camera that uses SDIO WiFi routes it via Port B (`MMC1_PB_4BIT`)**, never Port A
- **No existing camera in the entire project combines GMAC + MMC1_PA** — they are mutually exclusive

### 2.2 How Existing Cameras Avoid This Conflict

| Camera | WiFi Bus | MMC1 Config | GMAC Conflict? |
|:-------|:---------|:------------|:---------------|
| ZTE K540 (eth+atbm6032) | USB | No MMC1 at all | No — WiFi is USB, not SDIO |
| SIM SIMP400T (eth+ssv6155p) | USB | No MMC1 at all | No — WiFi is USB |
| Feisda HD620 (eth+rtl8188ftv) | USB | No MMC1 at all | No — WiFi is USB |
| TPTEK WOR504JCH (eth+rtl8731) | USB | No MMC1 at all | No — WiFi is USB |
| Vanhua H53E (eth+rtl8188ftv) | SDIO | **MMC1_PB_4BIT** | No — WiFi SDIO on Port B |
| 360 AP1PA3 (wifi only) | SDIO | **MMC1_PA_4BIT** | N/A — no ethernet |

### 2.3 Resolution: GMAC and ATBM6031 SDIO Cannot Run Simultaneously

Since both GMAC RMII and MMC1_PA_4BIT use the same Port A pins on the T31X, **they cannot be active at the same time**. The AP6PCM03 hardware has both an ethernet PHY and an ATBM6031 SDIO WiFi module wired to the board, but port-level pin muxing means only one can own Port A at a time.

**Two possible hardware scenarios for the AP6PCM03:**

1. **WiFi is on Port A, Ethernet is on Port A (shared pins)** — Only one can be active. The kernel must be compiled with either `MMC1_PA_4BIT` (WiFi) or `JZ_MAC_RMII` (Ethernet), not both. Switching requires runtime pin mux reconfiguration or loading/unloading kernel modules.

2. **WiFi is rewired to Port B on the AP6PCM03 PCB** — If the hardware designers moved WiFi SDIO to Port B (like the Vanhua H53E), then both can coexist. In this case, change `MMC1_PA_4BIT` to `MMC1_PB_4BIT`.

**Action required:** The user must determine from the AP6PCM03 hardware schematic or PCB inspection which port the ATBM6031 SDIO is wired to. The defconfig below assumes **Scenario 1** (shared Port A, mutual exclusion) since the AP6PCM03 is described as "similar to AP1PA3."

### 2.4 Chosen Approach: Build Both Modules, User Chooses at Runtime

Since the user wants to choose between WiFi and Ethernet as needed, the defconfig will:

1. **Include both WiFi SDIO and GMAC kernel modules** in the firmware image
2. **Only load one at a time** at boot, controlled by a runtime setting
3. The init system will check a persistent flag to decide which interface to activate

> **If** hardware testing confirms WiFi is on Port B (`MMC1_PB_4BIT`), change the defconfig accordingly and both interfaces can run simultaneously — no runtime switching needed.

---

## 3. Ethernet Boot-Blocking Prevention

### 3.1 The Problem

The current `S40network` init script (`overlay/etc/init.d/S40network`) calls `ifup -v -a` which brings up **all** interfaces marked `auto` in `/etc/network/interfaces`. The default `eth0` config (`overlay/etc/network/interfaces.d/eth0`) is:

```
auto eth0
iface eth0 inet dhcp
   dhcp-v6-enabled true
```

If ethernet is enabled but:
- The cable is unplugged
- The PHY is not responding
- DHCP server is unreachable

Then `udhcpc` will block the boot process while retrying DHCP lease acquisition. The `S40network` script has **no timeout** for this operation.

### 3.2 How S40network Works (current behavior)

```
S40network start
  ├── Check /run/wlan_ap_mode → exit if AP mode
  ├── Check gpio.eth.toggle in thingino.json → toggle PHY reset GPIO if configured
  └── ifup -v -a → brings up ALL "auto" interfaces (eth0, wlan0, etc.)
      ├── eth0: pre-up → set MAC address from ethaddr env
      ├── eth0: dhcp → udhcpc starts, can block if no link/DHCP
      └── wlan0: brought up by wpa_supplicant/wifi scripts
```

### 3.3 Solution: Disable eth0 Auto-Start, Add Timeout Protection

**Two-layer protection:**

#### Layer 1: Do NOT mark eth0 as `auto` — use `manual` instead

Create a custom `eth0` interface config in the camera overlay that replaces the default `auto eth0`:

**File:** camera overlay or `thingino-camera.json` approach — provide a custom `/etc/network/interfaces.d/eth0`:

```
# Ethernet is NOT auto-started to avoid blocking boot.
# Start manually: ifup eth0
# Or enable auto-start: change "manual" to "auto" below
manual eth0
iface eth0 inet dhcp
   pre-up timeout 5 ip link set dev eth0 up || true
   pre-up sleep 2
   pre-up carrier_check() { cat /sys/class/net/eth0/carrier 2>/dev/null; }; \
          [ "$(carrier_check)" = "1" ] || { echo "eth0: no link, skipping"; exit 1; }
```

This way:
- `ifup -a` during boot will **skip eth0** (it's `manual`, not `auto`)
- User can bring it up manually: `ifup eth0`
- User can switch to auto later by editing the file on the writable overlay

#### Layer 2: Add `gpio.eth.toggle` to thingino-camera.json (if PHY has reset GPIO)

If the AP6PCM03 has a GPIO-controlled ethernet PHY power/reset pin, add it to `thingino-camera.json`:

```json
{
  "gpio": {
    ...existing gpio entries...,
    "eth": {
      "toggle": true,
      "pin": <PHY_RESET_GPIO>,
      "active_low": true
    }
  }
}
```

This allows `S40network` to properly reset the PHY before bringing up the interface.

#### Layer 3: udhcpc timeout via command-line options

Even if eth0 is set to `auto`, the DHCP client can be configured with a timeout. The `udhcpc` default script (`overlay/usr/share/udhcpc/default.script`) already handles interface-type-based routing metrics (wired=100, wireless=200), so eth0 gets routing priority when active.

To add a DHCP timeout, the eth0 interface config can include:

```
auto eth0
iface eth0 inet dhcp
   udhcpc_opts -t 3 -T 3 -A 5 -b
```

Where:
- `-t 3` — max 3 DHCP discover retries
- `-T 3` — 3 second timeout between retries
- `-A 5` — 5 second timeout after failure before retry
- `-b` — background after failure (don't block)

This limits the boot delay to a maximum of ~15 seconds even if DHCP fails, then backgrounds.

---

## 4. Naming Convention

Following the project's dual-interface naming pattern:

**New config directory name:** `360_ap6pcm03_t31x_gc4653_eth+atbm6031`

Convention: `<vendor>_<model>_<soc>_<sensor>_eth+<wifi_chip>`

---

## 5. Files to Create

All files go under:
```
configs/cameras/360_ap6pcm03_t31x_gc4653_eth+atbm6031/
```

### 5.1 Defconfig (`360_ap6pcm03_t31x_gc4653_eth+atbm6031_defconfig`)

```
# NAME: 360 AP6PCM03 (T31X, GC4653, ETH, ATBM6031)
# FRAG: soc-xburst1 toolchain-xburst1 ccache brand rootfs kernel system target uboot ssl
BR2_ETHERNET=y
BR2_ISP_CLK_200MHZ=y
BR2_PACKAGE_THINGINO_KOPT_DWC2=y
BR2_PACKAGE_THINGINO_KOPT_DWC2_OTG=y
BR2_PACKAGE_THINGINO_KOPT_JZ_MAC=y
BR2_PACKAGE_THINGINO_KOPT_JZ_MAC_V13=y
BR2_PACKAGE_THINGINO_KOPT_MMC0=y
BR2_PACKAGE_THINGINO_KOPT_MMC0_PB_4BIT=y
BR2_PACKAGE_THINGINO_KOPT_MMC1=y
BR2_PACKAGE_THINGINO_KOPT_MMC1_PA_4BIT=y
BR2_PACKAGE_WIFI=y
BR2_PACKAGE_WIFI_ATBM6031=y
BR2_SENSOR_1_NAME="gc4653"
BR2_SENSOR_PARAMS="shvflip=1"
BR2_SOC_INGENIC_T31X=y
BR2_THINGINO_AUDIO=y
BR2_THINGINO_AUDIO_GPIO=63
BR2_THINGINO_BUTTON=y
BR2_THINGINO_MOTORS=y
BR2_THINGINO_PWM_ENABLE=y
BR2_THINGINO_RMEM_MB=48
BR2_THINGINO_SDCARD=y
FLASH_SIZE_MB=16
```

**Changes from `360_ap1pa3` base:**

| Line | Change | Reason |
|:-----|:-------|:-------|
| `# NAME:` | Updated to "360 AP6PCM03 (T31X, GC4653, ETH, ATBM6031)" | Identifies the new camera model |
| `BR2_ETHERNET=y` | **Added** | Enables ethernet, auto-selects U-Boot ETH + thingino-ethernet package |
| `BR2_PACKAGE_THINGINO_KOPT_JZ_MAC=y` | **Added** | Enables Ingenic MAC kernel module |
| `BR2_PACKAGE_THINGINO_KOPT_JZ_MAC_V13=y` | **Added** | Selects V1.3 MAC driver (standard for T31X, RMII mode, 50 MHz clock) |

Everything else (fragments, SoC, sensor, WiFi, audio, motors, MMC, PWM, flash size) is **identical** to the `360_ap1pa3` base.

> **GPIO Conflict Note:** Both `MMC1_PA_4BIT` (WiFi SDIO) and `JZ_MAC_RMII` (Ethernet) use Port A pins. At the kernel level, both modules are compiled into the image, but **only one should be loaded at runtime**. See Section 6 for runtime switching. If hardware testing confirms WiFi SDIO is wired to Port B, change `BR2_PACKAGE_THINGINO_KOPT_MMC1_PA_4BIT=y` to `BR2_PACKAGE_THINGINO_KOPT_MMC1_PB_4BIT=y` and both can coexist.

### 5.2 U-Boot Environment (`360_ap6pcm03_t31x_gc4653_eth+atbm6031.uenv.txt`)

```
gpio_button=7
gpio_button_call=47
gpio_default=25ID 26IDU 6i 48i 49o 50o 51o 53o 53O 54o 57o 58o 62o
```

Copied from `360_ap1pa3`. Adjust after hardware testing if the AP6PCM03 has different GPIO assignments.

### 5.3 Camera JSON (`thingino-camera.json`)

```json
{
  "gpio": {
    "button_reset": 7,
    "button_call": 47,
    "ir940": 14,
    "ircut": "59 60",
    "led_b": 51,
    "led_g": 54,
    "mmc_cd": 61,
    "mmc_power": {
      "active_low": true,
      "pin": 52
    },
    "wlan": 53
  }
}
```

Copied from `360_ap1pa3`. If the AP6PCM03 has a GPIO-controlled ethernet PHY reset, add:

```json
    "eth": {
      "toggle": true,
      "pin": <PHY_RESET_GPIO>,
      "active_low": true
    }
```

This is read by `S40network` to reset the PHY before bringing up eth0.

### 5.4 Motors JSON (`motors.json`)

```json
{
  "motors": {
    "gpio_invert": false,
    "gpio_limit_pan_1": 6,
    "gpio_limit_pan_2": 48,
    "gpio_pan": "49 50 57 58",
    "gpio_switch": 62,
    "gpio_tilt": "49 50 57 58",
    "homing": true,
    "is_spi": false,
    "pos_0": "3150,1250",
    "speed_pan": 700,
    "speed_tilt": 700,
    "steps_pan": 6300,
    "steps_tilt": 2500
  }
}
```

Copied directly from `360_ap1pa3`.

---

## 6. Runtime Interface Switching (WiFi vs Ethernet)

### 6.1 Why This Is Needed

Since GMAC RMII and MMC1_PA (ATBM6031 SDIO) share Port A pins, loading both kernel modules simultaneously will cause bus contention and undefined behavior. The user must choose one at boot time.

### 6.2 Recommended Approach: Init Script with Persistent Flag

Create a custom init script that runs before `S40network` (e.g., `S39netmode`) to configure which interface to activate:

**Persistent flag file:** `/etc/overlay/netmode` (on the writable JFFS2 overlay)
- Contains `wifi` (default) or `ethernet`

**Init script logic (`S39netmode`):**

```sh
#!/bin/sh
# S39netmode - Choose between WiFi and Ethernet before S40network runs

NETMODE_FILE="/etc/overlay/netmode"
NETMODE=$(cat "$NETMODE_FILE" 2>/dev/null || echo "wifi")

case "$NETMODE" in
    ethernet)
        # Disable WiFi SDIO (don't load MMC1/ATBM driver)
        # Remove wlan0 auto-start
        echo "Network mode: Ethernet"
        # Ensure eth0 is set to auto
        if [ -f /etc/network/interfaces.d/eth0 ]; then
            sed -i 's/^manual eth0/auto eth0/' /etc/network/interfaces.d/eth0
        fi
        # Prevent WiFi module from loading
        echo "blacklist atbm603x_wifi_sdio" > /tmp/wifi-blacklist.conf
        ;;
    wifi|*)
        # Default: WiFi mode, disable Ethernet MAC
        echo "Network mode: WiFi"
        # Ensure eth0 does NOT auto-start
        if [ -f /etc/network/interfaces.d/eth0 ]; then
            sed -i 's/^auto eth0/manual eth0/' /etc/network/interfaces.d/eth0
        fi
        # Prevent JZ_MAC from loading
        echo "blacklist jz_mac" > /tmp/eth-blacklist.conf
        ;;
esac
```

**User switches modes via:**
```sh
echo "ethernet" > /etc/overlay/netmode && reboot
echo "wifi" > /etc/overlay/netmode && reboot
```

### 6.3 Alternative: WebUI Toggle

If the thingino-webui supports custom settings pages, a toggle switch could be added to the web interface to set the `netmode` flag and trigger a reboot. This is a future enhancement.

---

## 7. Ethernet Boot-Blocking Prevention (Detailed)

### 7.1 Problem Analysis

The boot sequence relevant to networking:

```
S40network start
  └── ifup -v -a
      └── For each "auto" interface:
          └── udhcpc (DHCP client) starts
              └── Can block indefinitely if:
                  - No cable plugged in (no carrier)
                  - PHY not responding
                  - DHCP server unreachable
```

Current `overlay/etc/network/interfaces.d/eth0`:
```
auto eth0
iface eth0 inet dhcp
   dhcp-v6-enabled true
```

The `auto` keyword means `ifup -a` will try to bring up eth0 during boot. If DHCP fails, it blocks.

### 7.2 Solution: Custom eth0 Interface Config with Timeout + Carrier Check

Replace the default eth0 interface file with a safer version that:
1. Checks for link carrier before attempting DHCP
2. Uses udhcpc timeout options to prevent indefinite blocking
3. Backgrounds udhcpc on failure

**Custom `/etc/network/interfaces.d/eth0` (installed via camera overlay):**

```
auto eth0
iface eth0 inet dhcp
   # Timeout: max 3 retries, 3s between, background on failure
   udhcpc_opts -t 3 -T 3 -A 5 -b
   # Check for carrier (cable plugged in) before DHCP
   pre-up ip link set dev eth0 up
   pre-up sleep 2
   pre-up [ "$(cat /sys/class/net/eth0/carrier 2>/dev/null)" = "1" ] || { logger -t eth0 "No carrier, skipping DHCP"; exit 1; }
```

**Behavior:**
- `pre-up`: Brings eth0 link up, waits 2 seconds for carrier detection
- If no cable is plugged in (`carrier != 1`), skips DHCP entirely — **zero boot delay**
- If cable is present but DHCP fails: retries 3 times at 3s intervals, then backgrounds (`-b`) — **max ~15 seconds delay**, then boot continues
- The `exit 1` in pre-up causes `ifup` to skip this interface cleanly

### 7.3 Where to Place the Custom eth0 Config

**Option A (Recommended): Camera-specific overlay directory**

If Thingino supports per-camera overlay files, place it at:
```
configs/cameras/360_ap6pcm03_t31x_gc4653_eth+atbm6031/overlay/etc/network/interfaces.d/eth0
```

This overrides the default `overlay/etc/network/interfaces.d/eth0` for this camera only.

**Option B: Modify thingino-camera.json**

If overlay per-camera is not supported, the eth0 file can be installed via a post-build script or package hook.

**Option C: Use the writable overlay**

After first boot, the user can manually edit `/etc/network/interfaces.d/eth0` on the JFFS2 overlay — changes persist across reboots but not factory resets.

---

## 8. Implementation Steps

### Step 1: Create the config directory
```bash
mkdir -p configs/cameras/360_ap6pcm03_t31x_gc4653_eth+atbm6031
```

### Step 2: Create the defconfig
Create `360_ap6pcm03_t31x_gc4653_eth+atbm6031_defconfig` with the content from Section 5.1.

### Step 3: Copy supporting files from `360_ap1pa3`
- Create `360_ap6pcm03_t31x_gc4653_eth+atbm6031.uenv.txt` (Section 5.2)
- Create `thingino-camera.json` (Section 5.3)
- Create `motors.json` (Section 5.4)

### Step 4: Create custom eth0 interface config (boot-blocking prevention)
Place the custom eth0 config with carrier check and udhcpc timeout (Section 7.2) in the appropriate overlay location.

### Step 5: (Optional) Create S39netmode init script
If runtime WiFi/Ethernet switching is needed due to Port A pin conflict, create the switching script (Section 6.2).

### Step 6: Hardware testing
1. **Determine WiFi SDIO port**: Check if ATBM6031 is on Port A or Port B on the AP6PCM03 PCB
   - If Port B: change `MMC1_PA_4BIT` → `MMC1_PB_4BIT`, both interfaces can coexist, skip Step 5
   - If Port A: keep `MMC1_PA_4BIT`, runtime switching is required (Step 5)
2. **Test ethernet PHY**: Verify JZ_MAC V13 with 50 MHz clock works with the AP6PCM03's PHY
3. **Identify PHY reset GPIO**: If present, add `gpio.eth.toggle` to `thingino-camera.json`
4. **Test boot with cable unplugged**: Verify carrier check prevents DHCP blocking

### Step 7: Build and verify
```bash
make CAMERA=360_ap6pcm03_t31x_gc4653_eth+atbm6031
```

---

## 9. What Happens at Build Time

When `BR2_ETHERNET=y` is set, the build system automatically:

1. **U-Boot**: Enables ethernet support (`BR2_PACKAGE_THINGINO_UBOOT_ETH_ENABLE` auto-selected)
2. **Kernel**: Compiles `jz_mac` module with V1.3 driver, RMII mode, 50 MHz clock (default)
3. **Rootfs**: Installs `thingino-ethernet` package which enables BusyBox `IFPLUGD` for link detection
4. **Network**: `eth0` interface configured for DHCP with MAC from U-Boot env (`ethaddr`)
5. **Routing**: `udhcpc` default script (`overlay/usr/share/udhcpc/default.script`) sets route metrics — wired=100, wireless=200 — so eth0 gets routing priority when both are active

No fragment changes needed — the same `soc-xburst1 toolchain-xburst1 ccache brand rootfs kernel system target uboot ssl` fragment set handles everything.

---

## 10. Risks and Considerations

| Risk | Severity | Mitigation |
|:-----|:---------|:-----------|
| **GMAC and WiFi SDIO Port A pin conflict** | **HIGH** | Both use Port A on T31X. Only load one module at a time (Section 6). Verify hardware schematic — if WiFi is on Port B, no conflict exists. |
| **Ethernet DHCP blocking boot** | MEDIUM | Custom eth0 config with carrier check + udhcpc timeout `-t 3 -T 3 -b` limits max delay to ~15s, or 0s if no cable (Section 7). |
| **PHY not initializing without GPIO toggle** | MEDIUM | Check if AP6PCM03 has a PHY reset GPIO. If so, add `gpio.eth.toggle` to `thingino-camera.json` so `S40network` resets it before ifup. |
| **Different GPIO mappings on AP6PCM03 vs AP1PA3** | LOW | `uenv.txt` and JSON files may need adjustment. Start with AP1PA3 values, test on hardware. |
| **Flash size from added ethernet + both modules** | LOW | Ethernet adds ~50 KB; 16 MB flash has ample room. Both JZ_MAC and ATBM6031 modules are small. |
| **Routing conflicts when both eth0 and wlan0 active** | LOW | Already handled by `udhcpc/default.script` — wired metric=100, wireless=200. Eth0 wins by default. |

---

## 11. File Summary

| File | Action | Source |
|:-----|:-------|:-------|
| `configs/cameras/360_ap6pcm03_t31x_gc4653_eth+atbm6031/360_ap6pcm03_t31x_gc4653_eth+atbm6031_defconfig` | **Create** | Based on `360_ap1pa3` + 3 ethernet lines |
| `configs/cameras/360_ap6pcm03_t31x_gc4653_eth+atbm6031/360_ap6pcm03_t31x_gc4653_eth+atbm6031.uenv.txt` | **Create** | Copy from `360_ap1pa3` |
| `configs/cameras/360_ap6pcm03_t31x_gc4653_eth+atbm6031/thingino-camera.json` | **Create** | Copy from `360_ap1pa3`, add `gpio.eth` if PHY reset GPIO exists |
| `configs/cameras/360_ap6pcm03_t31x_gc4653_eth+atbm6031/motors.json` | **Create** | Copy from `360_ap1pa3` |
| Custom `eth0` interface config | **Create** | New file with carrier check + udhcpc timeout (Section 7.2) |
| `S39netmode` init script (if Port A conflict confirmed) | **Create** | New file for WiFi/Ethernet runtime switching (Section 6.2) |

---

## 12. Decision Tree Summary

```
Does the AP6PCM03 wire ATBM6031 SDIO to Port A or Port B?
│
├── Port A (same as AP1PA3) ──────────────────────────────────┐
│   GMAC RMII conflicts with WiFi SDIO.                       │
│   ├── Use MMC1_PA_4BIT in defconfig                         │
│   ├── Create S39netmode for runtime switching                │
│   ├── Default to WiFi mode (safer for IP camera)            │
│   └── User switches with: echo "ethernet" > /etc/overlay/netmode && reboot
│
└── Port B (different from AP1PA3) ───────────────────────────┐
    No conflict. Both can coexist.                            │
    ├── Change to MMC1_PB_4BIT in defconfig                   │
    ├── No switching script needed                            │
    ├── Both eth0 and wlan0 auto-start                        │
    └── Routing metrics handle priority (eth0=100, wlan0=200) │
```
