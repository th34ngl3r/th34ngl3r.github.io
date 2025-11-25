---
layout: post
title: "Installing Arch Linux with Pacstrap: A Step-by-Step Guide"
date: 2025-11-24
author: "Th3_4ngl3r"
categories: [Linux, Arch]
tags: [Arch] 
toc: true
---

# Installing Arch Linux with Pacstrap: A Step-by-Step Guide

Arch Linux is known for its simplicity, flexibility, and control. Unlike other distributions, Arch gives you the freedom to build your system from the ground up. In this guide, we’ll walk through the installation process using `pacstrap`, explaining each command in detail so you understand exactly what’s happening under the hood.

---

## Step 1: Ensure the System Clock is Correct

```bash
timedatectl set-ntp true
```

- Purpose: Synchronizes your system clock with network time servers.
- Why it matters: A correct system clock ensures proper package verification, SSL certificates, and smooth installation.

## Step 2: Partition, Format, and Mount the Disk

### Partition the disk

```bash
cfdisk /dev/nvme0n1
```

- Purpose: Opens a partition editor for the specified disk (/dev/nvme0n1).  I prefer cfdisk but you may choose your personal favorite.
- Action: Create partitions for EFI, root, home, and optionally swap.

### Format partitions

```bash
mkfs.fat -F32 /dev/nvme0n1p1
mkfs.ext4 -L ROOT /dev/nvme0n1p2
mkfs.ext4 -L HOME /dev/nvme0n1p3
```

- EFI partition: Formatted as FAT32 (mkfs.fat -F32).
- Root partition: Formatted as EXT4 with label ROOT.
- ome partition: Formatted as EXT4 with label HOME.

### Mount partitions

```bash
mount /dev/nvme0n1p2 /mnt
mkdir -p /mnt/home
mount /dev/nvme0n1p3 /mnt/home
mkdir -p /mnt/boot/efi
mount /dev/nvme0n1p1 /mnt/boot/efi
```

- Root: Mounted at /mnt.
- Home: Mounted at /mnt/home.
- EFI: Mounted at /mnt/boot/efi..

### Partition summary

| Partition | Device        | Filesystem | Mount point   |
|-----------|---------------|------------|---------------|
| EFI       | /dev/nvme0n1p1 | FAT32      | /mnt/boot/efi |
| Root      | /dev/nvme0n1p2 | EXT4       | /mnt          |
| Home      | /dev/nvme0n1p3 | EXT4       | /mnt/home     |

## Step 3: Install Base System with Pacstrap

```bash
pacstrap /mnt base linux linux-firmware
```

- Pacstrap: Installs packages into the mounted root filesystem.
- Packages:
  - base: Essential system utilities.
  - linux: Kernel.
  - linux-firmware: Firmware for hardware compatibility.

## Step 4: Generate fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

- Purpose: Generates filesystem table entries.
- Why: Ensures partitions auto-mount correctly at boot.

## Step 5: Change Root into the New System

```bash
arch-chroot /mnt /bin/bash
```

- Purpose: Switches into the new Arch environment.
- Effect: You’re now configuring the installed system as if it were booted.

## Step 6: Set Timezone

```bash
ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
hwclock --systohc
```

- Link timezone file.
- Synchronize hardware clock with system time.

## Step 7: Configure Locale

```bash
echo "en_US.UTF-8 UTF-8" > /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

- Locale: Defines language and character encoding.
- Generates UTF-8 locale for US English.

## Step 8: Set Hostname

```bash
echo "systemname" > /etc/hostname
cat >> /etc/hosts <<EOF
127.0.0.1   localhost
::1         localhost
127.0.1.1   systemname systemname.domain
EOF
```

- Hostname: Identifies your machine on a network.
- Hosts file: Maps hostnames to IP addresses.

## Step 9: Set Root Password

```bash
passwd root
```

- Purpose: Secures the root account with a password.

## Step 10: Create a Regular User

```bash
useradd -m -G wheel -s /bin/bash smokey
passwd smokey
```

- Creates user smokey.
  - -m: Creates home directory.
  - -G wheel: Adds user to wheel group (sudo privileges).
  - -s /bin/bash: Sets shell to Bash.

## Step 11: Install Essential Packages

```bash
pacman -S vim git sudo networkmanager
```

- vim: Text editor.
- git: Version control.
- sudo: Privilege escalation.
- networkmanager: Network management service.

## Step 12: Initialize and Populate Keyring

```bash
pacman-key --init
pacman-key --populate archlinux
pacman -Sy archlinux-keyring
```

- Purpose: Ensures package signatures can be verified.
- Refresh keys: Keeps system secure.

## Step 13: Install KDE Plasma (Wayland)

```bash
pacman -Syu --noconfirm
pacman -S --noconfirm \
    xorg-xwayland \
    plasma-meta \
    dolphin \
    kitty \
    sddm \
    wayland \
    qt6-wayland \
    pipewire pipewire-alsa pipewire-pulse pipewire-jack \
    networkmanager
systemctl enable sddm
systemctl enable NetworkManager
```

- Updates system.
- Installs Plasma desktop environment.
- Enables display manager (sddm) and networking.

## Step 14: Install GPU Drivers (Intel)

```bash
pacman -S mesa libva-intel-driver vulkan-intel
pacman -S libva-utils intel-media-driver
```

- Mesa: Open-source graphics drivers.
- VA-API / VDPAU: Hardware acceleration for video playback.

## Step 15: Install GRUB Bootloader (UEFI)

```bash
pacman -S grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

- GRUB: Bootloader for Linux.
- efibootmgr: Manages UEFI boot entries.

## Step 16: Create Swap Partition (optional)

```bash
mkswap /dev/nvme0n1p4
blkid /dev/nvme0n1p4
```

- mkswap: Initializes swap space.
- blkid: Retrieves UUID for fstab entry.

```bash
echo "UUID=6efd06e19-b378-4f8f-bdca-c3612d530c5e   none    swap    defaults    0 0" >> /etc/fstab
```

- Replace the UUID with the output from the blkid command

## Step 17: Unmount and Reboot

```bash
exit
umount -R /mnt
reboot
```

- Unmounts all partitions.
- Reboots into your new Arch Linux system.

## Final Thoughts

Installing Arch Linux with KDE is a fantastic way to build a stable, customizable desktop environment that balances power with user-friendliness. But if you’re ready to push your Linux journey even further, I highly recommend exploring Hyperland.

Hyperland is a dynamic Wayland compositor designed for speed, flexibility, and cutting-edge customization. Switching to Hyperland opens the door to:

- Performance gains: Lightweight and efficient, it can feel snappier than traditional desktop environments.
- Deep customization: From window animations to workspace layouts, you can tailor every detail to your workflow.
- Modern Wayland features: Better support for fractional scaling, touch gestures, and advanced rendering.
- Minimalist control: You decide exactly what runs, keeping your system lean and distraction-free.

By diving into Hyperland, you’re not just installing another desktop environment—you’re embracing a philosophy of total control and creativity. KDE gives you a polished starting point, but Hyperland lets you craft a workspace that’s uniquely yours. If you enjoy tinkering, experimenting, and squeezing every drop of potential out of your system, Hyperland is the next exciting step on your Arch Linux adventure.
