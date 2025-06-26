
### start condition

nvme0n1
	nvme0n1p1 - 300M (windows) uefi
	nvme0n1p2 -16M (windows) reserved
	nvme0n1p3- 196,9G(windows) disk C
	nvme0n1p4- 579M(windows) recovery
	nvme0n1p5- free

```
cfdisk -l /dev/nvme0n1 
```

### partition

```
nvme0n1p1 - 300M (windows) uefi
nvme0n1p2 - 16M (windows) reserved
nvme0n1p3 - 196,9G (windows) disk C:\
nvme0n1p4 - 579M (windows) recovery
nvme0n1p5 - remainder - Linux
```

### format partition

```
mkfs.btrfs -fL ARCH /dev/nvme0n1p5
```


### mount

```
mount /dev/nvme0n1p5 /mnt (linux)
btrfs su cr /mnt/@
btrfs su cr /mnt/@home
btrfs su cr /mnt/@var
umoutn /mnt

mount -vo noatime,compress=zstd:2,space_cache=v2,discard=async,subvol=@ /dev/nvme0n1p5 /mnt

mkdir -pv /mnt/{boot,home,var,windows}
mount -vo noatime,compress=zstd:2,space_cache=v2,discard=async,subvol=@home /dev/nvme0n1p5 /mnt/home

mount -vo noatime,compress=zstd:2,space_cache=v2,discard=async,subvol=@var /dev/nvme0n1p5 /mnt/var

mount /dev/nvme0n1p1 /mnt/boot (windows efi)
mount /dev/nvme0n1p3 /mnt/windows (windows disk C)

lsblk
```


### install uefi dual boot

~~~ 
pacstrap -K /mnt base base-devel linux linux-firmware linux-headers  intel-ucode grub efibootmgr os-prober iwd dhcpcd networkmanager git micro nano bluez bluez-utils cups btrfs-progs pipeware pipeware-pulse paucontrol alacritty nvidia-dkms nvidia-settings
~~~

[[arch btrfs general instruction#first settings|continue]]

### grub install uefi dual boot


```
grub-install  --efi-directory=/boot
```

[[arch btrfs general instruction#finish|finish]]