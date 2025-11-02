---
title: Understanding CPU Statistics in Linux (/proc/stat)
description: Deep technical dive into CPU time accounting in Linux, covering user, nice, system, idle, iowait, irq, softirq, steal, guest, and guest_nice statistics with practical examples and kernel internals.
date: 2025-01-14
type: docs
author: Samuel Matildes
tags: [linux, cpu, performance, monitoring, kernel, /proc/stat, system-administration, statistics]
---

<i class="fas fa-microchip" aria-hidden="true"></i> Kernel-Level CPU Time Accounting

## Summary

The Linux kernel maintains precise, per-CPU time accounting across ten distinct execution contexts. These statistics, exposed via `/proc/stat`, represent cumulative jiffy counters (typically 1/100th or 1/1000th of a second) since system boot. Understanding these counters is essential for performance analysis, capacity planning, and diagnosing CPU contention, I/O bottlenecks, interrupt storms, and virtualization overhead.

## The /proc/stat Interface

`/proc/stat` is a virtual file provided by the kernel's proc filesystem. It contains system-wide statistics aggregated across all CPUs and individual per-CPU lines. The format is non-blocking and updated atomically by the kernel scheduler's tick handler.

View the raw statistics:

```bash
cat /proc/stat
```

Example output:

```text
cpu  225748 1981 87654 3456789 1234 456 789 1011 2222 3333
cpu0 56387 495 21916 864197 308 114 197 252 555 833
cpu1 56390 495 21917 864198 309 114 197 253 556 833
cpu2 56385 495 21910 864197 308 114 197 252 555 833
cpu3 56386 496 21911 864197 309 114 198 254 556 834
intr 1234567890 123456 123456 123456 ...
ctxt 87654321
btime 1705123456
processes 23456
procs_running 2
procs_blocked 0
softirq 123456 789 12345 1234 5678 9012 3456 7890 123 456 789
```

The first line (`cpu`) aggregates all CPUs; subsequent `cpuN` lines show per-CPU statistics. Each CPU line contains ten fields:

```
cpuX user nice system idle iowait irq softirq steal guest guest_nice
```

**Note:** All values are cumulative counters measured in jiffies (kernel ticks). To calculate percentages or rates, you must sample at two points in time and compute deltas.

## Field-by-Field Breakdown

### 1. user (usr)

**Kernel context:** Time spent executing user-space code in normal priority processes.

**Increment condition:** Kernel tick handler (`account_process_tick()`) counts time when a process is running in user mode with default priority (nice value 0-0).

**Kernel source:** `kernel/sched/cputime.c::account_user_time()`

**Example interpretation:**
- High `user` time indicates CPU-bound applications
- Typical range: 20-80% on interactive systems
- Can exceed 100% per CPU if multiple threads run concurrently (SMP accounting)

**Practical example:**

```bash
# Monitor user CPU time growth over 5 seconds
T1=$(grep '^cpu ' /proc/stat | awk '{print $2}')
sleep 5
T2=$(grep '^cpu ' /proc/stat | awk '{print $2}')
DELTA=$((T2 - T1))
HZ=$(getconf CLK_TCK)  # Usually 100 or 1000
PERCENT=$(echo "scale=2; ($DELTA * 100) / ($HZ * 5)" | bc)
echo "User CPU usage: ${PERCENT}%"
```

**Kernel implementation detail:**

```c
// Simplified representation of kernel accounting
void account_user_time(struct task_struct *p, u64 cputime)
{
    p->utime += cputime;
    if (task_nice(p) <= 0)
        account_cputime_user(p, cputime);
}
```

### 2. nice

**Kernel context:** Time spent executing user-space code in niced processes (nice value < 0 means higher priority, > 0 means lower priority).

**Increment condition:** Same as `user`, but only when `task_nice(p) != 0` (non-zero nice value).

**Kernel source:** `kernel/sched/cputime.c::account_user_time()` with nice check

**Example interpretation:**
- Non-zero values indicate processes with adjusted priorities
- `nice < 0`: high-priority processes (e.g., RT priority mapped to nice)
- `nice > 0`: low-priority background tasks
- Modern kernels may report 0 if no niced processes exist

**Practical example:**

```bash
# Find processes contributing to nice time
ps -eo pid,ni,comm,pcpu --sort=-pcpu | head -10

# Generate nice time by running a niced process
nice -n 19 dd if=/dev/zero of=/dev/null bs=1M count=1000 &
NICE_PID=$!
sleep 2
# Check nice time increment
grep '^cpu ' /proc/stat | awk '{print $3}'
```

**Advanced: Nice value to priority mapping:**

```bash
# Show nice values and their scheduling impact
for nice in -20 -10 0 10 19; do
    echo "Nice $nice -> Priority: $((100 + nice))"
done
```

### 3. system (sys)

**Kernel context:** Time spent executing kernel code on behalf of user processes (system calls, kernel services).

**Increment condition:** Kernel tick handler counts time when the process is in kernel mode (handling syscalls, page faults, exceptions, etc.).

**Kernel source:** `kernel/sched/cputime.c::account_system_time()`

**Example interpretation:**
- High `system` time indicates frequent syscalls or kernel processing
- Typical range: 5-30% on normal systems
- Spikes suggest I/O-bound workloads, context switching, or kernel-intensive operations
- >50% may indicate kernel bottlenecks or driver issues

**Practical example:**

```bash
# Monitor system call rate (indirectly via system time)
T1=$(grep '^cpu ' /proc/stat | awk '{print $4}')
strace -c -e trace=all sleep 1 2>&1 | tail -1
T2=$(grep '^cpu ' /proc/stat | awk '{print $4}')
echo "System time delta: $((T2 - T1)) jiffies"

# High system time scenarios:
# 1. Frequent file I/O
dd if=/dev/urandom of=/tmp/test bs=4K count=10000

# 2. Network operations
curl -s https://example.com > /dev/null

# 3. Process creation
for i in {1..1000}; do true; done
```

**Kernel code path:**

```c
// System time accounting during syscall
long sys_xyz(...) {
    // Pre-syscall timestamp
    account_system_time(current, cputime_before);
    // ... kernel work ...
    account_system_time(current, cputime_after);
}
```

### 4. idle

**Kernel context:** Time the CPU spent idle (no runnable tasks, waiting in idle loop).

**Increment condition:** Kernel idle loop (`do_idle()`) executes when the runqueue is empty. The idle task (PID 0, `swapper`) runs and increments this counter.

**Kernel source:** `kernel/sched/idle.c::do_idle()`

**Example interpretation:**
- High `idle` = low CPU utilization
- Idle time should decrease under load
- `100% - idle%` ≈ total CPU utilization
- On SMP systems, one CPU can be idle while others are busy

**Practical example:**

```bash
# Calculate CPU utilization percentage
get_cpu_usage() {
    local cpu_line=$(grep '^cpu ' /proc/stat)
    local user=$(echo $cpu_line | awk '{print $2}')
    local nice=$(echo $cpu_line | awk '{print $3}')
    local system=$(echo $cpu_line | awk '{print $4}')
    local idle=$(echo $cpu_line | awk '{print $5}')
    local iowait=$(echo $cpu_line | awk '{print $6}')
    
    local total=$((user + nice + system + idle + iowait))
    local used=$((user + nice + system))
    
    echo "scale=2; ($used * 100) / $total" | bc
}

# Monitor over time
while true; do
    echo "$(date): CPU Usage: $(get_cpu_usage)%"
    sleep 1
done
```

**Idle loop internals:**

```c
// Simplified idle loop (arch-specific)
static void do_idle(void) {
    while (1) {
        if (need_resched()) {
            schedule_idle();
            continue;
        }
        // Enter low-power state (HLT, MWAIT, etc.)
        arch_cpu_idle();
        account_idle_time(cpu, cputime);
    }
}
```

### 5. iowait

**Kernel context:** Time the CPU spent idle while waiting for I/O operations to complete.

**Increment condition:** CPU is idle (`idle` would increment) but there are outstanding I/O requests in flight. This is a special case of idle time.

**Kernel source:** `kernel/sched/cputime.c::account_idle_time()` with I/O pending check

**Example interpretation:**
- Indicates I/O-bound workloads
- High `iowait` suggests disk/network bottlenecks
- **Important:** `iowait` does NOT mean the CPU is busy; it's idle time waiting for I/O
- Combined with low `user`/`system` = I/O bottleneck, not CPU bottleneck
- Can be misleading on systems with async I/O (io_uring, etc.)

**Practical example:**

```bash
# Generate iowait by saturating disk I/O
dd if=/dev/zero of=/tmp/stress bs=1M count=10000 oflag=direct &
DD_PID=$!

# Monitor iowait growth
watch -n 1 "grep '^cpu ' /proc/stat | awk '{print \"iowait: \" \$6 \" jiffies\"}'"

# Stop the stress
kill $DD_PID

# Compare with actual I/O stats
iostat -x 1 5
```

**Kernel accounting logic:**

```c
void account_idle_time(struct rq *rq, u64 cputime) {
    if (nr_iowait_cpu(smp_processor_id()) > 0)
        account_cputime_iowait(rq, cputime);
    else
        account_cputime_idle(rq, cputime);
}
```

**Common misconception:** `iowait` is NOT CPU time spent on I/O. The CPU is idle; the I/O device (disk controller, NIC) is busy.

### 6. irq (hardirq)

**Kernel context:** Time spent servicing hardware interrupts (IRQs).

**Increment condition:** Kernel interrupt handler executes. Each IRQ handler increments per-CPU and per-IRQ counters.

**Kernel source:** `kernel/softirq.c`, interrupt handlers

**Example interpretation:**
- High `irq` indicates interrupt-heavy workloads (network, storage, timers)
- Typical range: <1% on idle systems, 1-5% under load
- Spikes suggest hardware issues or misconfigured interrupt affinity
- Can be distributed via `smp_affinity` masks

**Practical example:**

```bash
# View interrupt distribution
cat /proc/interrupts

# Generate high IRQ load (network interrupts)
iperf3 -s &
SERVER_PID=$!
iperf3 -c localhost -t 30 -P 8  # 8 parallel streams
kill $SERVER_PID

# Monitor IRQ time
T1=$(grep '^cpu ' /proc/stat | awk '{print $7}')
sleep 5
T2=$(grep '^cpu ' /proc/stat | awk '{print $7}')
echo "IRQ time delta: $((T2 - T1)) jiffies"

# Set interrupt affinity (example: bind IRQ 24 to CPU 0)
echo 1 > /proc/irq/24/smp_affinity
```

**Interrupt handler accounting:**

```c
irqreturn_t handle_irq_event(struct irq_desc *desc) {
    u64 start = local_clock();
    // ... handle interrupt ...
    account_hardirq_time(current, local_clock() - start);
    return IRQ_HANDLED;
}
```

### 7. softirq

**Kernel context:** Time spent executing softirqs (deferred interrupt processing, bottom halves).

**Increment condition:** Kernel softirq daemon (`ksoftirqd`) or direct softirq execution in interrupt context processes pending softirqs.

**Kernel source:** `kernel/softirq.c::__do_softirq()`

**Softirq types (from `include/linux/interrupt.h`):**
- `HI_SOFTIRQ`: High-priority tasklets
- `TIMER_SOFTIRQ`: Timer callbacks
- `NET_TX_SOFTIRQ`: Network transmit
- `NET_RX_SOFTIRQ`: Network receive
- `BLOCK_SOFTIRQ`: Block device I/O completion
- `IRQ_POLL_SOFTIRQ`: IRQ polling
- `TASKLET_SOFTIRQ`: Normal tasklets
- `SCHED_SOFTIRQ`: Scheduler callbacks
- `HRTIMER_SOFTIRQ`: High-resolution timers
- `RCU_SOFTIRQ`: RCU callbacks

**Example interpretation:**
- High `softirq` indicates deferred processing load (network, timers, RCU)
- Network-heavy workloads show high `NET_RX_SOFTIRQ`/`NET_TX_SOFTIRQ`
- RCU-heavy systems (many CPUs) show high `RCU_SOFTIRQ`
- Softirq time can exceed 10% on network servers

**Practical example:**

```bash
# View softirq breakdown
cat /proc/softirq

# Example output:
#                    CPU0       CPU1       CPU2       CPU3
#          HI:          0          0          0          0
#       TIMER:     123456     123457     123458     123459
#      NET_TX:       1234       1235       1236       1237
#      NET_RX:      56789      56790      56791      56792
#       BLOCK:          0          0          0          0
#    IRQ_POLL:          0          0          0          0
#     TASKLET:         12         13         14         15
#       SCHED:      23456      23457      23458      23459
#     HRTIMER:          0          0          0          0
#         RCU:     345678     345679     345680     345681

# Generate NET_RX softirq load
iperf3 -s &
SERVER_PID=$!
iperf3 -c localhost -t 10 -P 16
kill $SERVER_PID

# Monitor softirq time growth
watch -n 1 "grep '^cpu ' /proc/stat | awk '{print \"softirq: \" \$8}'"
```

**Softirq execution contexts:**

```c
// Softirqs can run in:
// 1. Interrupt return path (if pending)
irq_exit() {
    if (in_interrupt() && local_softirq_pending())
        invoke_softirq();
}

// 2. ksoftirqd kernel thread
static int ksoftirqd(void *data) {
    while (!kthread_should_stop()) {
        __do_softirq();
        schedule();
    }
}

// 3. Explicit raise (local_bh_enable, etc.)
```

### 8. steal

**Kernel context:** Time "stolen" by the hypervisor from a virtual CPU (only in virtualized environments).

**Increment condition:** Hypervisor preempts the guest VM's virtual CPU to schedule other VMs or host tasks. The guest kernel detects this via paravirtualized time sources (e.g., KVM's `kvm_steal_time`).

**Kernel source:** `arch/x86/kernel/kvm.c::kvm_steal_time_setup()`

**Example interpretation:**
- Only non-zero in VMs (KVM, Xen, VMware, Hyper-V)
- High `steal` indicates host CPU overcommit or noisy neighbors
- >10% suggests VM should be migrated or host CPU capacity increased
- `steal` time is "lost" from the guest's perspective

**Practical example:**

```bash
# Check if running in a VM
if [ -d /sys/devices/virtual/dmi/id ]; then
    if grep -q "KVM\|VMware\|Xen\|Microsoft" /sys/devices/virtual/dmi/id/product_name 2>/dev/null; then
        echo "Running in virtualized environment"
    fi
fi

# Monitor steal time
watch -n 1 "grep '^cpu ' /proc/stat | awk '{print \"steal: \" \$9 \" jiffies\"}'"

# Calculate steal percentage
calc_steal_pct() {
    local cpu_line=$(grep '^cpu ' /proc/stat)
    local total=$(echo $cpu_line | awk '{sum=0; for(i=2;i<=NF;i++) sum+=$i; print sum}')
    local steal=$(echo $cpu_line | awk '{print $9}')
    echo "scale=2; ($steal * 100) / $total" | bc
}
```

**KVM steal time mechanism:**

```c
// Host side (KVM)
static void record_steal_time(struct kvm_vcpu *vcpu) {
    struct kvm_steal_time *st = vcpu->arch.st;
    st->steal += current->sched_info.run_delay;
}

// Guest side (Linux kernel)
static void kvm_steal_time_setup(void) {
    // Read steal time from shared page
    steal = st->steal;
    account_steal_time(steal);
}
```

### 9. guest

**Kernel context:** Time spent running a guest OS (nested virtualization or KVM guest time accounting).

**Increment condition:** Host kernel accounts time when a guest VM's virtual CPU is executing. This is the inverse of `steal` from the host's perspective.

**Kernel source:** `kernel/sched/cputime.c::account_guest_time()`

**Example interpretation:**
- Non-zero on hypervisor hosts running VMs
- Represents CPU time consumed by guest VMs
- In nested virtualization, a guest VM can itself host VMs
- Typically only relevant for hypervisor monitoring

**Practical example:**

```bash
# On a KVM host, monitor guest time
watch -n 1 "grep '^cpu ' /proc/stat | awk '{print \"guest: \" \$10}'"

# Compare with VM CPU usage (from host perspective)
virsh domstats --cpu <domain>
```

**Guest time accounting:**

```c
// When guest VM executes on host CPU
void account_guest_time(struct task_struct *p, u64 cputime) {
    account_cputime_guest(p, cputime);
    // Guest time is also counted as user time from host perspective
    account_user_time(p, cputime);
}
```

### 10. guest_nice

**Kernel context:** Time spent running niced guest OS processes (nested virtualization).

**Increment condition:** Same as `guest`, but for processes with non-zero nice values in the guest.

**Kernel source:** `kernel/sched/cputime.c::account_guest_time()` with nice check

**Example interpretation:**
- Parallel to `nice` but for guest VM processes
- Rarely non-zero unless guests run niced workloads
- Mostly relevant for hypervisor capacity planning

**Practical example:**

```bash
# Monitor guest_nice (typically zero)
grep '^cpu ' /proc/stat | awk '{print "guest_nice: " $11}'
```

## Complete CPU Utilization Calculation

To calculate accurate CPU percentages, sample `/proc/stat` at two points and compute deltas:

```bash
#!/bin/bash
# Comprehensive CPU statistics calculator

get_cpu_stats() {
    grep '^cpu ' /proc/stat | awk '{
        user=$2; nice=$3; system=$4; idle=$5;
        iowait=$6; irq=$7; softirq=$8; steal=$9;
        guest=$10; guest_nice=$11;
        
        # Total is sum of all fields (idle included)
        total = user + nice + system + idle + iowait + irq + softirq + steal
        
        # Active time (excluding idle and iowait)
        active = user + nice + system + irq + softirq
        
        # Idle time (true idle + iowait)
        idle_total = idle + iowait
        
        print user, nice, system, idle, iowait, irq, softirq, steal, guest, guest_nice, total, active, idle_total
    }'
}

# Sample 1
S1=($(get_cpu_stats))
sleep 1

# Sample 2
S2=($(get_cpu_stats))

# Calculate deltas
for i in {0..12}; do
    DELTA[$i]=$((${S2[$i]} - ${S1[$i]}))
done

# Calculate percentages (assuming HZ=100)
HZ=100
TOTAL_DELTA=${DELTA[10]}

echo "CPU Statistics (1 second sample):"
echo "================================"
printf "User:      %6.2f%%\n" $(echo "scale=2; (${DELTA[0]} * 100) / $TOTAL_DELTA" | bc)
printf "Nice:      %6.2f%%\n" $(echo "scale=2; (${DELTA[1]} * 100) / $TOTAL_DELTA" | bc)
printf "System:    %6.2f%%\n" $(echo "scale=2; (${DELTA[2]} * 100) / $TOTAL_DELTA" | bc)
printf "Idle:      %6.2f%%\n" $(echo "scale=2; (${DELTA[3]} * 100) / $TOTAL_DELTA" | bc)
printf "IOWait:    %6.2f%%\n" $(echo "scale=2; (${DELTA[4]} * 100) / $TOTAL_DELTA" | bc)
printf "IRQ:       %6.2f%%\n" $(echo "scale=2; (${DELTA[5]} * 100) / $TOTAL_DELTA" | bc)
printf "SoftIRQ:   %6.2f%%\n" $(echo "scale=2; (${DELTA[6]} * 100) / $TOTAL_DELTA" | bc)
printf "Steal:     %6.2f%%\n" $(echo "scale=2; (${DELTA[7]} * 100) / $TOTAL_DELTA" | bc)
printf "Guest:     %6.2f%%\n" $(echo "scale=2; (${DELTA[8]} * 100) / $TOTAL_DELTA" | bc)
printf "GuestNice: %6.2f%%\n" $(echo "scale=2; (${DELTA[9]} * 100) / $TOTAL_DELTA" | bc)
echo "================================"
printf "Total CPU Usage: %6.2f%%\n" $(echo "scale=2; (${DELTA[11]} * 100) / $TOTAL_DELTA" | bc)
```

## Per-CPU Analysis

Individual CPU cores can have vastly different statistics:

```bash
# Show per-CPU breakdown
for cpu in $(grep '^cpu[0-9]' /proc/stat | cut -d' ' -f1); do
    echo "=== $cpu ==="
    grep "^$cpu " /proc/stat | awk '{
        total = $2+$3+$4+$5+$6+$7+$8+$9
        printf "User: %.1f%%, System: %.1f%%, Idle: %.1f%%, IOWait: %.1f%%\n",
            ($2*100)/total, ($4*100)/total, ($5*100)/total, ($6*100)/total
    }'
done
```

## Kernel Implementation Details

### Tick Accounting

The kernel updates `/proc/stat` during periodic timer interrupts (tick handler):

```c
// Simplified tick handler flow
void update_process_times(int user_tick) {
    struct task_struct *p = current;
    u64 cputime = cputime_one_jiffy;
    
    if (user_tick) {
        account_user_time(p, cputime);
    } else {
        account_system_time(p, cputime);
    }
    
    // Update per-CPU stats
    account_cputime_index(cpu, index, cputime);
}
```

### Counter Precision

- **Jiffy resolution:** Typically 1ms (HZ=1000) or 10ms (HZ=100)
- **Cumulative counters:** Never reset, wrap around after ~497 days (32-bit) or effectively never (64-bit)
- **Atomic updates:** Counters updated atomically to prevent race conditions
- **Per-CPU storage:** Reduces cache line contention

### Reading /proc/stat Safely

The kernel provides a consistent snapshot:

```c
// Kernel side: seq_file interface ensures atomic reads
static int stat_show(struct seq_file *p, void *v) {
    // Lockless read of per-CPU counters
    for_each_possible_cpu(cpu) {
        sum_cpu_stats(cpu, &stats);
    }
    seq_printf(p, "cpu %llu %llu ...\n", stats.user, stats.nice, ...);
}
```

## Advanced Use Cases

### 1. Real-time CPU Monitoring Script

```bash
#!/bin/bash
# Continuous CPU statistics monitor

HZ=$(getconf CLK_TCK)
INTERVAL=1

while true; do
    clear
    echo "CPU Statistics (refreshing every ${INTERVAL}s)"
    echo "=============================================="
    
    # Aggregate all CPUs
    S1=$(grep '^cpu ' /proc/stat)
    sleep $INTERVAL
    S2=$(grep '^cpu ' /proc/stat)
    
    # Parse and calculate
    read -r u1 n1 s1 i1 w1 x1 y1 z1 st1 g1 gn1 <<< $(echo $S1 | awk '{print $2,$3,$4,$5,$6,$7,$8,$9,$10,$11}')
    read -r u2 n2 s2 i2 w2 x2 y2 z2 st2 g2 gn2 <<< $(echo $S2 | awk '{print $2,$3,$4,$5,$6,$7,$8,$9,$10,$11}')
    
    total=$(( (u2+n2+s2+i2+w2+x2+y2+z2) - (u1+n1+s1+i1+w1+x1+y1+z1) ))
    
    if [ $total -gt 0 ]; then
        printf "User:    %5.1f%%  [%8d jiffies]\n" $(echo "scale=1; (($u2-$u1)*100)/$total" | bc) $((u2-u1))
        printf "Nice:    %5.1f%%  [%8d jiffies]\n" $(echo "scale=1; (($n2-$n1)*100)/$total" | bc) $((n2-n1))
        printf "System:  %5.1f%%  [%8d jiffies]\n" $(echo "scale=1; (($s2-$s1)*100)/$total" | bc) $((s2-s1))
        printf "Idle:    %5.1f%%  [%8d jiffies]\n" $(echo "scale=1; (($i2-$i1)*100)/$total" | bc) $((i2-i1))
        printf "IOWait:  %5.1f%%  [%8d jiffies]\n" $(echo "scale=1; (($w2-$w1)*100)/$total" | bc) $((w2-w1))
        printf "IRQ:     %5.1f%%  [%8d jiffies]\n" $(echo "scale=1; (($x2-$x1)*100)/$total" | bc) $((x2-x1))
        printf "SoftIRQ: %5.1f%%  [%8d jiffies]\n" $(echo "scale=1; (($y2-$y1)*100)/$total" | bc) $((y2-y1))
        printf "Steal:   %5.1f%%  [%8d jiffies]\n" $(echo "scale=1; (($z2-$z1)*100)/$total" | bc) $((z2-z1))
        printf "Guest:   %5.1f%%  [%8d jiffies]\n" $(echo "scale=1; (($g2-$g1)*100)/$total" | bc) $((g2-g1))
    fi
    
    echo "=============================================="
    echo "Press Ctrl+C to exit"
    sleep $((INTERVAL - 1))
done
```

### 2. Historical CPU Statistics Collection

```bash
#!/bin/bash
# Collect CPU stats over time for analysis

LOG_FILE="/tmp/cpu_stats.log"
INTERVAL=5
DURATION=3600  # 1 hour

echo "timestamp,user,nice,system,idle,iowait,irq,softirq,steal,guest,guest_nice" > $LOG_FILE

START=$(date +%s)
while [ $(($(date +%s) - START)) -lt $DURATION ]; do
    TIMESTAMP=$(date +%s)
    STATS=$(grep '^cpu ' /proc/stat | awk '{print $2","$3","$4","$5","$6","$7","$8","$9","$10","$11}')
    echo "$TIMESTAMP,$STATS" >> $LOG_FILE
    sleep $INTERVAL
done

# Analyze collected data
echo "Analysis complete. Log: $LOG_FILE"
```

### 3. Detect CPU Anomalies

```bash
#!/bin/bash
# Alert on unusual CPU statistics

THRESHOLD_IOWAIT=20  # Alert if iowait > 20%
THRESHOLD_STEAL=10   # Alert if steal > 10%
THRESHOLD_SOFTIRQ=15 # Alert if softirq > 15%

check_cpu_stats() {
    S1=$(grep '^cpu ' /proc/stat)
    sleep 1
    S2=$(grep '^cpu ' /proc/stat)
    
    read -r u1 n1 s1 i1 w1 x1 y1 z1 st1 g1 gn1 <<< $(echo $S1 | awk '{print $2,$3,$4,$5,$6,$7,$8,$9,$10,$11}')
    read -r u2 n2 s2 i2 w2 x2 y2 z2 st2 g2 gn2 <<< $(echo $S2 | awk '{print $2,$3,$4,$5,$6,$7,$8,$9,$10,$11}')
    
    total=$(( (u2+n2+s2+i2+w2+x2+y2+z2) - (u1+n1+s1+i1+w1+x1+y1+z1) ))
    
    if [ $total -gt 0 ]; then
        iowait_pct=$(echo "scale=1; (($w2-$w1)*100)/$total" | bc)
        steal_pct=$(echo "scale=1; (($z2-$z1)*100)/$total" | bc)
        softirq_pct=$(echo "scale=1; (($y2-$y1)*100)/$total" | bc)
        
        if (( $(echo "$iowait_pct > $THRESHOLD_IOWAIT" | bc -l) )); then
            echo "WARNING: High IOWait detected: ${iowait_pct}%"
        fi
        
        if (( $(echo "$steal_pct > $THRESHOLD_STEAL" | bc -l) )); then
            echo "WARNING: High steal time detected: ${steal_pct}% (possible host overcommit)"
        fi
        
        if (( $(echo "$softirq_pct > $THRESHOLD_SOFTIRQ" | bc -l) )); then
            echo "WARNING: High softirq time detected: ${softirq_pct}% (possible interrupt storm)"
        fi
    fi
}

check_cpu_stats
```

## Relationship to Other Kernel Interfaces

### /proc/loadavg

Load average is related but distinct:
- Load average = runnable tasks + uninterruptible sleep (I/O wait)
- CPU statistics = actual CPU time spent in each state
- High load + low CPU usage = I/O bottleneck

### /proc/uptime

System uptime used to calculate rates:
```bash
uptime_seconds=$(awk '{print $1}' /proc/uptime)
cpu_jiffies=$(grep '^cpu ' /proc/stat | awk '{sum=0; for(i=2;i<=NF;i++) sum+=$i; print sum}')
HZ=$(getconf CLK_TCK)
cpu_time=$((cpu_jiffies / HZ))
echo "CPU time accumulated: ${cpu_time}s over ${uptime_seconds}s uptime"
```

### top/htop Internals

Tools like `top` read `/proc/stat` and `/proc/[pid]/stat` to compute CPU percentages:
```c
// Simplified top calculation
cpu_percent = (delta_user + delta_system) / delta_total * 100
```

## Troubleshooting Scenarios

### Scenario 1: High System Time

**Symptoms:** `system` time > 30%, slow application response

**Investigation:**
```bash
# Check syscall rate
strace -c -p $(pgrep -f <application>) 2>&1 | head -20

# Check context switch rate
vmstat 1 5

# Check kernel function profiling (requires perf)
perf top -g -p $(pgrep -f <application>)
```

**Common causes:**
- Frequent page faults
- Excessive system calls
- Kernel lock contention
- Driver issues

### Scenario 2: High IOWait

**Symptoms:** `iowait` > 20%, system feels sluggish

**Investigation:**
```bash
# Check I/O wait queue depth
iostat -x 1 5

# Check block device statistics
cat /proc/diskstats

# Identify I/O-intensive processes
iotop -o

# Check filesystem I/O
df -h
mount | grep -E '(noatime|nodiratime)'
```

**Common causes:**
- Slow storage (HDD vs SSD)
- Saturated I/O bandwidth
- Swapping (check `/proc/meminfo`)
- Network filesystem latency

### Scenario 3: High SoftIRQ Time

**Symptoms:** `softirq` > 15%, network performance issues

**Investigation:**
```bash
# Break down softirq types
watch -n 1 'cat /proc/softirq'

# Check network receive drops
cat /proc/net/softnet_stat

# Monitor network interrupts
cat /proc/interrupts | grep -i eth

# Check for receive livelock
ethtool -S <interface> | grep -i drop
```

**Common causes:**
- High network packet rate
- Small receive buffers
- Interrupt coalescing misconfiguration
- Network driver issues

### Scenario 4: High Steal Time in VM

**Symptoms:** `steal` > 10%, VM performance degradation

**Investigation:**
```bash
# Confirm virtualization
systemd-detect-virt

# Check CPU pinning
virsh vcpuinfo <domain>

# Monitor from host perspective
virsh domstats <domain> --cpu

# Check for CPU overcommit
# (On host) virsh nodeinfo
```

**Solutions:**
- Migrate VM to less loaded host
- Increase host CPU capacity
- Use CPU pinning/affinity
- Enable CPU features (virtio, paravirtualization)

## References and Further Reading

- **Kernel Documentation:** `Documentation/filesystems/proc.rst` (CPU statistics section)
- **Kernel Source:** `kernel/sched/cputime.c` (CPU time accounting implementation)
- **man proc(5):** `/proc/stat` format documentation
- **Linux Performance and Tuning Guide:** CPU accounting and analysis
- **Understanding the Linux Kernel (3rd ed.):** Chapter 4 (Interrupts and Exceptions), Chapter 7 (Kernel Synchronization)

---

<i class="fas fa-info-circle" aria-hidden="true"></i> **Note:** All statistics are cumulative since boot. To calculate rates or percentages, always sample at two points in time and compute deltas. The kernel tick rate (HZ) determines counter resolution and can be queried via `getconf CLK_TCK`.

