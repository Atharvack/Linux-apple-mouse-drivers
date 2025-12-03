# Apple Magic Mouse 2 Linux Driver

A Linux kernel module that enables full functionality for the Apple Magic Mouse 2, including scroll wheel emulation using the glass touch surface.

## Features

- **Scroll Wheel Emulation**: Use 2-finger swipe gestures on the glass surface to scroll
- **Vertical Scrolling**: Swipe up/down
- **Horizontal Scrolling**: Swipe left/right
- **Button Support**: Left click, right click, and middle click
- **Configurable Parameters**: Adjust scroll speed, acceleration, and button behavior
- **Bluetooth Support**: Works with Bluetooth-connected Magic Mouse 2

## Supported Devices

- **Apple Magic Mouse 2** (Product ID: 0x0323)
- **Connection**: Bluetooth
- **Tested on**: Ubuntu 24.04 with kernel 6.14.0-36-generic

## Requirements

- Linux kernel 4.18+ (tested on 6.14.0-36-generic)
- Kernel headers for your running kernel: `sudo apt install linux-headers-$(uname -r)`
- Build tools: `sudo apt install build-essential`
- Magic Mouse 2 paired and connected via Bluetooth

## Quick Start

### 1. Clone and Build

```bash
git clone <repository-url>
cd Linux-apple-mouse-drivers/magicmouse_driver
make
```

### 2. Install

**Backup your original module first:**

```bash
sudo cp /lib/modules/$(uname -r)/kernel/drivers/hid/hid-magicmouse.ko.zst \
        /lib/modules/$(uname -r)/kernel/drivers/hid/hid-magicmouse.ko.zst.backup
```

**Install the custom module:**

```bash
sudo cp hid-magicmouse.ko /lib/modules/$(uname -r)/kernel/drivers/hid/
sudo rm /lib/modules/$(uname -r)/kernel/drivers/hid/hid-magicmouse.ko.zst
sudo depmod -a
```

### 3. Load Module

```bash
sudo rmmod hid_magicmouse 2>/dev/null
sudo insmod /lib/modules/$(uname -r)/kernel/drivers/hid/hid-magicmouse.ko \
    emulate_scroll_wheel=1 scroll_speed=32 emulate_3button=1
```

### 4. Reconnect Mouse

Turn your Magic Mouse 2 **off and back on** to have the new driver claim the device.

### 5. Test

- **Scroll**: Swipe 2 fingers up/down or left/right on the glass surface
- **Click**: Touch down for left click
- **Right Click**: Two-finger touch down
- **Middle Click**: Single finger in middle area (or 3-finger click if configured)

## Configuration

### Module Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `emulate_scroll_wheel` | bool | `true` | Enable scroll wheel emulation |
| `scroll_speed` | uint | `32` | Scroll speed (0=slow, 63=fast) |
| `emulate_3button` | bool | `true` | Enable 3-button emulation |
| `scroll_acceleration` | bool | `false` | Enable scroll acceleration |
| `middle_click_3finger` | bool | `false` | Use 3-finger click for middle button |

### Make Configuration Persistent

Create a modprobe configuration file:

```bash
echo "options hid_magicmouse emulate_scroll_wheel=1 scroll_speed=32 emulate_3button=1" | \
    sudo tee /etc/modprobe.d/hid-magicmouse.conf
```

Update initramfs to apply at boot:

```bash
sudo update-initramfs -u
```

After reboot, the module will load automatically with your settings.

## Troubleshooting

### Check if Module is Loaded

```bash
lsmod | grep hid_magicmouse
```

### Verify Device Recognition

```bash
cat /proc/bus/input/devices | grep -A8 "Magic Mouse"
```

Expected output should show:
- Name: "Apple Inc. Magic Trackpad 2" (the driver uses trackpad mode for touch)
- Handlers: mouse3 event19 (numbers may vary)

### Check Module Parameters

```bash
cat /sys/module/hid_magicmouse/parameters/scroll_speed
cat /sys/module/hid_magicmouse/parameters/emulate_scroll_wheel
```

### Test Input Events

```bash
sudo apt install evtest
sudo evtest
```

Select your Magic Mouse device and test:
- REL_WHEEL events when scrolling vertically
- REL_HWHEEL events when scrolling horizontally
- BTN_LEFT, BTN_RIGHT events when clicking

### Mouse Not Working After Installation

1. **Restore original module:**
   ```bash
   sudo cp /lib/modules/$(uname -r)/kernel/drivers/hid/hid-magicmouse.ko.zst.backup \
           /lib/modules/$(uname -r)/kernel/drivers/hid/hid-magicmouse.ko.zst
   sudo rm /lib/modules/$(uname -r)/kernel/drivers/hid/hid-magicmouse.ko
   sudo depmod -a
   sudo rmmod hid_magicmouse
   sudo modprobe hid_magicmouse
   ```

2. **Reconnect your mouse**

### Scrolling Too Fast or Slow

Adjust scroll speed (0-63):

```bash
echo 50 | sudo tee /sys/module/hid_magicmouse/parameters/scroll_speed
```

Or reload with different speed:

```bash
sudo rmmod hid_magicmouse
sudo insmod /lib/modules/$(uname -r)/kernel/drivers/hid/hid-magicmouse.ko scroll_speed=50
```

## Uninstallation

### Restore Original Driver

```bash
sudo rm /lib/modules/$(uname -r)/kernel/drivers/hid/hid-magicmouse.ko
sudo cp /lib/modules/$(uname -r)/kernel/drivers/hid/hid-magicmouse.ko.zst.backup \
        /lib/modules/$(uname -r)/kernel/drivers/hid/hid-magicmouse.ko.zst
sudo depmod -a
sudo rmmod hid_magicmouse
sudo modprobe hid_magicmouse
```

### Remove Configuration

```bash
sudo rm /etc/modprobe.d/hid-magicmouse.conf
sudo update-initramfs -u
```

## How It Works

### Technical Overview

The Apple Magic Mouse 2 uses a touch-sensitive glass surface to detect finger gestures. This driver:

1. **Recognizes** the Magic Mouse 2 device (Product ID 0x0323)
2. **Processes** touch data from the glass surface (MOUSE2_REPORT_ID: 0x12)
3. **Converts** multi-finger swipe gestures into scroll wheel events
4. **Emulates** traditional mouse buttons based on touch patterns

### Key Implementation Details

- **Report Structure**: 14-byte prefix + 8 bytes per detected finger
- **Touch States**: START → DRAG → NONE
- **Scroll Calculation**: Delta between touch positions during DRAG state
- **Button Detection**: Based on number of fingers and position

For detailed technical documentation, see [IMPLEMENTATION.md](IMPLEMENTATION.md).

## Development

### Building from Source

```bash
cd magicmouse_driver
make clean
make
```

### Testing Changes

```bash
# Install your changes
sudo cp hid-magicmouse.ko /lib/modules/$(uname -r)/kernel/drivers/hid/
sudo depmod -a

# Reload module
sudo rmmod hid_magicmouse
sudo insmod /lib/modules/$(uname -r)/kernel/drivers/hid/hid-magicmouse.ko

# Reconnect mouse
```

### Code Structure

```
magicmouse_driver/
├── hid-magicmouse.c    # Main driver implementation
├── hid-ids.h           # Device ID definitions
├── Makefile            # Build configuration
└── CLAUDE.md           # AI assistant context
```

## Credits

This driver builds upon the excellent work of:

- **Michael Poole** <mdpoole@troilus.org> - Original hid-magicmouse driver
- **Chase Douglas** <chase.douglas@canonical.com> - Multi-touch support
- **Rohit Pidaparthi** <rohitkernel@gmail.com> - Magic Mouse 2 and Magic Trackpad 2 support

Reference implementation: [Linux-Magic-Trackpad-2-Driver](https://github.com/rohitpid/Linux-Magic-Trackpad-2-Driver)

## License

This program is free software; you can redistribute it and/or modify it under the terms of the **GNU General Public License version 2** (or later) as published by the Free Software Foundation.

This is the same license as the Linux kernel.

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly on your hardware
5. Submit a pull request with a clear description

## Support

- **Issues**: Report bugs or request features via GitHub Issues
- **Discussions**: Share your experience and ask questions in Discussions
- **Documentation**: Check IMPLEMENTATION.md for technical details

## Disclaimer

This driver is provided "as is" without warranty of any kind. Always backup your system before installing kernel modules. The authors are not responsible for any damage or data loss.

---

**Note**: This driver operates only on the HID input layer and does not modify any system-critical components. It is safe to uninstall at any time by restoring the original module.
