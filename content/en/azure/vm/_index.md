---
title: Virtual Machines
linkTitle: Virtual Machines
weight: 1
description: Azure VM troubleshooting guides, serial console recovery steps, and utilities for inspecting VM metadata.
type: docs
---

Use these notes to troubleshoot, recover, inspect, and automate your Azure VM
fleet.

## Featured guides

- [Enable Azure Serial Console and GRUB Menu on Linux VMs](/azure/vm/serial-console/) when SSH is unavailable or the VM is stuck early in boot
- [Profile Fetcher for Linux Azure VM](/azure/vm/getvmprofile/) when you need a fast local summary of Azure instance metadata

## Common use cases

- Recover a VM when only Azure Serial Console is available
- Validate bootloader and kernel console settings after image migration
- Capture instance metadata quickly during triage or support handoff
