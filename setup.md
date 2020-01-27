# Arch Linux Installation & Configuration Guide

This setup guide assumes:

* Recent Intel CPU based UEFI enabled PC or Laptop
* Using Intel iGPU, No discrete GPU
* SSD (in the following /dev/sda)
  
Features:

* LUKS encryption of root partition, no seperate /home, no swap
* Localized for Austria
* GNOME, systemd & wayland everything
* Vulkan & VA-API support
* AUR support
* ZSH Shell
* NTFS & Printers

# Base install

`loadkeys de-latin1-nodeadkeys`

`timedatectl set-ntp true`

## Partitioning

`cgdisk /dev/sda`

Select gpt partition table.

1. 512MB - EFI system partition
2. REST - Linux filesystem

## LUKS & Filesystems

`mkfs.fat -F32 /dev/sda1`

`cryptsetup -y -v luksFormat /dev/sda2`

`cryptsetup open /dev/sda2 cryptroot`

`mkfs.ext4 /dev/mapper/cryptroot`

## Mount

`mkdir /mnt/boot`

`mount /dev/sda1 /mnt/boot`

`mount /dev/mapper/cryptroot /mnt`

## Mirrorlist

`pacman -S reflector`

`reflector --age 6 --country 'Germany' --latest 30 --number 20 --sort rate --save /etc/pacman.d/mirrorlist`

## Pacstrap

`pacstrap /mnt base base-devel linux linux-firmware sudo zsh nano git dhcpcd intel-ucode`

## fstab

`genfstab -U /mnt >> /mnt/etc/fstab`

`nano /etc/fstab`

```sh
# {UUID} -> UUID of /dev/mapper/cryptroot
UUID={UUID}     /           ext4      	defaults,noatime	0 1

# {UUID} -> UUID of /dev/sda1
UUID={UUID}     /boot     	vfat      	defaults	        0 2

tmpfs		    /tmp		tmpfs		defaults,noatime	0 0
```

## Chroot

`arch-chroot /mnt`

## Time

`ln -sf /usr/share/zoneinfo/Europe/Vienna /etc/localtime`

`timedatectl set-ntp true`

`timedatectl set-local-rtc 0`

`hwclock --systohc`

## Locale

`nano /etc/locale.gen`

Uncomment en_US.UTF-8 and de_AT.UTF-8

`locale-gen`

`nano /etc/locale.conf`

```sh
LANG=en_US.UTF-8
LC_CTYPE=C
LC_NUMERIC=de_AT.UTF-8
LC_TIME=de_AT.UTF-8
LC_COLLATE=C
LC_MONETARY=de_AT.UTF-8
LC_MESSAGES=en_US.UTF-8
LC_PAPER=de_AT.UTF-8
LC_NAME=de_AT.UTF-8
LC_ADDRESS=de_AT.UTF-8
LC_TELEPHONE=de_AT.UTF-8
LC_MEASUREMENT=de_AT.UTF-8
LC_IDENTIFICATION=en_US.UTF-8
```

`localectl --no-convert set-keymap de-latin1-nodeadkeys`

## Hostname

`hostnamectl set-hostname my_hostname`

`hostnamectl --pretty set-hostname "My pretty hostname"`

## Users & Sudo

`passwd`

`useradd -m -g users -G wheel -s /bin/zsh -c "My Full Name" my_username`

`passwd my_username`

`EDITOR=/usr/bin/nano visudo`

Uncomment %wheel ALL=(ALL) ALL

## Initramfs

`nano /etc/mkinitcpio.conf`

```sh
HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block sd-encrypt filesystems fsck)
```

`mkinitcpio -P`

## Bootloader

`bootctl --path=/boot install`

`nano boot/loader/entries/arch.conf`

```sh
title	Arch Linux
linux	/vmlinuz-linux
initrd  /intel-ucode.img
initrd	/initramfs-linux.img
# {UUID} -> UUID of /dev/sda2
options rd.luks.name={UUID}=cryptroot root=/dev/mapper/cryptroot
```

`nano boot/loader/loader.conf`
```sh
default arch
timeout 3
editor 0
```
`exit`

`reboot`

# First boot
Login as root and run `dhcpcd` to configure the wired network connection.

## Desktop environment

`pacman -Syu gnome`

## Networking

`pacman -Syu networkmanager-openconnect networkmanager-openvpn`

## Vulkan

`pacman -Syu vulkan-intel vulkan-tools`

Verify that everything works correctly with `vulkaninfo`.

## Video Hardware Acceleration

For Broadwell CPU and newer:

`pacman -Syu intel-media-driver libva-utils`

Below Broadwell CPU:

`pacman -Syu libva-intel-driver libva-utils`

Verify that everything works correctly with `vainfo`.

## Printers

`pacman -Syu cups`

## NTFS

`pacman -Syu ntfs-3g`

## AUR support via yay

`git clone https://aur.archlinux.org/yay.git`

`cd yay`

`makepkg -si`

`rm -rf yay`

## Automatic mirrorlist updates

`yay -Syu reflector-timer`

`systemctl enable reflector.timer`

## Automatic bootloader updates

`yay -Syu systemd-boot-pacman-hook`

## Set GDM keyboard layout

`localectl --no-convert set-x11-keymap at pc105 nodeadkeys`

## Enable essential services

`systemctl enable --now NetworkManager`

`systemctl enable --now bluetooth`

`systemctl enable --now avahi-daemon`

`systemctl enable --now org.cups.cupsd.service`

`systemctl enable --now gdm`

# Desktop Environment

Login as my_username via GDM (Wayland session).

The following are my UI and UX adaptions, adapt to whatever you need.

## UX

`yay -Syu gnome-tweaks gnome-shell-extensions gnome-shell-extension-appindicator gnome-shell-extension-dash-to-dock`

## Design

`yay -Syu qogir-gtk-theme-git papirus-icon-theme`

## Fonts

`yay -Syu ttf-liberation noto-fonts noto-fonts-cjk noto-fonts-emoji  powerline-fonts`

## Shell

`yay -Syu oh-my-zsh-git`

Download .zshrc to ~

## Import settings

To import my settings for GNOME and Extensions (Locale, Themes, Fonts, Dash-to-Dock, Terminal, ...), download gnome_settings.ini and run:

`dconf load / < gnome_settings.ini`

# Apps

This section describes my standard apps and their configuration, adapt to whatever you need.

### GUI

`yay -Syu firefox thunderbird thunderbird-extension-enigmail telegram-desktop-bin whatsapp-nativefier visual-studio-code-bin spotify vlc mpv calibre-python3 libreoffice-fresh libreoffice-fresh-de gimp inkscape clockify-desktop dropbox nautilus-dropbox`

### CLI

`yay -Syu htop neofetch youtube-dl speedtest-cli wget pass`

### Dev

`yay -Syu jdk-openjdk go docker valgrind ghex insomnia filezilla wireshark-qt`

## Firefox

### Run natively under Wayland

`nano ~/.config/environment.d/envvars.conf`
```sh
MOZ_ENABLE_WAYLAND=1
```

### Performance enhancements

Visit `about:config` and set the following keys:

* extensions.pocket.enabled=false
* layers.acceleration.force-enabled=true
* gfx.webrender.all=true
* browser.cache.disk.enable=false
* browser.cache.memory.enable=true
* browser.cache.memory.capacity=512000

You can verify the changes by visiting `about:support`.

### Store profile in RAM

`yay -Syu profile-sync-daemon`

`nano ~/.config/psd/psd.conf`
```sh
USE_OVERLAYFS="yes"
BROWSERS="firefox"
```

`sudo visudo`
```sh
my_username ALL=(ALL) NOPASSWD: /usr/bin/psd-overlay-helper
```

`systemctl --user start psd`

`psd p`

`systemctl --user enable psd`

## Thunderbird

### Run natively under Wayland

This is the same as in Firefox, skip if already enabled.

`nano ~/.config/environment.d/envvars.conf`
```sh
MOZ_ENABLE_WAYLAND=1
```

### Performance enhancements

Visit Preferences > Advanced > Config Editor and set the following keys:

* layers.acceleration.force-enabled=true
* gfx.webrender.all=true

You can verify the changes by visiting Help > Troubleshooting Information.

## Libre Office

### Spellcheck

`yay -Syu hunspell-en_US hunspell-de`

Enable Tools > Options > Language Settings > Writing Aids > Hunspell SpellChecker

### UI & UX

`sudo nano /etc/libreoffice/sofficerc`
```sh
Logo=0
```

`yay -Syu papirus-libreoffice-theme`

Select Tools > Options > View > Icon style > Papirus

Select View > User interface > Tabbed

## Dropbox

### Disable non-pacman auto updates

`rm -rf ~/.dropbox-dist`

`install -dm0 ~/.dropbox-dist`

## Docker

`sudo gpasswd -a my_username docker`

`sudo systemctl enable --now docker`

Relogin needed before use.

## VScode

Install `pkief.material-icon-theme` and `azemoh.one-monokai` extensions.

Paste this into settings.json
```json
{
    "telemetry.enableTelemetry": false,
    "window.menuBarVisibility": "toggle",
    "workbench.startupEditor": "newUntitledFile",
    "workbench.colorTheme": "One Monokai",
    "editor.fontFamily": "'Noto Sans Mono'",
    "workbench.iconTheme": "material-icon-theme",
    "editor.fontWeight": "600",
}
```