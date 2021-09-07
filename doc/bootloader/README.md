# Update/Configure the bootloader on the CM4

At the time of writing, the factory bootloader does not support booting from a NVME drive (via PCIe).
To update the bootloader on the CM4, [raspberrypi/usbboot](https://github.com/raspberrypi/usbboot) is used.

### 1. Build `usbboot`
- Build `usbboot` on 'any other' GNU/Linux box, which will later be used to flash the EEPROM of the CM4. I'm using a spare RPi 3 with the latest Raspberry Pi OS (32 bit) installed.
  Besides a regular GCC/Make toolchain, `libusb-dev` is needed:
```
sudo apt install -y libusb-1.0-0-dev git
git clone --depth=1 https://github.com/raspberrypi/usbboot
cd usbboot
make
```

### 2. Configure the bootloader
- `recovery/boot.conf`

```
[all]
BOOT_UART=0
WAKE_ON_GPIO=1
POWER_OFF_ON_HALT=0

#
# NVME -> eMMC -> error
#
BOOT_ORDER=0xe16

#
# eMMC -> error (force boot from eMMC with a bootable NVME drive plugged in)
#
#BOOT_ORDER=0xe1

ENABLE_SELF_UPDATE=0
```

- `recovery/config.txt`

```
uart_2ndstage=1
eeprom_write_protect=1
```

With this configuration, the EEPROM will be write-protected when the jumper `EEPROM_nWP` (On the CM4IO Board) is fitted.

### 3. Build the bootlaoder
- `recovery/update-pieeprom.sh`

### 4. Flash the bootloader
- Power down the CM4IO Board
- Ensure `EEPROM_nWP` is not fitted
- Disable eMMC boot by fitting `nRPI_BOOT`
- Connect your box to the micro USB slave port on the CM4IO Board (`J11`)
- Initialize the flashing process `sudo ./rpiboot -d recovery` (It will wait for the CM4 to appear)
- Power up the CM4IO Board and wait for the flashing process to complete
- Remove `nRPI_BOOT` to enable the normal boot mode
- (Fit `EEPROM_nWP` to write-protect the EEPROM)

## Resources
- https://www.raspberrypi.org/documentation/computers/compute-module.html
- https://github.com/raspberrypi/usbboot
- https://datasheets.raspberrypi.org/cm4io/cm4io-datasheet.pdf
- https://www.raspberrypi.org/documentation/computers/raspberry-pi.html#BOOT_ORDER
