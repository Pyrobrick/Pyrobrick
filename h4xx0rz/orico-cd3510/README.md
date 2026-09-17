<!--
SPDX-FileCopyrightText: 2026 Pyrobrick and contributors
SPDX-License-Identifier: CC-BY-4.0
-->

# ORICO CD3510 notes

Reverse-engineering / mainline-Linux bring-up notes.

## Credits / provenance

- **Pavlo / [u/Hungry_Status_420](https://www.reddit.com/user/Hungry_Status_420/)**: device-side work, logs and DTB extraction.
- **Pyrobrick**: analysis, upstream comparison and notes.
- **Local Qwen3-14B**: helped sanitize DTB findings for device-unique data.
- **GPT-6 Astra**: used to summarize the notes.

Pavlo explicitly allowed publication and attribution.

## Platform

| Item | Finding |
|---|---|
| SoC | Realtek RTD1319 / Pym Particles |
| CPU | 4 × Cortex-A55, ARM64 |
| RAM | 1 GiB |
| Vendor kernel | Linux 5.10.107 |
| Userspace | OpenWrt-derived |
| Bootloader | U-Boot 2015.07 SDK-ge844cb2 |
| Root FS | SquashFS on `/dev/mmcblk0p1` |
| UART | `ttyS0`, `0x98007800`, 460800 baud |
| FDT load | `0x03000000` |

```dts
compatible = "realtek,pym-particles", "realtek,rtd1319";
model = "Realtek Pym Particles EVB (1GB, TEE)";
```

Kernel cmdline:

```text
earlycon=uart8250,mmio32,0x98007800 console=ttyS0,460800
uio_pdrv_genirq.of_id=generic-uio init=/etc/init
root=/dev/mmcblk0p1 rootfstype=squashfs rootwait loglevel=8
```

## DTB

Pavlo supplied the running-device DTB.

```text
Size:    40960 bytes
SHA-256: ce4c64562ed3e4f15822a65e296557fcd7e75957a6b1344fe6c4f19a92912c39
```

Confirmed:

- 4 × `arm,cortex-a55`;
- eMMC: `realtek,rtd13xx-emmc`;
- SD/MMC: `realtek,rtd13xx-sdmmc`;
- SATA: `realtek,ahci-sata`, 2 enabled ports;
- Ethernet: `realtek,rtd13xx-r8169soc`;
- USB2/USB3 DWC3 + Type-C;
- UART, I2C, SPI, RTC, watchdog, GPIO, LEDs/buttons;
- 3 PCIe controllers described, disabled.

Privacy:

- DTB contains a device-unique Ethernet `local-mac-address` → redacted.
- eFuse nodes describe UUID/chip-ID/calibration locations; actual UUID/chip-ID values are not present as plaintext DT properties.

Do not redistribute the original DTB unchanged.

## Boot chain / eMMC

Observed:

```text
/dev/mmcblk0
/dev/mmcblk0boot0
/dev/mmcblk0boot1
/dev/mmcblk0p1
/dev/mmcblk0p2
/dev/mmcblk0p3
/dev/mmcblk0rpmb
```

No U-Boot string found in `mmcblk0boot0`.

First 4 MiB of `/dev/mmcblk0` contains:

```text
U-Boot 2015.07 SDK-ge844cb2
U-Boot 2015.07 SDK-ge844cb2 (May 06 2023 - 02:57:09 +0000)
```

Likely: detected U-Boot image is in normal eMMC user area. Exact SPL/U-Boot offsets still unknown.

Useful environment:

```text
bootcmd=bootr
baudrate=460800
kernel_loadaddr=0x01008000
fdt_loadaddr=0x03000000
rootfs_loadaddr=0x03100000
rescue_vmlinux=emmc.uImage
rescue_dtb=rescue.emmc.dtb
rescue_rootfs=rescue.root.emmc.cpio.gz_pad.img
```

## UART

Unpopulated pads:

```text
GND / RXO / TXO
460800 baud, 8N1, no flow control
```

`F` during vendor boot reportedly enters recovery/failsafe.

## Mainline status

First target: **mainline kernel + serial console + built-in initramfs**.

Likely possible now with generic support only:

```text
Cortex-A55 -> PSCI -> GICv3 -> ARM timer -> DW APB UART -> initramfs
```

Assumption: U-Boot leaves UART/clock state usable.

Blockers for normal boot from `/dev/mmcblk0p1`:

1. RTD1319 CRT/ISO clocks + resets;
2. plain RTD1319 pinctrl;
3. `rtd13xx-emmc`.

Later:

- `rtd13xx-r8169soc` Ethernet;
- SATA PHY/AHCI;
- remaining board peripherals.

Already upstream:

- RTD1319 ISO GPIO;
- substantial RTD1319 USB2/USB3/DWC3/Type-C support.

## Upstream relation

Realtek submitted RTD1319/Pym Particles DTS support in 2020. The board DTS did not land in current mainline.

- [RTD1319/Pym Particles submission](https://lists.infradead.org/pipermail/linux-realtek-soc/2020-June/000033.html)
- [Linux Realtek ARM64 DTS](https://github.com/torvalds/linux/tree/master/arch/arm64/boot/dts/realtek)
- [RTD GPIO](https://github.com/torvalds/linux/blob/master/drivers/gpio/gpio-rtd.c)
- [USB2 PHY](https://github.com/torvalds/linux/blob/master/drivers/phy/realtek/phy-rtk-usb2.c)
- [USB3 PHY](https://github.com/torvalds/linux/blob/master/drivers/phy/realtek/phy-rtk-usb3.c)

Upstream OpenWrt `realtek` targets managed-switch SoCs, not this ARM64 RTD1319 platform.

## Next useful captures

- exact eMMC size + partition roles;
- SPL/U-Boot offsets;
- kernel `.config`;
- full `dmesg`;
- rescue DTB/kernel/rootfs;
- clock/reset/pinctrl dependencies;
- Ethernet/SATA driver behaviour;
- controlled packet capture of vendor services.

Avoid writes to eMMC/U-Boot environment until recovery is proven.

## Vendor network notes

Reported names:

```text
disk_m7
sdvnfs
syncthing
transmission
/sata/public
/sata/home
/sata/group
port 9898
```

Pavlo reports strong dependence on vendor Internet connectivity and many outbound connections.

Observed capability:

- the vendor software can access/download the entire disk contents.

Not observed:

- abnormally large outgoing traffic;
- evidence that the entire disk contents were actually uploaded;
- proof that `transmission` is ORICO's main cloud daemon.

Need process/binary/DNS/packet evidence for stronger claims.

## Security

Firmware 1.9.12 and earlier: **CVE-2025-69429 / JVNDB-2026-003443** (symlink handling).

- [JVNDB-2026-003443](https://jvndb.jvn.jp/en/contents/2026/JVNDB-2026-003443.html)

## Licensing

- Notes/research: **CC BY 4.0**.
- Original utility code/shell snippets: additionally **MIT**.
- Vendor firmware, binaries, bootloaders and extracted vendor DT material are **not relicensed**.
