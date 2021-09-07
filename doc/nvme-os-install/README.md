# Install RaspiOS / Gentoo on the NVME drive

## Preparation
- Update the bootloader (EEPROM) and set `BOOT_ORDER=0xe1`(We DON'T want to boot from NVME right now): [Update/Configure the bootloader on the CM4](../bootloader/README.md)
- Install RaspiOS on the CM4's eMMC drive: [Install RaspiOS (arm64) on the CM4's eMMC](../raspios-emmc/README.md)

## A. RaspiOS
Booting the CM4 from the eMMC drive, you should have `/dev/nvme0n1` available.
1. Download/Verify RaspiOS and write it to the NVME drive
2. Currently the RaspiOS image does NOT contain the latest firmware/kernel, which is required to boot from NVME. Update via:
```
sudo mkdir -p /mnt/{boot,root}
sudo mount /dev/nvme0n1p1 /mnt/boot
sudo mount /dev/nvme0n1p2 /mnt/root
sudo ROOT_PATH=/mnt/root BOOT_PATH=/mnt/boot rpi-update
```
(Details at https://www.raspberrypi.org/documentation/computers/raspberry-pi.html#nvme-ssd-boot)
3. Enable the USB ports as described in [Install RaspiOS (arm64) on the CM4's eMMC](../raspios-emmc/README.md)
4. Shutdown: `systemctl poweroff`
5. Set `BOOT_ORDER=0xe16`: [Update/Configure the bootloader on the CM4](../bootloader/README.md)
6. Boot your RaspiOS 64-bit from NVME

## B. Gentoo
- Boot RaspiOS on your CM4 (from eMMC)
- Setup Networking, Locales etc.
- Continue at [Install Gentoo (arm64)](../gentoo/README.md)

## Resources
- https://www.raspberrypi.org/documentation/computers/raspberry-pi.html#nvme-ssd-boot
