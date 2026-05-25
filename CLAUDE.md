# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

Linux kernel driver for the AIC8800D80 USB WiFi+Bluetooth chipset (and variants: D80N, D80X2, DC, DLN). The repo ships two kernel modules and a DKMS-based installer.

Supported USB devices: Tenda U11, AX913B, and adapters with Vendor IDs `a69c`, `368b`, and `1111` (clone/Pandora devices that need USB mode switching).

## Build & Install Commands

### Build drivers manually (without DKMS)

```sh
cd drivers/aic8800
make -j$(nproc) KDIR=/lib/modules/$(uname -r)/build
```

### Install via DKMS (recommended)

```sh
sudo bash install.sh
```

This copies firmware to `/lib/firmware/`, installs udev rules, registers the DKMS package, builds both modules, and loads them.

### Check DKMS status and build log

```sh
dkms status aic8800
cat /var/lib/dkms/aic8800/1.0.0/build/make.log
```

### Build test utilities

```sh
cd aicrf_test
make
sudo make install   # installs wifi_test and bt_test to /sbin/
```

### Bluetooth diagnostics

```sh
sudo bash diagnose_bt.sh
sudo bash diagnostic_build.sh
```

### Reload modules manually

```sh
sudo modprobe -r aic8800_fdrv aic_load_fw
sudo modprobe aic8800_fdrv
```

## Architecture

### Kernel Modules

| Module | Source | Purpose |
|---|---|---|
| `aic8800_fdrv` | `drivers/aic8800/aic8800_fdrv/` | WiFi driver (cfg80211, ~120 source files) |
| `aic_load_fw` | `drivers/aic8800/aic_load_fw/` | Bluetooth firmware uploader |

The **WiFi driver** (`aic8800_fdrv`) is based on the RivieraWaves RWNX framework. It handles USB/SDIO host interfaces and registers with `cfg80211`. The `rwnx_main.c` file is the entry point.

The **Bluetooth firmware loader** (`aic_load_fw`) uploads BT firmware to the chip over USB, then the standard Linux `btusb` driver takes over the HCI interface. The old custom `aic_btusb` module (still present in `drivers/aic8800/aic_btusb/`) is deprecated — do not use it.

### Chip Variant Compatibility

Each chip variant has its own compatibility layer file:
- `aic_compat_8800d80.c/h` — D80
- `aic_compat_8800d80x2.c/h` — D80X2

Firmware is organized per variant under `fw/aic8800D80/`, `fw/aic8800D80X2/`, etc. The `rwnx_platform.h` file defines firmware filenames and mode constants.

### Bluetooth Flow

1. Device plugs in → udev rule (`aic.rules`) triggers firmware load via `aic_load_fw`
2. Firmware upload completes → standard `btusb` driver binds to the HCI interface
3. `modprobe/aic8800-bt.conf` ensures `aic_load_fw` loads before `btusb`

For "Pandora" clone devices (USB ID `1111:1111`), `usb_modeswitch` first switches the device from mass-storage mode to WiFi+BT mode (`a69c:8d80`) before drivers bind.

### DKMS Package

- Package name: `aic8800`, version `1.0.0`
- Source copied to `/usr/src/aic8800-1.0.0/` by `install.sh`
- Modules install to `/updates/dkms/` and automatically rebuild on kernel upgrade

### Key Files for Common Tasks

- **Adding a new chip variant**: add compat files in `aic_load_fw/`, add firmware filenames in `rwnx_platform.h`, add firmware binaries under `fw/`
- **Changing modprobe/load order**: edit `modprobe/aic8800-bt.conf`
- **Changing udev binding rules**: edit `aic.rules`
- **Debugging BT firmware load**: `dmesg | grep -E 'fw_patch|fw_adid|aic'`
- **Install script logic**: `install.sh` (~650 lines) — logs to `/tmp/aic8800d80_install.log`
