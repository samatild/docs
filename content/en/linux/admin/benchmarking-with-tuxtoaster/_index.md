---
title: Linux Benchmarking Made Easy with Tux Toaster
description: A practical guide to stress testing and benchmarking Linux systems using the Tux Toaster toolkit.
date: 2025-10-17
type: docs
author: Samuel Matildes
tags: [linux, benchmarking, performance, stress-test, cpu, memory, disk, network]
---

<i class="fas fa-tachometer-alt" aria-hidden="true"></i> Benchmark smarter, not harder — with Tux Toaster.

## What is Tux Toaster?

[`Tux Toaster`](https://github.com/samatild/tuxtoaster) is an all-in-one performance toolkit for Linux. It triggers various load tests ("toasters") to help you evaluate the performance and stability of your system across CPU, memory, disk, and network. It offers an interactive terminal menu with multi-select support and clear, stoppable workloads.

![Preview](images/tuxtoaster_preview.gif)

## When to use it

- Hardware bring-up and burn-in
- Post-maintenance validation (kernel/firmware/driver updates)
- Capacity planning and instance comparison
- Performance regressions investigations
- Reproducible stress scenarios for bug reports

## Requirements

Tux Toaster targets Linux and relies on:

- Python 3.8+
- Python package: `psutil`
- System utilities: `dd`, `lsblk`, `taskset`, `pkill`
- Internet connectivity for network tests

Optional/privileged:

- Root privileges for the "Unclean GC" runaway memory test to adjust `oom_score_adj`

Install `psutil` if needed:

```bash
pip3 install psutil
```

## Installation

Clone and run the toolkit locally:

```bash
# Clone the repository
git clone https://github.com/samatild/tuxtoaster.git

# Navigate to the project directory
cd tuxtoaster

# Run the main Python script (interactive menu)
python3 tuxtoaster.py
```

Menu controls:

- Use arrow keys to navigate, Enter to select.
- Many submenus support multi-select; hints appear in the UI.
- Press `q`, `x`, or `Esc` in a menu to go back.
- During tests, press Enter to stop.

## Quick start

From the main menu, pick a category and test(s) to run.

### CPU

- Single Core
- All Cores
- Custom Number of Cores (uses `taskset`; experimental)

### Memory

- Single Runaway Thread
- Multiple Runaway Threads
- Memory spikes
- Unclean GC (requires root to set `oom_score_adj`)

### Disk

- IOPS Reads (4K, direct I/O)
- IOPS Writes (4K, direct I/O)
- Random IOPS R/W (4K, direct I/O)
- IOPS 50-50 R/W (4K, direct I/O)
- Throughput Reads (4MB, direct I/O)
- Throughput Writes (4MB, direct I/O)
- Random Throughput R/W (4MB, direct I/O)
- Throughput 50-50 R/W (4MB, direct I/O)
- Read while write cache is getting flushed

### Network

- Network IN (Single) — downloads `https://proof.ovh.net/files/100Mb.dat`
- Network OUT (Single) — UDP to `8.8.8.8:53`
- Network IN (Multiple) — N parallel downloads
- Network OUT (Multiple) — N parallel UDP senders
- Socket Exhaustion (under development)
- Simulate Latencies/Disconnects/Packet Loss (under development)

## Reading results

Tux Toaster prints live progress and a summary when you stop a test. Disk tests create temporary files under a dedicated directory on the selected mount points and clean up on exit. Network tests report bandwidth per socket in multi-socket modes.

Tips:

- Run tests at least 3 times and use medians for comparisons.
- Keep a record of CPU governor, kernel version, microcode, and thermal state.
- Pin CPU frequency when comparing hardware to reduce variance.

## Good benchmarking hygiene

- Stop noisy services (package updates, indexing, backup agents)

## Troubleshooting

- Missing `psutil`: `pip3 install psutil`
- Permission errors: some memory tests and `taskset` pinning may require `sudo`
- Inconsistent results: check CPU governor, temperature, and background load
- Direct I/O errors: some filesystems/containers may not honor `oflag=direct`

## Learn more

- Project: [`github.com/samatild/tuxtoaster`](https://github.com/samatild/tuxtoaster)
- Issues/feedback: open a GitHub issue with your logs and command line

---



