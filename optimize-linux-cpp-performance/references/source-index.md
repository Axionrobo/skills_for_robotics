# Source coverage and modernization notes

## Source

The skill synthesizes and paraphrases every Markdown article in [zhiyong0804/tech_blog — Linux性能优化实战](https://github.com/zhiyong0804/tech_blog/tree/master/Linux%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96%E5%AE%9E%E6%88%98), retrieved 2026-07-12. It does not copy the articles verbatim.

Current kernel behavior must be checked against [Linux kernel documentation](https://docs.kernel.org/), especially [IP sysctls](https://docs.kernel.org/networking/ip-sysctl.html), [block I/O schedulers](https://docs.kernel.org/block/index.html), [Transparent Huge Pages](https://docs.kernel.org/admin-guide/mm/transhuge.html), and [HugeTLB pages](https://docs.kernel.org/admin-guide/mm/hugetlbpage.html).

## Coverage: CPU, scheduling, and general method

- `CPU优化套路.md`: compiler, algorithms, async work, threads, caches, CPU binding/groups, priorities, limits, NUMA, IRQ balance.
- `CPU使用率.md`: CPU accounting, changing PIDs, crashes/restarts, short-lived processes, exec tracing.
- `快速分析CPU性能瓶颈.md`: utilization, load, context switches, cache hit rate, task states.
- `上下文切换-CPU使用率.md`: process/thread/interrupt switches, TLB, voluntary/involuntary switches, pools, DMA/zero-copy.
- `平均负载.md`: runnable and uninterruptible tasks, CPU versus I/O load, `mpstat`/`pidstat`.
- `软中断.md`: hard/soft interrupt split, `ksoftirqd`, packet-rate observation.
- `Linux内核线程.md`: reclaim, workers, migration, journal/writeback, softirq call paths, flame graphs.
- `Linux性能优化总结.md`: CPU, memory, disk, network, application, and architecture checklist.
- `Linux性能工具总结.md`: CPU, memory, disk, network, and benchmark tool map.
- `Linux性能答疑.md`: reclaim, memory accounting, cache/buffer, memory tools and tuning.
- `Linux问题答疑.md`: blocking/nonblocking and sync/async I/O, dentry/inode cache, database cache behavior.
- `性能问题学习.md`: metrics, targets, baselines, bottleneck analysis, optimization, monitoring.
- `系统监控的思路.md`: USE and RED, process/dependency/internal metrics.
- `内核动态追踪.md`: DTrace/SystemTap/ftrace/perf/eBPF/BCC, probes, strace overhead, sysdig.

## Coverage: memory and storage

- `Linux内存工作机制.md`: mappings, page faults, page tables, large pages, malloc/brk/mmap, reclaim, RSS/PSS/USS, OOM.
- `Linux buffer和cache.md`: buffer/page/slab cache, write coalescing, cache hit tools.
- `Swap内存.md`: direct/kswapd reclaim, watermarks, NUMA reclaim, file/anonymous pages, swappiness.
- `内存泄露实战.md`: heap/mapped leaks, BCC memleak, writeback, allocation flame graphs.
- `不可中断进程和僵尸进程.md`: D/Z states, I/O wait, profiling, parent reaping.
- `Linux IO操作.md`: VFS/block layer, media units, merging/reordering, IOPS/bandwidth/latency.
- `IO磁盘性能优化.md`: fio, latency fields, tracing, append/cache/mmap/batching, cgroups, scheduler/filesystem/device tuning.
- `IO延迟很高.md`: top/iostat/pidstat/strace/filetop/opensnoop localization.
- `系统IO瓶颈.md`: filesystem/device metrics and bottleneck workflow.

## Coverage: network and high concurrency

- `C10K和C10M.md`: event-driven handling, thundering herd, `EPOLLEXCLUSIVE`, `SO_REUSEPORT`, descriptors, conntrack, queues, affinity, RPS/RFS, offload, DPDK/XDP.
- `Linux网络.md`: receive path, DMA/interrupts/sk_buff, bandwidth/throughput/latency/PPS, errors/drops, socket and listen queues.
- `网络性能优化套路.md`: layer-specific targets/tools, long connections, caching/serialization/DNS, socket/TCP/UDP/sysctl, queues, IRQs, steering, offloads, QoS.
- `评估系统的网络性能.md`: pktgen, iperf/netperf, web load generators.
- `网络延迟变大.md`: NODELAY/QUICKACK, request/route/packet/syscall diagnosis.
- `服务器丢包严重.md`: link, qdisc, IP/transport, firewall, and capture checks.
- `网卡中断过高优化.md`: IRQ affinity, RPS/RFS, RSS, queue-to-CPU mapping.
- `优化NAT性能.md`: NAT/conntrack/bridge overhead, table sizing, stateless alternatives.
- `DNS.md`: record types and DNS instability context.
- `应对DDos攻击和TFO.md`: SYN controls/cookies and TCP Fast Open.
- `防止DDos攻击.md`: rate limits, SYN parameters, XDP/DPDK, scrubbing/firewalls.
- `网络性能答疑.md`: ring/sk_buff/socket buffers, ksoftirqd, connection identity/limits, MASQUERADE.
- `tcpdump和wireshark.md`: packet capture and disabling name resolution during measurement.

## Coverage: cases, containers, and dependencies

- `服务吞吐量下降.md`: conntrack, worker limits, listen queues, ephemeral ports, TIME_WAIT reuse.
- `容器性能分析.md`: cold-start phases, CPU/memory flame graphs, debug symbols, RED, BCC.
- `容器应用启动慢.md`: logs and container state inspection.
- `Redis 性能调优.md`: fork memory/overcommit/maxmemory and AOF rewrite context.
- `Redis延迟高.md`: scoped strace/nsenter, process I/O wait and synchronous writes.

## Modernization corrections

- Replace deprecated `oom_adj` guidance with `oom_score_adj` and a system-wide OOM plan.
- Replace legacy `noop`/`deadline` prescriptions with the schedulers actually exposed by blk-mq on the target host.
- Treat swap-off, THP, huge pages, allocator thresholds, cache dropping, affinity, buffer sizing, and fixed utilization thresholds as experiments, not universal rules.
- Do not describe `tcp_fin_timeout` as a TIME_WAIT control; current kernel documentation defines it for orphaned FIN_WAIT_2.
- Do not present SYN cookies as mutually exclusive with backlog sizing or as ordinary capacity scaling; they are a fallback under overload.
- Do not promise that `TCP_QUICKACK` remains enabled indefinitely.
- Verify current kernel semantics for `tcp_tw_reuse`, offloads, conntrack, cgroups, and network queue parameters before applying changes.
- Distinguish lowering CPU usage from increasing throughput; batching, polling, compression, caching, and offload can move costs and alter latency.
