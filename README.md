# Arch-root-on-ZFS-intall
## Introduction
You should follow the guide on the [wiki](https://wiki.archlinux.org/title/Install_Arch_Linux_on_ZFS) alongside this guide as it will be certantly more uptodate. The purpose of this guid is to provide a more straight forward installation guide, as the arch wiki's guide is sometimes vague as its designed to be usefull for any installation.

### Goal
The goal is to install ArchLinux with its root on a [ZFS](https://wiki.archlinux.org/title/ZFS) filesystem, using [rEFInd](https://wiki.archlinux.org/title/REFInd) as the boot manager to support easy dualbooting from seperate HardDrives. Please keep in mind that this guide assumes a system using UEFI.

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
\
\>`o` => to start new gpt partition scheme \
\>`y` => to confirm \
\>`n` => to create a new partition (this will be our /boot partition) \
\>`​` or `1` => to make it the first partition (`​` means nothing e.g. just press ENTER) \
\>`​` => to start at the first usable sector \
\>`+xGiB` or `+xMiB` => to make the partition x GiB/MiB \
\>`ef00` => to set the partition type to EFI system partition \
\
now create the second partiton for the system \
\
\>`n` => to create a new partition (this will be our / partition) \
\>`​` or `2` => to make it the second partition (`​` means nothing e.g. just press ENTER) \
\>`​` => to make it start at the first usable sector
\>`​` => to make it end at the last usable sector
\>`bf00` => to set the partition type to Solaris root \
\
then type `p` to check the partition layout and then type `w` and `y` to confirm. \
\
and format the /boot partition to FAT32 with `mkfs.vfat -F 32 /dev/DISKp1`, \
for zfs we need to reference the disk by its id, to get it type `ls -al /dev/disk/by-id` and remember the id of your root partition e.g. `DISK.xxxxxxxxxxxxxxxx-part2` (can have a different format depending on your disk) \
\
now create the pool for your root with: \
<table>
<tr>
<td>
       
```
zpool create -f -o ashift=X         \
       -O acltype=posixacl       \
       -O relatime=on            \
       -O xattr=sa               \
       -O dnodesize=legacy       \
       -O normalization=formD    \
       -O mountpoint=none        \
       -O canmount=off           \
       -O devices=off            \
       -R /mnt                   \
       -O encryption=aes-256-gcm \
       -O keyformat=passphrase   \
       -O keylocation=prompt     \
       zroot /dev/disk/by-id/DISK.xxxxxxxxxxxxxxxx-part2
```

</td>
<td>

```
X = 9 for 512 bit or 12 for 4096 bit sector size, 
       find the sector size with "lsblk -o NAME,PHY-SEC"
​
​
​
​
​
​
​
​
remove if you don't want encryption
remove if you don't want encryption
remove if you don't want encryption
replace with your root partition id
```

</td>
</tr>
</table>

Note that in this case the root pool is called zroot, same as in the arch wiki. But one may change it to somthing like rpool, like its typicaly called in solaris systems. \
\
create the root and home datasets for `zroot` with

```
zfs create -o mountpoint=none zroot/data; \
zfs create -o mountpoint=none zroot/ROOT; \
zfs create -o mountpoint=/ -o canmount=noauto zroot/ROOT/default; \
zfs create -o mountpoint=/home zroot/data/home; \
zfs create -o mountpoint=/root zroot/data/home/root;
```

now check if the any datasets are mounted with `zfs get mounted` and if so unmount them with `zfs umount -a` and optionally delete the created folders with `rm -rf /mnt/*`, then check if everything worked so far with `zfs list`. The output should look similar to this:

```
NAME                   USED  AVAIL     REFER  MOUNTPOINT
zroot                  xxxK   xxxG       xxK  none
zroot/ROOT             xxxK   xxxG       xxK  none
zroot/ROOT/default     xxxK   xxxG       xxK  /mnt
zroot/data             xxxK   xxxG       xxK  none
zroot/data/home        xxxK   xxxG       xxK  /mnt/home
zroot/data/home/root   xxxK   xxxG       xxK  /mnt/root
```

### Mounting (useful to remember, fixing broken systems)
To confirm everything so far worked export and reimport the pool (in this case `zroot`). To do this type `zpool export zroot` to export the pool (to be able to export a pool make sure all datasets are unmounted, which we already made sure) and reimport it with `zpool import -d /dev/disk/by-id -R /mnt zroot -N
`. \
\
After beeing imported, the pool still needs to be mounted. In case you encrypted your pool, it first needs to be unlocked with `zfs load-key zroot`, which will prompt you for a password assuming you chose promt as the keylocation for your root pool. Now make sure to first only mount the `ROOT/default` dataset with `zfs mount zroot/ROOT/default` as the order is important, then the remaining datasets with `zfs mount -a`.

### Finishing touches before actual install
create the mount mountpoint at /mnt/boot with `mkdir /mnt/boot`. Its worth mentioning that nowdays its more common to mount uefi systems at `/efi` but in our case `/boot` works better with rEFInd. \
\
mount the efi partiton at `/mnt/boot` with `mount /dev/DISKp1 /mnt/boot` and set the root to boot from for `zroot` with `zpool set bootfs=zroot/ROOT/default zroot`. \
\
finally "install" the system with `pacstrap /mnt PACKAGES` where `PACKAGES` is a space separated list of packages you need in the new system, it should only contain nessesary packages, the rest should be installed when chrooted. An example would be `base base-devel neovim openssh opendoas` and in some cases a networkmanager like NetworkManager or iwctl if you want to use wifi with systemd-networkmanager.

### configuring the system for zfs
generate a fstab with `genfstab -U -p /mnt >> /mnt/etc/fstab` and edit it to exclude the all zfs datasets (should only contain efi partition) by commenting them out, the lines to comment out are most likely:
```
>comment out zroot/ROOT/default
 >comment out zroot/data/home
 >comment out zroot/data/home/root
 ```
 
 chroot into the new system with `arch-chroot /mnt`.
