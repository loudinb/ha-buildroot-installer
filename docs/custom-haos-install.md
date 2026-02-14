# Automated Custom Home Assistant Deployment on the Green

This document describes a fully automated, zero-interaction pipeline for
deploying a **custom Home Assistant Core** to a Home Assistant Green device.
The pipeline has three layers, each built once and chained together:

```
┌─────────────────────────────┐
│  1. Custom HA Core          │  Build a modified Core container image
│     (Docker image)          │  and push to a container registry
└──────────────┬──────────────┘
               │ image reference
               ▼
┌─────────────────────────────┐
│  2. Custom HAOS             │  Build HAOS configured to pull the
│     (disk image)            │  custom Core image on first boot
└──────────────┬──────────────┘
               │ hosted URL
               ▼
┌─────────────────────────────┐
│  3. Custom Boot Installer   │  Installer SD card that flashes the
│     (this repo)             │  custom HAOS image to eMMC automatically
└─────────────────────────────┘
```

Insert the SD card, power on the Green, walk away. When the yellow LED goes
solid, the device has a fully customized Home Assistant stack on eMMC.

## Architecture

The Home Assistant Green runs four distinct software layers at runtime:

```
┌──────────────────────────────────────┐
│  Home Assistant Core                 │  Python app — your custom version
│  (Docker container)                  │
├──────────────────────────────────────┤
│  Home Assistant Supervisor           │  Container orchestrator — manages
│  (Docker container)                  │  Core, add-ons, updates
├──────────────────────────────────────┤
│  Home Assistant OS (HAOS)            │  Minimal Linux — kernel, systemd,
│  (on eMMC)                           │  Docker, networking
├──────────────────────────────────────┤
│  Hardware (RK3566 SoC)               │  Rockchip RK3566, 4GB RAM, eMMC
└──────────────────────────────────────┘
```

Key relationships:
- The **installer** (this repo) writes the **HAOS image** to eMMC. It has no
  knowledge of Core or the Supervisor.
- **HAOS** boots, starts Docker, and launches the **Supervisor**.
- The **Supervisor** reads its configuration to determine which **Core**
  container image to pull and run.

To get a custom Core running with zero interaction, every layer in the chain
must be configured before the device boots.

---

## Layer 1: Custom Home Assistant Core

Home Assistant Core is a Python application distributed as a Docker container
image. The stock image is published at:

```
ghcr.io/home-assistant/home-assistant:YYYY.M.N
```

### Building a Custom Core Image

Fork or clone the [home-assistant/core](https://github.com/home-assistant/core)
repository, make your modifications, then build and push:

```bash
# Clone your fork
git clone https://github.com/YOUR_ORG/core.git
cd core

# Make your changes (custom integrations, patches, etc.)
# ...

# Build the container image for aarch64 (the Green's architecture)
docker buildx build \
  --platform linux/arm64 \
  --tag ghcr.io/YOUR_ORG/homeassistant-green:YYYY.M.N \
  --push \
  .
```

The image must be:
- Built for `linux/arm64` (aarch64) — the Green's CPU architecture
- Pushed to a registry accessible from the Green's network (GHCR, Docker Hub,
  or a self-hosted registry)
- Tagged with a version the Supervisor will accept

Note your full image reference (e.g.,
`ghcr.io/YOUR_ORG/homeassistant-green:2024.12.1`). You will need it when
configuring the HAOS layer.

---

## Layer 2: Custom Home Assistant OS

HAOS is built from the
[home-assistant/operating-system](https://github.com/home-assistant/operating-system)
repository. It is also a Buildroot-based build, like this installer repo.

The goal at this layer is to produce an HAOS disk image where the Supervisor is
pre-configured to pull your custom Core container image instead of the stock
one.

### How the Supervisor Resolves the Core Image

On first boot, the Supervisor reads its machine configuration and consults
version endpoints to decide which Core image to pull. The key configuration
lives in:

```
/etc/hassio.json
```

This file is baked into the HAOS image at build time and contains fields
including:

| Field           | Purpose                                       |
|-----------------|-----------------------------------------------|
| `machine`       | Hardware identifier (e.g., `green`)           |
| `image`         | Supervisor container image reference           |
| `data`          | Path to persistent data (`/mnt/data`)         |

The Supervisor itself determines the Core image based on the machine type and
the configured update channel. To override the Core image, you need to
either:

**Option A: Pre-populate Supervisor configuration on the data partition**

The Supervisor stores its runtime state in `/mnt/data/supervisor/`. By placing
a pre-configured `updater.json` or modifying the Supervisor's startup
configuration in the HAOS build, you can pin the Core image reference.

**Option B: Use a custom Supervisor build**

Fork the [home-assistant/supervisor](https://github.com/home-assistant/supervisor)
and modify the default image resolution to point at your custom Core registry.
Then configure the HAOS build to use your custom Supervisor image.

**Option C: Overlay files on the data partition**

The HAOS build process supports injecting files into the image. Add
configuration files to the data partition during the HAOS build that tell the
Supervisor to use your custom Core image on first boot.

### Building Custom HAOS

```bash
# Clone the operating-system repo
git clone https://github.com/home-assistant/operating-system.git
cd operating-system

# Configure for the Green board with your customizations
# (refer to the operating-system repo's documentation for board-specific
#  build instructions and where to inject custom Supervisor configuration)

# Build the Green image
make green
```

The output is a disk image, e.g.:

```
release/haos_green-XX.Y.img.xz
```

Host this image on an HTTP/HTTPS server accessible from the Green's network
during installation. Note the URL — you will need it for the installer layer.

---

## Layer 3: Custom Boot Installer (This Repository)

This Buildroot repository builds an SD card image that boots on the Green and
flashes an HAOS image to eMMC. The modified `haos-flash` script supports
a `haos_url` kernel command-line parameter that points the installer at your
custom HAOS image.

### Building the Installer

```bash
# Load the Green installer defconfig
make green_installer_defconfig

# Build (cross-compiles toolchain, kernel 6.1.46, U-Boot, rootfs)
make

# Output:
#   output/images/boot.img.xz
```

### SD Card Image Layout

The output `boot.img.xz` is a GPT disk image:

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

### Writing the SD Card

```bash
xzcat output/images/boot.img.xz | sudo dd of=/dev/sdX bs=4M status=progress
sync
```

### Configuring the Custom HAOS URL

Mount the SD card's FAT32 boot partition and edit `extlinux/extlinux.conf`.

The stock file:

```
label Home Assistant Green Installer linux
  kernel /Image
  devicetree /rk3566-ha-green.dtb
  initrd /rootfs.cpio.zst
  append console=ttyS2,1500000 console=tty1
```

Add `haos_url=` to the `append` line, pointing at your custom HAOS image:

```
  append console=ttyS2,1500000 console=tty1 haos_url=https://your-server.example.com/haos_green-custom.img.xz
```

### Kernel Command-Line Parameters

| Parameter       | Purpose                                          |
|-----------------|--------------------------------------------------|
| `haos_url=`     | Full URL to a custom HAOS disk image (any host)  |
| `haos_version=` | Pin a specific official HAOS release version      |

**Priority order** (highest first):

| Priority | Source                            | Behavior                       |
|----------|-----------------------------------|--------------------------------|
| 1        | `haos_url=` on kernel cmdline     | Use URL directly               |
| 2        | `haos_version=` on kernel cmdline | Build URL from official release|
| 3        | Script argument (URL)             | Used if called manually        |
| 4        | Script argument (channel name)    | Query version API              |
| 5        | Default                           | Stable channel                 |

The systemd service passes `stable` as a script argument (priority 4), so
kernel command-line parameters (priorities 1-2) always take precedence.

### Supported Image Formats

The installer auto-detects compression from the URL file extension:

| Extension  | Decompression |
|------------|---------------|
| `.img.xz`  | xz            |
| `.img.gz`  | gzip          |
| `.img`     | none          |
| other      | xz (default)  |

---

## How `haos-flash` Works

### Boot Sequence

```
Power on with SD card inserted
  → Rockchip BootROM loads idbloader from SD
    → SPL initializes DDR, loads U-Boot
      → U-Boot reads extlinux/extlinux.conf from FAT32
        → Loads kernel + DTB + initramfs into RAM
          → Boots Linux with the append line as /proc/cmdline
            → systemd starts install-autostart.service
              → /usr/bin/haos-flash runs
                → Parses /proc/cmdline for haos_url / haos_version
                  → curl downloads image, pipes through decompressor
                    → dd writes to /dev/mmcblk0 (eMMC)
                      → Powers off on success
```

### Kernel Command-Line Parsing

The script extracts parameters from `/proc/cmdline` using a POSIX-compatible
helper:

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

Any `key=value` pair appended to the `append` line in `extlinux.conf` becomes
accessible through this function.

### URL Validation

The resolved URL must start with `https://` or `http://`. HTTP URLs produce a
warning but proceed. Invalid URLs (empty, local paths, unsupported schemes)
cause the script to exit with an error and turn off the yellow LED.

### LED Indicators

| LED State              | Meaning                              |
|------------------------|--------------------------------------|
| Rapid blink (100ms)    | Download and flash in progress       |
| Solid on               | Flash succeeded, system powering off |
| Off                    | Error occurred                       |

---

## End-to-End Automation

Putting it all together, the one-time setup for a fully automated pipeline:

### 1. Build and publish custom Core

```bash
cd /path/to/your/core-fork
docker buildx build --platform linux/arm64 \
  --tag ghcr.io/YOUR_ORG/homeassistant-green:YYYY.M.N \
  --push .
```

### 2. Build custom HAOS (configured for your Core image)

```bash
cd /path/to/operating-system-fork
# Configure Supervisor to use ghcr.io/YOUR_ORG/homeassistant-green:YYYY.M.N
make green
# Host the output image:
#   scp release/haos_green-custom.img.xz your-server:/var/www/images/
```

### 3. Build the installer and prepare SD cards

```bash
cd /path/to/ha-buildroot-installer
make green_installer_defconfig
make

# Write to SD card
xzcat output/images/boot.img.xz | sudo dd of=/dev/sdX bs=4M status=progress
sync

# Configure the SD card to point at your custom HAOS
sudo mount /dev/sdX3 /mnt
```

Edit `/mnt/extlinux/extlinux.conf`:

```
label Home Assistant Green Installer linux
  kernel /Image
  devicetree /rk3566-ha-green.dtb
  initrd /rootfs.cpio.zst
  append console=ttyS2,1500000 console=tty1 haos_url=https://your-server.example.com/haos_green-custom.img.xz
```

```bash
sudo umount /mnt
```

### 4. Deploy

Insert the SD card into the Green and power on. The device will:

1. Boot from SD card
2. Download and flash your custom HAOS to eMMC
3. Power off (yellow LED goes solid)
4. Remove SD card, power on again
5. HAOS boots from eMMC, starts Supervisor
6. Supervisor pulls your custom Core container image
7. Custom Home Assistant Core is running

No human interaction required after inserting the SD card.

### Subsequent Deployments

The installer SD card is reusable. To deploy a different custom image, just
remount the SD card's boot partition and change the `haos_url` in
`extlinux.conf`. The installer itself never needs to be rebuilt.
