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

same as the LFS instruction above, but using sh instead of bash.

and here's the list for additional program that doesn't provided by busybox by default:
- [sysklogd](https://github.com/troglobit/sysklogd/releases)
- [sysvinit]()

## Link and references
- [LFS Guide](https://www.linuxfromscratch.org/lfs/view/stable)
- [tldp HOWTO](https://tldp.org/HOWTO/Bootdisk-HOWTO)
- [Arch Wiki](https://wiki.archlinux.org/title/Installation_guide) - installation guide
- Manual Page
    - `man mount`
    - `cat /proc/filesystems` or `ls /lib/modules/$(uname -r)/kernel/fs`

## Question
is it true that transfering the kernel [directly](https://tldp.org/HOWTO/Bootdisk-HOWTO/x703.html) without bootloader [no longer possible](https://superuser.com/questions/415429/how-to-boot-linux-kernel-without-bootloader)?

