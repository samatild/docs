---
title: Why Kernel Crash Dumps Are Critical for Root Cause Analysis
description: Deep-dive on using vmcore crash dumps for postmortem kernel debugging, including real-world kernel bug and OOM workflows.
date: 2025-10-14
type: docs
author: Samuel Matildes
tags: [linux, kernel, kdump, vmcore, crash-dump, kernel-panic, oom, memory-management, debugging, postmortem, makedumpfile, crash-utility]
---

<i class="fas fa-microscope" aria-hidden="true"></i> Postmortem Kernel Forensics with vmcore

## Summary

When the Linux kernel panics, there is no userspace stack, no application logs, and often no intact filesystems. The only canonical, lossless record of the kernel’s terminal state is the crash dump (vmcore). Without vmcore, you are constrained to heuristics and guesswork; with vmcore, you can deterministically reconstruct CPU state, task scheduling, memory allocators, locks, timers, and subsystems at the exact point of failure. This is the difference between timeline narratives and hard proof.

## What a vmcore Captures (and Why It Matters)

- CPU architectural state: general-purpose registers, control registers, MSRs, per-CPU contexts.
- Full kernel virtual memory snapshot: page tables, slab caches, VFS dentries/inodes, networking stacks, block layer queues, and device driver state.
- Task list and scheduler state: `task_struct`, runqueues, RT/DL classes, stop machine contexts.
- Lock state: `mutex`, `spinlock_t` owners, wait queues, and contention points.
- Timers/workqueues/interrupts: pending timers, softirqs, tasklets, IRQ threads, NMI backtraces.

With unstripped `vmlinux` and kernel debuginfo, these structures become symbol-resolved and type-aware in tools like `crash` and `gdb`.

## Minimal Prerequisites for a Useful Dump

- Reserve crash kernel memory at boot: `crashkernel=auto` (or a fixed size appropriate to RAM and distro guidance).
- Ensure `kdump` service is active and the dump target has write bandwidth and space (prefer raw disk/LVM or fast local FS; only use NFS/SSH if necessary).
- Keep exact-matching debuginfo for the running kernel build:
  - Uncompressed `vmlinux` with full DWARF and symbols.
  - Matching `System.map` and all loaded module debuginfo (e.g., `kernel-debuginfo`, `kernel-debuginfo-common` on RHEL/Fedora; `linux-image-…-dbgsym` on Debian/Ubuntu repositories).
- Persist critical panic policies:

```bash
sysctl -w kernel.panic_on_oops=1
sysctl -w kernel.unknown_nmi_panic=1
sysctl -w kernel.panic_on_unrecovered_nmi=1
sysctl -w vm.panic_on_oom=2   # 1=panic on OOM, 2=panic if no killable task
sysctl -w kernel.panic=10     # auto-reboot N seconds after panic
```

Persist via `/etc/sysctl.d/*.conf` as needed. For manual testing, enable SysRq and force a controlled crash:

```bash
echo 1 | sudo tee /proc/sys/kernel/sysrq
echo c | sudo tee /proc/sysrq-trigger
```

## Acquisition Pipeline and Size Reduction

`makedumpfile` can filter non-essential pages to reduce vmcore size and I/O time without destroying forensics value. Recommended options:

```bash
makedumpfile -l --message-level 1 \
  -d 31 \
  /proc/vmcore /var/crash/vmcore.filtered
```

- `-d 31` drops cache pages, free pages, user pages, and unused memory; tune masks per incident.
- Always retain an unfiltered copy during critical investigations if space allows.

## Core Tooling

- `crash`: purpose-built kernel postmortem shell using `vmlinux` DWARF types.
- `gdb` with `vmlinux`: useful for advanced symbol work and scripted analysis.
- `vmcore-dmesg`: extracts oops logs and last-kmsg from the dump.

Launch crash with debuginfo and module path:

```bash
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /var/crash/vmcore
```

Typical initial commands in `crash`:

```bash
sys                # kernel, uptime, panic info
ps                 # task list summary
bt                 # backtrace of current task (set with 'set' or '-p PID')
log                # kernel ring buffer extracted from vmcore
kmem -i            # memory info: zones, nodes, reclaimers
files -p <PID>     # per-process file descriptors
dev -d             # device list & drivers
irq                # IRQ and softirq state
foreach bt         # backtrace all tasks (can be heavy on large systems)
```

## Example 1 — Kernel Bug/Oops Leading to Panic

Symptoms at runtime: abrupt reboot, serial console shows BUG/oops with taint flags; no userspace core dumps.

Postmortem workflow:

```bash
vmcore-dmesg /var/crash/vmcore | less
```

Look for signatures such as:

```
BUG: unable to handle kernel NULL pointer dereference at 0000000000000010
RIP: 0010:driver_xyz_process+0x5a/0x120 [driver_xyz]
Call Trace:
  worker_thread+0x8f/0x1a0
  kthread+0xef/0x120
  ret_from_fork+0x2c/0x40
Tainted: G    B  W  OE  5.14.0-xyz #1
```

Correlate symbols and inspect the faulting frame:

```bash
crash> sym driver_xyz_process
crash> dis -l driver_xyz_process+0x5a
crash> bt
crash> set -p <pid_of_worker>
crash> bt -f            # show full frames with arguments
crash> struct task_struct <task_addr>
```

Representative outputs:

```text
# vmcore-dmesg (panic excerpt)
[ 1234.567890] Kernel panic - not syncing: Fatal exception
[ 1234.567891] CPU: 7 PID: 4123 Comm: kworker/u16:2 Tainted: G    B  W  OE   5.14.0-xyz #1
[ 1234.567893] Hardware name: Generic XYZ/ABC, BIOS 1.2.3 01/01/2025
[ 1234.567895] Workqueue: events_unbound driver_xyz_wq
[ 1234.567897] RIP: 0010:driver_xyz_process+0x5a/0x120 [driver_xyz]
[ 1234.567901] Call Trace:
[ 1234.567905]  worker_thread+0x8f/0x1a0
[ 1234.567906]  kthread+0xef/0x120
[ 1234.567907]  ret_from_fork+0x2c/0x40
```

```text
crash> sys
      KERNEL: /usr/lib/debug/lib/modules/5.14.0-xyz/vmlinux
    DUMPFILE: /var/crash/vmcore  [PARTIAL DUMP]
        CPUS: 32
        DATE: Tue Oct 14 10:22:31 2025
      UPTIME: 02:14:58
LOAD AVERAGE: 6.14, 6.02, 5.77
       PANIC: "Kernel panic - not syncing: Fatal exception"
         PID: 4123
     COMMAND: "kworker/u16:2"
        TASK: ffff8b2a7f1f0c00  [THREAD_INFO: ffffb2f1c2d2a000]
         CPU: 7
       STATE: TASK_RUNNING (PANIC)
```

```text
crash> ps
   PID    PPID  CPU       TASK        ST  %MEM     VSZ    RSS  COMM
> 4123       2    7  ffff8b2a7f1f0c00  RU   0.1   0      0    kworker/u16:2
   1         0    0  ffff8b2a70000180  IN   0.0  16272  1308  systemd
   532       1    2  ffff8b2a703f9b40  IN   0.2  912312 80324 containerd
   987     532    5  ffff8b2a7a2fcd00  IN   0.4  1452312 231212 kubelet
```

```text
crash> bt
PID: 4123  TASK: ffff8b2a7f1f0c00  CPU: 7  COMMAND: "kworker/u16:2"
 #0 [ffffb2f1c2d2be78] machine_kexec at ffffffff914b3e10
 #1 [ffffb2f1c2d2bec8] __crash_kexec at ffffffff915a1c32
 #2 [ffffb2f1c2d2bf28] panic at ffffffff914c2a9d
 #3 [ffffb2f1c2d2bf80] oops_end at ffffffff9148df90
 #4 [ffffb2f1c2d2bfb0] page_fault_oops at ffffffff9148e4b5
 #5 [ffffb2f1c2d2bfe0] exc_page_fault at ffffffff91abc7e1
 #6 [ffffb2f1c2d2c018] asm_exc_page_fault at ffffffff91c0133e
 #7 [ffffb2f1c2d2c048] driver_xyz_process+0x5a/0x120 [driver_xyz]
 #8 [ffffb2f1c2d2c0a0] worker_thread+0x8f/0x1a0
 #9 [ffffb2f1c2d2c0e0] kthread+0xef/0x120
#10 [ffffb2f1c2d2c110] ret_from_fork+0x2c/0x40
```

```text
crash> kmem -i
                 PAGES        TOTAL      PERCENTAGE
    TOTAL MEM    3276800      12.5 GB   100%
         FREE     152345       595 MB     4%
         USED    3124455      11.9 GB    96%
       SHARED      80312       313 MB     2%
      BUFFERS      49152       192 MB     1%
       CACHED     842304       3.2 GB    26%
        SLAB      921600       3.5 GB    28%
      PAGECACHE   655360       2.5 GB    20%
ZONE DMA32: min 16224, low 20280, high 24336, scanned 1e6, order 3 allocs failing
Reclaimers: kswapd0: active, direct reclaim: observed
```

```text
crash> log | head -n 6
<0>[ 1234.567890] Kernel panic - not syncing: Fatal exception
<4>[ 1234.567900] CPU: 7 PID: 4123 Comm: kworker/u16:2 Tainted: G    B  W  OE
<4>[ 1234.567905] RIP: 0010:driver_xyz_process+0x5a/0x120 [driver_xyz]
<6>[ 1234.567950] Workqueue: events_unbound driver_xyz_wq
```

Actionable patterns:

- Null-dereference at a deref site → check expected invariants and lifetime rules for the object; validate RCU usage (`rcu_read_lock()`/`_unlock()` pairs) and reference counting (`kref`, `refcount_t`).
- Use-after-free → examine slab allocator metadata around the pointer; `kmem` and `rd -p` (raw reads) can validate freelist poisoning.
- Interrupt vs thread context → verify hardirq/softirq context in `bt`; ensure lock acquisition order obeys documented lockdep dependencies.

If tainted by proprietary modules (`OE`), ensure matching module debuginfo is loaded so frames resolve cleanly. Validate module list:

```bash
crash> mod
```

From here, produce a minimal repro and map the faulting path to specific source lines using `dis -l` and DWARF line tables; attach exact register state and call trace to the fix.

## Example 2 — Out-Of-Memory (OOM) and Panic-on-OOM

By default, OOM does not produce a vmcore because the kernel kills a task to free memory and continues. For deterministic forensics on pathological memory pressure, set `vm.panic_on_oom=1` or `2` so the system panics and kdump captures a vmcore.

Pre-incident configuration:

```bash
sysctl -w vm.panic_on_oom=2
sysctl -w kernel.panic=15
```

After the event, extract the OOM report:

```bash
vmcore-dmesg /var/crash/vmcore | grep -A40 -B10 -n "Out of memory"
```

Example OOM excerpt:

```text
[ 4321.000001] Out of memory: Killed process 29876 (jav a) total-vm:16384000kB, anon-rss:15500000kB, file-rss:12000kB, shmem-rss:0kB, UID:1000 pgtables:30240kB oom_score_adj:0
[ 4321.000015] oom_reaper: reaped process 29876 (java), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB
[ 4321.000120] Memory cgroup out of memory: Killed process 30555 (python) in cgroup /kubepods/besteffort/pod123/xyz
[ 4321.000200] Node 0 DMA32: free:152kB min:16224kB low:20280kB high:24336kB active_anon:10123456kB inactive_anon:1123456kB active_file:123456kB inactive_file:654321kB unevictable:0kB
[ 4321.000250] kswapd0: node 0, oom: task failed order:3, mode:0x24201ca(GFP_HIGHUSER_MOVABLE|__GFP_ZERO)
```

Selected crash views for OOM analysis:

```text
crash> kmem -i
                 PAGES        TOTAL      PERCENTAGE
    TOTAL MEM    3276800      12.5 GB   100%
         FREE      20480        80 MB     0%
         USED    3256320      12.4 GB    99%
       CACHED     131072       512 MB     4%
        SLAB      983040       3.7 GB    30%
Direct reclaim active; high-order allocations failing (order:3)
```

```text
crash> ps -m | head -n 5
   PID    VSZ      RSS COMM
 29876  16384000 15500000 java
 30555   2048000  1800000 python
   987   1452312   231212 kubelet
```

Interpretation checklist inside `crash`:

```bash
crash> kmem -i               # zones, watermarks, reclaimers state
crash> kmem -s               # slab usage; look for runaway caches
crash> ps -m                 # memory stats per task
crash> vtop <task> <va>      # translate VA to PFN to inspect mapping
crash> files -p <PID>        # fd pressure and mmaps
crash> p sysctl_oom_dump_tasks
crash> log                   # OOM killer selection rationale, constraints
```

Indicators:

- `oom-kill:constraint=CONSTRAINT_NONE,nodemask=(null),cpuset=...` shows the policy path; `score`/`oom_score_adj` determine the victim.
- Stalled reclaim (`kswapd`, `direct reclaim`) with high `order` allocations failing → likely hugepages, GFP_ATOMIC depletion, or CMA stress.
- One slab consuming disproportionate memory → e.g., runaway `dentry` or `kmalloc-64` due to leak; confirm with `kmem -S` and inspect suspects via object walkers if available.

If OOM was triggered by a specific container/cgroup, use cgroup-aware views (kernel dependent):

```bash
crash> p memory.stat @<memcg_addr>
```

## Correlating vmcore with Source and Binaries

Always analyze with the exact build artifacts of the panicked kernel:

- `vmlinux` and module `.debug` files must match the `uname -r` and build ID of the running kernel at the time of panic.
- Mismatches lead to wrong type layouts, invalid offsets, and misleading backtraces.
- On distros with split debuginfo, install the `debuginfo` packages for the precise NVR (Name-Version-Release) string.

## Crash Analysis Cheat Sheet

- `vmcore-dmesg`: panic reason, oops, OOM logs; fastest high-signal overview.
- `sys`: kernel build, CPU count, uptime, panic string, current task/CPU.
- `ps` / `ps -m`: runnable tasks; `-m` adds memory stats per task.
- `bt` / `bt -f`: backtrace of current or selected task with frames/args.
- `kmem -i` / `-s` / `-S`: memory inventory; slabs by cache; cache detail.
- `log`: kernel ring buffer reconstructed from vmcore.
- `mod`: loaded modules and taint state.
- `files -p <PID>`: file descriptors and mmaps for a task.
- `irq`: hardirq/softirq state.
- `vtop <task> <va>`: VA→PFN translation; inspect mappings around suspect pages.

## References and Further Reading

- Crash utility: https://crash-utility.github.io/
- makedumpfile project: https://github.com/makedumpfile/makedumpfile
- Upstream kdump guide: https://www.kernel.org/doc/html/latest/admin-guide/kdump/kdump.html
- Ubuntu Kernel Crash Dump Recipe: https://wiki.ubuntu.com/Kernel/CrashdumpRecipe
- Fedora kdump guide: https://docs.fedoraproject.org/en-US/fedora-coreos/debugging-kernel-crashes/

---

