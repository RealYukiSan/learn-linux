# A note on building LFS-like system
This repository is an expansion version of [this gist](https://gist.github.com/RealYukiSan/c69d9cc9120c1e5d7b5afcf371e3f79d)

The computer have several stage on booting process, the way to achieve or perform the operation are variant but the overall general stage are same:
- System startup
- Bootloader stage (BIOS and GRUB stuff?)
- Kernel
- Process init

see the [wiki](https://en.wikipedia.org/wiki/Booting_process_of_Linux) for more information.

## The requirement
- busybox for userspace program
- latest stable linux Kernel

## Partition layout and File system
- 1 GB for / with ext4 format
- 1 GB for / /home with ext4 format
- a few MB for /boot with ext2 format (?)
- optional swap partition (?)

## Building LFS-like in the stages
1. Create disk image file instead of formatting physical drive
2. Prepare the linux system on mounted drive
3. Test the newly created linux system on QEMU emulator
4. Transfer the newly created linux system to USB flash drive

## Create the disk image

```bash
dd if=/dev/zero of=root.img bs=1G count=1
mkfs -v -t ext4 ./root.img
```

## Mount and prepare the linux system

We'll populate the disk image with minimum requirement for building a linux system.

```bash
mkdir -v /mnt/newsystem
mount -v -t ext4 root.img /mnt/newsystem
extlinux -i /mnt/newsystem
```

install the busybox, see the gist above.

make sure the owner of `/mnt/newsystem` are root (though by default it's already root), because when chroot, the only user that available was `root`

### Preparing Virtual Kernel File Systems

To be able to perform chroot, we need to prepare the virtual kernel file systems so the kernel will be able to communicate with the kernel itself, see the LFS chapter 7.3 for the rest of instruction.

Just remember to replace the `$LFS` variable with `/mnt/newsystem`.

### `chroot` into the system that have been prepared

same as the LFS instruction above, but using `sh` instead of `bash`.

and here's the list for additional program that doesn't provided by busybox by default:
- [sysklogd](https://github.com/troglobit/sysklogd/releases)
- [sysvinit](https://github.com/slicer69/sysvinit)

### Configure the fstab

## Run it on QEMU

```
qemu-system-x86_64 \
-drive format=raw,file=./boot.img,index=0,media=disk \
-nographic -enable-kvm
```

You also able to using physical device instead of disk image by `-hdb <device>` option

## Misc

### Make a backup using disk image format

transfer device to disk image file:

```
dd if=<device> of=<file.img> oflag=direct conv=fsync bs=4M status=progress
```

to mount, use offset to specify which partition to be mounted.

you can also run the image on QEMU! if the disk img using partition, don't forget to use `/dev/sda1` instead of `/dev/sda` for `root` kernel parameter

### MBR bootstrap code

The file system has unallocated space at the front of the disk, here's the proof:

```
dd if=/dev/zero of=test.img status=progress bs=200M count=1
mkfs.ext2 test.img
hexdump -C test.img | less # check the first offset
dd bs=440 count=1 conv=notrunc if=/usr/lib/syslinux/bios/mbr.bin of=test.img
hexdump -C test.img | less # re-check the first offset
```

see the [MBR bootstrap code creation](https://superuser.com/questions/1206396/mbr-bootstrap-code-creation).

### Useful command for debugging

```
findmnt -A
mount
losetup -a
df -h
```

## Link and references
- [linuxfromscratch](https://www.linuxfromscratch.org/lfs/view/stable) - LFS Guide
- [tldp](https://tldp.org/HOWTO/Bootdisk-HOWTO) - HOWTO create rescue disk
- [Arch Wiki](https://wiki.archlinux.org/title/Installation_guide) - installation guide
- Manual Page
    - `man mount`
    - `cat /proc/filesystems` or `ls /lib/modules/$(uname -r)/kernel/fs`
- [Arch Wiki](https://wiki.archlinux.org/title/Syslinux#BIOS_systems) - BIOS System
- [Syslinux Wiki](https://wiki.syslinux.org/wiki/index.php?title=Library_modules#Syslinux_modules_working_dependencies) - Syslinux modules working dependencies
- [UNIX Stackexchange](https://unix.stackexchange.com/a/151483/606032) - Who provide the UUID in the `root` kernel parameter? initramfs - PARTUUID is a good alternative too!
- [stackoverflow](https://stackoverflow.com/questions/10603104/the-difference-between-initrd-and-initramfs) - initramfs vs initrd
- [askubuntu](https://askubuntu.com/q/1511094/1783505) - create partition on disk image

## Question
is it true that transfering the kernel [directly](https://tldp.org/HOWTO/Bootdisk-HOWTO/x703.html) without bootloader [no longer possible](https://superuser.com/questions/415429/how-to-boot-linux-kernel-without-bootloader)?

