---
title: Enabling Automatic Kernel Crash Collection with kdump
description: How to automatically enable and configure kdump crash collection on Linux systems using the kdump-enabler script.
date: 2025-10-13
type: docs
author: Samuel Matildes
tags: [linux, kernel, kdump, vmcore, crash-dump, kernel-panic, oom, memory-management, debugging, postmortem, makedumpfile, crash-utility]
---

<i class="fas fa-microchip" aria-hidden="true"></i> Automatic Enablement of Kernel Crash Dump Collection with kdump-enabler

This article explains how to automatically enable and configure kernel crash dump (kdump) collection on Linux systems using the `kdump-enabler` script. This approach works across multiple distributions and simplifies the process of preparing your system to collect crash dumps for troubleshooting and analysis.

## Overview

`kdump-enabler` is a Bash script that automates the setup of kdump:
- Installs required packages
- Configures the crashkernel parameter in GRUB
- Enables and starts the kdump service
- Sets up SysRq for manual crash triggering
- Creates backups of configuration files before changes
- Supports Ubuntu, Debian, RHEL, CentOS, Fedora, openSUSE, Arch Linux, and more

## Prerequisites
- Root privileges (run with `sudo`)
- systemd-based Linux distribution
- GRUB bootloader
- Sufficient disk space in `/var/crash` for crash dumps

## Installation

Clone the repository and run the script:

```bash
git clone https://github.com/samatild/kdump-enabler.git
cd kdump-enabler
sudo ./kdump-enabler.sh
```

Or download and run directly:

```bash
curl -O https://raw.githubusercontent.com/samatild/kdump-enabler/main/kdump-enabler.sh
chmod +x kdump-enabler.sh
sudo ./kdump-enabler.sh
```

## Usage

Run the script interactively:

```bash
sudo ./kdump-enabler.sh
```

Or use options for automation:

```bash
sudo ./kdump-enabler.sh -y           # Non-interactive mode
sudo ./kdump-enabler.sh --check-only  # Only check current configuration
sudo ./kdump-enabler.sh --no-sysrq    # Skip SysRq crash enablement
```


## What the Script Does

1. Detects your Linux distribution and package manager
2. Checks current kdump status
3. Installs required packages
4. Configures crashkernel parameter in GRUB based on system RAM
5. Enables kdump service at boot and starts it
6. Enables SysRq for manual crash triggering
7. Creates crash dump directory at `/var/crash`

## Post-Installation

After running the script, **reboot your system** for the crashkernel parameter to take effect:

```bash
sudo reboot
```

Verify kdump is working:

- Ubuntu/Debian:
  ```bash
  sudo kdump-tools test
  sudo systemctl status kdump-tools
  ```
- RHEL/CentOS/Fedora:
  ```bash
  sudo kdumpctl showmem
  sudo systemctl status kdump
  ```
- Check crashkernel:
  ```bash
  cat /proc/cmdline | grep crashkernel
  ```
- Check SysRq:
  ```bash
  cat /proc/sys/kernel/sysrq
  # Should output: 1
  ```

## Examples

Below are examples of running the script on different distributions and with various options, along with the kinds of output you can expect.

### Interactive run (Ubuntu/Debian)

```bash
sudo ./kdump-enabler.sh

# Output (abridged):
╔══════════════════════════════════════════════════════════════╗
║                        KDUMP ENABLER v1.0.0                  ║
╚══════════════════════════════════════════════════════════════╝
[INFO] Detecting Linux distribution...
[SUCCESS] Detected: Ubuntu 22.04
[INFO] Package manager: apt

[INFO] Checking current kdump configuration...
[WARNING] No crashkernel parameter found in kernel command line
[WARNING] kdump service exists but is not active
[WARNING] System requires kdump configuration

[WARNING] This script will:
  1. Install kdump packages (linux-crashdump kdump-tools kexec-tools)
  2. Configure crashkernel parameter in GRUB
  3. Enable and start kdump service
  4. Enable SysRq crash trigger
  5. Require a system reboot to complete setup
Do you want to continue? [y/N] y

[INFO] Installing required packages...
... apt-get update -qq
... apt-get install -y linux-crashdump kdump-tools kexec-tools
[SUCCESS] Packages installed successfully

[INFO] Configuring crashkernel parameter...
[INFO] Recommended crashkernel size: 384M (Total RAM: 12GB)
... updating /etc/default/grub
... running update-grub
[SUCCESS] GRUB configuration updated

[INFO] Configuring kdump settings...
... setting USE_KDUMP=1 in /etc/default/kdump-tools
[SUCCESS] kdump-tools configured

[INFO] Enabling kdump service...
[SUCCESS] kdump service enabled at boot
[WARNING] kdump service will start after reboot (crashkernel parameter needs to be loaded)

[INFO] Enabling SysRq crash trigger...
[SUCCESS] SysRq enabled for current session
[SUCCESS] SysRq configuration persisted to /etc/sysctl.conf

╔════════════════════════════════════════════════════════════════╗
║                    KDUMP SETUP COMPLETED                        ║
╚════════════════════════════════════════════════════════════════╝
IMPORTANT: A system reboot is required to apply all changes!
```

### Non-interactive run (auto-confirm)

```bash
sudo ./kdump-enabler.sh -y

# Output differences:
# - Skips confirmation prompts
# - Performs install/configuration immediately
```

### Check-only mode (no changes)

```bash
sudo ./kdump-enabler.sh --check-only

# Output (abridged):
[INFO] Checking current kdump configuration...
[WARNING] No crashkernel parameter found in kernel command line
[WARNING] kdump service not found
[INFO] Crash dump directory: /var/crash (0 dumps found)

# Exits after status check without installing or modifying anything
```

### Skip SysRq enablement

```bash
sudo ./kdump-enabler.sh -y --no-sysrq

# Output differences:
# - Does not enable SysRq or persist sysctl settings
# - All other steps (packages, GRUB, service) proceed
```

### RHEL/Fedora example highlights

```bash
sudo ./kdump-enabler.sh -y

# Output (abridged):
[SUCCESS] Detected: Red Hat Enterprise Linux 9
[INFO] Package manager: yum

[INFO] Installing required packages...
... yum install -y kexec-tools
[SUCCESS] Packages installed successfully

[INFO] Configuring crashkernel parameter...
... updating /etc/default/grub
... grub2-mkconfig -o /boot/grub2/grub.cfg
[SUCCESS] GRUB2 configuration updated

[INFO] Configuring kdump settings...
... ensuring path /var/crash in /etc/kdump.conf
... setting core_collector makedumpfile -l --message-level 1 -d 31
[SUCCESS] kdump.conf configured
```

## Testing Crash Dumps

⚠️ **Warning:** The following will immediately crash your system and generate a dump.

```bash
echo c | sudo tee /proc/sysrq-trigger
```

After reboot, check for crash dumps:

```bash
ls -lh /var/crash/
```

## Troubleshooting

- Ensure crashkernel is loaded: `cat /proc/cmdline | grep crashkernel`
- Reboot after running the script
- Check available memory and disk space
- View service logs: `sudo journalctl -u kdump -xe`
- Update GRUB if needed and reboot

## References
- [Ubuntu Kernel Crash Dump Recipe](https://wiki.ubuntu.com/Kernel/CrashdumpRecipe)
- [RHEL kdump Documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/installing-and-configuring-kdump_managing-monitoring-and-updating-the-kernel)
- [Fedora kdump Guide](https://docs.fedoraproject.org/en-US/fedora/latest/system-administrators-guide/kernel-module-driver-configuration/Working_with_Kernel_Crash_Dumps/)
- [Arch Linux kexec](https://wiki.archlinux.org/title/Kexec)

---

For more details, see the [kdump-enabler GitHub repository](https://github.com/samatild/kdump-enabler).
