# Third-party source and artifact notice

This repository includes a patched `rtk_hciattach` source snapshot and a tested aarch64 binary for convenience when bringing up Bluetooth on the Tencent Aurora 5 Pro box.

- `src/rtk_hciattach-rtl8852bs-maxpatch-src.tar.gz` is a patched source snapshot derived from Realtek/Radxa-style `rtkbt` userspace tooling.
- `artifacts/rtk_hciattach-aarch64-rtl8852bs-maxpatch` is built from that patched source.
- `patches/rtl8852bs-max-patch-size.patch` documents the local change required for RTL8852BS firmware/config payload size.

Third-party source and binary artifacts retain their original upstream licensing terms. The files in this repository are intended as a practical hardware bring-up reference for Armbian/Linux on Tencent Aurora 5 Pro hardware.
