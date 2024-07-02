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
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
make bcm2711_defconfig
./scripts/kconfig/merge_config.sh .config ../acpi.conf ../uroot_initramfs.conf
make Image -j$(nproc)
make dtbs -j$(nproc)
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

## Workflow

This project uses git subtrees. The Idea is that you only use this repository to make changes and leave the u-boot-acpi repo alone.

If you do changes inside of the u-boot repository you can commit them like usual. In order to push a commit that makes changes inside the u-boot directory do the following command:
```
git subtree push --prefix u-boot [remote that points to u-boot-acpi repo] master
```
This will automatically push the changes to the u-boot-acpi repository as well as keep the change in this repository.

## Testing in QEMU:

Starting with QEMU 9.0 you can run the RPI 4 in qemu:

Update boot_cmd.txt:
```
fatload mmc 1:1 ${kernel_addr_r} Image
fatsize mmc 1:1 Image
#setenv bootargs "console=ttyS0,115200 console=tty1"
bootefi ${kernel_addr_r}:${filesize} -
```

To run u-boot and boot from MMC run:
```
qemu-system-aarch64 -machine raspi4b -kernel u-boot/u-boot.bin -cpu cortex-a72 -smp 4 -m 2G -drive file=raspbian.img,format=raw,index=0 -dtb linux/arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dtb -nographic
```
Note: Assumes you have *Image* copied into the raspbian.img image.
