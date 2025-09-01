
# ArchLinux+KVM+NixOS+Hyprland GPU-Passthrough  

> ✅ **UEFI включён**, **OVMF**, **NVIDIA**, **Arch/EndeavourOS host**, **NixOS guest**  
> Всё делается на физическом Arch, внутри — NixOS c Hyprland и полным passthrough NVIDIA.

---

## 1. Пакеты хост-системы
```bash
sudo pacman -Syu
sudo pacman -S qemu virt-manager ebtables dnsmasq dmidecode ovmf swtpm
````

---

## 2. IOMMU + VFIO (UEFI)

### 2.1 Убедиться, что IOMMU включён

bash

Copy

```bash
dmesg | grep -i iommu
```

Если нет — добавить в параметры загрузки.

### 2.2 GRUB (UEFI)

Открыть `/etc/default/grub`:

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

Сохранить и обновить:

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

Добавить модули:

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

## 7. Виртуальная машина (UEFI + OVMF)

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

### 7.3 Создание через virt-manager

1. `virt-manager` → «Create a new VM»  
    • Import existing disk → `nixos-gpu.qcow2`  
    • OS Type: Linux → Version: generic  
    • RAM ≥ 8 GB, CPUs ≥ 8 cores  
    • **Firmware**: UEFI x86_64: `/usr/share/edk2-ovmf/x64/OVMF_CODE.fd`  
    • Chipset: Q35  
    • **Video**: remove QXL/Virtio (none)  
    • Add Hardware → PCI Host Device → GPU + audio NVIDIA  
    • CPUs → host-passthrough  
    • TPM v2.0 (swtpm) — по желанию
    
2. Подключить ISO и установить NixOS.
    

---

## 8. Установка NixOS внутри VM

bash

Copy

```bash
sudo nixos-install
```

Пример `configuration.nix` (UEFI, GPT, systemd-boot):

nix

Copy

```nix
{ config, pkgs, ... }:

{
  imports = [ ./hardware-configuration.nix ];

  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;

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

Hyprland должен запуститься на **реальной NVIDIA** через VFIO passthrough.

---

## 10. Отладка

- `dmesg | grep -i vfio`
    
- `lspci -k | grep -A3 -i nvidia`
    
- `journalctl -xe`
    
- `sudo virsh dumpxml nixos-gpu | grep -i hostdev`