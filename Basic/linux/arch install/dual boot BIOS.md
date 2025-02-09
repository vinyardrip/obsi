


### start condition

sda
	sda1 - 579M (windows) boot
	sda2 - 100G (windows) disk C\
	sda3 - free


```
cfdisk -l /dev/sda
```

### partition

```
sda1 - 579M (windows) boot
sda2 - 100G (windows) disk C:\
sda3 - remainder - Linux(primary)
```

### format partition

```
mkfs.btrfs -fL ARCH /dev/sda3
```


### mount

```
mount /dev/sda3 /mnt (linux)
btrfs su cr /mnt/@
btrfs su cr /mnt/@home
btrfs su cr /mnt/@var
umoutn /mnt

mount -vo noatime,compress=zstd:2,space_cache=v2,discard=async,subvol=@ /dev/nvme0n1p5 /mnt

mkdir -pv /mnt/{boot,home,var,windows}
mount -vo noatime,compress=zstd:2,space_cache=v2,discard=async,subvol=@home /dev/nvme0n1p5 /mnt/home

mount -vo noatime,compress=zstd:2,space_cache=v2,discard=async,subvol=@var /dev/nvme0n1p5 /mnt/var

mount /dev/sda2 /mnt/windows (windows disk C)

lsblk
```


### install bios dual boot

~~~ 
pacstrap -K /mnt base base-devel linux linux-firmware linux-headers  intel-ucode grub os-prober iwd dhcpcd networkmanager git micro nano bluez bluez-utils cups btrfs-progs pipeware pipeware-pulse paucontrol alacritty nvidia-dkms nvidia-settings
~~~

[[arch btrfs general instruction#first settings|continue]]

### grub install bios dual boot


```
grub-install /dev/sda
```

[[arch btrfs general instruction#finish|finish]]