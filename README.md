# Arch-root-on-ZFS-install
## Introduction
You should follow the guide on the [wiki](https://wiki.archlinux.org/title/Install_Arch_Linux_on_ZFS) alongside this guide as it will be certainly more up to date. The purpose of this guid is to provide a more straight forward installation guide, as the one in the arch wiki is sometimes a bit vague as it's designed to be useful for any installation.

### Goal
The goal is to install ArchLinux with its root on a [ZFS](https://wiki.archlinux.org/title/ZFS) filesystem, using [rEFInd](https://wiki.archlinux.org/title/REFInd) as the boot manager to support easy dual-booting from separate HardDrives. Please keep in mind that this guide assumes a system using UEFI.

## Installation
### Partitioning
The Partition layout will be as follows:
| Partition | Filesystem | Size          | Partition type | Mount point |
| --------- | ---------- | ------------- | ------------- | ----------- |
| DISKp1   | FAT32      | 512MiB - 1GiB | ef00          | /boot       |
| DISKp2   | ZFS        | REST          | bf00          | /           |

First, boot from the PC you are going to install ArchLinux to, and boot from the USB Stick. I would recommend starting an ssh server, to do that type `systemctl start sshd.service` then set a root password with `passwd`. To connect to the server type `ssh root@IP` where IP is the IP of your ssh server, to find it, type `ip addr`. \
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
now create the second partition for the system \
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
< remove if you don't want encryption
<
<
replace with your root partition id
```

</td>
</tr>
</table>

Note that in this case the root pool is called zroot, same as in the arch wiki. But one may change it to something like rpool, like its typically called in solaris systems. \
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
After being imported, the pool still needs to be mounted. In case you encrypted your pool, it first needs to be unlocked with `zfs load-key zroot`, which will prompt you for a password assuming you chose prompt as the key location for your root pool. Now make sure to first only mount the `ROOT/default` dataset with `zfs mount zroot/ROOT/default` as the order is important, then the remaining datasets with `zfs mount -a`.

### Finishing touches before actual install
create the mount mountpoint at /mnt/boot with `mkdir /mnt/boot`. Its worth mentioning that nowadays its more common to mount uefi systems at `/efi` but in our case `/boot` works better with rEFInd. \
\
mount the efi partition at `/mnt/boot` with `mount /dev/DISKp1 /mnt/boot` and set the root to boot from for `zroot` with `zpool set bootfs=zroot/ROOT/default zroot`. \
\
finally "install" the system with `pacstrap /mnt PACKAGES` where `PACKAGES` is a space separated list of packages you need in the new system, it should only contain necessary packages, the rest should be installed when chrooted. An example would be `base base-devel neovim openssh opendoas` and in some cases a networkmanager like NetworkManager or iwctl if you want to use wifi with systemd-networkmanager. You could also copy the ArchISO `pacman.conf` to the new system with `cp /etc/pacman.conf /mnt/etc/`

### configuring the system for zfs
generate a fstab with `genfstab -U -p /mnt >> /mnt/etc/fstab` and edit it to exclude the all zfs datasets (should only contain efi partition) by commenting them out, the lines to comment out are most likely:
```
zroot/ROOT/default
zroot/data/home
zroot/data/home/root
```
 
chroot into the new system with `arch-chroot /mnt`.

now follow the [instructions](https://github.com/archzfs/archzfs/wiki) to add the arch-zfs repo to pacman. Do it by adding ```[archzfs]
Server = https://archzfs.com/$repo/$arch``` to the `pacman.conf`. Then if the versions of archzfs and linux match just type `pacman -S linux archzfs-linux`. *Solution needed:* If the versions don't match install the right version. \
\
generate an appropriate host id with: `zgenhostid $(hostid)` and create a zfs cache file with `zpool set cachefile=/etc/zfs/zpool.cache zroot`. Your system won't boot if you forget this step. \
\
modify the kernel hooks in `/etc/mkinitcpio.conf` to `HOOKS=(base udev autodetect modconf block keyboard zfs filesystems)`, that means adding `zfs` in front of `filesystems` and moving `keyboard` in front of both. You may optionally also remove fsck, as it's not needed when the root is on a zfs filesystem. Then generate the new hooks with `mkinitcpio -p KERNEL` where kernel is the name of your kernel, in most cases `linux`. \
\
Also enable these services `zfs.target`, `zfs-import-cache`, `zfs-mount`, `zfs-import.target` with these commands
```
systemctl enable zfs.target
systemctl enable zfs-import-cache
systemctl enable zfs-mount
systemctl enable zfs-import.target
```
\
install rEFInd with `pacman -S refind` and let it install its files to /boot with `refind-install`, then add `"Standard boot options"     "rw zfs=bootfs"` to `/boot/refind_linux.conf` for further install options consider [the arch wiki](https://wiki.archlinux.org/title/REFInd). \
\
lastly set your root password with `passwd`. And don't forget to export and unmount the pools and partitions with `umount /mnt/boot`, `zfs umount -a`, `zpool export zroot`. If that fails consider using the `-f`option to force the unmount and export and if that also failse try using the legacy unmount option with the force flag like `umount -l /mnt` \
\
Your system won't boot if you forget this!!! \
\
the system is now ready to use and bootable but it should still be configured as shown in the [vanilla arch install guide](https://wiki.archlinux.org/title/installation_guide#Time_zone) to be properly usable.
