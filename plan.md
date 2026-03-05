# Plan: Add defconfig for 360 AP6PCM03 (T31X, GC4653, ETH+ATBM6031)

## 1. Background

The **360 AP6PCM03** is a camera based on the **T31X** SoC with a **GC4653** image sensor and **ATBM6031** WiFi, identical to the existing `360_ap1pa3_t31x_gc4653_atbm6031` configuration but with the **addition of an Ethernet port**.

### 1.1 Reference Configurations

| Config | SoC | Sensor | WiFi | Ethernet | Role |
|:-------|:----|:-------|:-----|:---------|:-----|
| `360_ap1pa3_t31x_gc4653_atbm6031` | T31X | GC4653 | ATBM6031 | No | **Primary base** — same camera family |
| `zte_k540_t31x_sc4336_eth+atbm6032` | T31X | SC4336 | ATBM6032 | Yes | **Ethernet pattern** — T31X with ATBM WiFi + ETH |
| `sim_simp400t_t31x_gc4653_eth+ssv6155p` | T31X | GC4653 | SSV6155P | Yes | **Same sensor + ETH** — GC4653 with ethernet |
| `feisda_hd620_t31x_jxq03_eth+rtl8188ftv` | T31X | JXQ03 | RTL8188FTV | Yes | **Full-featured ETH** — DWC2 + JZ_MAC + ETH |
| `tptek_wor504jch_t31x_gc4653_eth+rtl8731` | T31X | GC4653 | RTL8731 | Yes | **Same sensor + ETH** — GC4653 with ethernet |

### 1.2 Ethernet Support Pattern (learned from existing cameras)

Adding Ethernet to a T31X camera requires exactly **3 additions** to the defconfig:

1. **`BR2_ETHERNET=y`** — Master ethernet enable flag (defined in `Config.soc.in:1089`).
   - Automatically selects `BR2_PACKAGE_THINGINO_UBOOT_ETH_ENABLE` (U-Boot ethernet support).
   - Via `Config.in:34`, auto-selects `BR2_PACKAGE_THINGINO_ETHERNET` which in turn selects `BR2_PACKAGE_THINGINO_KOPT` and `BR2_PACKAGE_THINGINO_KOPT_JZ_MAC`.
   - The `thingino-ethernet` package enables BusyBox `IFPLUGD` for link detection.

2. **`BR2_PACKAGE_THINGINO_KOPT_JZ_MAC=y`** — Enables the Ingenic MAC kernel module (auto-selected, but explicitly set in most configs).

3. **`BR2_PACKAGE_THINGINO_KOPT_JZ_MAC_V13=y`** — Selects the V1.3 MAC driver variant (used by all T31X ethernet cameras). The V13 variant also provides clock speed selection (25 MHz or 50 MHz, defaulting to 50 MHz).

All examined T31X+ETH cameras consistently use this exact set of additions. No fragment changes, no additional kernel patches, and no extra JSON configuration keys are needed for basic ethernet.

---

## 2. Naming Convention

Following the project's dual-interface naming pattern (e.g., `zte_k540_t31x_sc4336_eth+atbm6032`):

**New config directory name:** `360_ap6pcm03_t31x_gc4653_eth+atbm6031`

This follows the convention: `<vendor>_<model>_<soc>_<sensor>_eth+<wifi_chip>`

---

## 3. Files to Create

All files go under:
```
configs/cameras/360_ap6pcm03_t31x_gc4653_eth+atbm6031/
```

### 3.1 Defconfig (`360_ap6pcm03_t31x_gc4653_eth+atbm6031_defconfig`)

Start from the `360_ap1pa3` defconfig and add ethernet support:

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
| `# NAME:` | Updated to "360 AP6PCM03 ... ETH, ATBM6031" | Identifies the new camera model |
| `BR2_ETHERNET=y` | **Added** | Enables ethernet support, auto-selects U-Boot ETH and thingino-ethernet package |
| `BR2_PACKAGE_THINGINO_KOPT_JZ_MAC=y` | **Added** | Enables Ingenic MAC kernel module |
| `BR2_PACKAGE_THINGINO_KOPT_JZ_MAC_V13=y` | **Added** | Selects the V1.3 MAC driver (standard for T31X) |

Everything else (fragments, SoC, sensor, WiFi, audio, motors, MMC, PWM, flash size) is **identical** to the `360_ap1pa3` base.

### 3.2 U-Boot Environment (`360_ap6pcm03_t31x_gc4653_eth+atbm6031.uenv.txt`)

Copy directly from `360_ap1pa3` — the GPIO mappings for buttons and defaults should be the same hardware:

```
gpio_button=7
gpio_button_call=47
gpio_default=25ID 26IDU 6i 48i 49o 50o 51o 53o 53O 54o 57o 58o 62o
```

> **Note:** If the AP6PCM03 has different GPIO assignments for the ethernet PHY reset or LEDs, this file will need to be updated with the correct pin mappings after hardware testing. The ethernet PHY itself is typically handled by the JZ_MAC driver and doesn't need GPIO entries in `uenv.txt`.

### 3.3 Camera JSON (`thingino-camera.json`)

Copy from `360_ap1pa3` — same GPIO layout assumed:

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

> **Note:** If the AP6PCM03 has an ethernet link LED on a GPIO, it can be added here (e.g., `"eth_led": <pin>`). Most cameras rely on the PHY's built-in LED, so this is usually not needed.

### 3.4 Motors JSON (`motors.json`)

Copy directly from `360_ap1pa3` — same motor hardware assumed:

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

---

## 4. Implementation Steps

### Step 1: Create the config directory
```bash
mkdir -p configs/cameras/360_ap6pcm03_t31x_gc4653_eth+atbm6031
```

### Step 2: Create the defconfig
Create `360_ap6pcm03_t31x_gc4653_eth+atbm6031_defconfig` with the content from Section 3.1.

### Step 3: Copy supporting files from `360_ap1pa3`
- Copy and adapt `uenv.txt` (Section 3.2)
- Copy `thingino-camera.json` (Section 3.3)
- Copy `motors.json` (Section 3.4)

### Step 4: Verify the build (optional)
```bash
make CAMERA=360_ap6pcm03_t31x_gc4653_eth+atbm6031
```

---

## 5. What Happens at Build Time

When `BR2_ETHERNET=y` is set, the build system automatically:

1. **U-Boot**: Enables ethernet support (`BR2_PACKAGE_THINGINO_UBOOT_ETH_ENABLE` is auto-selected)
2. **Kernel**: Loads the `jz_mac` module with V1.3 driver and 50 MHz reference clock (default)
3. **Rootfs**: Installs the `thingino-ethernet` package which enables BusyBox `IFPLUGD` for automatic `eth0` link detection
4. **Network**: The `eth0` interface is configured for DHCP with hardware MAC address from U-Boot env (`ethaddr`)

No fragment changes are needed — the same `soc-xburst1 toolchain-xburst1 ccache brand rootfs kernel system target uboot ssl` fragment set handles everything.

---

## 6. Risks and Considerations

| Risk | Mitigation |
|:-----|:-----------|
| GPIO pin conflicts between ethernet PHY and existing peripherals | The T31X has dedicated MII/RMII pins managed by the JZ_MAC driver; no GPIO overlap with motors/LEDs/buttons expected |
| Different hardware revision may have different GPIO mappings | Document that `uenv.txt` and JSON files may need adjustment after hardware testing |
| Flash size pressure from added ethernet packages | Ethernet support adds minimal size (~50 KB); 16 MB flash has ample room |
| Network priority/routing when both eth0 and wlan0 are active | Handled by standard Linux routing; `eth0` typically gets priority via metric. BusyBox `IFPLUGD` manages link state |

---

## 7. File Summary

| File | Action | Source |
|:-----|:-------|:-------|
| `configs/cameras/360_ap6pcm03_t31x_gc4653_eth+atbm6031/360_ap6pcm03_t31x_gc4653_eth+atbm6031_defconfig` | **Create** | Based on `360_ap1pa3` + 3 ethernet lines |
| `configs/cameras/360_ap6pcm03_t31x_gc4653_eth+atbm6031/360_ap6pcm03_t31x_gc4653_eth+atbm6031.uenv.txt` | **Create** | Copy from `360_ap1pa3` |
| `configs/cameras/360_ap6pcm03_t31x_gc4653_eth+atbm6031/thingino-camera.json` | **Create** | Copy from `360_ap1pa3` |
| `configs/cameras/360_ap6pcm03_t31x_gc4653_eth+atbm6031/motors.json` | **Create** | Copy from `360_ap1pa3` |
