---
name: optimize-linux-cpp-performance
description: Diagnose, benchmark, and optimize C++ applications and services running on Linux across compiler and algorithm choices, CPU scheduling and cache behavior, memory allocation and NUMA, storage I/O, networking, concurrency, containers, and kernel limits. Use when Codex must investigate high CPU, latency, low throughput, context switches, memory growth, page faults, swap, slow disk I/O, packet loss, softirq load, C10K/C10M scalability, or produce a measurable Linux C++ performance plan.
---

# Optimize Linux C++ Performance

## Operating principles

Optimize from evidence, not folklore. Preserve correctness, durability, security, and operability. Treat every compiler flag, allocator choice, kernel parameter, affinity rule, offload feature, and scheduler change as a workload-specific hypothesis.

Measure both throughput and latency distribution. Do not hide p95/p99/p999 regressions behind a better average. Run the same workload, dataset, warm-up, CPU frequency policy, machine placement, and concurrency before and after each change.

## Workflow

1. Define the service-level target: throughput, p99 latency, CPU/request, memory/request, startup time, IOPS, bandwidth, or maximum concurrent connections.
2. Record the environment: kernel, CPU topology, NUMA layout, memory, storage, NIC queues/offloads, compiler, standard library, allocator, build flags, container limits, and dependency versions.
3. Establish a release-build baseline with symbols. Capture workload parameters and at least three stable runs.
4. Apply RED to the service (rate, errors, duration) and USE to each resource (utilization, saturation, errors).
5. Classify the leading constraint as CPU, scheduling/locking, memory, file I/O, network, dependency, or capacity/architecture.
6. Collect the narrowest evidence that can confirm the hypothesis. Prefer sampling (`perf`) over intrusive tracing; use eBPF, tracepoints, or `strace` only when needed.
7. Change one primary variable. Record the exact code/config diff and its expected causal chain.
8. Re-run the baseline and correctness tests. Report effect size, variance, tail latency, resource movement, and any tradeoff.
9. Keep only repeatable wins. Add production monitoring and rollback criteria.

## Route to the right reference

- Read [cpp-code-and-build.md](references/cpp-code-and-build.md) for algorithms, data layout, compiler/linker optimization, allocation, synchronization, concurrency, caching, and system-call reduction.
- Read [linux-runtime.md](references/linux-runtime.md) for CPU affinity, scheduling, NUMA, huge pages, swap, cgroups, OOM behavior, containers, and runtime isolation.
- Read [io-network.md](references/io-network.md) for page cache, filesystems, block I/O, asynchronous I/O, `mmap`, epoll, socket settings, queues, offloads, C10K/C10M, NAT, and packet-loss handling.
- Read [profiling.md](references/profiling.md) for the symptom-to-tool workflow, commands, benchmark design, and interpretation.
- Read [source-index.md](references/source-index.md) when tracing a recommendation back to the 41 source articles or checking modernization notes.

## Required decision discipline

### CPU-bound

Confirm with `perf stat`, `perf record`, flame graphs, per-core utilization, run-queue length, IPC, cache misses, branch misses, and context switches. Optimize the dominant code path or data movement before adding threads. Separate useful parallelism from oversubscription.

### Memory-bound

Distinguish footprint, leak, allocator contention, fragmentation, page faults, cache/TLB misses, remote NUMA access, reclaim, and swap. Prefer fewer allocations, reuse, locality, and bounded caches. Test huge pages; never assume they help.

### I/O-bound

Distinguish filesystem cache behavior from actual device I/O. Measure IOPS, bandwidth, queue depth, utilization, and latency. Batch, coalesce, cache, append, or use asynchronous I/O only when semantics permit. Do not trade away required durability.

### Network-bound

Measure at application, transport, IP, and NIC layers. Separate connection setup, serialization, syscalls/copies, socket queueing, retransmission, packet drops, softirq pressure, and peer latency. Tune buffers or backlogs only after observing saturation or drops.

## Guardrails

- Do not copy sysctl values across hosts. First inspect current kernel documentation and defaults.
- Do not use `drop_caches` as a production optimization. Use it only for controlled cold-cache experiments after `sync`, with explicit risk acknowledgement.
- Do not disable swap, THP, ASLR, security checks, journaling, or durability globally without workload proof and operational approval.
- Use `oom_score_adj`, not deprecated `oom_adj`. Protecting one process can cause a worse victim selection elsewhere.
- Modern block devices use blk-mq schedulers such as `none`, `mq-deadline`, `bfq`, or `kyber`; do not prescribe legacy `noop`/`deadline` names blindly.
- `tcp_fin_timeout` governs orphaned `FIN_WAIT_2`, not general `TIME_WAIT`. `tcp_tw_reuse` is kernel- and topology-sensitive.
- SYN cookies are an overload fallback, not a substitute for capacity. Validate TCP-option and CPU tradeoffs.
- `TCP_NODELAY` and `TCP_CORK` express opposite batching goals; select per message pattern. `TCP_QUICKACK` is not a permanent mode.
- Treat DPDK/XDP, busy polling, real-time priorities, CPU isolation, and IRQ pinning as advanced designs requiring dedicated cores and rollback plans.

## Deliverable format

Return:

1. target and baseline;
2. evidence and bottleneck classification;
3. ranked hypotheses;
4. minimal proposed changes with tradeoffs;
5. exact validation experiment;
6. before/after table including tail latency and errors;
7. production rollout, monitoring, and rollback criteria.

