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

>$ cryptsetup luksFormat /dev/nvme2n1p2

3. I initialize it as a virtual device
>$ cryptsetup open /dev/nvme2n1p2 rootpart

4. I format the LUKS virtual partition as BTRFS then mount it.
>$ mkfs.btrfs /dev/mapper/rootpart

5. I mount the BTRFS partition and create my top-level subvolume.
>$ mount /dev/mapper/rootpart /mnt
>$ btrfs subvolume create /mnt/archtop
>$ umount /dev/mapper/rootpart /mnt

6. I re-mount into my top level subvolume, and create the root subvolume.

@ = /

>$ mount -o relatime,space_cache=v2,ssd,compress=zstd:1,subvol=archtop /dev/mapper/rootpart /mnt
>$ btrfs subvolume create /mnt
>$ umount /mnt

7. I re-mount into my root subvolume @, and create the rest of the directories and subvolumes I think I'll need.

>$ mount -o relatime,space_cache=v2,ssd,compress=zstd:1,subvol=archtop/@ /dev/mapper/rootpart /mnt

>$ btrfs subvolume create /mnt/.snapshots
>$ btrfs subvolume create /mnt/home
>$ btrfs subvolume create /mnt/opt
>$ btrfs subvolume create /mnt/root
>$ btrfs subvolume create /mnt/srv
>$ btrfs subvolume create /mnt/tmp
>$ btrfs subvolume create /mnt/usr
>$ btrfs subvolume create /mnt/var
>$ btrfs subvolume create /mnt/var/cache
>$ btrfs subvolume create /mnt/var/log
>$ btrfs subvolume create /mnt/var/tmp

8. Format and mount the EFI partition:
mkfs.fat -F32 /dev/nvme2n1p1
mkdir /mnt/boot
mount /dev/nvme2n1p1 /mnt/boot


That's the initial disk management out of the way. Now we're ready to install Linux into the root volume.

9. Install Base System
>$ pacstrap -K /mnt base base-devel linux linux-firmware networkmanager cryptsetup lvm2 neovim vim grub efibootmgr wget curl man-db man-pages openssh tree btrfs-progs


10. Generate fstab
>$ genfstab -U /mnt >$>$ /mnt/etc/fstab

11. Chroot into installed system
>$ arch-chroot /mnt

12. Timezone and Locale Stuff
>$ ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
>$ hwclock --systohc
>$ vim /etc/locale.gen
>$ locale-gen
>$ vim /etc/locale.conf


13. Set hosname
>$ vim /etc/hostname

14. Add "encrypt" flag to mkinitcpio.conf
>$ pacman -S btrfs-progs intel-ucode amd-ucode cryptsetup
>$ vim /etc/mkinitcpio.conf
>$ pacman -S --noconfirm btrfs-progs intel-ucode amd-ucode cryptsetup:
```bash
MODULES=(btrfs)
HOOKS=(base udev autodetect modconf keyboard keymap consolefont block encrypt filesystems fsck) # Added "encrypt" between "block" and "filesystems"
```

>$ mkinitcpio -P


15. Install and configure GRUB

>$ pacman -S --noconfirm grub efibootmgr
>$ grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

16. Get Drive UUID
>$ blkid -s UUID -o value /dev/nvme2n1p2

17. Make decrypt on boot work properly
>$ vim /etc/default/grub
```bash
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=YOUR_UUID:rootpart rootflags=subvol=archtop/@"
GRUB_PRELOAD_MODULES="part_gpt part_msdos luks"
GRUB_ENABLE_CRYPTODISK=y
```
>$ grub-mkconfig -o /boot/grub/grub.cfg


18. VERIFY that grub worked properly. 
There should be a new boot entry.
>$ efibootmgr

Check which kernels are in GRUB's list.
>$ grep 'menuentry' /boot/grub/grub.cfg | sed -e "s/menuentry '\([^']*\)'.*/\1/"


19. Set new Root Password
>$ passwd

20. Add new wheel user
>$ useradd -m -G wheel -s /bin/bash Pyrus
>$ passwd Pyrus

21. Configure sudo access for that nerd
Install the sudo package
>$ pacman -S --noconfirm sudo

Uncomment the "%wheel ALL=(ALL) ALL" line in /etc/sudoers
>$ EDITOR=vim visudo
```bash
"%wheel ALL=(ALL) ALL"
```

22. Boot and login into the new system
>$ exit
>$ umount -R /mnt
>$ restart

# Configuration (from the moment we log in)

1. Error Checking

Verify that there's no failed services
>$ systemctl --failed

Check the overall error logs. Work through them a bit if needed.
>$ journalctl -p 3 -xb

2. Give color coding to Pacman for visibility
>$ sudo vim /etc/pacman.conf
```bash
# Misc options
Color
ILoveCandy
```

3. Enable the multilib repository
To enable multilib repository, uncomment the [multilib] section in /etc/pacman.conf:
```bash
[multilib]
Include = /etc/pacman.d/mirrorlist
```

4. Add whatever users you want to autologin

>$ sudo groupadd autologin

>$ sudo groupadd -a -G autologin [username]

>$ sudo vim /etc/sddm.conf
```bash
[Autologin]
User=pyrus
Session=hyprland
```

5. yay configuration for AUR packages
https://github.com/Jguer/yay

Need git and base-devel first.
>$ sudo pacman -Syu git base-devels

Download and build yay from repo
>$ mkdir yay-install; cd yay-install
>$ git clone https://aur.archlinux.org/yay.git
>$ cd yay
>$ makepkg -si

Configure
>$ yay -Y --gendb
>$ yay -Syu --devel
>$ yay -Y --devel --save

## Display and Desktop Environment
1. (IF using an nvidia GPU) Install Nvidia Drivers
https://youtu.be/AOjOd3wIPu8

>$ sudo pacman -S nvidia nvidia-utils nvidia-settings

Optional support for 32-bit applications
>$ sudo pacman -Sy lib32-nvidia-utils

2. Nvidia Driver Configuration

>$ vim /etc/mkinitcpio.conf
Remove ``kms`` from the HOOKS array in /etc/mkinitcpio.conf

Add ``nvidia``, ``nvidia_modeset``, ``nvidia_uvm`` and ``nvidia_drm`` to the initramfs.

Regenerate the initramfs.
>$ mkinitcpio -P

Make NVIDIA's drivers aware of the change.

>$ vim /etc/modprobe.d/nvidia.conf
```bash
options nvidia-drm modeset=1
```

3. Create a pacman hook to update the NVIDIA Module whenever there's a driver update

```bash
/etc/pacman.d/hooks/nvidia.hook
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia
Target=linux
# Change the linux part above and in the Exec line if a different kernel is used

[Action]
Description=Update NVIDIA module in initcpio
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'
```

4. Hyprland's Recommended NVIDIA Configs
https://wiki.hyprland.org/Nvidia/

Make a global config like the following:
>$ sudo vim /etc/profile.d/nvidia_hyprland.sh
```bash
export LIBVA_DRIVER_NAME=nvidia
export XDG_SESSION_TYPE=wayland
export GBM_BACKEND=nvidia-drm
export __GLX_VENDOR_LIBRARY_NAME=nvidia
export WLR_NO_HARDWARE_CURSORS=1
```
>$ sudo chmod +x /etc/profile.d/nvidia_hyprland.sh
>$ source /etc/profile

Install additional packages
Pacman: qt5-wayland, qt5ct, libva
Yay: nvidia-vaapi-driver-git (AUR)
>$ sudo pacman -Syu qt5-wayland qt5ct libva
>$ yay -S nvidia-vaapi-driver

>$ reboot

TODO: BUILD FROM SOURCE AND FIX THE FLICKERING

5. Hyprland Required Software
https://wiki.hyprland.org/Useful-Utilities/Must-have/

Kitty is the default terminal emulator.
>$ sudo pacman -Syu kitty


## Audio
1. Install ``alsa-utils``.
>$ sudo pacman -S alsa-utils

# TODO

Desktop environments

BTRFS Snapshotting

Managing flatpaks

Configure PipeWire

Virtual Machines
> LookingGlass


Contingency if hyprland don't work with the closed-source NVIDIA Drivers:
https://archlinux.org/packages/extra/x86_64/nvidia-open/
https://github.com/NVIDIA/open-gpu-kernel-modules