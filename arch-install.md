# 1. Prepare the drives

Use `# lsblk` to check drive partitions. Use `# gdisk` to "zap" the drives before use, erasing the data in the process.

## 1.1. Partition the drives

```
# cgdisk /dev/*name_of_dive*
```
| Type | Hex code | Name | Size | Mount point |
| --- | --- | --- | --- | --- |
| EFI system partition | EF00 | esp | 1GiB | /mnt/boot |
| Linux swap | 8200 | swap | (RAM size) | [SWAP] |
| Linux filesystem | 8300 | root | 48GiB | /mnt |
| Linux filesystem | 8300 | home | Rest of the drive | /mnt/home |

## 1.2. Format the partitions

Format the partitions created in the previous step with `# mkfs`.

```
# mkfs.fat -F32 /dev/*esp_partition*
# mkfs.ext4 /dev/*root_partition*
# mkfs.ext4 /dev/*home_partition*
# mkswap /dev/*swap_partition*
```

## 1.3. Mount the partitions

Mount the formatted partitions with `# mount`.

```
# mount /dev/*root_partition* /mnt
# mount /dev/*esp_partition* /mnt/boot --mkdir -o umask=0077
# mount /dev/*home_partition* /mnt/home --mkdir
# swapon /dev/*swap_partition*
```

> **Warning:** Make sure to mount the root partition **first** to avoid issues.

# 2. Installation

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

## 3.2. Chroot

Change root into the new system:
```
# arch-chroot /mnt
```

## 3.3. Pacman

Install some essential packages with pacman:
```
# pacman -S bashtop git neofetch neovim pacman-contrib openssh tree zsh
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

Run the following command:

```
# echo LANG=en_US.UTF-8 > /etc/locale.conf
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

Run the following command:

```
# echo arch-luka > /etc/hostname
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
# bootctl install
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
# echo "options root=PARTUUID=$(blkid -s PARTUUID -o value /dev/*root_partition*) rw" >> /boot/loader/entries/arch.conf
```

## 3.10. Adding user(s)

Add a new user and change it's password:
```
# useradd -m -G wheel -s /bin/zsh luka
# passwd luka
# chfn luka
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

## 5.1. AMD driver

Install the following packages:
* mesa
* lib32-mesa
* vulkan-radeon
* lib32-vulkan-radeon
* libva-mesa-driver
* lib32-libva-mesa-driver
* mesa-vdpau
* lib32-mesa-vdpau

## 5.2. NVIDIA driver

The NVIDIA proprietary driver is the most tedious to install. **Do not** use the open source driver.

### 5.2.1. Packages to install

Install the following packages:
* nvidia-dkms
* nvidia-utils
* lib32-nvidia-utils
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

## 5.3. Intel driver

Install the following packages:
* mesa
* lib32-mesa
* vulkan-intel
* lib32-vulkan-intel

# 6. Graphical User Interface

## 6.1. Display server and manager

Install the `xorg-server` and `xorg-apps` packages:
```
$ sudo pacman -S xorg-server xorg-xinit xorg-xrandr
```

Install the `sddm` package:
```
$ sudo pacman -S sddm
```
Enable the `sddm` systemd service:
```
$ sudo systemctl enable sddm
```

## 6.3. Desktop environment or window manager\

Install the `plasma` package group:
```
$ sudo pacman -S plasma
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
$ cd ..
$ rm -rf yay
```
