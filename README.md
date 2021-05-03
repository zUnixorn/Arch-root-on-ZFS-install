# Arch-root-on-ZFS-intall
## Introduction
You should follow the guide on the [wiki](https://wiki.archlinux.org/title/Install_Arch_Linux_on_ZFS) alongside this guide as it will be certantly more uptodate. The purpose of this guid is to provide a more straight forward installation guide, as the arch wiki's guide is sometimes vague as its designed to be usefull for any installation.

### Goal
The goal is to install ArchLinux with its root on a [ZFS](https://wiki.archlinux.org/title/ZFS) filesystem, using [rEFInd](https://wiki.archlinux.org/title/REFInd) as the boot manager to support easy dualbooting from seperate HardDrives. Note that we are installing on an UEFI system.

## Prequisites
* USB Drive with an ArchISO that has ZFS installed (can be downloaded unofficialy or made using [ArchISO](https://wiki.archlinux.org/title/Archiso))
* A HardDrive, in this guide refered to as `DRIVE`

## Installation
The Partition layout in this guide will be as follows:
| Partition | Filesystem | Size          | Partitiontype |
| --------- | ---------- | ------------- | ------------- |
| DRIVEp1   | FAT32      | 512MiB - 1GiB | ef00          |
| DRIVEp2   | ZFS        | REST          | bf00          |
