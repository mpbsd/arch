My Arch Linux Installation
==========================

As stated by the project's website, [Arch Linux][] is a lightweight and flexible
Linux distribution that tries to Keep it Simple.

Download the ISO
----------------

```shell
$ wget
```

Burn the ISO to a USB with dd
-----------------------------

```shell
$ sudo dd if=archlinux.iso of=/dev/sda
```

Installation
------------

Here I adjust the font to my liking:

```shell
# fontset ter-u28n
```
I don't need to loadkeys as I use an US keyboard layout. Take a look at network
interfaces. The wireless network interface is generally called `wlan0`.

```shell
# ip a
```

Connect to my WiFi network using `iwctl` like this

```shell
# iwctl
[iwctl]# station wlan0 connect <SSID>
password:
```
```shell
# reflector --country Brazil --age 12 --sort rate --save /etc/pacman.d/mirrorlist
```

```shell
# lsblk
```

```shell
# wipefs --all /dev/nvme0n1
```

```shell
# gdisk /dev/nvme0n1
```

| Partition      | Type       | Code | Size        |
| :------------- | :--------- | :--- | :---------- |
| /dev/nvme0n1p1 | EFI System | ef00 | 2G          |
| /dev/nvme0n1p2 | Linux LVM  | 8e00 | `remainder` |

```shell
# lsblk
```

```shell
# dd if=/dev/zero of=/dev/nvme0n1p1 bs=1M count=2048
```

This can take quite some time.

```shell
# dd if=/dev/urandom of=/dev/nvme0n1p2
```

```shell
# cryptsetup luksFormat /dev/nvme0n1p2
```

answer `YES` with all capital letters, enter a password and confirm it by
retyping.

```shell
# cryptsetup luksOpen /dev/nvme0n1p2 crypticarch
# pvcreate /dev/mapper/cypticarch
# vgcreate crypticarchvg /dev/mapper/crypticarch
# lvcreate -L 256G -n root crypticarchvg
# lvcreate -L 32G -n swap crypticarchvg
# lvcreate -l 100%FREE home crypticarchvg
# lsblk
# mkfs.fat -F 32 /dev/nvme0n1p1
# mkfs.ext4 /dev/crypticarchvg/root
# mkswap /dev/crypticarchvg/swap
# mkfs.ext4 /dev/crypticarchvg/home
# mount /dev/crypticarchvg/root /mnt
# mkdir /mnt/{boot,home}
# mount /dev/nvme0n1p1 /mnt/boot
# mount /dev/crypticarchvg/home /mnt/home
# swapon /dev/crypticarchvg/swap
# lsblk
# pacstrap /mnt base base-devel linux-lts linux-lts-headers linux-firmware intel-ucode lvm2 vi
# genfstab -U /mnt >> /mnt/etc/fstab
# cat /mnt/etc/fstab
# arch-chroot /mnt
# ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
# hwclock --systohc
# vi /etc/locale.gen (en_US.UTF-8 UTF-8)
# locale-gen
# vi/etc/locale.conf (LANG=en_US.UTF-8)
# vi /etc/hostname (arch)
# vi /etc/hosts (127.0.0.1 hostname...)
# passwd
# pacman -S grub efibootmgr networkmanager polkit
# vi /etc/mkinitcpio.conf (HOOKS=(... encrypt lvm2 ...))
# mkinitcpio -p linux-lts
# grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
# blkid (copy the /dev/nvme0n1p2 uuid code)
# vi /etc/default/grub (GRUB_CMDLINE_LINUX=cryptdevice=UUID=<block device uuid>:crypticarch root=/dev/crypticarchvg/root
# grub-mkconfig -o /boot/grub/grub.cfg
# useradd -m -G wheel `archie`
# passwd `archie`
# EDITOR=vi visudo
# systemctl enable NetworkManager
# exit
# umount -R /mnt
# reboot
```

Install Some Packages
---------------------

```shell
$ sudo pacman -Syu
$ sudo pacman -S xorg xord-xdm
$ sudo pacman -S i3 dmenu rxvt-unicode terminus-font tmux 
$ sudo pacman -S xdg-utils xdg-user-dirs arc-gtk-theme papirus-icon-theme
$ sudo pacman -S firefox gvim ranger libreoffice-still
```

[Arch Linux]: https://archlinux.org
[Wiki]: https://wiki.archlinux.org
[Download]: https://archlinux.org/download