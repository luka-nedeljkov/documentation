
# 1. Prepare the drives

Use `# lsblk` to check drive partitions. Use `# gdisk` to "zap" the drives before use, erasing the data in the process.

## 1.1. Partition the drives

```
# cgdisk /dev/*name_of_dive*
```
| Type | Hex code | Name | Size | Mount point |
| --- | --- | --- | --- | --- |
| EFI system partition | EF00 | efi | 512MiB | /mnt/efi |
| XBOOTLDR partition | EA00 | boot | 1GiB | /mnt/boot |
| Linux swap | 8200 | swap | (RAM size) | [SWAP] |
| Linux filesystem | 8300 | root | 32GiB | /mnt |
| Linux filesystem | 8300 | home | Rest of the drive | /mnt/home |

## 1.2. Format the partitions

Format the partitions created in the previous step with `# mkfs`.

```
# mkfs.fat -F32 /dev/*efi_partition*
# mkfs.fat -F32 /dev/*boot_partition*
# mkfs.ext4 /dev/*root_partition*
# mkfs.ext4 /dev/*home_partition*
# mkswap /dev/*swap_partition*
```

## 1.3. Mount the partitions

Mount the formatted partitions with `# mount`.

```
# mount /dev/*root_partition* /mnt
# mount /dev/*efi_partition* /mnt/efi --mkdir
# mount /dev/*boot_partition* /mnt/boot --mkdir
# mount /dev/*home_partition* /mnt/home --mkdir
# swapon /dev/*swap_partition*
```

> **Warning:** Make sure to mount the root partition **first** to avoid issues.

# 2. Installation

## 2.1. Updating the mirror list

This step is optional.

First backup your mirror list by copying `/etc/pacman.d/mirrorlist` to `/etc/pacman.d/mirrorlist.bak`.
```
# cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
```

Update the pacman repos and install the 'pacman-contrib' package with `# pacman -Sy pacman-contrib`.
```
# rankmirrors -n 6 /etc/pacman.d/mirrorlist.bak > /etc/pacman.d/mirrorlist
```

## 2.2. Run `pacstrap` to install the system

Install the initial system packages with `# pacstrap`. Package list:
* amd-ucode
* base
* base-devel
* linux
* linux-firmware

Install command:
```
# pacstrap -K /mnt amd-ucode base base-devel linux linux-firmware
```

# 3. Configuration

## 3.1. Fstab

Generate an fstab file.
```
# genfstab -U /mnt >> /mnt/etc/fstab
```
Check the resulting `/mnt/etc/fstab` file, and edit it in case of errors.

Edit the options for the `/efi` entry: `fmask=0137,dmask=0027`
Remount `/efi`:
```
# umount /efi
# mount /efi
```

## 3.2. Chroot

Change root into the new system:
```
# arch-chroot /mnt
```

## 3.3. Pacman

Install some essential packages with pacman:
```
# pacman -S bashtop git neofetch neovim pacman-contrib openssh zsh
```

Edit the `/etc/pacman.conf` file and uncomment or add the following lines:
```
...
Color
ParallelDownloads=5
ILoveCandy
...
[multilib]
Include = /etc/pacman.d/mirrorlist
...
```

Refresh the repos and upgrade the system:
```
# pacman -Syyuu
```

## 3.4. Localization

Edit `/etc/locale.gen` and uncomment `en_US.UTF-8 UTF-8`. Generate the locales by running:
```
# locale-gen
```

Create the locale.conf file, and set the LANG variable accordingly:

`/etc/locale.conf`
```
LANG=en_US.UTF-8
```

## 3.5. Time

Set the time zone:
```
# ln -sf /usr/share/zoneinfo/Europe/Belgrade /etc/localtime
```

Run `hwclock` to generate `/etc/adjtime`:
```
# hwclock --systohc
```

## 3.6. Enable the fstrim timer

Enable the fstrim timer if the system is installed on an **SSD**:
```
# systemctl enable fstrim.timer
```

## 3.7. Network Configuration

### 3.7.1. Hostname

Create the hostname file:

`/etc/hostname`
```
arch-luka
```

### 3.7.2 NetworkManager

Install the `networkmanager` package with pacman:
```
# pacman -S networkmanager
```

Enable the `NetworkManager` and `systemd-resolved` services.
```
# systemctl enable NetworkManager
# systemctl enable systemd-resolved
```

## 3.8. Root password

Set the root password:
```
# passwd
```

## 3.9. Boot loader (systemd-boot)

Install the boot loader with:
```
# bootctl --esp-path=/efi --boot-path=/boot install
```

### 3.9.1 Loader Configuration

The loader configuration is stored in the file `/efi/loader/loader.conf`.

`/efi/loader/loader.conf`
```
default arch.conf
timeout 0
editor no
```

> **Note:** Set `timeout` to `menu-force` if dual booting

### 3.9.2 Adding loaders

Create and edit a file at `/boot/loader/entries/arch.conf`

`/boot/loader/entries/arch.conf`
```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /amd-ucode.img
initrd  /initramfs-linux.img
```
Run the following command to make Arch boot from the root volume:
```
# echo "options root=PARTUUID=$(blkid -s PARTUUID -o value /dev/*root_partition*) rw quiet splash" >> /boot/loader/entries/arch.conf
```

## 3.10. Adding user(s)

Add a new user and change it's password:
```
# useradd -m -G wheel -s /bin/zsh luka
# passwd luka
```

Run the following command:
```
# EDITOR=nvim visudo
```
Uncomment the `%wheel ALL=(ALL:ALL) ALL` line and add `Defaults rootpw` to the end of the file.

# 4. Reboot

Exit the chroot environment by typing `exit` or pressing `Ctrl+d`.

Optionally manually unmount all the partitions with `umount -R /mnt`.

Finally, restart the machine by typing `reboot`. Remember to remove the installation medium and then login into the new system with your user account.

# 5. Graphics drivers

This step is optional. It is however reccomended to install the proper drivers for your graphics card.

## 5.1. Intel driver

Install the following packages:
* mesa
* lib32-mesa
* vulkan-intel
* lib32-vulkan-intel

## 5.2. NVIDIA driver

The NVIDIA proprietary driver is the most tedious to install. **Do not** use the open source driver.

### 5.2.1. Packages to install

Install the following packages:
* nvidia-dkms
* nvidia-utils
* lib32-nvidia-utils
* ~~opencl-nvidia~~
* ~~lib32-opencl-nvidia~~
* nvidia-settings

```
$ sudo pacman -S nvidia-dkms nvidia-utils lib32-nvidia-utils nvidia-settings
```

### 5.2.2. mkinitcpio.conf

Edit `/etc/mkinitcpio.conf` with the following lines:

`/etc/mkinitcpio.conf`
```
...
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
...
```

Run the following command to regenerate the initramfs:
```
$ sudo mkinitcpio -P
```

### 5.2.3. Boot loader entry

Edit `/boot/loader/entries/arch.conf` and add the following string to the end of the `options` line:

`/boot/loader/entries/arch.conf`
```
... nvidia-drm.modeset=1
```

### 5.2.4 pacman hook

Create a directory in `/etc/pacman.d/` called `hooks`. Create and edit a file called `/etc/pacman.d/hooks/nvidia.hook`:

`/etc/pacman.d/hooks/nvidia.hook`
```
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia

[Action]
Depends=mkinitcpio
When=PostTransaction
Exec=/usr/bin/mkinitcpio -P
```

# 6. Graphical User Interface

## 6.1. Display server and manager

Install the `xorg-server` and `xorg-apps` packages:
```
$ sudo pacman -S xorg-server xorg-apps
```

Install the `sddm` package:
```
$ sudo pacman -S sddm
```
Enable the `sddm` systemd service:
```
$ sudo systemctl enable sddm
```

## 6.3. Desktop environment

Choose whether you want a desktop environment or tiling window manager.

For the desktop environment, install the `plasma` package group:
```
$ sudo pacman -S plasma
```

For the tiling window manager, install the `i3` package group:
```
$ sudo pacman -S i3
```

## 6.4. Essential applications

Install the following packages:
* alacritty (terminal)
* dolphin (file explorer)
* firefox (browser)
```
$ sudo pacman -S alacritty dolphin firefox
```

# 7. Installing an AUR helper

Installation:
```
$ git clone https://aur.archlinux.org/yay.git
$ cd yay
$ makepkg -si
$ rm yay
```