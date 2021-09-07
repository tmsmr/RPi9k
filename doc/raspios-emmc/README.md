## Install RaspiOS (arm64) on the CM4's eMMC

To access the eMMC storage of the CM4 on a other box, it has to be booted in MSD mode. Check https://www.raspberrypi.org/documentation/computers/compute-module.html#steps-to-flash-the-emmc for details.

### 1. Set up `usbboot`
Follow the instructions at [Update/Configure the bootloader on the CM4](../bootloader/README.md)

### 2. Download/Verify RaspiOS
- Download required files:
```
wget https://downloads.raspberrypi.org/raspios_lite_arm64/images/raspios_lite_arm64-2021-05-28/2021-05-07-raspios-buster-arm64-lite{.zip,.zip.sha256,.zip.sig}
```

- Verify the image:
```
sha256sum -c 2021-05-07-raspios-buster-arm64-lite.zip.sha256
gpg --no-default-keyring --keyring=/tmp/keyring.gpg --keyserver keyserver.ubuntu.com --recv 54C3DD610D9D1B4AF82A37758738CD6B956F460C # cross-check the key id
gpg --no-default-keyring --keyring=/tmp/keyring.gpg --verify 2021-05-07-raspios-buster-arm64-lite.zip.sig
```

- Unzip the image:
```
unzip 2021-05-07-raspios-buster-arm64-lite.zip
```

### 3. Write RaspiOS
- Power down the CM4IO Board
- Disable eMMC boot by fitting `nRPI_BOOT`
- Connect your box to the micro USB slave port on the CM4IO Board (`J11`)
- Wait for the CM4 to appear and boot it in MSD mode: `sudo ./rpiboot`
- Power up the CM4IO Board and wait for the command to complete
- Check if the new device is available: `dmesg`, `lsblk`
- Double-check if you got the correct device: `cat /sys/class/block/DEVICE/device/vendor` should be `RPi-MSD-`
- Write image: `sudo dd if=2021-05-07-raspios-buster-arm64-lite.img of=/dev/DEVICE bs=4M status=progress`
- Remove `nRPI_BOOT` to enable the normal boot mode

### 4. Enable USB ports
The USB ports on the CM4 / CM4IO are disabled by default. If you are not going to run the OS headless, you probably want to enable the USB ports:
1. Mount the target's `/boot`: e.g. `sudo mount /dev/sda1 /mnt`
2. Append `dtoverlay=dwc2,dr_mode=host` to `/mnt/config.txt`
3. Unmount: `sudo umount /mnt`

## Resources
- https://www.raspberrypi.org/documentation/computers/compute-module.html#steps-to-flash-the-emmc
- https://github.com/raspberrypi/usbboot
- https://datasheets.raspberrypi.org/cm4io/cm4io-datasheet.pdf
