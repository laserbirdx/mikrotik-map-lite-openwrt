![OpenWrt on MikroTik mAP lite](assets/banner.webp)

# OpenWrt on MikroTik mAP lite (RBmAPL-2nD)

![OpenWrt: 25.12.6 (ath79/mikrotik)](https://img.shields.io/badge/OpenWrt-25.12.6%20(ath79%2Fmikrotik)-00B5E2?logo=openwrt&logoColor=white) ![SoC: Qualcomm QCA9533](https://img.shields.io/badge/SoC-Qualcomm%20QCA9533-3253DC?logo=qualcomm&logoColor=white) ![install: Netboot (No Disassembly)](https://img.shields.io/badge/install-Netboot%20(No%20Disassembly)-brightgreen) ![license: CC BY 4.0](https://img.shields.io/badge/license-CC%20BY%204.0-lightgrey)

Installation guide for official OpenWrt 25.12.6 on the MikroTik RouterBOARD mAP lite
(`RBmAPL-2nD`, Qualcomm Atheros QCA9533, 64 MiB RAM, 16 MiB SPI-NOR flash). OpenWrt is
loaded into RAM over Ethernet via RouterBOOT netboot (BOOTP + TFTP) and then written to
flash with `sysupgrade`, without opening the case.

The firmware images are unmodified official OpenWrt builds. They are attached to the
[Releases](../../releases/latest) page for convenience and are also available from
[downloads.openwrt.org](https://downloads.openwrt.org/releases/25.12.6/targets/ath79/mikrotik/).

> **Unofficial guide. You do this at your own risk.** Installing OpenWrt overwrites the
> RouterOS `firmware` partition. Back up your RouterOS license key before flashing (see Step 1).

## Hardware

Measured on a RouterBOARD mAP L-2nD running RouterOS `v6.36.3` / RouterBOOT `3.33`:

| Component | Details |
|---|---|
| Model | MikroTik RouterBOARD mAP L-2nD (`mAP lite`) |
| SoC | Qualcomm Atheros QCA9533, MIPS 24Kc @ 650 MHz |
| RAM | 64 MiB DDR2 |
| Flash | 16 MiB SPI-NOR |
| Ethernet | 1x 10/100 Mbps (`ether1`, 802.3af/at + passive PoE-in) |
| Wi-Fi | QCA9533 built-in, 2.4 GHz 802.11b/g/n 2x2 |
| Power | Micro-USB 5 V or PoE-in (10–60 V) |
| Stock firmware | RouterOS v6 (`mipsbe`), RouterBOOT `3.33` |
| OpenWrt target | `ath79/mikrotik`, profile `mikrotik_routerboard-mapl-2nd` |

### Flash layout (16 MiB SPI-NOR)

| MTD | Offset | Size | Name | Notes |
|---|---|---|---|---|
| `mtd0` | `0x000000` | 56 KiB | `routerboot` | Primary RouterBOOT bootloader (read-only) |
| `mtd1` | `0x00e000` | 4 KiB | `hard_config` | MAC addresses, Wi-Fi calibration data (read-only) |
| `mtd2` | `0x00f000` | 4 KiB | `bios` | |
| `mtd3` | `0x010000` | 60 KiB | `routerboot2` | Backup RouterBOOT bootloader |
| `mtd4` | `0x01f000` | 4 KiB | `soft_config` | RouterBOOT settings (`boot_device`, `boot_protocol`, etc.) |
| `mtd5` | `0x020000` | 16256 KiB | `firmware` | OpenWrt kernel + squashfs rootfs + overlay |

`hard_config` and the RouterBOOT partitions remain untouched during installation.

## Tools

| File | Use |
|---|---|
| [`tools/netboot_server.py`](tools/netboot_server.py) | Standalone Python 3 BOOTP/DHCP (UDP 67) + RFC 2347 TFTP (UDP 69, `blksize`/`tsize` OACK) server for MikroTik RouterBOOT. Filters by MikroTik MAC OUI and serves the `initramfs` image directly from memory. |

## Installation

### 1. Back up the RouterOS license key

Before replacing RouterOS, export the factory license key over SSH (or WinBox `System -> License -> Export Key`). Older RouterOS 6 builds use `ssh-rsa` and `hmac-sha1`:

```sh
ssh -o HostKeyAlgorithms=+ssh-rsa -o MACs=+hmac-sha1 admin@192.168.88.1 \
  "/system license print; /system routerboard print"
```

To export and copy the `.key` file:

```sh
ssh -o HostKeyAlgorithms=+ssh-rsa -o MACs=+hmac-sha1 admin@192.168.88.1 \
  "/system license output"
scp -O -o HostKeyAlgorithms=+ssh-rsa -o MACs=+hmac-sha1 \
  admin@192.168.88.1:*.key ./
```

### 2. Boot the OpenWrt `initramfs` image over Ethernet

1. Connect the PC directly to the mAP lite `ether1` port and assign a static IP on the Ethernet adapter:
   - IP address: `192.168.88.253`
   - Subnet mask: `255.255.255.0`
2. Start the bundled BOOTP + TFTP server in an elevated terminal (root / Administrator is required to bind to UDP ports 67 and 69):

   ```sh
   python3 tools/netboot_server.py \
     openwrt-25.12.6-ath79-mikrotik-mikrotik_routerboard-mapl-2nd-initramfs-kernel.bin \
     --bind-ip 192.168.88.253
   ```

3. Trigger RouterBOOT network boot using **either** method below:
   - **Option A — Over SSH from RouterOS (no button press needed):**

     ```sh
     ssh -o HostKeyAlgorithms=+ssh-rsa -o MACs=+hmac-sha1 admin@192.168.88.1 \
       "/system routerboard settings set boot-protocol=bootp boot-device=try-ethernet-once-then-nand; /system reboot"
     ```

   - **Option B — Using the Reset button:**
     Unplug power, hold the **Reset** button, plug power back in, and keep holding for ~15 seconds until the LED stops flashing and stays solid (Netinstall / Etherboot mode), then release.

4. `netboot_server.py` answers the BOOTP request with `192.168.88.200` and transfers the `initramfs` kernel over TFTP (`*** FIRMWARE TRANSFER COMPLETE! ***`).
5. Wait ~60 seconds for OpenWrt to boot in RAM.

### 3. Flash the `sysupgrade` image permanently

Once OpenWrt boots from `initramfs`, it configures `ether1` as LAN at `192.168.1.1/24`.

1. Set your PC's Ethernet adapter to `192.168.1.2/24` (or DHCP).
2. Copy the `squashfs-sysupgrade.bin` image to `/tmp` on the router and run `sysupgrade`:

   ```sh
   scp -O openwrt-25.12.6-ath79-mikrotik-mikrotik_routerboard-mapl-2nd-squashfs-sysupgrade.bin \
     root@192.168.1.1:/tmp/sysupgrade.bin

   ssh root@192.168.1.1 "sysupgrade -v -n /tmp/sysupgrade.bin"
   ```

The router writes the image to the 16 MiB SPI-NOR `firmware` partition (`mtd5`) and reboots into OpenWrt 25.12.6.

## Default configuration after install

| Setting | Value |
|---|---|
| LAN | `ether1` (`eth0`), `192.168.1.1/24`, DHCP server enabled |
| Login | `root`, no password (set one first) |
| Web UI | LuCI (`http://192.168.1.1`) |
| Wi-Fi | Disabled by default |

> **Single-port note:** Because the mAP lite has only one Ethernet port (`eth0`), OpenWrt assigns it to `lan` so you are not locked out after flashing.

## Reverting to RouterOS

RouterBOOT (`mtd0` / `mtd3`) and the hardware configuration (`mtd1`) are preserved. To return to stock RouterOS at any time, put the device into Etherboot mode and flash a RouterOS `mipsbe` `.npk` package using MikroTik **Netinstall**.

## References

- [OpenWrt Techdata: MikroTik RBmAPL-2nD (mAP lite)](https://openwrt.org/toh/hwdata/mikrotik/mikrotik_rbmapl-2nd_map_lite)
- [OpenWrt Wiki: MikroTik Common Procedures](https://openwrt.org/toh/mikrotik/common)

## License

This guide is licensed under [CC BY 4.0](LICENSE). The scripts in [`tools/`](tools) are
licensed under the [MIT License](LICENSE-tools).

The firmware images on the Releases page are unmodified official OpenWrt 25.12.6 builds,
licensed under GPL-2.0. Source code:
[openwrt/openwrt @ v25.12.6](https://github.com/openwrt/openwrt/tree/v25.12.6).
