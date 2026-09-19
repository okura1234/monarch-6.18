# SPDX-License-Identifier: GPL-2.0-only

dtb-$(CONFIG_ARCH_REALTEK) += rtd1293-ds418j.dtb

dtb-$(CONFIG_ARCH_REALTEK) += rtd1295-mele-v9.dtb
dtb-$(CONFIG_ARCH_REALTEK) += rtd1295-probox2-ava.dtb
dtb-$(CONFIG_ARCH_REALTEK) += rtd1295-wd-mycloud-home.dtb
dtb-$(CONFIG_ARCH_REALTEK) += rtd1295-qua-station.dtb
dtb-$(CONFIG_ARCH_REALTEK) += rtd1295-xnano-x5.dtb
dtb-$(CONFIG_ARCH_REALTEK) += rtd1295-zidoo-x9s.dtb

# 16 KiB of trailing free space so U-Boot's rescue path can grow the FDT in
# place (it tries to add a /factory subnode and set linux,stdout-path on
# every boot). Discovered on the sibling Duo board port
# (symops/pelican-6.18) as the explanation for its own FDT_ERR_NOSPACE
# boot warnings, once it was noticed that a stale-but-still-working
# Monarch .dtb happened to carry this exact amount of headroom by
# accident, from some earlier ad-hoc build never refreshed since. Applied
# here properly via this flag rather than relying on that stale artifact.
DTC_FLAGS_rtd1295-wd-mycloud-home := -p 0x4000
DTC_FLAGS_rtd1295-qua-station := -p 0x4000

dtb-$(CONFIG_ARCH_REALTEK) += rtd1296-ds418.dtb

dtb-$(CONFIG_ARCH_REALTEK) += rtd1395-bpi-m4.dtb
dtb-$(CONFIG_ARCH_REALTEK) += rtd1395-lionskin.dtb

dtb-$(CONFIG_ARCH_REALTEK) += rtd1619-mjolnir.dtb
