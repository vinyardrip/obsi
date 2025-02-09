h# Install arch btrfs

## connect wi-fi
```
iwctl
station wlan0 connect vinyardrip
pass
quit
ping -c3 google.com

setfont ter-128n
timedatectl set-ntp true
```

 ## setting mirrors & pacman

```
reflector -c Belarus,Russia,Ukraine -a 5 --sort rate --verbose --save /etc/pacman.d/mirrorlist

nano /etc/pacman.conf
```

```
Color
ILoveCandy

ParallelDownloads = 15

[multilib]
Iinclude = /etc/pacman.d/mirrorlist

pacman -Sy
```


[[single UEFI]]

[[single BIOS]]

[[dual boot UEFI]]

[[dual boot BIOS]]

### start condition

sda
	sda1 - (windows) recovery
	sda2 - (windows) EFI
	sda3 - (windows) reserved
	sda4 - (windows) DATA
	sda5 - free


### cfdisk empty disk

[[single BIOS#cfdisk|single boot]]
[[dual boot BIOS#cfdisk|dual boot]]


```
cfdisk -z /dev/sda 
gpt 
```

### partition

```
sda1 - 512M - type EFI- boot(for kernel)
sda2 -  remainder - Linux
```

### format partition

```
mkfs.vfat -n boot /dev/sda1
mkfs.btrfs -fL arch /dev/sda2
```
clearl all partition

```
wipefs -- all /dev/sda
```


### first settings
~~~ 
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt

micro /etc/locale.gen
ru_RU.UTF-8
en_US.UTF-8
locale-gen

ln -sf /usr/share/zoneinfo/Europe/Minsk /etc/localtime
hwclock --systohc
echo "LANG=en_US.UTF-8" >> /etc/locale.conf
echo "KEYMAP=ruwin_alt_sh-UTF-8 \nFONT=cyr-sun16" >> /etc/vconsole.conf
echo "PCNAME" >> /etc/hostname

micro /etc/hosts
~~~

### edit file hosts

127.0.0.1    localhost
::1                 localhost
127.0.1.1    PCNAME.localdomain    PCNAME

```
passwd
useradd -mG wheel,users -s /bin/bash USERNAME
passwd USERNAME
systemctl enable dhcpcd
systemctl enable iwd.service
systemctl enable bluetooth
systemctl enable cups
systemctl enable fstrim.timer
systemctl enable reflector.timer
EDITOR=micro visudo
uncomment %wheel ALL=(ALL) ALL

micro /etc/mkinitcpio.conf
MODULES=(btrfs)

mkinitcpio -p linux
```


###  grub install

[[single UEFI#grub install uefi|grub install UEFI]]

[[single BIOS#grub install bios|grub install BIOS]]

[[dual boot UEFI#grub install uefi dual boot|grub install dual boot UEFI]]

[[dual boot BIOS#grub install bios dual boot|grub install dual boot BIOS]]


### finish

```
grub-mkconfig -o /boot/grub/grub.cfg

exit

umount -a

reboot
```