---
layout: post
title: How to restore an accidentally deleted Windows Boot Manager with a Windows / Arch Linux dual-boot installation
tags: windows arch linux dual-boot bootloader systemd-boot
---


During my Arch Linux installation, I accidentally formatted the EFI system partition created by my existing Windows installation and replaced it with a new EFI system that starts Arch Linux using `systemd-boot`.
While my initial goal was to have a Windows 10 / Arch Linux dual boot, I am currently only able to boot under Arch since I deleted my Windows Boot Manager files.

In this article, I will describe how I managed to reinstall Windows Boot Manager on my EFI system and create a Windows 10 entry in systemd-boot.

## Current partitions

Here is the list of my partitions after my Arch installation:
```bash
$ sudo fdisk -l
Device          Start        End    Sectors   Size Type
/dev/sda1        2048     534527     532480   260M EFI System
/dev/sda2      534528     567295      32768    16M Microsoft reserved
/dev/sda3      567296  926358296  925791001 441.5G Microsoft basic data
/dev/sda4   926359552  927473663    1114112   544M Windows recovery environment
/dev/sda5   927475712 1951475711 1024000000 488.3G Linux filesystem
/dev/sda6  1951475712 1953523711    2048000  1000M Windows recovery environment
```

As you can see, my Linux filesystem is `/dev/sda5` and my Windows filesystem is `/dev/sda3`. The other partitions are managed by the Windows installation: `/dev/sda2` is a [Microsoft reserved partition](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-uefigpt-based-hard-drive-partitions#microsoft-reserved-partition-msr) and both `/dev/sda4` and `/dev/sda6` are recovery partitions ([Windows RE](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-uefigpt-based-hard-drive-partitions#recovery-tools-partition)) automatically created by Windows 10.

Here, the interesting partition is `/dev/sda1`, which is the EFI system mounted as `/boot` for my Linux installation. As I mentioned before, this partition was initially a [System partition](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-uefigpt-based-hard-drive-partitions#system-partition) managed by Windows, but since I formatted the partition during my Linux installation, I now need to restore my Windows Boot Manager files on the partition in order to be able to boot both Windows 10 and Linux.

## Restoring the Windows Boot Manager files

The first step is to create a bootable USB drive for Windows 10. The Windows 10 ISO can be downloaded from the [Microsoft website](https://www.microsoft.com/en-us/software-download/windows10ISO). Since the image uses around 6GB disk space, I got a 16GB USB drive to make sure I have enough disk space for the ISO image. It is important that this USB drive contains no useful data since the filesystem will be completely overwritten by the Windows ISO image.

As shown in the following command output, my USB drive is detected as the new device `/dev/sdb`:
```bash
$ lsblk
NAME           MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda              8:0    0 931.5G  0 disk
├─sda1           8:1    0   260M  0 part  /boot
├─sda2           8:2    0    16M  0 part
├─sda3           8:3    0 441.5G  0 part
├─sda4           8:4    0   544M  0 part
├─sda5           8:5    0 488.3G  0 part
│ └─LVM        254:0    0 488.3G  0 crypt
│   ├─mvg-swap 254:1    0    20G  0 lvm   [SWAP]
│   ├─mvg-root 254:2    0    40G  0 lvm   /
│   └─mvg-home 254:3    0 428.3G  0 lvm   /home
└─sda6           8:6    0  1000M  0 part
sdb              8:16   1  14.8G  0 disk
└─sdb1           8:17   1  14.8G  0 part
```

To override all data within the USB drive with the Windows 10 ISO, I installed `woeusb` from AUR. `woeusb` is a tool to create a bootable Windows USB drive since [standard ways of copying ISO to a USB drive](https://wiki.archlinux.org/index.php/USB_flash_installation_medium#BIOS_and_UEFI_bootable_USB) do not work with Windows images.

```bash
$ sudo woeusb --target-filesystem NTFS --device Win10_20H2_v2_French_x64.iso /dev/sdb
```

Once the command has finished its execution, I rebooted from the newly created Windows installation drive (by changing the UEFI boot order in BIOS) and landed in the Windows installation screen.

I then found a [thread](https://answers.microsoft.com/en-us/windows/forum/all/accidentally-deleted-windows-10-boot-partition-how/181745f9-3303-4968-9851-5c213db9c89c) where Mark S. described how to create a lost EFI partition from an installation media. Since I already had an existing EFI system partition, I skipped the steps used to create and format the partition and only run the following steps:
- Press `SHIFT+F10` on the Windows installation screen to open a command-line interpreter.
- Run `diskpart` to launch the disk management tool DiskPart.
- Run `list disk` in DiskPart to get a list of existing disks.
- Run `select disk N` where `N` refers to the number of the disk which contains our EFI system partition (for me it was Disk 0 so I ran `select disk 0`).
![](/assets/bootloader-01.jpg)
- Run `list volume` to get the list of the partitions in this disk.
![](/assets/bootloader-02.jpg)
- Find the letter corresponding to the volume with the existing Windows filesystem (in my case it is `C` since `Volume 0` is listed as my Windows filesystem).
- Exit DiskPart by running `exit`.
- Run `bcdboot X:\Windows` where `X` is the letter found in the previous step. This command should copy the boot files from the Windows partition to our existing EFI system partition so that an entry will be created by `systemd-boot`.
![](/assets/bootloader-03.jpg)

I then restarted and went back to the BIOS to change the default boot device back to `Linux Boot Loader`.

![](/assets/bootloader-04.jpg)

An new entry called `Windows Boot Manager` has been created right below my `Arch Linux` entry and I now have an operational dual boot ! :D
