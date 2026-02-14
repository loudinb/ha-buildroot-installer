# Custom Home Assistant OS Installation on the Green

This document describes the modifications to the Home Assistant Green installer
boot image that allow flashing a **custom HAOS image** without rebuilding the
installer each time. After building the installer once, any future custom image
can be selected by editing a single text file on the SD card.

## Overview

The stock installer boots from an SD card, downloads the latest stable HAOS
release, and writes it to the Green's eMMC. The modified installer adds support
for two kernel command-line parameters that override this default behavior:

| Parameter       | Purpose                                    |
|-----------------|--------------------------------------------|
| `haos_url=`     | Flash an arbitrary image from any URL       |
| `haos_version=` | Flash a specific official HAOS version      |

These parameters are read from `/proc/cmdline` at runtime. Because U-Boot on
the Green loads its kernel command line from `extlinux/extlinux.conf` on the
FAT32 boot partition, the entire customization reduces to editing a text file.

## Building the Installer (One Time)

### Prerequisites

A Linux host with standard Buildroot dependencies (GCC, make, wget, etc.).
See the Buildroot manual for the full list.

### Build Steps

```bash
# 1. Load the Green installer defconfig
make green_installer_defconfig

# 2. Build everything (toolchain, kernel 6.1.46, U-Boot, rootfs)
make

# 3. The output image is at:
#    output/images/boot.img.xz
```

The build cross-compiles an entire aarch64 toolchain, Linux kernel, U-Boot, and
a minimal rootfs containing `curl`, `jq`, `xz`, and the `haos-flash` script.
This takes a while on the first run, but only needs to be done **once**.

### What's in the Image

The output `boot.img.xz` is a GPT disk image with this layout:

```
+------------------+----------+--------------------------------------------+
| Partition        | Offset   | Contents                                   |
+------------------+----------+--------------------------------------------+
| loader1          | 32 KiB   | idbloader.img (Rockchip SPL + TPL)         |
| uboot            | 8 MiB    | u-boot.itb (U-Boot proper + ATF)           |
| boot (FAT32)     | after    | Image, DTB, rootfs.cpio.zst, extlinux/     |
+------------------+----------+--------------------------------------------+
```

The FAT32 boot partition contains:

```
/
├── Image                    # Linux kernel (AArch64)
├── rk3566-ha-green.dtb      # Device tree blob
├── rootfs.cpio.zst          # Compressed initramfs (the installer OS)
└── extlinux/
    └── extlinux.conf        # Boot configuration (kernel command line)
```

## Writing the Image to an SD Card

Decompress and write the image to an SD card (replace `/dev/sdX`):

```bash
xzcat output/images/boot.img.xz | sudo dd of=/dev/sdX bs=4M status=progress
sync
```

## Customizing the Install via `extlinux.conf`

After writing the image, mount the SD card's FAT32 boot partition and edit
`extlinux/extlinux.conf`. The stock file looks like this:

```
label Home Assistant Green Installer linux
  kernel /Image
  devicetree /rk3566-ha-green.dtb
  initrd /rootfs.cpio.zst
  append console=ttyS2,1500000 console=tty1
```

All customization is done by appending parameters to the `append` line.

### Option 1: Flash a Custom Image URL (`haos_url`)

Point the installer at any HTTP/HTTPS URL hosting a raw disk image:

```
  append console=ttyS2,1500000 console=tty1 haos_url=https://example.com/my-custom-haos.img.xz
```

This is the most flexible option. The URL can point to:
- A self-built HAOS image on your own server
- A CI artifact from a development branch
- A pre-release image hosted anywhere

The installer auto-detects the compression format from the URL file extension:

| Extension  | Decompression |
|------------|---------------|
| `.img.xz`  | xz            |
| `.img.gz`  | gzip          |
| `.img`     | none          |
| other      | xz (default)  |

### Option 2: Flash a Specific Official Version (`haos_version`)

Install a pinned version from the official GitHub releases:

```
  append console=ttyS2,1500000 console=tty1 haos_version=12.1
```

This constructs the download URL automatically:

```
https://github.com/home-assistant/operating-system/releases/download/12.1/haos_green-12.1.img.xz
```

### Option 3: Default (No Parameters)

With no extra parameters, the installer behaves exactly like the stock image:
it queries `https://version.home-assistant.io/stable.json` and installs the
latest stable release.

## Parameter Priority

When multiple sources of configuration exist, `haos-flash` resolves them in
this order (highest priority first):

```
1.  haos_url=...      on kernel command line   →  use this URL directly
2.  haos_version=...  on kernel command line   →  build URL from version
3.  URL as script argument                     →  (systemd service default)
4.  Channel name as script argument            →  query version API
5.  (none)                                     →  stable channel
```

In practice, the systemd service passes `stable` as a script argument
(priority 4), so adding `haos_url` or `haos_version` to the kernel command
line (priorities 1-2) always takes precedence.

## How It Works Internally

### Boot Sequence

```
Power on
  → Rockchip BootROM loads idbloader from SD card
    → SPL initializes DDR, loads U-Boot
      → U-Boot reads extlinux/extlinux.conf
        → Loads kernel + DTB + initramfs into RAM
          → Boots Linux with the `append` line as /proc/cmdline
            → systemd starts install-autostart.service
              → /usr/bin/haos-flash runs
                → Reads /proc/cmdline for haos_url / haos_version
                  → Downloads image, pipes through decompressor, dd to eMMC
                    → Powers off on success
```

### The `cmdline_param` Helper

The script reads kernel parameters by parsing `/proc/cmdline`:

```sh
cmdline_param() {
    set -- $(cat /proc/cmdline)
    for arg in "$@"; do
        case "$arg" in
            "$1"=*)
                echo "${arg#*=}"
                return 0
                ;;
        esac
    done
    return 1
}
```

This means any `key=value` pair on the kernel command line is accessible. The
script checks for `haos_url` first, then `haos_version`, falling back to the
channel-based resolution if neither is present.

### URL Validation

The script validates that the resolved URL starts with `https://` or
`http://`. HTTP URLs produce a warning but are allowed. Anything else (empty
string, local path, ftp, etc.) is rejected, the yellow LED is turned off, and
the script exits with an error.

### LED Indicators

| LED State                | Meaning                              |
|--------------------------|--------------------------------------|
| Rapid blinking (100ms)   | Download + flash in progress         |
| Solid on                 | Flash completed, shutting down       |
| Off                      | Error (no storage, invalid URL, etc.)|

## Examples

### Flash a locally-hosted custom build

Host the image on a machine on your LAN:

```
  append console=ttyS2,1500000 console=tty1 haos_url=https://192.168.1.50:8080/haos_green-custom.img.xz
```

### Flash an older official release

Roll back to a specific version:

```
  append console=ttyS2,1500000 console=tty1 haos_version=11.5
```

### Flash an uncompressed image

The installer handles `.img` files without compression:

```
  append console=ttyS2,1500000 console=tty1 haos_url=https://my-server.local/haos_green-dev.img
```

### Revert to stock behavior

Remove any `haos_url` or `haos_version` parameters from the `append` line,
leaving just the console settings:

```
  append console=ttyS2,1500000 console=tty1
```

## Quick Reference

```bash
# Build (once)
make green_installer_defconfig && make

# Write to SD
xzcat output/images/boot.img.xz | sudo dd of=/dev/sdX bs=4M status=progress && sync

# Mount and edit
sudo mount /dev/sdX3 /mnt
sudo vi /mnt/extlinux/extlinux.conf
# → add haos_url=... or haos_version=... to the append line
sudo umount /mnt

# Boot the Green from the SD card — installation is fully automatic
```
