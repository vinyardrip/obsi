

### start condition

sda

## partition single or empty disk

### cfdisk

```
cfdisk -z /dev/sda
gpt
```

### partition

```
sda1 - 512M - type EFI- boot(for kernel)
sda2 -  50G - (root)
```


## mount

```
mount /dev/sda2 /mnt
btrfs su cr /mnt/@
btrfs su cr /mnt/@home
btrfs su cr /mnt/@var

umoutn /mnt

mount -vo noatime,compress=zstd:2,space_cache=v2,discard=async,subvol=@ /dev/sda2 /mnt

mkdir -pv /mnt/{boot/uefi,home,var}
mount -vo noatime,compress=zstd:2,space_cache=v2,discard=async,subvol=@home /dev/sda2 /mnt/home

mount -o noatime,compress=zstd:2,space_cache=v2,discard=async,subvol=@var /dev/sda2 /mnt/var

mount /dev/sda1 /mnt/boot/uefi
```


### install uefi

~~~ 
pacstrap -K /mnt base base-devel linux linux-firmware linux-headers  intel-ucode grub efibootmgr os-prober iwd dhcpcd networkmanager git micro nano bluez bluez-utils cups btrfs-progs pipeware pipeware-pulse paucontrol alacritty nvidia-dkms nvidia-settings
~~~

[[arch btrfs general instruction#first settings|continue]]

### grub install uefi


```
grub-install  --efi-directory=/boot/uefi
```

[[arch btrfs general instruction#finish|finish]]
