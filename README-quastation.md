# au Qua Station (RTD1295) port — fork notes

This branch (`qua-station`) ports the [symops/monarch-6.18](https://github.com/symops/monarch-6.18)
tree (originally for the WD My Cloud Home, same Realtek RTD1295 SoC) to
au/KDDI's **Qua Station** (photo storage NAS with LTE, model HVD1).

The base `monarch-6.18` README above still applies to the underlying
SoC/board port. This file only covers what's specific to the Qua Station
board itself.

## What's different on this board

Qua Station shares the RTD1295 SoC with the WD My Cloud Home but has a
different board layout: LTE modem, 11 LEDs, physical buttons, a standalone
USB 2.0 socket (EHCI/OHCI, not present on the WD board's DT), and a
different GPIO/LED wiring. The vendor kernel for this specific board is
[tsukumijima/QuaStation-Kernel-BPi](https://github.com/tsukumijima/QuaStation-Kernel-BPi)
(4.9.119) — that tree, plus a serial-console session against the running
4.9 kernel, was the primary hardware reference for everything board-specific
in this fork (DTS, LED/button mapping, clock-gate bit assignments).

## Status

Verified end-to-end on real hardware, booting from the internal SSD
(rootfs on SATA, not just USB):

- SATA (1TB SSD, root filesystem)
- PCIe (both Wi-Fi slots: RTL8192EE 2.4GHz + RTL8812AE 5GHz, simultaneous link-up)
- USB 2.0 (dwc3 port + standalone EHCI/OHCI socket) and USB 3.0
- 11 LEDs + buttons (physical mapping confirmed by hand, COPY button repurposed as KEY_POWER)
- Bluetooth (RTL8761ATV over UART, via vendor `rtk_hciattach`; not yet mainlined as serdev)
- Clock gate driver for RTD129x (57 gates, ported from vendor BSP `cgc.c`, machine-checked against the 4.9 DTB)

Known gaps:

- MSI interrupts on PCIe regress on real hardware (root cause unresolved); currently pinned to INTx
- systemd `.link`-based MAC address pinning doesn't take effect on this board (worked around via `wpa_supplicant@.service.d`)
- Internal GMAC probes successfully but this board has no physical RJ45 port (never worked under the vendor 4.9 kernel either)

## Write-up

The full story — including a serial-less debugging methodology, the exact
register-level root causes for SATA/PCIe/USB2.0 issues, an ext4 corruption
incident and recovery, and a Bluetooth SPP emergency-console setup — is
written up as a 5-part series on Qiita (Japanese):

1. Diagnosing a silent boot failure without a serial console
2. Bringing up SATA / PCIe / Wi-Fi / LEDs
3. Migrating rootfs to the internal SSD
4. A Bluetooth SPP emergency console
5. Recovering from an ext4 corruption incident, plus a USB2.0/PCIe log-flood fix

(Links will be added once the series is published publicly.)

## Building

Standard cross-build from an x86_64 host:

```sh
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- rtd1295-qua-station_defconfig  # or reuse the shipped .config
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) Image dtbs modules
```

Output DTB: `arch/arm64/boot/dts/realtek/rtd1295-qua-station.dtb`.

A pre-built `uImage` + DTB, verified booting on real hardware, is attached
to the [`qua-run41` release](../../releases/tag/qua-run41) (sha256sums in
the release notes). No rootfs is distributed — build one yourself from
`ubuntu-base-*-arm64` (see the Qiita series, part 1 and 3, for the exact
steps used here) or any other arm64 distro; this repo only covers the
kernel/DTB side.

## License

GPL-2.0, same as upstream Linux and `monarch-6.18`.
