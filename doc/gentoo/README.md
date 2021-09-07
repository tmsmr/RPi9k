# Install Gentoo (arm64)

## WARNING
This is no detailed guide, neither regarding the Raspberry Pi nor Gentoo. You should have some experience working with modern GNU/Linux environments and tools to find this sufficient!

## General
While the 'official' Gentoo 64-bit guides for the RPi suggest cross-compiling the kernel (RaspiOS arm64 is pretty new and usually you don't have two storage devices attached to a RPi),
we can perform a more convenient install since we have RaspiOS arm64 running on the eMMC drive already.

Besides some RPi-specific resources (kernel, firmware, wifi-driver), a 'normal' `chroot`-installation is possible.

## Install base system

### Create Partitions/Filesystems on the NVME drive
- Create a MS-DOS partition table
- Create the partitions:
```
Device         Boot   Start      End  Sectors  Size Id Type
/dev/nvme0n1p1 *       2048   526335   524288  256M  c W95 FAT32 (LBA)             # Bootable, Type c
/dev/nvme0n1p2       526336  4720639  4194304    2G 82 Linux swap / Solaris        # Type 82
/dev/nvme0n1p3      4720640 21497855 16777216    8G 83 Linux
```
- Make the filesystems:
```
mkfs -t vfat -F 32 /dev/nvme0n1p1
mkswap /dev/nvme0n1p2
mkfs -t ext4 /dev/nvme0n1p3
```
After completing the installation, i'm going to create a backup of the used space with `dd` as 'baseline image', that's why `nvme0n1p3` is so small. After creating the backup, we will resize the partition/filesystem...

Check https://wiki.gentoo.org/wiki/Raspberry_Pi_3_64_bit_Install#Partition_the_microSD_card for details.

### Activate swap, Mount root
```
swapon /dev/nvme0n1p2
mkdir -p /mnt/gentoo
mount /dev/nvme0n1p3 /mnt/gentoo
```

### Install stage 3
- Download and verify stage3-arm64-systemd (If you prefer OpenRC, you probably know what are you doing and should be able to adjust the following steps accordingly):
```
wget http://distfiles.gentoo.org/releases/arm64/autobuilds/20210905T230622Z/stage3-arm64-systemd-20210905T230622Z{.tar.xz,.tar.xz.DIGESTS.asc}

gpg --keyserver hkps://keys.gentoo.org --recv 13EBBDBEDE7A12775DFDB1BABB572E0E2D182910 # cross-check the key id
gpg --verify stage3-arm64-systemd-20210905T230622Z.tar.xz.DIGESTS.asc

# only SHA512 on stage3-arm64-systemd-20210905T230622Z.tar.xz is expected to match...
sha512sum -c stage3-arm64-systemd-20210905T230622Z.tar.xz.DIGESTS.asc
```

- Extract stage3-arm64-systemd:
```
tar xpf stage3-arm64-systemd-*.tar.xz --xattrs-include='*.*' --numeric-owner -C /mnt/gentoo
```

### Copy some files from the running RaspiOS we need later

```
mkdir /mnt/gentoo/install
cp /boot/{config.txt,cmdline.txt} /mnt/gentoo/install/
cp /lib/firmware/brcm/brcmfmac43455-sdio* /mnt/gentoo/install/
```

### Mount boot, chroot into Gentoo
You probably want to use `screen` or `tmux` from now on if you are connected via `ssh`...
```
mount /dev/nvme0n1p1 /mnt/gentoo/boot

mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount -t proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /tmp /mnt/gentoo/tmp

cp /etc/resolv.conf /mnt/gentoo/etc

chroot /mnt/gentoo /bin/bash

source /etc/profile
export PS1="(chroot) ${PS1}"
```

### Prepare Portage
- Generate locales:
  - Select locales in `/etc/locale.gen`
  - Generate them with `locale-gen`
- Configure `/etc/portage/make.conf` (adjust to your needs):
```
COMMON_FLAGS="-march=native -O2 -pipe"
...
MAKEOPTS="-j5"
GENTOO_MIRRORS="https://ftp-stud.hs-esslingen.de/pub/Mirrors/gentoo/"
```
- Populate `repos.conf`
```
mkdir -p /etc/portage/repos.conf
cp /usr/share/portage/config/repos.conf /etc/portage/repos.conf/gentoo.conf
```
- Install ebuild repository snapshot and sync it
```
emerge-webrsync
emerge --sync --quiet
```
- Ensure the correct profile is selected
```
eselect profile list
(eselect profile set X)
```

### Update @world

```
emerge --update --deep --newuse @world
```

## Build and install RPi kernel
- Install dependencies

```
emerge dev-vcs/git sys-devel/bc
```

- Get the sources (*Default branch currently rpi-5.10.y, HEAD at 1c384f032011fe012db04802c09c3b0437d92215*)
```
cd /usr/src
git clone --depth=1 https://github.com/raspberrypi/linux 
```
- Generate default config for the RPi 4

```
cd linux
make bcm2711_defconfig
```

- Adjust `.config` to your needs. The only change i made was setting the default CPUFreq Governor to `ondemand`:
```
--- .config.orig        2021-09-07 14:54:22.217232482 -0000
+++ .config     2021-09-07 14:54:39.520907181 -0000
@@ -529,9 +529,9 @@
 CONFIG_CPU_FREQ_GOV_COMMON=y
 CONFIG_CPU_FREQ_STAT=y
 # CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE is not set
-CONFIG_CPU_FREQ_DEFAULT_GOV_POWERSAVE=y
+# CONFIG_CPU_FREQ_DEFAULT_GOV_POWERSAVE is not set
 # CONFIG_CPU_FREQ_DEFAULT_GOV_USERSPACE is not set
-# CONFIG_CPU_FREQ_DEFAULT_GOV_ONDEMAND is not set
+CONFIG_CPU_FREQ_DEFAULT_GOV_ONDEMAND=y
 # CONFIG_CPU_FREQ_DEFAULT_GOV_CONSERVATIVE is not set
 # CONFIG_CPU_FREQ_DEFAULT_GOV_SCHEDUTIL is not set
 CONFIG_CPU_FREQ_GOV_PERFORMANCE=y
```

- Build the kernel, the kernel modules and the device tree binaries/overlays:
```
make -j5 Image modules dtbs
```

- Get yourself a coffee ... do the dishes ... read a book ...

- Install the kernel, the kernel modules, the device tree binary for the CM4 and all device tree overlays:
```
cp arch/arm64/boot/Image /boot/kernel8.img
make modules_install
cp arch/arm64/boot/dts/broadcom/bcm2711-rpi-cm4.dtb /boot/
mkdir -p /boot/overlays
cp arch/arm64/boot/dts/overlays/*.dtbo /boot/overlays/
```
*Don't forget to update all these files when building a new kernel!*

## Install RPi firmware
- Clone repository: `git clone --depth=1 https://github.com/raspberrypi/firmware.git /opt/raspi-firmware`
- Install firmware: `cp /opt/raspi-firmware/boot/{fixup*,start*} /boot/`

*master, HEAD at ac362357ef910d2fd2b688abef5e5fbb875d98a5*

## Configure the bootloader
We copied `config.txt` and `cmdline.txt` from RaspiOS before. Place them at `/boot`
```
mv /install/{config.txt,cmdline.txt} /boot/
```
- `config.txt` should contain `dtoverlay=dwc2,dr_mode=host` if you need the USB ports on the CM4/CM4IO
- Adjust the `root=` parameter in `cmdline.txt` to point to the correct device, e.g.:
```
console=serial0,115200 console=tty1 root=/dev/nvme0n1p3 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait
```

## Install RPi userland
Some tools are not working on 64-bit systems, see https://github.com/raspberrypi/userland/blob/master/README.md.

- Clone repositorty: `git clone --depth=1 https://github.com/raspberrypi/userland.git /opt/raspi-userland`
- Build and install userland tools:
```
cd /opt/raspi-userland
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release -DARM64=ON ../
make
make install
```

- You have to add `/opt/vc/lib` to your `LD_LIBRARY_PATH` to run those commands. e.g.:
```
LD_LIBRARY_PATH=/opt/vc/lib /opt/vc/bin/vcgencmd measure_temp
```

## Install the wifi driver
We are using the firmware files from RaspiOS we copied earlier:
```
mkdir -p /lib/firmware/brcm
mv /install/brcmfmac43455-sdio* /lib/firmware/brcm/
rmdir /install # should be empty now
```

## Generate `/etc/fstab`
```
/dev/nvme0n1p1          /boot           vfat            noauto,noatime  1 2
/dev/nvme0n1p2          none            swap            sw              0 0
/dev/nvme0n1p3          /               ext4            noatime         0 1
```
Note that `/boot` is not mounted automatically (on purpose). If you want to upgrade the kernel, device tree files or firmware files, mount it manually...

## Install NetworkManager
*If you don't hate me because of stage3-arm64-systemd yet, most likely you will because i'm using NetworkManager :). Skip it, configure the network however you want...*

- Append `USE="${USE} networkmanager"` to `/etc/portage/make.conf`
- Create `/etc/portage/package.use/networkmanager`:
```
net-misc/networkmanager tools
net-wireless/wpa_supplicant dbus
```
- Install NetworkManager: `emerge net-misc/networkmanager`

## Finalizing
- Set a password for root
- Exit the `chroot`
- Power down the system: `systemctl poweroff`
- Set `BOOT_ORDER=0xe16`: [Update/Configure the bootloader on the CM4](../bootloader/README.md) to boot from the NVME drive
- Boot your Gentoo 64-bit from NVME
- After the box booted Gentoo with systemd, we can complete the configuration:
```
systemd-machine-id-setup
hostnamectl set-hostname RPi9k
localectl set-locale LANG=en_US.utf8
localectl set-keymap de
env-update && source /etc/profile
timedatectl set-timezone Europe/Berlin
timedatectl set-ntp true
systemctl enable NetworkManager && systemctl start NetworkManager
```

## Connect to wifi using nmcli
```
nmcli device wifi rescan
nmcli device wifi list
nmcli device wifi connect SSID-Name --ask
```

## Create a backup of the base system
T.B.D

## Packages for RPi specific resources
I have not testet them yet, but there are some packages available which would make the installation and maintenance easier:
- https://packages.gentoo.org/packages/sys-kernel/raspberrypi-sources
- https://packages.gentoo.org/packages/sys-kernel/raspberrypi-image
- https://packages.gentoo.org/packages/sys-boot/raspberrypi-firmware
- https://packages.gentoo.org/packages/sys-firmware/raspberrypi-wifi-ucode
- https://packages.gentoo.org/packages/media-libs/raspberrypi-userland-bin

## Resources
- https://wiki.gentoo.org/wiki/Raspberry_Pi4_64_Bit_Install
- https://wiki.gentoo.org/wiki/Raspberry_Pi_3_64_bit_Install
- https://www.raspberrypi.org/documentation/computers/linux_kernel.html
- https://github.com/raspberrypi/linux
- https://github.com/raspberrypi/firmware
- https://wiki.gentoo.org/wiki/Systemd

