# Linux 6.18 for WD My Cloud Home (Realtek RTD1295 "Monarch")

This tree is vanilla `v6.18` plus a board port for the WD My Cloud Home
(single-bay), a NAS built around Realtek's RTD1295 SoC. It replaces the
vendor's stock kernel, `linux-4.9.330`, which shipped as GPL source at
[symops/monarch-4.9.330](https://github.com/symops/monarch-4.9.330) — that
tree is this port's primary hardware reference throughout.

Mainline already carries generic RTD1295/RTD129x SoC support (maintained by
Andreas Färber): GIC, ARM timer, three UARTs, and not much else. Every
peripheral this board actually needs — SATA, USB, Ethernet, RTC, PWM,
cpufreq, SMP bring-up — had no mainline driver at all before this port, and
had to be forward-ported from the vendor's 4.9-era Realtek BSP.

## Status

Verified end-to-end on real hardware: SMP (4 cores), SATA (real disk,
6.0 Gbps link, RAID assembly, root mount), USB 2.0/3.0 (host, mass
storage), Ethernet, RTC (flash-backed, survives reboot), PWM/LED,
cpufreq (all 3 board OPPs), watchdog, thermal.

Not done: GPIO/pinctrl (no mainline `gpio-rtd129x`/`pinctrl-rtd129x`
driver exists yet — see below), I2C PMIC regulator stack (built but
doesn't probe, see below), full CRT clock controller (only the one CPU
clock needed for cpufreq is ported, not the whole vendor
`clk_pll_div`/mux/gate framework).

## Where this code came from

Three sources, in order of how much of each thing is used:

1. **This port's own new code**, written directly against the vendor
   4.9.330 driver as a functional/register-level reference, targeting
   modern kernel APIs from the start rather than adapting someone else's
   forward-port. This is the SATA glue driver, the SCPU (CPU) clock
   driver, the PWM controller driver, the flash-backed RTC + SPI-NOR
   controller, and the IRQ-affinity fix.
2. **[Fireblossom/wd-mch-kernel](https://github.com/Fireblossom/wd-mch-kernel)**,
   an existing community 4.9→6.18 port for this exact device
   (`docs/PORTING_GUIDE_4.9_to_6.18.md` there documents its own
   per-file reasoning). Adopted largely as-is for SMP bring-up, GMAC
   Ethernet, USB dwc3/PHY, watchdog, thermal, IRQ mux, I2C, and the
   MFD/regulator stack, plus the board DTS skeleton.
3. **The vendor 4.9.330 tree itself**, as the ground-truth register
   reference for everything above, and as the direct source for the
   parts of (1) that needed genuinely new forward-porting.

## Change summary (relative to vanilla v6.18)

### Board

- `arch/arm64/boot/dts/realtek/rtd1295-wd-mycloud-home.dts` (new, ~730
  lines) — SoC/board description forward-ported from the vendor's
  `rtd-1295-monarch-1GB.dts`, including board-specific SATA PHY
  calibration tables (`phy-param`/`tx-driving-tbl`/`rx-sense-tbl` —
  without these the mainline PHY driver falls back to generic defaults
  for a *different* reference board and the disk never links, see
  "SATA" below).
- `arch/arm64/kernel/smp_spin_table.c` — this SoC's secondary CPUs spin
  on a device register, not a normal RAM address; small patch to the
  generic spin-table code to support that.

### Storage (SATA)

- `drivers/ata/ahci_rtd1295.c` (new) — platform driver on top of the
  standard `ahci_platform`/`libahci_platform` framework, not
  `generic-ahci`: this SoC's AHCI block needs several vendor-specific
  register pokes generic-ahci knows nothing about (MMIO offsets
  `0xf20`/`0xc`), a drive-bay power GPIO raised by raw MMIO poke (no
  mainline `gpio-rtd129x` driver/binding exists to do this properly
  yet), and a deferred host-init delay matching the vendor driver's
  default. Most importantly: an explicit deassert of the
  `SATA_PHY_POW_0` reset bit (CRT offset `0x00`, bit 10) — the SATA
  analog PHY power macro is left in hardware reset by the boot loader
  on a cold boot path (reproducible via the physical USB-Install
  button, which skips U-Boot's own SATA scan), and every *other*
  register write that looks plausible (MDIO calibration, SB2 gate
  bits, the AHCI vendor registers above) silently no-ops against it
  because they live in unrelated reset/clock domains. This one bit was
  the actual root cause after several other SATA-link-down leads
  (calibration tables, GPIO, timing) turned out to be red herrings.
- `drivers/phy/realtek/phy-rtk-sata.c` — SATA PHY, already written
  against the modern generic PHY framework (`devm_phy_create()`) in
  the vendor tree, so this forward-port only needed API-surface
  updates, not a rewrite.
- `drivers/mtd/spi-nor/controllers/rtk-sfc.c` (new) — the boot SPI-NOR
  controller, forward-ported to `spi_nor_controller_ops`. Its RDID
  register read is the one genuinely hardware-quirky part: the whole
  (up to 4-byte) JEDEC ID response is latched by a single pulse of
  `SFC_CTL` into one 32-bit word at a fixed IO address, not one byte
  per read/pulse as a naive port of the vendor's `memcpy`-based
  `read_reg()` would suggest — confirmed by hand via `devmem` after
  three different byte-wise read strategies all came back identical
  and wrong (`ef ef ef ef ef ef` instead of the real `ef 40 14`).

### RTC

- `drivers/rtc/rtc-wd-mch-flash.c` (new) — this board has no
  battery-backed hardware RTC. The vendor firmware persists the clock
  as 8 bytes (seconds + inverted-checksum) in a dedicated area of the
  boot SPI-NOR instead; this driver reads/writes the same on-flash
  format as the vendor's `rtc-rtk.c`, so the timestamp stays valid
  across a dual-boot back to stock firmware. Registers under a new
  `wd,mycloud-home-rtc` compatible rather than binding to
  `realtek,rtd1295-rtc`: the real hardware RTC counter at that
  register offset free-runs from an arbitrary reset value with no
  battery backup and is useless for keeping actual time (this is why
  mainline's `rtc-rtd119x.c`, which drives that counter directly,
  always reports 2014-01-01 on this board) — the vendor firmware
  never uses it either. Periodic write-back to flash needs no extra
  code: it's the kernel's ordinary `CONFIG_RTC_SYSTOHC` (NTP → RTC
  sync, ~every 11 minutes once synced), already enabled and pointed at
  `rtc0`.

### PWM / front LED

- `drivers/pwm/pwm-rtd129x.c` (new) — forward-ported from the vendor's
  `pwm-rtk.c` to `pwm_ops.apply()`/`.get_state()`. The vendor driver's
  ~700 lines of custom per-channel sysfs attributes are dropped —
  mainline already exposes the same controls generically under
  `/sys/class/pwm/` — keeping only the register math and probe core.
  `get_state()` reads live hardware registers rather than mirroring a
  software-shadow array, because `leds-pwm`'s `default-state = "keep"`
  calls `get_state()` once at probe to seed the LED's initial
  brightness from whatever is already running.
- Board DTS front SYS LED default: `"keep"` turned out to be the wrong
  choice for this channel. The raw pre-driver register state is reset/
  BootROM leftovers (~1.6 Hz, visibly flickering), not anything the
  boot loader meaningfully configured — the vendor driver
  unconditionally reprograms all 4 channels at probe regardless of
  what it finds there. Its logged defaults for this channel
  (`clksrc_div=1 clkout_div=255 duty_rate=10`, ~26367 Hz) are the real
  known-good operating point, so the DTS now uses
  `default-state = "on"` with an explicit `default-brightness`
  reproducing that duty cycle.

### CPU clock / cpufreq

- `drivers/clk/realtek/clk-rtd129x-scpu.c` (new) — a single-purpose
  driver for just the one clock cpufreq needs (the CPU PLL), rather
  than a port of the vendor's generic `clk_pll_div`/mux/gate framework
  for the SoC's whole CRT clock tree (~1600 lines across the vendor's
  `clk-pll.{c,h}` + `common.{c,h}`, none of it needed for this one
  clock). Both lookup tables (PLL N/F "SSC1" register values,
  post-divider values) are exact copies of the vendor's; the
  anti-glitch divider/PLL write ordering is kept; the vendor's
  `val==1` workaround is dropped as dead code for this specific
  divider table (never has a value of 1); there is no
  `.enable`/`.disable`/`.is_enabled` because the vendor's own gating
  hooks are no-ops for this particular clock (no `pow_loc`
  configured) — it is the CPU's own clock, `CLK_IGNORE_UNUSED` in the
  vendor tree too.
- Board DTS `scpu_clk` node has no `reg`: the divider register (CRT
  offset `0x030`) and the PLL/SSC1 registers (`0x500+`) sit far apart
  in the same block, so the driver reaches both through `&crt`'s
  syscon regmap instead of a private MMIO sub-window that could only
  cover one of them.
- Three board OPPs in the DTS (300375/600750/1100000 kHz) match the
  vendor's boot-time PLL setting and its double/half.

### IRQ mux

- `drivers/irqchip/irq-rtd129x.c` — this SoC's has more interrupt
  sources than GIC lines, multiplexed through a vendor IRQ-mux block
  (adopted from the community port). One additional fix on top: its
  `set_affinity` delegated to the parent GIC IRQ but never recorded
  the result on the muxed (virtual) `irq_data` itself, so genirq
  warned `"did not update eff. affinity mask"` for every muxed IRQ
  (e.g. the UART's) whenever affinity was set. Fixed by recording the
  parent's effective affinity on the muxed IRQ too, since all IRQs on
  one mux instance share that single parent GIC line anyway.

### Everything else from the community port, adopted as-is

SMP bring-up, GMAC Ethernet (`r8169soc.c` — note the driver name; the
matching Kconfig symbol is gated behind `ARCH_RTD129x`, a Kconfig trap
worth knowing about if it ever looks unselectable), USB dwc3 + PHY,
watchdog, thermal, I2C, and the MFD/regulator stack for the board's
I2C PMIC. The I2C/regulator stack builds and is wired into the DTS but
does not currently probe successfully — not investigated further since
nothing on this board depends on it (the vendor boot log shows it
initializing, but the board runs fine without it).

## Building and booting

This board's U-Boot (`2015.07`, `Realtek QA Board`, 2016 build) has two
requirements that are very easy to miss and produce total, silent boot
failure — no console output, nothing — if skipped:

1. **`arch/arm64/boot/Image`'s header must be patched** before
   packaging. This loader copies the Image to the address given by its
   own header `text_offset` field; a normal relocatable Image has
   `text_offset=0`, i.e. "copy the kernel on top of itself at physical
   address 0". Run `tools/monarch/patch-header.py arch/arm64/boot/Image`
   on every build before flashing/testing. (This was mistaken for a
   kernel-image-size limit for a long time before the real cause was
   found — several rounds of shrinking the kernel "fixed" it only by
   accident, by changing unrelated build parameters alongside it.)
2. **Package the patched Image raw, never gzip.** This loader's
   built-in gzip decompression is unreliable above a few MB and fails
   in different, non-obvious ways depending on size (`inflate()`
   buffer errors, or silent hangs) — raw avoids the decompression step
   in the loader entirely. `dd`-append 512 KB of zero padding after
   the patched Image (`xbuild.sh` in the vendor tree does the same;
   exact reason not confirmed, but required by their tooling).

The safe way to test any of this without touching the board's internal
storage: this loader has a `boot_rescue_from_usb` path (triggered by
the physical USB-Install button, or automatically when no factory
partition is found) that reads fixed filenames from a GPT/FAT USB
stick — `rescue.sata.dtb`, `sata.uImage` (despite the name, no
mkimage/FIT wrapping — just the raw patched `Image` + padding, see
above), and either a combined `CONFIG_INITRAMFS_SOURCE`-baked initramfs
or a separate `rescue.root.sata.cpio.gz_pad.img` (gzip'd cpio,
zero-padded to exactly 4194304 bytes regardless of real payload size —
the loader reads that fixed block size unconditionally). Nothing on
the board's own flash/disk is touched by this path.
