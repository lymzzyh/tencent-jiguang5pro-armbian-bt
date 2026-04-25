# Tencent Aurora 5 Pro Armbian Bluetooth UART helper

This repository contains the bring-up artifacts used to enable the onboard Realtek Bluetooth controller on the **Tencent Aurora 5 Pro TV box** (腾讯极光 5 Pro 盒子) when running Armbian or other Linux systems based on the Rockchip RK3588 vendor kernel.

The Bluetooth part on this board is handled as:

- Hardware/platform: Tencent Aurora 5 Pro box
- SoC family: Rockchip RK3588S/RK3588 vendor-kernel platform
- Wi-Fi/Bluetooth combo: Realtek RTL8852BE class device
- Bluetooth transport: **UART9**, not USB
- UART node observed on Linux: `/dev/ttyS9` / `febc0000.serial`
- Bluetooth protocol: Realtek H5 / three-wire
- Firmware/config names: `rtl8852bs_fw`, `rtl8852bs_config`

## What is included

```text
scripts/install_rtl8852bs_bt_uart9.sh
    One-click installer for a fresh Armbian system.
    It installs a patched rtk_hciattach binary, installs/enables a systemd service,
    installs BlueZ/rfkill when missing, and verifies hci0.

patches/rtl8852bs-max-patch-size.patch
    Minimal patch for radxa/rtkbt rtk_hciattach so CHIP_8852BS accepts a larger
    firmware/config patch payload.

src/rtk_hciattach-rtl8852bs-maxpatch-src.tar.gz
    Patched rtk_hciattach source snapshot used to build the tested binary.

artifacts/rtk_hciattach-aarch64-rtl8852bs-maxpatch
    Tested aarch64 rtk_hciattach binary built from the patched source.
    The installer embeds this payload directly, so a target board does not need
    to compile rtk_hciattach during installation.

SHA256SUMS
    Checksums for the shipped files.
```

## Quick install on a fresh Armbian system

Copy the installer to the board, then run it as root:

```bash
chmod +x scripts/install_rtl8852bs_bt_uart9.sh
sudo ./scripts/install_rtl8852bs_bt_uart9.sh install
```

If you copy only the script to the target:

```bash
chmod +x install_rtl8852bs_bt_uart9.sh
sudo ./install_rtl8852bs_bt_uart9.sh install
```

The default settings are:

```bash
TTY_DEV=/dev/ttyS9
INIT_SPEED=115200
PROTO=rtk_h5
```

They can be overridden if your device tree exposes the Bluetooth UART under a different tty:

```bash
TTY_DEV=/dev/ttyS9 INIT_SPEED=115200 PROTO=rtk_h5 sudo -E ./install_rtl8852bs_bt_uart9.sh install
```

## Verify

```bash
sudo ./scripts/install_rtl8852bs_bt_uart9.sh verify
systemctl status rtk-hciattach.service bluetooth.service
btmgmt info
hciconfig -a
bluetoothctl show
readlink -f /sys/class/bluetooth/hci0
```

Expected result:

```text
rtk-hciattach.service: active (running), enabled
bluetooth.service: active (running), enabled
btmgmt info: Index list with 1 item, hci0 Primary controller
hciconfig -a: hci0 Type: Primary Bus: UART, UP RUNNING
Manufacturer: Realtek Semiconductor Corporation
bluetoothctl show: Powered: yes
/sys/class/bluetooth/hci0 -> .../febc0000.serial/tty/ttyS9/hci0
```

## What the installer does

The installer performs these steps:

1. Checks that it is running on `aarch64`.
2. Checks `/dev/ttyS9` exists by default.
3. Checks for RTL8852BS Bluetooth firmware/config under `/lib/firmware/rtlbt/`.
4. Installs missing BlueZ/rfkill tools using `apt-get` when available.
5. Installs the patched `rtk_hciattach` to `/usr/local/bin/rtk_hciattach`.
6. Installs helper scripts:
   - `/usr/local/sbin/rtl8852bs-bt-reset.sh`
   - `/usr/local/sbin/rtl8852bs-bt-up.sh`
7. Installs and enables `/etc/systemd/system/rtk-hciattach.service`.
8. Starts `rtk-hciattach.service`, waits for `hci0`, brings it up, then starts `bluetooth.service`.
9. Prints service and HCI status for verification.

The service command is:

```bash
/usr/local/bin/rtk_hciattach -n -s 115200 /dev/ttyS9 rtk_h5
```

## Why a patched rtk_hciattach is needed

The tested Armbian firmware provides RTL8852BS Bluetooth firmware/config files. The original `rtk_hciattach` source used during bring-up rejected the firmware/config payload for `CHIP_8852BS` with:

```text
Total length of fwc is larger than allowed
```

The patch moves `CHIP_8852BS` to a larger `max_patch_size` branch:

```c
case CHIP_8852BS:
case CHIP_8852CS:
    max_patch_size = 128 * 1024;
    break;
```

With this change, the board successfully initializes the controller as Realtek H5 over UART9.

## Rebuilding rtk_hciattach from the patched source

On an aarch64 Linux system with build tools installed:

```bash
tar -xzf src/rtk_hciattach-rtl8852bs-maxpatch-src.tar.gz
cd rtk_hciattach
make clean
make
```

The resulting `rtk_hciattach` can replace the binary embedded in the installer if needed.

## Tested status

Validated on a Tencent Aurora 5 Pro box running an Armbian vendor-kernel image:

```text
Linux rk3588s-box-v10 6.1.115-vendor-rk35xx aarch64
UART: /dev/ttyS9 -> febc0000.serial
Bluetooth: hci0, Bus UART, UP RUNNING
Manufacturer: Realtek Semiconductor Corporation
Controller: Powered yes
```

The install flow was tested on a freshly flashed minimal Armbian rootfs. A reboot test was also performed; after reboot, both `rtk-hciattach.service` and `bluetooth.service` came up automatically and `hci0` remained available.

## Notes

- Do **not** use USB Bluetooth assumptions for this hardware. The working path here is UART9 + Realtek H5.
- Generic BlueZ `btattach -P 3wire` is not enough for this device because the Realtek vendor initialization and firmware download are required.
- If `hci0` does not appear, first check:
  - `/dev/ttyS9` exists
  - RTL8852BS firmware/config files exist
  - `rfkill list` is not blocking Bluetooth
  - `journalctl -u rtk-hciattach.service --no-pager -n 160`
- Device-tree RTS/GPIO details may differ across board ports. This repo only ships the user-space attach/configuration path that has been validated on the Tencent Aurora 5 Pro box.

## Provenance and licensing

The `rtk_hciattach` source snapshot is derived from Realtek/Radxa-style `rtkbt` userspace tooling used for RK3588 Bluetooth bring-up. Any third-party source retains its upstream licensing terms. The installer and repository documentation are provided for hardware bring-up and Armbian integration reference.
