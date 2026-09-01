# POCO X3 Pro (vayu) double-tap-to-wake

## Scope

This documents the working double-tap-to-wake (DT2W) implementation for the
Xiaomi POCO X3 Pro (`vayu`) running postmarketOS with Hyprland and the Novatek
NT36672 TDDI touchscreen.

The implementation was physically validated on a **Huaxing** panel. Tianma's
panel driver has the matching reset/power handling, but was not physically
tested. Do not install Tianma firmware on a Huaxing device.

Kernel packaging lives in:

```text
device/testing/linux-postmarketos-qcom-sm8150/
├── APKBUILD
└── 0006-novatek-dt2w-support.patch
```

## Result

A physical double tap while DPMS is off now:

1. reaches the Novatek interrupt handler;
2. is recognized as two nearby contacts;
3. emits `KEY_WAKEUP`;
4. reaches Hyprland; and
5. enables DPMS.

Final controlled test:

```text
before_irq=0
second=1 dpms=0
second=2 dpms=1
after_irq=38 delta=38 woke=1
[nt36672-spi] Software gesture: Double Tap
```

Validated incremental package:

```text
linux-postmarketos-qcom-sm8150-6.17.0_p20260901081036-r10.apk
SHA-256: 01431562bc076839bcb304fb8d11cad83c90d21045f4d51eaee16f86883676b0
```

The final recipe was also rebuilt cleanly with `pmbootstrap build --force`.

## Root causes

DT2W crossed four independently broken boundaries.

### 1. Userspace wake handling

Hyprland initially received no usable wake action. Synthetic `KEY_WAKEUP`
injection proved the userspace path after adding a wake binding and correcting
the Lua DPMS dispatch syntax.

The required behavior is equivalent to:

```text
KEY_WAKEUP -> hl.dsp.dpms({ action = "enable" })
```

Synthetic wake worked before physical touch wake did, proving the remaining
fault was in the kernel/TDDI path.

### 2. Shared TDDI reset

The panel and touchscreen are integrated. Pulling panel reset low during DPMS
off also resets touch, making off-screen gestures impossible.

The panel drivers now consult `nvt_ts_wakeup_enabled()`. When DT2W is enabled,
they preserve reset and avoid shutting down the integrated controller.

Correct SM8150 tiled TLMM registers used during diagnosis:

| Signal | GPIO | Register |
|---|---:|---:|
| Panel reset | 6 | `0x03d06004` |
| Touch reset | 12 | `0x0390c004` |
| Touch IRQ | 122 | `0x03d7a004` |
| Display VCI | 21 | `0x03515004` |
| Touch VDDIO | 100 | `0x03d64004` |

Panel reset remained high across DPMS off:

```text
0x00000003 -> 0x00000003
```

### 3. Display/touch shutdown ordering

The DSI manager originally emitted only an early power-down notification. The
touch driver suspended before panel/DSI shutdown completed. TDDI transitions
then generated repeated `FD`/`FE` watchdog packets and firmware recovery,
destroying the selected wake mode.

The fix adds a normal `MI_DRM_EVENT_BLANK` power-down notification after DSI
host/PHY shutdown. Touch handling is split into two phases:

1. `MI_DRM_EARLY_EVENT_BLANK`: mask the touchscreen IRQ;
2. `MI_DRM_EVENT_BLANK`: reload touch firmware, establish the off-screen mode,
   re-enable the IRQ, and arm IRQ wake.

This prevents stale reset packets from racing the suspend path.

### 4. Native firmware gesture mode did not work

Xiaomi/Novatek downstream uses host command `0x13` for wake gestures. It was
tested with both Huaxing firmware variants:

| Firmware | Version | SHA-256 | Result |
|---|---:|---|---|
| Packaged Huaxing `fw02.fd13` | 198 | `688e6ef88151f41bb09cfed8fdad0d94e7edf67a9e3bdba30a385eaa477f0086` | No gesture IRQ |
| LineageOS Huaxing `fw02/.71e7` | 9 | `6b269eb7a25204434674c53c6e4f11870965a7b784e62da00797283628b9c966` | No native gesture packet |

After fixing shutdown ordering, command `0x13` entered cleanly without a
watchdog storm, but physical double taps still emitted no gesture IRQ with
either firmware. The LineageOS firmware only produced useful interrupts while
the controller remained in normal scan mode.

The LineageOS firmware was therefore **not** packaged. The original Huaxing
firmware was restored.

## Working implementation

The final implementation uses software recognition over normal Novatek touch
reports while the display is off.

### Sysfs control

The touchscreen exposes:

```text
/sys/bus/spi/devices/spi0.0/double_tap
```

Enable DT2W with:

```sh
echo 1 | sudo tee /sys/bus/spi/devices/spi0.0/double_tap
```

Setting this also updates the device wakeup state.

### Panel behavior

When DT2W is disabled, panel unprepare follows the regular TDDI off/sleep and
reset path.

When DT2W is enabled, Huaxing and Tianma panel unprepare skip the TDDI
`DISPLAY_OFF`/`ENTER_SLEEP_MODE` sequence and preserve reset. DSI host shutdown
and DPMS still blank the visible display, while the touch portion remains able
to scan.

### Touch behavior

During DT2W suspend, the driver:

1. masks IRQ before panel/DSI shutdown;
2. reloads the original Novatek firmware after shutdown;
3. keeps the controller in normal scan mode;
4. resets software tap state;
5. re-enables IRQ; and
6. enables IRQ wake.

The software recognizer:

- scans every touch slot;
- accepts contact states `1` (enter) and `2` (moving), because the first
  observed off-screen report is not guaranteed to be state `1`;
- detects only a new contact transition, not every movement report;
- requires the second tap within **100–700 ms**;
- requires taps within **200 pixels** on each axis; and
- emits `KEY_WAKEUP` after a valid pair.

Scanning all slots and accepting states 1/2 was the final functional fix. The
first implementation assumed slot 0/state 1 and missed real taps despite a
large IRQ count.

## Power trade-off

This is not the Novatek firmware's low-power gesture mode. The touchscreen
remains in normal scan mode while DT2W is enabled, and the panel's TDDI sleep
sequence is skipped. This will consume more screen-off power than native
firmware DT2W.

That trade-off is intentional: native command `0x13` produced no physical
gesture IRQ on this Huaxing hardware with either known firmware. Disable DT2W
through sysfs when maximum standby time matters.

## Files changed by the patch

```text
drivers/gpu/drm/msm/dsi/dsi_manager.c
drivers/gpu/drm/panel/panel-huaxing-nt36672.c
drivers/gpu/drm/panel/panel-tianma-nt36672.c
drivers/input/touchscreen/nt36672/nt36xxx.c
include/linux/input/nt36672.h
```

`include/linux/input/nt36672.h` exports `nvt_ts_wakeup_enabled()` so panel
drivers can preserve the shared TDDI state only when required.

## Build and deploy

Clean recipe build:

```sh
orb -m postmarket-builder bash -lc '
  pmbootstrap checksum linux-postmarketos-qcom-sm8150
  pmbootstrap build linux-postmarketos-qcom-sm8150 --force
'
```

Incremental source-tree build:

```sh
orb -m postmarket-builder bash -lc '
  pmbootstrap build linux-postmarketos-qcom-sm8150 \
    --src=/home/steven/src/sm8150-linux
'
```

Host source edits are not automatically reflected in the VM source tree. Sync
changed files into `/home/steven/src/sm8150-linux` before an incremental build.
A stale VM source tree caused an earlier package to appear successful while
omitting the new symbols.

Deploy the resulting APK:

```sh
scp linux-postmarketos-qcom-sm8150-*.apk vayu:/tmp/
ssh -tt vayu \
  "printf '1432\n' | sudo -S apk add --allow-untrusted /tmp/linux-postmarketos-qcom-sm8150-*.apk"
ssh -tt vayu "printf '1432\n' | sudo -S reboot"
```

Confirm the installed package version before rebooting. Do not interrupt APK
triggers: `mkinitfs` and `boot-deploy` must finish and flash the complete
`boot.img`.

## Validation checklist

```sh
# Kernel and package
ssh vayu 'uname -a; apk list --installed linux-postmarketos-qcom-sm8150'

# DT2W and wakeup state
ssh vayu 'cat /sys/bus/spi/devices/spi0.0/double_tap; \
  cat /sys/bus/spi/devices/spi0.0/power/wakeup'

# Firmware identity
ssh vayu 'sha256sum /lib/firmware/novatek/j20s_novatek_ts_fw02.bin'

# Gesture result and watchdog regressions
ssh vayu "dmesg | grep -E 'Software gesture: Double Tap|Recover for fw reset|FD|FE'"

# Storage stability regression check
ssh vayu "dmesg | grep -E 'pa_err|dl_err|LINERESET|dev_reset|ufs_eh_wq' || \
  echo 'ZERO UFS ERRORS'"
```

A complete physical test must verify all three outcomes:

1. touchscreen IRQ count increases during the taps;
2. the kernel logs `Software gesture: Double Tap`; and
3. Hyprland DPMS changes from off to on.
