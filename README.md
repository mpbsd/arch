How I Installed Arch on My Personal Laptop
==========================================

I maintain these notes as a reminder to myself in case I need to reinstall Arch.

Here I install Arch as the sole operating system on my machine with full disk
encryption (LVM on LUKS).

Turn `secure boot` off in the machine BIOS in case you haven't already.

1. Downloading the ISO:

    ```shell
    $ wget http://br.mirror.archlinux-br.org/iso/2022.02.01/archlinux-2022.02.01-x86_64.iso
    $ wget http://br.mirror.archlinux-br.org/iso/2022.02.01/archlinux-2022.02.01-x86_64.iso.sig
    $ wget http://br.mirror.archlinux-br.org/iso/2022.02.01/sha1sums.txt
    ```

2. Verify the integrity of the downloaded ISO:

    ```shell
    $ sha1sum -c --ignore-missing sha1sum.txt
    ```

3. Verify the signature of the ISO:

    ```shell
    $ gpg --keyserver-options auto-key-retrieve --verify archlinux-2022.02.01.iso.sig
    ```

4. Burn the ISO into a USB stick (be careful with this one):

    ```shell
    $ sudo dd if=archlinux.iso of=/dev/sdx bs=4M conv=fsync oflag=direct status=progress
    ```

5. Reboot the machine and enter the installer.

6. Adjust the family and size of the terminal fonts to your liking:

    ```shell
    # fontset ter-u28n
    ```

7. Adjust the layout of the keyboard with the `loadkeys` command.

8. Take a look at your network interfaces:

    ```shell
    # ip -c a
    ```

9. Connect to your network using `iwctl` (I don't have a wired connection just yet):

    ```shell
    # iwctl
    > station wlan0 get-networks
    ...
    > station wlan0 connect <SSID>
    password:
    > exit
    ```

10. Test your connection:

    ```shell
    # ping -c 3 archlinux.org
    ```

11. Update system's clock:

    ```shell
    timedatectl set-ntp true
    ```

12. Take a look at your disks:

    ```shell
    # lsblk
    ```

13. Get rid of existing partition tables on the target disk (here it's an nvme):

    ```shell
    # wipefs --all /dev/nvme0n1
    ```

14. Partition the target disk with `gdisk`:

    ```shell
    # gdisk /dev/nvme0n1
    ```

    Here's an example:

    | Partition      | Type       | Code | Size                       |
    | :------------- | :--------- | :--- | :------------------------- |
    | /dev/nvme0n1p1 | EFI System | ef00 | 1G                         |
    | /dev/nvme0n1p2 | Linux LVM  | 8e00 | `whatever is left on disk` |

15. Take another look at your disks:

    ```shell
    # lsblk
    ```

16. Write data to the newly created partitions:

    ```shell
    # dd if=/dev/zero of=/dev/nvme0n1p1 bs=1M count=1024
    ```

    This one is for paranoiac minds out there and could take quite a while
    depending on your hardware.

    ```shell
    # dd if=/dev/urandom of=/dev/nvme0n1p2
    ```

17. Encrypt the LVM partition with LUKS:

    ```shell
    # cryptsetup luksFormat /dev/nvme0n1p2
    ```

18. Open the encrypted partition with luksOpen:

    ```shell
    # cryptsetup luksOpen /dev/nvme0n1p2 crypticarch
    ```

19. Create the physical volume:

    ```shell
    # pvcreate /dev/mapper/cypticarch
    ```

20. Create the volume group:

    ```shell
    # vgcreate crypticarchvg /dev/mapper/crypticarch
    ```

21. Create separate root, swap and home partitions:

    ```shell
    # lvcreate -L 64G -n root crypticarchvg
    # lvcreate -L 4G -n swap crypticarchvg
    # lvcreate -l 100%FREE home crypticarchvg
    ```

22. It's probably a good idea to have yet another look at your disks at this moment:

    ```shell
    # lsblk
    ```

23. Format your recently created partitions:

    ```shell
    # mkfs.vfat /dev/nvme0n1p1
    # mkfs.ext4 /dev/crypticarchvg/root
    # mkswap /dev/crypticarchvg/swap
    # mkfs.ext4 /dev/crypticarchvg/home
    ```

24. Mount the file systems:

    ```shell
    # mount /dev/crypticarchvg/root /mnt
    # mkdir /mnt/{boot,home}
    # mount /dev/nvme0n1p1 /mnt/boot
    # mount /dev/crypticarchvg/home /mnt/home
    # swapon /dev/crypticarchvg/swap
    ```

25. Make sure everything is in proper place:

    ```shell
    # lsblk
    ```

26. Select the mirrors:

    ```shell
    # reflector --country Brazil --age 12 --sort rate --save /etc/pacman.d/mirrorlist
    ```

27. Install the basic system:

    ```shell
    # pacstrap /mnt base base-devel linux-lts linux-lts-headers linux-firmware intel-ucode lvm2 vi
    ```

28. Generate an fstab with `UUID`'s:

    ```shell
    # genfstab -U /mnt >> /mnt/etc/fstab
    ```
    
    Make sure everything is in place:

    ```shell
    # cat /mnt/etc/fstab
    ```

29. Change root:

    ```shell
    # arch-chroot /mnt
    ```

30. Select the machine's time zone and set the hardware's clock:

    ```shell
    # ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
    # hwclock --systohc
    ```

31. Localization:

    Uncomment that one line of `/etc/locale.conf` containing `en_US.UTF-8 UTF-8`:

    ```shell
    # vi /etc/locale.gen 
    # locale-gen
    ```

    Set the `LANG` variable by adding `LANG=en_US.UTF-8` to `/etc/locale.conf`:

    ```shell
    # vi /etc/locale.conf 
    ```

32. Network configuration:

    Set the machine's hostname:

    ```shell
    # vi /etc/hostname
    ```

    ```shell
    # vi /etc/hosts
    ```

33. Install and configure a boot loader (I mean, grub):

    ```shell
    # pacman -S grub efibootmgr
    ```

    ```shell
    # vi /etc/mkinitcpio.conf (HOOKS=(... encrypt lvm2 ...))
    ```

    ```shell
    # mkinitcpio -p linux-lts
    ```

    ```shell
    # grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
    ```

    Get the `UUID` of the `/dev/nvme0n1p2` partition:

    ```shell
    # blkid 
    ```

    Add

    > GRUB_CMDLINE_LINUX="cryptdevice=UUID=</dev/nvme0n1p2 UUID>:crypticarch root=/dev/crypticarchvg/root"

    to `/etc/default/grub`:

    ```shell
    # vi /etc/default/grub
    ```

    ```shell
    # grub-mkconfig -o /boot/grub/grub.cfg
    ```

33. Choose a root password for the root user:

    ```shell
    # passwd
    ```
```shell
# useradd -m -G wheel archie
# passwd archie
# EDITOR=vi visudo
```

```shell
# pacman -S networkmanager polkit
# systemctl enable NetworkManager
```

```shell
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
