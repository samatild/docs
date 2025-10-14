---
title: Enable Azure Serial Console and GRUB Menu on Linux VMs
linkTitle: Serial Console (Linux)
description: Configure GRUB and systemd to expose the GRUB menu and kernel logs over Azure Serial Console on Linux VMs, including migrated images.
weight: 20
type: docs
date: 2025-10-14
author: Samuel Matildes
tags: [azure, linux, vm, grub, serial-console, ttyS0, boot, troubleshooting]
keywords: ["Azure Serial Console", "serial console", "ttyS0", "GRUB", "getty", "serial-getty@ttyS0", "earlyprintk", "earlycon", "kernel cmdline", "boot", "Azure VM", "Linux"]
---

Azure Serial Console is invaluable when SSH or the network is unavailable. This guide shows how to configure Linux distributions to expose the GRUB menu and kernel boot logs over the VM’s serial port on Azure, fixing cases where migrated images or some distros don’t show GRUB or any serial output.

> For proactive setup, follow Microsoft’s official guide: [Proactive GRUB and serial console configuration for Linux on Azure](https://learn.microsoft.com/en-us/troubleshoot/azure/virtual-machines/linux/serial-console-grub-proactive-configuration). This article is intended for cases where you’re facing issues getting serial console and the GRUB menu to display and need troubleshooting-oriented configuration.

### What you’ll do

- Enable GRUB output/input over serial
- Ensure kernel logs go to serial early in boot
- Enable a login prompt on `ttyS0`
- Regenerate GRUB config per distro

### Prerequisites

- VM is running in Azure
- You have admin/root access inside the VM
- In the Azure portal, ensure Serial Console access is permitted for your subscription/VM and that you have sufficient RBAC permissions

---

## 1) Edit `/etc/default/grub`

Back up the current file:

```bash
sudo cp -a /etc/default/grub /etc/default/grub.bak.$(date +%Y%m%d-%H%M%S)
```

Replace or add the following settings. The kernel command line can vary slightly by distro (see next section), but the following is a solid baseline for Azure VMs:

```bash
GRUB_TIMEOUT=10
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_CMDLINE_LINUX="crashkernel=auto console=tty1 console=ttyS0 earlyprintk=ttyS0 rootdelay=300"
GRUB_DISABLE_RECOVERY="true"
GRUB_ENABLE_BLSCFG=true
GRUB_TIMEOUT_STYLE=menu
GRUB_TERMINAL="serial console"
GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
GRUB_TERMINAL_OUTPUT="serial console"
GRUB_TERMINAL_INPUT="serial console"
```

Notes:

- The last `GRUB_TERMINAL_*` and `GRUB_TERMINAL` lines ensure both serial and local console are active; Azure uses `ttyS0` by default.
- `console=` parameters are processed in order; the last one becomes the primary console for messages. Keeping `console=ttyS0` last prioritizes Azure Serial Console output.
- `earlyprintk=ttyS0` helps very early boot logs appear; some newer kernels use `earlycon` instead (optional).

## 2) Distro-specific kernel cmdline and GRUB regeneration

After updating `/etc/default/grub`, regenerate GRUB with the appropriate command for your distro and firmware type.

### Ubuntu/Debian

- Prefer to include serial console in both lines:

```conf
GRUB_CMDLINE_LINUX="crashkernel=auto console=tty1 console=ttyS0,115200n8 rootdelay=300"
GRUB_CMDLINE_LINUX_DEFAULT="console=tty1 console=ttyS0,115200n8"
```

Regenerate GRUB:

```bash
sudo update-grub
```

### RHEL, CentOS, AlmaLinux, Rocky Linux, Fedora

- Keep `GRUB_ENABLE_BLSCFG=true` on BLS-based distros (RHEL8+/Fedora). Ensure `GRUB_CMDLINE_LINUX` includes serial console:

```conf
GRUB_CMDLINE_LINUX="crashkernel=auto console=tty1 console=ttyS0,115200n8 rootdelay=300"
```

Regenerate GRUB (choose the correct target for your firmware):

```bash
# BIOS
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# UEFI (RHEL/Alma/Rocky)
sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg

# UEFI (Fedora)
sudo grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg
```

### SUSE (SLES, openSUSE)

```conf
GRUB_CMDLINE_LINUX="crashkernel=auto console=tty1 console=ttyS0,115200n8 rootdelay=300"
```

Regenerate GRUB (choose the correct target for your firmware):

```bash
# BIOS
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# UEFI (SUSE)
sudo grub2-mkconfig -o /boot/efi/EFI/opensuse/grub.cfg
```

## 3) Enable a serial login on `ttyS0`

Most Azure images already spawn a getty on `ttyS0`, but some migrated images do not. Ensure it is enabled:

```bash
sudo systemctl enable --now serial-getty@ttyS0.service
```

Verify:

```bash
systemctl status serial-getty@ttyS0.service
```

## 4) Reboot and test

```bash
sudo reboot
```

In the Azure portal, open the VM and select Serial console. You should see the GRUB menu and kernel boot logs over serial. Use the arrow keys to select entries if needed.

---

## Verification checklist

- Kernel cmdline contains serial console:

```bash
cat /proc/cmdline
# expect ... console=tty1 console=ttyS0,115200n8 ...
```

- Serial device is present and initialized:

```bash
dmesg | grep -Ei "(ttyS0|console)"
```

- GRUB environment saved entry (optional):

```bash
sudo grub2-editenv list 2>/dev/null || true
```

---

## Troubleshooting

- No GRUB menu over serial
  - Ensure `GRUB_TIMEOUT_STYLE=menu` and `GRUB_TIMEOUT>0`
  - Ensure `GRUB_TERMINAL="serial console"` and `GRUB_SERIAL_COMMAND` are set
  - Regenerate GRUB for the correct firmware path (BIOS vs UEFI)

- No kernel logs over serial
  - Ensure `console=ttyS0,115200n8` appears in the kernel cmdline and is last
  - Consider adding `earlycon` on very new kernels if `earlyprintk` is ignored

- No login prompt in Serial Console
  - Enable `serial-getty@ttyS0.service`
  - Check `journalctl -u serial-getty@ttyS0.service -b`

- Migrated image still not working
  - Double-check `/etc/default/grub` matches the settings above
  - Reinstall/refresh the bootloader if necessary (distro-specific)
  - Verify the correct GRUB config path was regenerated

---

## References

- Azure Serial Console for Linux (Microsoft Learn): https://learn.microsoft.com/en-us/troubleshoot/azure/virtual-machines/linux/serial-console-linux


