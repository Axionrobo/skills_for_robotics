# Profiling and benchmark workflow

## Contents

- Baseline contract
- Symptom-to-tool map
- CPU workflow
- Memory workflow
- I/O workflow
- Network workflow
- Dynamic tracing
- Experiment report

## Baseline contract

Record:

- exact binary, build flags, compiler, allocator, config, kernel, and hardware;
- workload generator, concurrency, payload/data distribution, duration, and warm-up;
- affinity, cgroup/container limits, frequency/thermal state, and colocated workloads;
- requests/s, errors, latency percentiles, CPU time, memory high-water mark, I/O, and network;
- run-to-run variance across at least three stable samples.

Do not benchmark on a debug/sanitized build unless that build is the target. Do not compare a warm-cache run with a cold-cache run.

## Symptom-to-tool map

| Symptom | First checks | Narrowing tools |
|---|---|---|
| High CPU | `top`, `mpstat`, `pidstat` | `perf stat`, `perf record/report`, flame graph |
| High load, moderate CPU | `vmstat`, task states, PSI | `pidstat -w/-d`, `iostat`, blocked-stack tracing |
| Context switches | `vmstat`, `pidstat -w -t` | lock/futex profiling, scheduler tracepoints |
| Memory growth | `free`, PSS/USS, cgroup events | heap profiler, BCC `memleak`, allocation flame graph |
| Page faults/reclaim | `pidstat -r`, `vmstat`, PSI | `perf`, mm tracepoints, `numastat` |
| Slow disk I/O | `iostat -xz`, `pidstat -d` | `filetop`, `opensnoop`, `blktrace`, eBPF |
| Network latency | `ping`/`hping3`, `ss`, `sar` | `traceroute`, `tcpdump`, Wireshark, syscall tracing |
| Packet loss/softirq | link/NIC stats, `/proc/softirqs` | `ethtool -S`, IRQ mapping, drop tracing |
| Short-lived processes | process tree, logs | `execsnoop` |
| Container issue | cgroup metrics, container logs | host PID profiling, `nsenter`, `sysdig` |

## CPU workflow

1. Confirm which cores and threads consume CPU and whether steal/throttling exists.
2. Check run queue, voluntary/involuntary switches, migrations, and interrupts.
3. Use `perf stat` for cycles, instructions, IPC, branch misses, cache misses, faults, switches, and migrations.
4. Sample call stacks with `perf record -g -p PID -- sleep 30`; use a flame graph if aggregation helps.
5. Map the dominant stack to algorithm, data layout, lock, allocator, syscall, or kernel path.
6. Re-measure cycles/request and instructions/request after the change.

Interpretation:

- low IPC plus cache misses suggests memory/locality pressure;
- high branch misses suggests unpredictable control flow;
- high involuntary switches suggests CPU competition;
- high voluntary switches suggests waits, locks, I/O, or explicit yielding;
- high system CPU suggests syscalls, networking, memory management, or kernel work.

## Memory workflow

1. Separate RSS, PSS, USS, mappings, anonymous memory, file cache, slab, and cgroup accounting.
2. Check allocation rate, retained heap, fragmentation, faults, reclaim, swap, and NUMA locality.
3. Use an allocation profiler or BCC `memleak` for outstanding stacks; compare snapshots over time.
4. Use `perf record -e 'kmem:*'` or relevant mm tracepoints only when allocation/reclaim paths need kernel-level evidence.
5. Validate footprint, allocation throughput, pause/tail latency, and OOM headroom.

## I/O workflow

1. Use `iostat -xz 1` to identify device utilization, queueing, throughput, and latency.
2. Use `pidstat -d 1` to identify the process.
3. Use file and syscall tracing to find the file/table/path and access pattern.
4. Reproduce safely with `fio` using the same block size, mix, direct/buffered behavior, depth, and concurrency.
5. Test batching, cache, async I/O, layout, scheduler, or hardware changes one at a time.

## Network workflow

1. Compare single-request and concurrent latency with an application-level load generator.
2. Measure RTT and route; distinguish local processing from remote/network delay.
3. Inspect connection states, queues, retransmits, drops, socket overflow, and NIC stats.
4. Inspect packets when protocol behavior is unclear.
5. Inspect application socket syscalls and event-loop delay.
6. Benchmark TCP/UDP with `iperf`/`netperf`, HTTP with `wrk`/`ab`, and PPS with a suitable packet generator only in an isolated environment.

## Dynamic tracing

- Prefer `perf` for broad sampling and hardware/software counters.
- Use ftrace/trace-cmd for kernel function and function-graph tracing.
- Use eBPF/BCC for targeted histograms, latency distributions, short-lived processes, file activity, allocations, or drops.
- Use uprobes/USDT for selected userspace functions when code modification is undesirable.
- Use `strace` sparingly: ptrace stops and user/kernel transitions can perturb a busy process. Filter calls and bound duration.
- Use SystemTap when its probe model fits and compatible debug data is available.
- Use `sysdig` for cross-container system-call and resource views when appropriate.

## Experiment report

| Field | Baseline | Candidate | Delta |
|---|---:|---:|---:|
| Throughput | | | |
| p50/p95/p99/p999 | | | |
| Error rate | | | |
| CPU/request | | | |
| Instructions/request | | | |
| Context switches/request | | | |
| Allocations or bytes/request | | | |
| RSS/PSS high-water mark | | | |
| Disk I/O/request | | | |
| Network bytes/packets/request | | | |

State confidence, number of runs, variance, side effects, rollback threshold, and which production metric confirms the win.

