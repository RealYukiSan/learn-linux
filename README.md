# A note on building LFS-like system
This repository is an expansion version of [this gist](https://gist.github.com/RealYukiSan/c69d9cc9120c1e5d7b5afcf371e3f79d)

The computer have several stage on booting process, the way to achieve or perform the operation are variant but the overall general stage are same:
- System startup
- BIOS and Bootloader stage
- Kernel
- Process init

see the [wiki](https://en.wikipedia.org/wiki/Booting_process_of_Linux) for more information.

## The requirement
- busybox for userspace program
- latest stable linux Kernel

## Partition layout and File system
- 200 MB for root filesystem with ext4 format

## Building LFS-like in the stages
1. Create disk image file instead of formatting physical drive
2. Prepare the linux system on mounted drive
3. Test the newly created linux system on QEMU emulator
4. Transfer the newly created linux system to USB flash drive

## Create the disk image 

```bash
dd if=/dev/zero of=linux.img bs=200M count=1 status=progress
```

### Setup partition

enable boot flag and create a partition with whole disk image size.

```bash
fdisk ./linux.img
```

after adding partition on the disk image:
- map the partition to loop-back device
- format the partition
- mount the partition

```bash
sudo losetup -v -P -f linux.img
sudo mkfs.ext4 /dev/loop<n>p<n>
sudo mkdir -v /mnt/newsystem
sudo mount -v /dev/loop<n>p<m> /mnt/newsystem
```

replace <n> with the proper device number, and <m> with partition number.

check if it's successfuly mounted by `findmnt -T /mnt/newsystem`.

later after you've done with the disk image, execute:
```bash
sudo umount -v /mnt/newsystem
sudo losetup -dv /dev/loop<n>
```

in case you want to mount the image, mount the partition by specifying the offset:
```bash
sudo mount linux.img /mnt/newsystem -o loop,offset=$((512*2048))
```

## Prepare the linux system

We'll populate the disk image with minimum requirement for building a linux system.

install the busybox, see the gist above.

make sure the owner of `/mnt/newsystem` are root (though by default it's already root), because when chroot, the only user that available was `root`

### Preparing Virtual Kernel File Systems

To be able to perform chroot, we need to prepare the virtual kernel file systems so the kernel will be able to communicate with the kernel itself, see the LFS chapter 7.3 for the rest of instruction.

```bash
sudo mkdir -pv /mnt/newsystem/{dev,proc,sys,run,tmp}
```

### Preparing standard directory

```bash
sudo mkdir -pv /mnt/newsystem/{etc,home,root,boot/syslinux}
```

Bash is really cool and badass! You can create multiple directories in all sorts of ways, even with just a single line of code.

### Configure the `/etc/fstab`

```bash
sudo cat > /mnt/newsystem/etc/fstab << "EOF"
proc           /proc          proc     nosuid,noexec,nodev 0     0
sysfs          /sys           sysfs    nosuid,noexec,nodev 0     0
tmpfs          /run           tmpfs    defaults            0     0
devtmpfs       /dev           devtmpfs mode=0755,nosuid    0     0
tmpfs          /dev/shm       tmpfs    nosuid,nodev        0     0
devpts         /dev/pts       devpts   gid=5,mode=620      0     0
EOF
```

add `mount -a` later in your init program?

### Install the syslinux bootloader

```bash
extlinux -i /mnt/newsystem/boot/syslinux --device=/dev/loop<n>
```

### Configure syslinux

```bash
sudo cat > /mnt/newsystem/boot/syslinux/extlinux.conf << "EOF"
TIMEOUT 300
ONTIMEOUT limnux

UI vesamenu.c32
MENU TITLE Boot

LABEL limnux
	MENU LABEL Limnux ayooo
	LINUX /boot/bzImage
	APPEND root=/dev/sda1 console=ttyS0 rw
EOF
sudo cp -v /usr/lib/syslinux/bios/{vesamenu.c32,libutil.c32,libcom32.c32}  /mnt/usb/boot/syslinux
```

delete the console and replace the root kernel parameter with PARTUUID if you want to use it with usb flash drive

and the last thing, copy the compiled kernel into `/boot` directory

## Run it on QEMU

```bash
qemu-system-x86_64 \
-drive format=raw,file=./linux.img,index=0,media=disk \
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

### Additional program
the list for additional program that doesn't provided by busybox by default:
- [sysklogd](https://github.com/troglobit/sysklogd/releases)
- [sysvinit](https://github.com/slicer69/sysvinit)

## Link and references
- [linuxfromscratch](https://www.linuxfromscratch.org/lfs/view/stable) - LFS Guide
- [tldp](https://tldp.org/HOWTO/Bootdisk-HOWTO) - HOWTO create rescue disk
- [wiki.archlinux](https://wiki.archlinux.org/title/Installation_guide) - installation guide
- Manual Page
    - `man mount`
    - `cat /proc/filesystems` or `ls /lib/modules/$(uname -r)/kernel/fs`
- [wiki.archlinux](https://wiki.archlinux.org/title/Syslinux#BIOS_systems) - BIOS System
- [wiki.syslinux](https://wiki.syslinux.org/wiki/index.php?title=Library_modules#Syslinux_modules_working_dependencies) - Syslinux modules working dependencies
- [unix](https://unix.stackexchange.com/a/151483/606032) - Who provide the UUID in the `root` kernel parameter? initramfs - PARTUUID is a good alternative too!
- [stackoverflow](https://stackoverflow.com/questions/10603104/the-difference-between-initrd-and-initramfs) - initramfs vs initrd
- [askubuntu](https://askubuntu.com/q/1511094/1783505) - create partition on disk image
- [unix](https://unix.stackexchange.com/q/774947/606032) - install bootloader on unpartitioned disk
- [joe-bergeron](https://www.joe-bergeron.com/posts/Writing%20a%20Tiny%20x86%20Bootloader/) - what is bootloader? tiny bootloader
- [superuser](https://superuser.com/a/1297351/1867794) - auto-mount partition on loop back device

## Question
is it true that transfering the kernel [directly](https://tldp.org/HOWTO/Bootdisk-HOWTO/x703.html) without bootloader [no longer possible](https://superuser.com/questions/415429/how-to-boot-linux-kernel-without-bootloader)?

