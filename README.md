# Building the project

In order to build and test the project, we need a couple of ingridients.
The ingridients have been added as git subtrees in this repository and they have the following remotes:

branch: rpi-5.10.y, remote: https://github.com/raspberrypi/linux.git
branch: main,       remote: https://github.com/u-root/u-root.git
branch: acpi-max,   remote: https://github.com/9elements/u-boot-acpi

It is expected by the reader, that he/she has already installed the neccessary toolchain (aarch64-linux-gnu).
This can be as easy as installing it using your package manager of choice.
It is also required that the `go` toolchain is installed as well as some typical system binaries that Linux needs in order to compile.

## Building u-root
```
cd u-root
go build .
GOARCH=arm64 ./u-root -defaultsh gosh -o initramfs.cpio boot coreboot-app ./cmds/core/* ./cmds/boot/*
cd -
```

## Building Linux kernel

```
cd linux
echo "CONFIG_INITRAMFS_SOURCE=\"../u-root/initramfs.cpio\"" >> arch/arm64/configs/bcm2711_defconfig
echo "CONFIG_CMDLINE_FORCE=y"                               >> arch/arm64/configs/bcm2711_defconfig
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make bcm2711_defconfig
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j$(nproc)
cd -
```

## Building u-boot

```
cd u-boot
ARCH=arm CROSS_COMPILE=aarch64-linux-gnu- make rpi_4_defconfig
ARCH=arm CROSS_COMPILE=aarch64-linux-gnu- make -j$(nproc)
cd -
```

## Build u-boot enviroment

```
cd u-boot
./tools/mkimage -A arm64 -O linux -T script -C none -d ../boot_cmd.txt boot.scr
cd -
```

## Flash contents to sdcard

Your SD card should have at least one partition:
512M FAT32 EFI partition

mount your sd card and copy all ingridients onto the SD Card
```
mount /dev/mmcblk0p0 /mnt
cp linux/arch/arm64/boot/Image /mnt
cp linux/arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dtb /mnt
cp u-boot/u-boot.bin /mnt
cp u-boot/boot.scr /mnt
cp config.txt /mnt
cp bootcode.bin /mnt
cp start4.elf /mnt
unmount /mnt
```

Now plug out the SD card and put it into the raspberry PI 4 B
