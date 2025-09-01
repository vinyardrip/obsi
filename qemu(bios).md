
````markdown
# ArchLinux+KVM+NixOS+Hyprland GPU-Passthrough  
> ⚠️ **Legacy BIOS**, **UEFI отключён**, **NVIDIA**  
> Всё делается на **физическом Arch/EndeavourOS** (host), внутри — **NixOS** с **Hyprland** и **GPU passthrough**.

---

## 1. Пакеты хост-системы
```bash
sudo pacman -Syu
sudo pacman -S qemu virt-manager dnsmasq dmidecode ovmf swtpm
````

---

## 2. IOMMU + VFIO (Legacy BIOS)

### 2.1 Проверить группы IOMMU

bash

Copy

```bash
sudo dmesg | grep -i iommu
```

Если **IOMMU не виден**, добавь параметры вручную.

### 2.2 GRUB (Legacy BIOS)

Отредактируй `/etc/default/grub`:

bash

Copy

```bash
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt vfio-pci.ids=10de:XXXX,10de:YYYY kvm.ignore_msrs=1"
```

> Заменить `10de:XXXX,10de:YYYY` на **GPU + аудио**:

bash

Copy

```bash
lspci -nn | grep -i nvidia
```

Сохранить и пересобрать:

bash

Copy

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 3. VFIO раньше nvidia

Создать `/etc/modprobe.d/vfio.conf`:

bash

Copy

```bash
options vfio-pci ids=10de:XXXX,10de:YYYY
softdep nvidia pre: vfio-pci
```

---

## 4. Mkinitcpio

Добавить модули раньше nvidia:

bash

Copy

```bash
sudo nano /etc/mkinitcpio.conf
```

В секцию `MODULES`:

`MODULES=(vfio_pci vfio vfio_iommu_type1 vfio_virqfd)`

Пересобрать:

bash

Copy

```bash
sudo mkinitcpio -P
```

---

## 5. Проверка после перезагрузки

bash

Copy

```bash
dmesg | grep -i vfio
ls -l /sys/bus/pci/drivers/vfio-pci/ | grep 10de
```

---

## 6. Libvirt

bash

Copy

```bash
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt $USER
```

---

## 7. Виртуальная машина (Legacy BIOS + SeaBIOS)

### 7.1 Скачать ISO

bash

Copy

```bash
wget https://channels.nixos.org/nixos-24.05/latest-nixos-minimal-x86_64-linux.iso
```

### 7.2 Диск

bash

Copy

```bash
qemu-img create -f qcow2 nixos-gpu.qcow2 60G
```

### 7.3 Запуск без virt-manager (CLI)

> Прямой запуск, если virt-manager не умеет Legacy GPU passthrough.

bash

Copy

```bash
sudo qemu-system-x86_64 \
  -enable-kvm \
  -m 8192 \
  -smp 8 \
  -cpu host \
  -drive file=nixos-gpu.qcow2,if=virtio,cache=writeback \
  -drive file=nixos-minimal.iso,media=cdrom \
  -boot d \
  -vga none \
  -device vfio-pci,host=01:00.0,multifunction=on \
  -device vfio-pci,host=01:00.1 \
  -display none \
  -serial mon:stdio
```

> Поменять `01:00.0` и `01:00.1` на свои `lspci` адреса.

---

## 8. Установка NixOS внутри VM

1. Запустить CLI установщик:
    
    bash
    
    Copy
    
    ```bash
    sudo nixos-install
    ```
    
2. Пример `configuration.nix` (копировать в `/etc/nixos/`):
    

nix

Copy

```nix
{ config, pkgs, ... }:

{
  imports = [ ./hardware-configuration.nix ];

  boot.loader.grub.enable = true;
  boot.loader.grub.device = "/dev/vda";   # virtio
  boot.loader.grub.useOSProber = false;

  boot.kernelParams = [ "nvidia-drm.modeset=1" ];

  services.xserver = {
    enable = true;
    displayManager.gdm.enable = true;
    videoDrivers = [ "nvidia" ];
  };

  hardware = {
    opengl = {
      enable = true;
      driSupport = true;
    };
    nvidia = {
      modesetting.enable = true;
      powerManagement.enable = false;
      open = false;
    };
  };

  programs.hyprland = {
    enable = true;
    xwayland.enable = true;
  };

  users.users.nixos = {
    isNormalUser = true;
    extraGroups = [ "wheel" "video" ];
    password = "nixos";
  };

  system.stateVersion = "24.05";
}
```

---

## 9. Перезагрузка внутри VM

bash

Copy

```bash
sudo reboot
```

Если всё правильно — Hyprland запустится на **реальной NVIDIA** через VFIO passthrough.

---

## 10. Отладка

- `dmesg | grep -i vfio`
    
- `lspci -k | grep -A3 -i nvidia`
    
- `journalctl -xe`