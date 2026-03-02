# pdcMeta-ECU1

Yocto meta layer repository for ECU1 in the PDC (Parking Distance Control) project.

## Repository Structure

```
pdcMeta-ECU1/
└── meta-vehiclecontrol/    # Custom Yocto layer for Raspberry Pi 4 ECU1
```

## What This Repository Contains

This repository holds the `meta-vehiclecontrol` Yocto layer, which defines how to build the ECU1 system image for the Raspberry Pi 4.

The layer includes recipes for:
- VehicleControl ECU application (vehicle control, gamepad input, battery monitoring)
- vsomeip / CommonAPI middleware (IPC between ECU1 and ECU2)
- CAN bus setup (MCP2518FD at 1000 kbps)
- Camera streaming (GStreamer RTP/UDP → Jetson Orin Nano)
- System configuration (network, udev rules, systemd services)

## How to Use

This layer is used as part of a Yocto build. Clone it alongside the other Yocto layers and add it to your `bblayers.conf`.

See [meta-vehiclecontrol/README.md](meta-vehiclecontrol/README.md) for the full build guide, including:
- Host prerequisites
- Yocto layer setup
- `local.conf` configuration
- Runtime verification
- Troubleshooting

## Quick Reference

```bash
# In your Yocto build directory
source poky/oe-init-build-env build-ecu1

# Add meta-vehiclecontrol to bblayers.conf, then:
bitbake vehiclecontrol-image
```

## Target Hardware

- **Board**: Raspberry Pi 4 (64-bit)
- **IP**: 192.168.1.100
- **Camera**: OV5647 (RPi Camera Module v1.3)
- **CAN**: MCP2518FD via SPI

## License

MIT License — SEA:ME DES Project Team
