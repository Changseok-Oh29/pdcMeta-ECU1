# meta-vehiclecontrol

Yocto layer for ECU1 (VehicleControl ECU) — PiRacer vehicle control system running on Raspberry Pi 4.

## Overview

This layer builds a minimal Linux image for the VehicleControl ECU. It uses vsomeip/CommonAPI middleware to expose vehicle control services to the head-unit (ECU2/Jetson Orin Nano) over Ethernet.

## System Architecture

| Role | Hardware | IP |
|------|----------|----|
| **ECU1** — Service Provider + Routing Manager | Raspberry Pi 4 + PiRacer AI Kit | 192.168.1.100 |
| **ECU2** — Service Consumer | Jetson Orin Nano | 192.168.1.101 |

**ECU1 capabilities:**
- Vehicle control (steering, throttle, gear management)
- Battery monitoring (INA219)
- Gamepad input handling
- Reverse camera streaming (GStreamer → Jetson via RTP/UDP)
- vsomeip service provider (Service ID: `0x1234`)
- CAN interface setup (MCP2518FD at 1000 kbps)

## Layer Dependencies

Yocto **4.0 Kirkstone (LTS)**:
- `meta` (poky) — `kirkstone` branch
- `meta-raspberrypi` — `kirkstone` branch
- `meta-openembedded/meta-oe` — `kirkstone` branch
- `meta-openembedded/meta-python` — `kirkstone` branch
- `meta-openembedded/meta-multimedia` — `kirkstone` branch (for libcamera, GStreamer)

## Build

### Prerequisites (host)

```bash
# Ubuntu 20.04 or 22.04 recommended
sudo apt-get install -y \
    gawk wget git diffstat unzip texinfo gcc build-essential chrpath socat \
    cpio python3 python3-pip python3-pexpect xz-utils debianutils iputils-ping \
    python3-git python3-jinja2 libegl1-mesa libsdl1.2-dev pylint xterm \
    python3-subunit mesa-common-dev zstd liblz4-tool file
```

Disk space: ~50 GB free recommended (first build downloads ~10 GB of sources).

### Step 1 — Clone Yocto layers

```bash
mkdir -p ~/yocto && cd ~/yocto

# Poky
git clone -b kirkstone git://git.yoctoproject.org/poky

# meta-raspberrypi
git clone -b kirkstone git://git.yoctoproject.org/meta-raspberrypi

# meta-openembedded
git clone -b kirkstone git://git.openembedded.org/meta-openembedded
```

### Step 2 — Initialize build environment

```bash
cd ~/yocto
source poky/oe-init-build-env build-ecu1
```

### Step 3 — Configure bblayers.conf

Add to `build-ecu1/conf/bblayers.conf`:

```
BBLAYERS ?= " \
    /home/seame/yocto/poky/meta \
    /home/seame/yocto/poky/meta-poky \
    /home/seame/yocto/meta-raspberrypi \
    /home/seame/yocto/meta-openembedded/meta-oe \
    /home/seame/yocto/meta-openembedded/meta-python \
    /home/seame/yocto/meta-openembedded/meta-multimedia \
    /home/seame/PDC/headunit/DES_Head-Unit/meta/meta-vehiclecontrol \
"
```

### Step 4 — Configure local.conf

Add to `build-ecu1/conf/local.conf`:

```
MACHINE = "raspberrypi4-64"

# Enable camera
ENABLE_UART = "1"
ENABLE_I2C = "1"

# GPU memory for camera
GPU_MEM = "128"

# Parallel build tuning (adjust to your CPU)
BB_NUMBER_THREADS = "8"
PARALLEL_MAKE = "-j8"

# Enable systemd
DISTRO_FEATURES:append = " systemd"
VIRTUAL-RUNTIME_init_manager = "systemd"

# RPI-specific
RPI_USE_U_BOOT = "0"
ENABLE_SPI_BUS = "1"
```

### Step 5 — Build

```bash
cd ~/yocto/build-ecu1
bitbake vehiclecontrol-image
```

First build: 2–4 hours. Subsequent builds: 10–30 minutes.

### Step 6 — Flash SD card

```bash
cd ~/yocto/build-ecu1/tmp/deploy/images/raspberrypi4-64/

# Check your SD card device
lsblk

# Flash (replace /dev/sdX with actual device)
sudo dd if=vehiclecontrol-image-raspberrypi4-64.rootfs.rpi-sdimg \
    of=/dev/sdX bs=4M status=progress conv=fsync && sync
```

## Included Packages

| Category | Package | Version |
|----------|---------|---------|
| Communication | vsomeip | 3.5.8 |
| Communication | commonapi-core | 3.2.4 |
| Communication | commonapi-someip | 3.2.4 |
| Communication | boost | system |
| Hardware | pigpio | latest |
| Hardware | i2c-tools | latest |
| Hardware | can-utils | latest |
| Camera | libcamera + libcamera-gst | latest |
| Camera | gstreamer1.0-plugins-{base,good,bad,ugly} | latest |
| System | systemd, openssh, bash, iproute2 | latest |
| App | vehiclecontrol-ecu | 1.0 |

## Network

- ECU1 fixed IP: `192.168.1.100/24` (configured via systemd-networkd)
- Multicast routing: `224.0.0.0/4` for vsomeip Service Discovery
- eth0 power management: set to `on` at boot (prevents packet loss)

## Default Login

| Field | Value |
|-------|-------|
| Username | `root` |
| Password | `raspberry` |

## Hardware Configuration

- I2C enabled at 400 kHz
- Supported I2C devices:
  - PCA9685 `0x40` — steering servo
  - PCA9685 `0x60` — motor controller
  - INA219 `0x41` — battery monitor
- CAN: MCP2518FD via SPI, 1000 kbps (`can0`)
- Camera: OV5647 (RPi Camera Module v1.3) on i2c bus 10

## Systemd Services

| Service | Description | Auto-start |
|---------|-------------|-----------|
| `vehiclecontrol-ecu.service` | Main vehicle control app | enabled |
| `can-setup.service` | CAN interface initialization | enabled |
| `camera-streaming.service` | GStreamer RTP stream to Jetson | enabled |

## Runtime Verification

After booting the RPi:

```bash
# Check all services
systemctl status vehiclecontrol-ecu
systemctl status can-setup
systemctl status camera-streaming

# Verify CAN interface
ip link show can0

# Verify eth0 power management
cat /sys/class/net/eth0/device/power/control  # should be "on"

# Verify vsomeip routing
ls /tmp/vsomeip-0   # socket created by routing manager

# Check camera stream (on Jetson side)
gst-launch-1.0 udpsrc port=5000 \
    caps="application/x-rtp,encoding-name=H264,payload=96" \
    ! rtph264depay ! h264parse ! nvv4l2decoder ! nv3dsink
```

## Troubleshooting

### Build Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `boost` fetch fails | Network timeout | Re-run `bitbake vehiclecontrol-image` |
| `vsomeip` compile fails | Missing boost dependency | Check RDEPENDS in vsomeip recipe |
| Layer compatibility error | Wrong branch | Ensure all layers on `kirkstone` branch |
| Disk space error | Build cache full | `du -sh ~/yocto/build-ecu1/tmp` — clean sstate if >30 GB |
| Recipe parse error | `.bb` syntax issue | `bitbake -p` to validate all recipes |

### Runtime Errors

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `vehiclecontrol-ecu` fails to start | vsomeip socket missing | Check `journalctl -u vehiclecontrol-ecu` |
| I2C devices not found | I2C not enabled | Verify `dtparam=i2c_arm=on` in `/boot/config.txt` |
| No network | IP not assigned | Check `systemctl status systemd-networkd` |
| vsomeip no service found | Multicast routing | Run `ip route add 224.0.0.0/4 dev eth0` |
| pigpio daemon error | Permission issue | Check `/var/log/pigpiod.log` |
| Camera stream no output | libcamera IPA | Check `libcamera` IPA modules installed at `/usr/lib/libcamera/` |
| 40–60% packet loss | eth0 power management | Verify `/sys/class/net/eth0/device/power/control` = `on` |

### Rebuild Workflow

```bash
# Rebuild only the main app
bitbake vehiclecontrol-ecu -c compile -f && bitbake vehiclecontrol-image

# Clean and full rebuild of one package
bitbake <package-name> -c cleansstate && bitbake vehiclecontrol-image

# Check sstate cache size
du -sh ~/yocto/build-ecu1/sstate-cache/
```

## Layer Structure

```
meta-vehiclecontrol/
├── conf/
│   └── layer.conf
├── recipes-connectivity/
│   ├── vsomeip/          # vsomeip 3.5.8 cross-compile
│   ├── commonapi/        # CommonAPI Core + SOMEIP binding
│   └── can-setup/        # CAN interface systemd service
├── recipes-multimedia/
│   ├── libcamera/        # libcamera bbappend (raspberrypi IPA)
│   └── camera-streaming/ # GStreamer RTP streaming service
├── recipes-vehiclecontrol/
│   └── vehiclecontrol-ecu/  # Main application recipe
├── recipes-core/
│   ├── images/           # vehiclecontrol-image.bb
│   ├── udev/             # udev rules (eth0 power, GPIO)
│   └── packagegroups/    # packagegroup-vehiclecontrol.bb
└── README.md
```

## License

MIT License — SEA:ME DES Project Team
