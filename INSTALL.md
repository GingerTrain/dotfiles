# Steps for Installing Arch Linux on a UEFI-GPT System with systemd-boot

## Table of Contents
---
1. [Notes](#notes)
   - [General](#general)
   - [Partition Scheme](#partition-scheme)
2. [Boot](#boot)
   - [Arch Linux Image](#arch-linux-image)
   - [Mount](#mount)
   - [Boot](#boot)
3. [Pre-installation](#pre-installation)
   - [Boot Mode](#boot-mode)
   - [Internet Connection](#internet-connection)
   - [Partition](#partition)
   - [File System](#file-system)
4. [Installation](#installation)
5. [Configuration](#configuration)
   - [Fstab File](#fstab-file)
   - [Chroot](#chroot)
   - [Network](#network)
   - [Localization](#localization)
   - [System](#system)
   - [Bootloader](#bootloader)
   - [Swap File](#swap-file)
6. [Reboot](#reboot)
7. [Post-installation](#post-installation)
   - [Network](#network) `TODO: Jump to correct network section`
8. [References](#references)
   - [Special Thanks](special-thanks)

## Notes
---
`TODO: wayland > x11 wherever possible`

`TODO: Made for 1 hard drive raw and full install, but I plan on adding single and separate hard drive dualboot install ~ current Same HD Dual-boot notes are for single hard drive dualboot (I believe)`  
### General `TODO: Write`
This guide will take you from downloading the Arch Liunx Image to getting a basic desktop environment. There are references linked at the end of the guide that are very helpful if you get stuck.

'#' means the command is being entered by root.  
'$' means the command is being entered by a user.

The commands in this guide are mostly fine to copy and paste but some are specific to my system or personal preference; they are meant to be used as a reference not followed step-by-step. I recommend researching what each command does before you try to execute it.

All non-essential packages are for my system (AMD CPU and AMD GPU) and are my personal preference for the environment I want.

My localization configurations are for Ontario, Canada.

### Partition Scheme
Below is my preferred partition scheme, and the one I will be following in this guide; there is no swap partition because I prefer to use a swap file.

| Partition |   Size  |         Name         |
| --------- | ------- | -------------------  |
| /dev/sda1 | 512MiB  | EFI system partition |
| /dev/sda2 | 32GiB   | Linux x86-64 root(/) |
| /dev/sda3 |    *    | Linux /home          |

## Boot
---

### Arch Linux Image
Download the Arch Linux image from the following website.
```
https://www.archlinux.org/download/
```

**Optional:** Verify the image signature of the Arch Linux image file.

### Mount
Download a software to create a bootable USB flash drive and put the Arch Linux ISO file on the USB.
```
https://www.pendrivelinux.com/yumi-multiboot-usb-creator/
```

### Boot
Plug the USB into the system and reboot the system.  
**Note: The boot device order may need to be modified in BIOS.**

## Pre-installation
---

### Boot Mode
Verify that UEFI is enabled on the motherboard by checking if the following folder exists.
```
# ls /sys/firmware/efi/efivars
```

### Internet Connection
Make sure the desired network interface is listed and enabled.  
**Note: enpXsX/ethX is wired, wlpXsX/wlanX is wireless.**
```
# ip link
```

If the system is using a wired connection then plug in an Ethernet cable.

If the system is using a wireless connection then use a network manager to connect to a wireless network.
```
# wifi-menu
```

**Optional:** Configure the network connection to have a static IP address or a dynamic IP address.

Verify that an internet connection was established.
```
# ping -c 4 archlinux.org
```

### Partition
Verify that the disks are recognized by the system.
```
# lsblk
# cat /proc/partitions
```

Clear any previous partitions.
```
# gdisk /dev/sda
command: o
proceed: y
```
Create new partitions and write to a GPT table.  
**Same HD Dual-boot: Install Windows first and use the Windows EFI System partition instead of creating a new one.**  
**Note: `RET` means hit enter and accept the default.**
```
command: n
partition #: RET
first sector: RET
last sector: 512M
GUID: EF00

command: n
partition #: RET
first sector: RET
last sector: 32G
GUID: 8304

command: n
partition #: RET
first sector: RET
last sector: RET
GUID: 8302

command: w
proceed: y
```

Verify that the partitions were created correctly.
```
# gdisk -l /dev/sda
```

Format each partition with an appropriate file system.  
**Same HD Dual-boot: Don't format the Windows EFI System partition.**
```
# mkfs.fat -F32 /dev/sda1
# mkfs.ext4 /dev/sda2
# mkfs.ext4 /dev/sda3
```

### File System
Mount the file systems to their respective directories.  
**Same HD Dual-boot: Mount the EFI partition that Windows created to boot.**
```
# mount /dev/sda2 /mnt
# mkdir /mnt/boot
# mount /dev/sda1 /mnt/boot
# mkdir /mnt/home
# mount /dev/sda3 /mnt/home
```

## Installation
---

Select the 5 best HTTP mirror servers based on geographic region and move them to the top of the mirrorlist file.  
**Note: Check the [mirror status](https://www.archlinux.org/mirrors/status/) page to find the fastest mirrors.**
```
# nano /etc/pacman.d/mirrorlist
## Canada
Server = http://mirror.sergal.org/archlinux/$repo/os/$arch
## Canada
Server = http://mirror.csclub.uwaterloo.ca/archlinux/$repo/os/$arch
## Canada
Server = http://muug.ca/mirror/archlinux/$repo/os/$arch
## Canada
Server = http://mirror.scd31.com/arch/$repo/os/$arch
## Canada
Server = http://archlinux.mirror.colo-serv.net/$repo/os/$arch
## Canada
Server = http://archlinux.mirror.rafal.ca/$repo/os/$arch
## Canada
Server = http://mirror2.evolution-host.com/archlinux/$repo/os/$arch
## Canada
Server = http://mirror.its.dal.ca/archlinux/$repo/os/$arch
```

Install the essential packages (Linux kernel, firmware, etc).  
**Note: base-devel is a group of packages that are required for building things on linux (gcc, make, autoconf, etc.); it isn't required but it is very convenient.**
```
# pacstrap /mnt base base-devel linux linux-firmware
```

**Optional:** Install any other packages.  
**IMPORTANT: Research packages the system might require; network managers and some hardware drivers NEED to be installed before reboot.** `TODO: Reword.`
```
# pacstrap /mnt git nano gdisk iwd dhcpcd amd-ucode mesa libva-mesa-driver mesa-vdpau xf86-video-amdgpu vulkan-radeon man-db man-pages
```

**Honourable Mentions**
```
# pacstrap /mnt wpa_supplicant networkmanager openssh reflector intel-ucode 
```

## Configuration
---

### Fstab File
Generate an Fstab file to define how disk partitions should be mounted.
```
# genfstab -U -p /mnt >> /mnt/etc/fstab
```

### Chroot
Change root into the new system.
```
arch-chroot /mnt
```

### Network
**Note: Using systemd-networkd and iwd.**  

Wired network with DHCP.  
**Note: Replace [NAME] with the device name (wildcards are accepted e.g. enp0s3 or en\*).**
```
# echo -e "[Match]\nName=[NAME]\n\n[Network]\nDHCP=ipv4" > /etc/systemd/network/50-dhcp-wired.network
```
Wired network with static IP address.  
**Note: Replace [NAME] with the device name (wildcards are accepted e.g. enp0s3 or en\*), [ADDRESS] with the static IP address (e.g. 192.168.0.25), and [START] with the start address of that IP range (e.g. 192.168.0.0).**
```
# echo -e "[Match]\nName=[NAME]\n\n[Network]\nAddress=[ADDRESS]/24\nGateway=[START]\n#DNS=8.8.8.8" > /etc/systemd/network/50-static-wired.network
```
Wireless network with DHCP.  
**Note: Replace [NAME] with the device name (wildcards are accepted e.g. wlp0s3 or wl\*).**
```
# echo -e "[Match]\nName=[NAME]\n\n[Network]\nDHCP=ipv4" > /etc/systemd/network/50-dhcp-wireless.network
# systemctl enable iwd.service
```
Wireless network with static IP address.  
**Note: Replace [NAME] with the device name (wildcards are accepted e.g. wlp0s3 or wl\*), [ADDRESS] with the static IP address (e.g. 192.168.0.25), and [START] with the start address of that IP range (e.g. 192.168.0.0).**
```
# echo -e "[Match]\nName=[NAME]\n\n[Network]\nAddress=[ADDRESS]/24\nGateway=[START]\n#DNS=8.8.8.8" > /etc/systemd/network/50-static-wireless.network
# systemctl enable iwd.service
```

Start systemd-networkd services to enable networking on boot.
```
# systemctl enable systemd-networkd.service
# systemctl enable systemd-resolved.service
```

### Localization
Update the system clock and verify.
```
# timedatectl set-timezone Canada/Eastern
# timedatectl status
```

Set the time zone and generate a hardware clock.  
**Same HD Dual-boot: Add `--localtime` to the hwclock command so the clocks sync correctly between OS.**
```
# ln -sf /usr/share/zoneinfo/Canada/Eastern /etc/localtime
# hwclock --systohc
```

Generate other locales.
```
# sed -i 's/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
# locale-gen
# echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

### System
Create the hostname file and update the entries list in the hosts file.  
**Note: If the system has a static IP address then it should be used instead of 127.0.0.1**
```
# echo archy > /etc/hostname
# echo -e "127.0.0.1\tlocalhost\n::1\t\tlocalhost\n127.0.0.1\tarchy.localdomain archy" >> /etc/hosts
```

Create an initial ramdisk environment.
```
# mkinitcpio -P
```

Set the root password.
```
# passwd
```

Add a new user and grant them sudo access.
```
# useradd -m -g users -G wheel,systemd-journal -s /bin/bash etaylor
# passwd etaylor
# EDITOR=nano visudo
%wheel ALL=(ALL) ALL
```

### Bootloader
Install systemd-boot.
```
# bootctl install
```

Check that Linux exists in the bootloader and create loader configurations.
```
# cd /boot/
# ls -l
# sed -i 's/default/#default/' loader/loader.conf
# echo -e "default arch\ntimeout 4" >> loader/loader.conf
```
Add loaders.  
**IMPORTANT: Replace [SDA2] with the PARTUUID of the root partition (not a string).**
```
# blkid > loader/entries/arch.conf
# nano loader/entries/arch.conf
title Arch Linux
linux /vmlinuz-linux
initrd /amd-ucode.img
initrd /initramfs-linux.img
options root=PARTUUID=[SDA2] rw
```

### Swap File
Create a swap file on the root partition with the proper permissions.  
**Note: Replace [SIZE] with the size of the swap file. To use suspend-to-disk (hibernate) make the swap file the same size as the system RAM or larger; otherwise it should be adjusted based on the system workload.**
```
# fallocate -l [SIZE] /swap
# chmod 600 /swap
# mkswap /swap
# swapon /swap
```

Create an fstab entry so the swap file is loaded on boot.
```
# echo -e "# /swap\n/swap\tnone\tswap\tdefaults\t0\t0" >> /etc/fstab
```

**Optional:** Adjust sysctl accordingly if the swap file was created on an SSD.
```
# echo "vm.swappiness=1" >> /etc/sysctl.d/99-sysctl.conf
```

Verify the swap file is working.
```
# free -m
```

**Note: Removing a swap file and the relevant fstab entry.**
```
# swapoff /swap
# rm -f /swap
# sed -i '/swap\t/d' /etc/fstab
```

## Reboot
---

Exit chroot and reboot the system after unmounting all the partitions.  
**IMPORTANT: Remove the installation media when the system is off or it will boot into the live environment.**
```
# exit
# umount -R /mnt/{boot,home,swap,} && reboot
```

## Post-installation
---

Ensure the system is up-to-date.
```
$ sudo pacman -Syu
```

### Network
Create a symbolic link from the systemd-resolved to the system version for DNS resolution.
```
$ sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

Set systemd-resolved as the default DNS manager.
```
# echo -e "[Network]\nNameResolvingService=systemd" > /etc/iwd/main.conf
```

**Optional: Connect to a wireless network.**  
**Note: Replace [DEVICE] with the wireless device name and [SSID] with the network.**
```
$ iwctl
[iwd]$ device list
[iwd]$ station [DEVICE] scan
[iwd]$ station [DEVICE] get-networks
[iwd]$ station [DEVICE] connect [SSID]
```

**Note: Disconnecting from a network.**
```
[iwd]# station device disconnect
```






`` TODO: Move to SETUP.md or something.``
### Replacement for pacman
`TODO: Install yay`

### Display Server
**Optional: xorg-apps**
```
pacman -S xorg-server xorg-xinit
cp /etc/X11/xinit/xinitrc ~/.xinitrc
```
`TODO: Wayland and sway`

### Console Improvements
Coloured outputs.
```
$ sudo sed -i 's/#Color/Color/' /etc/pacman.conf
$ alias diff='diff --color=auto'
$ alias grep='grep --color=auto'
$ alias ls='ls --color=auto'
```

### Emacs
Build Emacs master from source.
```
$ git clone -b master git://git.sv.gnu.org/emacs.git

$ ./autogen.sh
$ ./configure
--with-x-toolkit=lucid
--with-modules
--with-x
--with-file-notification=no
--with-sound=no
--without-compress-install
--without-dbus
--without-gif
--without-gpm
--without-imagemagick
--without-jpeg
--without-lcms2
--without-libgmp
--without-libotf
--without-libsystemd
--without-m17n-flt
--without-makeinfo
--without-png
--without-rsvg
--without-selinux
--without-tiff
--without-toolkit-scroll-bars
--without-gsettings
--without-gconf
--without-xaw3d
--without-xim
--without-xpm
--without-zlib
'CFLAGS= -O2 -g3'

$ make -j
$ sudo make install
```
Download .emacs.d. `TODO: Finish`
```
$ git clone https://gitlab.com/GingerTrain/emacs.d.git .emacs.d
```
Run Emacs as a daemon.
```
emacs --daemon
```
Connect to Emacs daemon.  
**Note: `-n` is short for `--no-wait` and tells Emacs to not hog the terminal, `-c` creates a new frame, and `-t` runs Emacs in the terminal.**
```
emacsclient -nc
```
Install packages from config
```
M-x list-packages `TODO: Finish`
```

## References
---

- [ArchWiki](https://wiki.archlinux.org/index.php/Installation_guide) - This is the first place I go when I am having Arch issues. Their installation guide was very useful and beginner friendly.
- [Arch Linux Forums](https://bbs.archlinux.org/) - This is the second place I go when I am having Arch issues. The community is very welcoming to beginners but only if you have put in a solid effort on your own to solve the problem, which means Google searching, reading the ArchWiki, reading the manual/documentation, and reading other forum posts related to the topic.
- [AUR](https://aur.archlinux.org/) - Arch User Repository, here you can find all the available packages.
- [Wiki³](https://kyau.net/wiki/ArchLinux:Installation) - A useful installation guide made by [Sean "kyau" Bruen](https://kyau.net/)

### Special Thanks
My brother Randy for giving me tons of advice and answering a lot of my questions.
