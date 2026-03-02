# meta-vehiclecontrol

Custom Yocto layer for building the ECU1 system image (VehicleControl ECU) on Raspberry Pi 4.

For application-level details (architecture, hardware, vsomeip/CommonAPI, GStreamer, gamepad mapping), see the VehicleControlECU README.md file.

---

## Layer Dependencies

Yocto **4.0 Kirkstone (LTS)**:

| Layer | Branch |
|-------|--------|
| `meta` (poky) | `kirkstone` |
| `meta-poky` | `kirkstone` |
| `meta-raspberrypi` | `kirkstone` |
| `meta-openembedded/meta-oe` | `kirkstone` |
| `meta-openembedded/meta-python` | `kirkstone` |
| `meta-openembedded/meta-networking` | `kirkstone` |
| `meta-openembedded/meta-multimedia` | `kirkstone` |
| `meta-qt5` | `kirkstone` |

---

## Build Setup

### Host Prerequisites

Ubuntu 20.04 or 22.04 recommended. Disk space: ~50 GB free.

```bash
sudo apt-get install -y \
    gawk wget git diffstat unzip texinfo gcc build-essential chrpath socat \
    cpio python3 python3-pip python3-pexpect xz-utils debianutils iputils-ping \
    python3-git python3-jinja2 libegl1-mesa libsdl1.2-dev pylint xterm \
    python3-subunit mesa-common-dev zstd liblz4-tool file
```

### bblayers.conf

```
BBLAYERS ?= " \
    /path/to/yocto/poky/meta \
    /path/to/yocto/poky/meta-poky \
    /path/to/yocto/poky/meta-yocto-bsp \
    /path/to/yocto/meta-raspberrypi \
    /path/to/yocto/meta-openembedded/meta-oe \
    /path/to/yocto/meta-openembedded/meta-python \
    /path/to/yocto/meta-openembedded/meta-networking \
    /path/to/yocto/meta-openembedded/meta-multimedia \
    /path/to/yocto/meta-qt5 \
    /path/to/meta-vehiclecontrol \
"
```

### local.conf

```
MACHINE = "raspberrypi4-64"

# Init manager
DISTRO_FEATURES:append = " systemd"
DISTRO_FEATURES_BACKFILL_CONSIDERED:append = " sysvinit"
VIRTUAL-RUNTIME_init_manager = "systemd"
VIRTUAL-RUNTIME_initscripts = "systemd-compat-units"

# Hardware interfaces
ENABLE_I2C = "1"
ENABLE_SPI_BUS = "1"
ENABLE_UART = "1"

# Bootloader
RPI_USE_U_BOOT = "0"
DISABLE_RPI_BOOT_LOGO = "1"

# GPU memory for camera
GPU_MEM = "128"

# OpenGL (required for Qt5)
PREFERRED_PROVIDER_virtual/egl = "mesa"
PREFERRED_PROVIDER_virtual/libgles2 = "mesa"
PREFERRED_PROVIDER_virtual/libgl = "mesa"
DISTRO_FEATURES:append = " opengl"
MACHINE_FEATURES:append = " vc4graphics"

# Kernel
PREFERRED_PROVIDER_virtual/kernel = "linux-raspberrypi"
PREFERRED_VERSION_linux-raspberrypi = "6.1%"

# Cache directories
DL_DIR ?= "${TOPDIR}/../downloads"
SSTATE_DIR ?= "${TOPDIR}/../sstate-cache"

# Licenses
LICENSE_FLAGS_ACCEPTED = "commercial synaptics-killswitch"

# GStreamer x264 support
PACKAGECONFIG:append:pn-libcamera = " gst"
PACKAGECONFIG:append:pn-gstreamer1.0-plugins-ugly = " x264"

# Build parallelism
BB_NUMBER_THREADS = "8"
PARALLEL_MAKE = "-j 8"
```

### Build

```bash
source sources/poky/oe-init-build-env build
bitbake vehiclecontrol-image
```

First build: 2–4 hours. Subsequent builds: 10–30 minutes.

---

## Included Packages

| Category | Package | Notes |
|----------|---------|-------|
| Communication | vsomeip 3.5.8 | SOME/IP middleware |
| Communication | commonapi-core 3.2.4 | CommonAPI C++ runtime |
| Communication | commonapi-someip 3.2.4 | SOME/IP binding |
| Hardware | pigpio | GPIO control |
| Hardware | i2c-tools, can-utils | I2C/CAN debugging |
| Camera | libcamera, libcamera-gst | With raspberrypi IPA |
| Camera | gstreamer1.0-plugins-{base,good,bad,ugly} | RTP/UDP streaming |
| System | systemd, openssh, iproute2, bash | Base system |
| Wireless | wpa-supplicant, bluez5, pi-bluetooth | WiFi + Bluetooth |
| Application | vehiclecontrol-ecu | Main ECU1 app |

---

## Default Login

| Field | Value |
|-------|-------|
| Username | `root` |
| Password | `raspberry` |

---

## Systemd Services

| Service | Description |
|---------|-------------|
| `vehiclecontrol-ecu.service` | Main vehicle control application |
| `can-setup.service` | CAN interface initialization (MCP2518FD, 1000 kbps) |
| `camera-streaming.service` | GStreamer RTP stream to ECU2 |

---

## Runtime Verification

After flashing and booting the RPi:

```bash
# Check services
systemctl status vehiclecontrol-ecu.service
systemctl status can-setup.service
systemctl status camera-streaming.service

# CAN interface
ip link show can0

# eth0 power management (must be "on" to prevent packet loss)
cat /sys/class/net/eth0/device/power/control

# vsomeip routing socket
ls /tmp/vsomeip-0

# Service logs
journalctl -u vehiclecontrol-ecu.service -f
```

---

## Troubleshooting

### Build Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `boost` fetch fails | Network timeout | Re-run `bitbake vehiclecontrol-image` |
| `vsomeip` compile fails | Missing boost | Check RDEPENDS in vsomeip recipe |
| Layer compatibility error | Wrong branch | Ensure all layers on `kirkstone` branch |
| Disk space error | Build cache full | `du -sh tmp/` — clean sstate if >30 GB |
| Recipe parse error | `.bb` syntax issue | `bitbake -p` to validate all recipes |

### Runtime Errors

| Symptom | Cause | Fix |
|---------|-------|-----|
| `vehiclecontrol-ecu` fails | vsomeip socket missing | `journalctl -u vehiclecontrol-ecu` |
| I2C not found | I2C not enabled | Check `dtparam=i2c_arm=on` in `/boot/config.txt` |
| No network | IP not assigned | `systemctl status systemd-networkd` |
| vsomeip no service | Multicast routing | `ip route add 224.0.0.0/4 dev eth0` |
| 40–60% packet loss | eth0 power management | Set `/sys/class/net/eth0/device/power/control` to `on` |
| Camera no output | libcamera IPA missing | Check `/usr/lib/libcamera/` for IPA modules |

### Rebuild Specific Package

```bash
# Recompile and reinstall one package
bitbake vehiclecontrol-ecu -c compile -f && bitbake vehiclecontrol-image

# Full clean rebuild of one package
bitbake <package-name> -c cleansstate && bitbake vehiclecontrol-image

# Check sstate cache size
du -sh sstate-cache/
```

---

## License

MIT License — SEA:ME DES Project Team
