These are the notes on my complex Arch Linux install

# General Links
https://wiki.archlinux.org/title/installation_guide
https://wiki.archlinux.org/title/Arch_boot_process#Boot_loader
https://wiki.archlinux.org/title/Data-at-rest_encryption
https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface
https://wiki.archlinux.org/title/General_recommendations
https://wiki.archlinux.org/title/List_of_applications
https://wiki.archlinux.org/title/Snapper
https://archive.kernel.org/oldwiki/btrfs.wiki.kernel.org/index.php/SysadminGuide.html#Layout

# Disk Management
https://github.com/Deebble/arch-btrfs-install-guide

## Misc Links
https://wiki.archlinux.org/title/dm-crypt/Encrypting_an_entire_system#Btrfs_subvolumes_with_swap
https://gist.github.com/MaxXor/ba1665f47d56c24018a943bb114640d7

## Device Partitioning

This install is on my NVME SSD.

I create a 1 GB EFI partition for the UEFI boot loader, and one primary partition for the entire rest of the device.

```
fdisk /dev/nvme2n1

n
1
[enter]
+1G

n
2
[enter]
[enter]
```

## LUKS

### Initialization

I create a block-level encryption scheme on the second partition of my SSD. The EFI partition cannot be encrypted.

```cryptsetup luksFormat /dev/nvme2n1p2```

I initialize it as a virtual device
```cryptsetup open /dev/nvme2n1p2 rootpart```

PS: I can verify how fast (or slow) the encryption is going to be with the following two commands
``cryptsetup --help`` --> see the default settings at the bottom (currently aes-xts with 256bit key)
``crypsetup benchmark`` --> get a speed test of each standard's encrypt/decrypt speeds. Compare these values to the top speed of your SSD in MiB/s and deem if they are acceptable. If the value here is GREATER than the read/write of your SSD, then LUKS will have pretty much no overhead aside from CPU usage.

### Persistence and GRUB Decrypt
TODO:
???



## Formatting

I format the efi boot partition as Fat32.
```mkfs.fat -F 32 /dev/nvme2n1p1```

I format the LUKS virtual partition as BTRFS.
```mkfs.btrfs /dev/mapper/rootpart```

# BTRFS
## BTRFS Essential commands

See all current subvolumes
```btrfs subvolume list -p /```

Create a new subvolume
```btrfs subvolume create [path]```

Get a summary of a mounted directory's usage (including snapshots and COW files)
```btrfs fi du / -s```

See mounting file (not a btrfs command)
```cat /etc/fstab```



## BTRFS Mounting and Subvolumes
https://wiki.archlinux.org/title/Btrfs#Creating_a_subvolume
https://christitus.com/btrfs-guide/
https://wiki.archlinux.org/title/Snapper#Suggested_filesystem_layout

First, we need to mount the device to actually use it.
```mount /dev/mapper/rootpart /mnt```


### BTRFS Subvolumes (Nested Architecture)

```btrfs subvolume create archtop```

```umount /mnt```
```mount -o relatime,space_cache=v2,ssd,subvol=archtop /dev/mapper/rootpart /mnt```


```btrfs subvolume create /mnt/@```

[TODO: fill in the rest of this section]


### BTRFS Subvolumes (Semi-Nested Architecture)

[[example_btrfs_schema.png]]

Now, we can create each of our subvolumes, such as home, root, etc.

First, we create a top-level subvolume as a container for the whole installation
```btrfs subvolume create archtop```

Then we can mount that container, and create the rest of our subvolumes inside of it.
```umount /mnt```
```mount -o relatime,space_cache=v2,ssd,compress=zstd:1,subvol=archtop /dev/mapper/rootpart /mnt```

Root Subvolume ``/``
```btrfs subvolume create /mnt/@```

Home ``/home``
```btrfs subvolume create /mnt/@home```

Snapshots ``/.snapshots``
```btrfs subvolume create /mnt/@snapshots```

Logs ``/var/log``
```btrfs subvolume create /mnt/@var_log```

Pacman Cache ``/var/cache/pacman/pkg``
```btrfs subvolume create /mnt/@var_cache_pacman_pkg``

Tmp ``/tmp``
```btrfs subvolume create /mnt/@tmp```

That's all the BTRFS volumes for now.

### Filesystem Mounting (Semi-Nested Architecture)

Mounting the root ``/`` subvolume:
```mount -o relatime,space_cache=v2,ssd,subvol=@ /dev/mapper/rootpart /mnt```

At this point, we need to create the directories we're mounting to.
```mkdir -p /mnt/{boot/efi,home,var/log,btrfs,tmp}```

Mounting the @home ``/home`` subvolume:
```mount -o relatime,space_cache=v2,ssd,subvol=@home /dev/mapper/rootpart /mnt/home```

Mounting the @tmp ``/tmp`` subvolume:
```mount -o relatime,space_cache=v2,ssd,subvol=@tmp /dev/mapper/rootpart /mnt/tmp```


Mounting the @var_log ``/var/log`` subvolume:
```mount -o relatime,space_cache=v2,ssd,subvol=@var_log /dev/mapper/rootpart /mnt/var/log```

Mounting the @snapshots ``/.snapshots`` subvolume:
```mount -o relatime,space_cache=v2,ssd,subvol=@snapshots /dev/mapper/rootpart /mnt/.snapshots```

### Mount ESP
```mount /dev/nvme2n1p1 /mnt/boot/efi```

## BTRFS Compression
https://btrfs.readthedocs.io/en/latest/Compression.html
https://www.reddit.com/r/btrfs/comments/oom46x/i_would_like_to_set_up_an_encrypted_external/
https://www.reddit.com/r/btrfs/comments/qklux7/learning_btrfs_basics_subvolume_compression/

You mount a compress-on-the-fly subvolume with the ``compress-force=zstd`` flag.
eg:
```mount -t btrfs -o defaults,noatime,compress-force=zstd:2 /dev/mapper/data1 /mnt/data```

## BTRFS Swap File

https://wiki.archlinux.org/title/Btrfs#Swap_file
https://btrfs.readthedocs.io/en/latest/Swapfile.html

A swap file should be 4GB.

[TODO: SET THIS UP ONCE THE SYSTEM IS UP AND RUNNING]


# Initial Install (from USB to Booted)

I am setting up an Arch Linux install.

My 2Tb NVMe SSD is labeled ``/dev/nvme2n1``.
The Fat32 EFI boot partition, is empty and not set up yet. It is ``/dev/nvme2n1p1``.
The LUKS device is ``/dev/nvme2n1p2``.

Here are the things I've done so far since starting:

1. I create a 1 GB EFI partition for the UEFI boot loader, and one primary partition for the entire rest of the device.

```
fdisk /dev/nvme2n1

n
1
[enter]
+1G

n
2
[enter]
[enter]
```

2. I create a block-level encryption scheme on the second partition of my SSD. The EFI partition cannot be encrypted.

```cryptsetup luksFormat /dev/nvme2n1p2```

3. I initialize it as a virtual device
```cryptsetup open /dev/nvme2n1p2 rootpart```

4. I format the LUKS virtual partition as BTRFS then mount it.
```mkfs.btrfs /dev/mapper/rootpart```

5. I mount the BTRFS partition and create my top-level subvolume.
```mount /dev/mapper/rootpart /mnt```
```btrfs subvolume create /mnt/archtop```
```umount /dev/mapper/rootpart /mnt```

6. I re-mount into my top level subvolume, and create the root subvolume.
@ = /
```mount -o relatime,space_cache=v2,ssd,compress=zstd:1,subvol=archtop /dev/mapper/rootpart /mnt```
```btrfs subvolume create /mnt/@```
```umount /mnt```

7. I re-mount into my root subvolume @, and create the rest of the directories and subvolumes I think I'll need.
```mount -o relatime,space_cache=v2,ssd,compress=zstd:1,subvol=archtop/@ /dev/mapper/rootpart /mnt```
```btrfs subvolume create /mnt/.snapshots```
```btrfs subvolume create /mnt/home```
```btrfs subvolume create /mnt/opt```
```btrfs subvolume create /mnt/root```
```btrfs subvolume create /mnt/srv```
```btrfs subvolume create /mnt/tmp```
```btrfs subvolume create /mnt/usr```
```btrfs subvolume create /mnt/var```
```btrfs subvolume create /mnt/var/cache```
```btrfs subvolume create /mnt/var/log```
```btrfs subvolume create /mnt/var/tmp```

8. Format and mount the EFI partition:
```bash
mkfs.fat -F32 /dev/nvme2n1p1
mkdir /mnt/boot
mount /dev/nvme2n1p1 /mnt/boot
```

9. Install Base System
> pacstrap -K /mnt base base-devel linux linux-firmware networkmanager cryptsetup lvm2 neovim vim grub efibootmgr wget curl man-db man-pages openssh tree btrfs-progs


10. Generate fstab
> genfstab -U /mnt >> /mnt/etc/fstab

11. Chroot into installed system
> arch-chroot /mnt

12. Timezone and Locale Stuff
> ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
> hwclock --systohc
> vim /etc/locale.gen
> locale-gen
> vim /etc/locale.conf


13. Set hosname
> vim /etc/hostname

14. Add "encrypt" flag to mkinitcpio.conf
> pacman -S btrfs-progs intel-ucode amd-ucode cryptsetup
> vim /etc/mkinitcpio.conf
> pacman -S --noconfirm btrfs-progs intel-ucode amd-ucode cryptsetup:
``MODULES=(btrfs)``
``HOOKS=(base udev autodetect modconf keyboard keymap consolefont block **encrypt** filesystems fsck)``

> mkinitcpio -P


15. Install and configure GRUB

> pacman -S --noconfirm grub efibootmgr
> grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

16. Get Drive UUID
> blkid -s UUID -o value /dev/nvme2n1p2

17. Make decrypt on boot work properly
> vim /etc/default/grub
``GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=YOUR_UUID:rootpart rootflags=subvol=archtop/@"``
``GRUB_PRELOAD_MODULES="part_gpt part_msdos luks"``
``GRUB_ENABLE_CRYPTODISK=y``

18. VERIFY that grub worked properly. There should be a new boot entry.
> efibootmgr

19. Set new Root Password
> passwd

20. Add new wheel user
> useradd -m -G wheel -s /bin/bash Pyrus
> passwd Pyrus

21. Configure sudo access for that nerd
Install the sudo package
> pacman -S --noconfirm sudo

Uncomment the "%wheel ALL=(ALL) ALL" line in /etc/sudoers
> EDITOR=vim visudo
```"%wheel ALL=(ALL) ALL"```

22. Boot and login into the new system
> exit
> umount -R /mnt
> restart