---
title: Kernel
linkTitle: Kernel
weight: 1
description: Linux kernel troubleshooting guides, crash dump content, CPU behavior, lockup analysis, and low-level execution concepts.
type: docs
---

Use this section to understand kernel behavior, investigate failures, and go
deeper into the low-level mechanics behind Linux systems.

## Featured guides

- [Kernel Mode vs User Mode: Privilege Levels and System Call Execution](/linux/kernel/kernel-mode-vs-user-mode/) for a deep explanation of execution context, memory protection, and system calls
- [Why Kernel Crash Dumps Matter](/linux/kernel/why-kernel-crash-dumps-matter/) for practical reasons to collect crash dumps before the next failure
- [Enabling Kdump Crash Collection](/linux/kernel/enabling-kdump-crash-collection/) for getting crash dumps configured on Linux systems
- [Soft vs Hard Lockups](/linux/kernel/softhardlockups/) for watchdog-driven lockup detection and troubleshooting

## Common tasks

- Prepare a system to collect crash dumps before a kernel panic occurs
- Understand why a machine hung, soft-locked, or hard-locked
- Learn how privilege levels and system calls shape performance and security behavior
- Investigate CPU statistics and kernel symptoms with better context
