
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
sda1 - 32M - type BIOS boot
sda2 - 512M - type EFI System
sda3 - remainder - Linux
```

### format partition

```
mkfs.vfat -n boot /dev/sda2
mkfs.btrfs -fL home /dev/sda3
```

### mount

```
mount /dev/sda3 /mnt
btrfs su cr /mnt/@
btrfs su cr /mnt/@home
btrfs su cr /mnt/@var

umoutn /mnt

mount -vo noatime,compress=zstd:2,space_cache=v2,discard=async,subvol=@ /dev/sda3 /mnt

mkdir -pv /mnt/{boot,home,var}
mount -vo noatime,compress=zstd:2,space_cache=v2,discard=async,subvol=@home /dev/sda3 /mnt/home

mount -vo noatime,compress=zstd:2,space_cache=v2,discard=async,subvol=@var /dev/sda3 /mnt/var

mount /dev/sda2 /mnt/boot
```

### install  arch bios

~~~ 
pacstrap -K /mnt base base-devel linux linux-firmware linux-headers intel-ucode git micro nano grub os-prober iwd dhcpcd networkmanager
~~~

[[arch btrfs general instruction#first settings|continue]]

### grub install bios

```
grub-install /dev/sda  
```


[[arch btrfs general instruction#finish|finish]]