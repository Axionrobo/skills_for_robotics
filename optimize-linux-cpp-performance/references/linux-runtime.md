# Linux runtime optimization

## Contents

- CPU and scheduler
- Interrupts and softirqs
- Memory, reclaim, and NUMA
- Resource control and containers
- Operational caveats

## CPU and scheduler

- Inspect load average together with CPU utilization. Load includes runnable tasks and uninterruptible sleep, so high load can mean CPU contention or blocked I/O.
- Use per-CPU metrics. One saturated core can bottleneck a serial stage while aggregate utilization looks low.
- Distinguish voluntary switches (often waiting for locks/I/O/resources) from involuntary switches (CPU competition/time-slice expiry).
- Reduce process/thread count, blocking, wakeups, timer frequency, and migration when context-switch cost is material.
- Bind latency-critical workers to selected CPUs only after proving migrations or cache locality are a problem. Leave capacity for kernel work, IRQs, and housekeeping.
- Use CPU affinity and NUMA placement together. CPU pinning without memory locality can worsen remote-memory traffic.
- Adjust nice or scheduling policy only with starvation analysis. Real-time policy can block essential kernel and operational work.
- Use cgroup CPU controls for isolation, quotas, and weights; verify throttling metrics because quotas can create periodic tail-latency spikes.
- Keep CPU frequency governor, turbo state, thermal state, and virtualization steal time stable during comparisons.

Useful observations:

```bash
uptime
mpstat -P ALL 1
pidstat -u -w -t -p PID 1
vmstat 1
perf stat -p PID -e cycles,instructions,cache-misses,branches,branch-misses,context-switches,cpu-migrations,page-faults
```

## Interrupts and softirqs

- Identify the interrupt type and affected CPU before changing affinity: `/proc/interrupts`, `/proc/softirqs`, `mpstat -P ALL`, and `sar -n DEV`.
- Spread multi-queue NIC IRQs across suitable CPUs or use `irqbalance`; avoid colliding high-rate IRQs with a latency-critical worker unless deliberate cache affinity helps.
- Prefer hardware RSS when available. Use RPS to distribute receive processing in software and RFS to steer flows toward the consuming application CPU.
- Size `rps_sock_flow_entries` and per-queue `rps_flow_cnt` coherently; divide the global flow count among active RX queues.
- Confirm improvement through softirq time, packets/s, drops, migrations, cache misses, and application tail latency.

## Memory, reclaim, and NUMA

- Interpret VIRT, RSS/RES, PSS, USS, shared mappings, anonymous pages, file pages, and allocator retention separately. Sum PSS rather than RSS for a closer system-wide process total.
- Track minor versus major faults. Major faults imply storage involvement; bursts of minor faults can still hurt latency.
- Use bounded pools/arenas to reduce allocation churn. Do not preallocate so much that reclaim or OOM risk rises.
- Use page cache intentionally for file workloads. Monitor hit rate with `cachestat`/`cachetop` when available.
- Diagnose reclaim with `vmstat`, PSI, `/proc/vmstat`, zone watermarks, and `kswapd` activity.
- Keep swap policy workload-specific. Latency-sensitive services often lower swap tendency or reserve memory, but a blanket swap-off can turn pressure into abrupt OOM.
- Set memory cgroup limits with headroom; inspect `memory.events`, `memory.current`, pressure stalls, and OOM kills.
- Adjust `oom_score_adj` only as part of a system-level victim policy.
- Measure THP modes and defragmentation behavior. Consider `madvise(MADV_HUGEPAGE)` for selected large regions rather than globally forcing THP.
- Use explicit HugeTLB pages only when reservation, alignment, deployment, and failure behavior are managed.
- On NUMA systems, inspect topology and placement with `lscpu`, `numactl --hardware`, and `numastat -p PID`; use first-touch, `numactl`, or memory policy deliberately.

Useful observations:

```bash
free -h
vmstat 1
pidstat -r -p PID 1
cat /proc/PID/status
pmap -x PID
smem -p -k
numastat -p PID
cat /proc/pressure/memory
```

## Resource control and containers

- Record cpuset, CPU quota/weight, memory limit, I/O weight/max, PID limit, and file-descriptor limit. Host capacity does not equal container capacity.
- Detect CPU throttling and memory pressure before changing code.
- Use cgroup I/O controls to protect shared storage from one service; validate latency and throughput for every tenant.
- Cache hot container images, preallocate network resources, or reuse warm containers when cold-start latency is the target.
- Profile the host PID or enter the correct namespaces when container tools cannot see the process.
- Use RED for the service and USE for the host/cgroup. Include dependency latency so container symptoms are not mistaken for local resource issues.

## Operational caveats

- Never clear page cache on a production host as a routine fix.
- Do not infer a leak from free-memory decline alone; Linux uses available RAM for reclaimable cache.
- Do not protect all important processes from OOM; the system must retain a viable victim policy.
- Do not pin every thread and IRQ. Reserve housekeeping capacity and document the topology.
- Do not treat a 70%, 80%, or any fixed utilization number as universal. Queueing and SLOs determine the real threshold.

