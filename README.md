# Arch-root-on-ZFS-intall
## Introduction
You should follow the guide on the [wiki](https://wiki.archlinux.org/title/Install_Arch_Linux_on_ZFS) alongside this guide as it will be certantly more uptodate. The purpose of this guid is to provide a more straight forward installation guide, as the arch wiki's guide is sometimes vague as its designed to be usefull for any installation.

### Goal
The goal is to install ArchLinux with its root on a [ZFS](https://wiki.archlinux.org/title/ZFS) filesystem, using [rEFInd](https://wiki.archlinux.org/title/REFInd) as the boot manager to support easy dualbooting from seperate HardDrives. Note that we are installing on an UEFI system.

## Prequisites
* USB Stick with an ArchISO that has ZFS installed (can be downloaded unofficialy or made using [ArchISO](https://wiki.archlinux.org/title/Archiso))
* A HardDrive

## Installation
### Partitioning
The Partition layout in this guide will be as follows:
| Partition | Filesystem | Size          | Partitiontype | Mount point |
| --------- | ---------- | ------------- | ------------- | ----------- |
| DISKp1   | FAT32      | 512MiB - 1GiB | ef00          | /boot       |
| DISKp2   | ZFS        | REST          | bf00          | /           |

First, boot from the PC you are going to install ArchLinux to, and boot from the USB Stick. I would recommend to start an ssh server, to do that type `systemctl start sshd.service` then set a root password with `passwd`. To connect to the server type `ssh root@IP` where IP is the IP of your ssh server, to find it, type `ip addr`. \
\
To start Partitioning type `lsblk` to find the device you want to partition. It will be referenced with `DISK` from now on. \

start partitioning with `gdisk /dev/DISK` (or any other tool) \
in gdisk type: \
\>`o` => to start new gpt partition scheme \
\>`y` => to confirm \
\>`n` => to create a new partition (this will be our /boot partition) \
\>`​` or `1` => to make it the first partition (`​` means nothing e.g. just press ENTER) \
\>`​` => to start at the first usable sector \
\>`+xGiB` or `+xMiB` => to make the partition x GiB/MiB \
\>`ef00` => to set the partition type to EFI system partition \
\
now create the second partiton for the system \
\>`n` => to create a new partition (this will be our / partition) \
\>`​` or `2` => to make it the second partition (`​` means nothing e.g. just press ENTER) \
\>`​` => to make it start at the first usable sector
\>`​` => to make it end at the last usable sector
\>`bf00` => to set the partition type to Solaris root \
\
then type `p` to check the partition layout and then type `w` and `y` to confirm. \
\
now format the /boot partition to FAT32 with `mkfs.vfat -F 32 /dev/DISKp1` \
for zfs we need to reference the disk by its id, to get it type `ls -al /dev/disk/by-partuuid` and remember the partuuid of your root partition e.g. `DISKp2` \

