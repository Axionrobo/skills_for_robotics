# Storage I/O and network optimization

## Contents

- File and block I/O
- I/O optimization methods
- Network application layer
- TCP/UDP and connection scale
- NIC and packet processing
- NAT, loss, and extreme packet rates

## File and block I/O

Measure device utilization, queueing, IOPS, throughput, and latency. Random small I/O is usually IOPS-sensitive; sequential large I/O is usually bandwidth-sensitive.

```bash
iostat -xz 1
pidstat -d -p PID 1
fio --name=case --filename=/path/testfile --rw=randread --bs=4k --ioengine=io_uring --iodepth=32 --direct=1 --runtime=60 --time_based --group_reporting
```

- Use `fio` only on a safe test file/device. Match read/write mix, block size, queue depth, direct/buffered mode, concurrency, and dataset size to production.
- Separate submission latency, completion latency, and end-to-end application latency.
- Use `filetop`, `opensnoop`, `lsof`, tracepoints/eBPF, or carefully scoped `strace` to map device pressure back to files and call sites.
- Inspect filesystem capacity, inode usage, page/dentry/inode caches, dirty pages, and writeback.

## I/O optimization methods

- Replace scattered random writes with append/sequential writes when the format permits.
- Coalesce small reads/writes and durable commits. Preserve transaction boundaries and crash guarantees.
- Use buffered I/O to exploit page cache when reuse exists; use direct I/O only when double caching or application-managed alignment/queueing is justified.
- Use `mmap` for repeated access to mapped regions when its faulting, lifetime, truncation, and consistency semantics are acceptable.
- Use asynchronous I/O (`io_uring`, native AIO where appropriate) to overlap work; set a bounded queue depth to avoid latency inflation.
- Tune read-ahead for demonstrated sequential patterns. Excess read-ahead wastes bandwidth and cache on random workloads.
- Increase block request queue depth only when the device can benefit and latency remains acceptable.
- Select from the schedulers exposed by `/sys/block/DEV/queue/scheduler`; compare `none`, `mq-deadline`, `bfq`, or `kyber` on the actual device/workload.
- Use I/O priorities only with a scheduler that supports them; real-time I/O can starve the host.
- Use SSD/NVMe, RAID, sharding, or separate devices when the workload exceeds physical capacity. Include failure and rebuild behavior.
- Use tmpfs only for disposable/reconstructable data and account for memory pressure.

## Network application layer

- Reuse connections to amortize TCP/TLS handshakes; bound pools and handle stale connections.
- Cache stable data and DNS responses. Honor invalidation and TTL.
- Reduce payload with a compact serialization format such as Protocol Buffers, but measure CPU-versus-bandwidth tradeoffs.
- Batch messages and use vectored or zero-copy I/O when it reduces syscalls/copies without harming latency.
- Use nonblocking sockets plus `epoll` for many concurrent mostly idle connections. Avoid one thread per connection at large scale.
- Prevent accept wakeup contention with `EPOLLEXCLUSIVE`, `SO_REUSEPORT`, or a deliberate accept-dispatch design; measure load distribution.
- Apply backpressure and bounded queues. An unbounded queue converts overload into memory growth and tail latency.

## TCP/UDP and connection scale

- Choose `TCP_NODELAY` for latency-sensitive small messages or `TCP_CORK`/application batching for packet aggregation. Do not enable both without a precise design.
- Size per-socket `SO_SNDBUF`/`SO_RCVBUF` and system maxima from bandwidth-delay product and observed queue drops; larger buffers also consume more memory and can increase queueing delay.
- Raise file-descriptor limits, ephemeral-port range, listen backlog, SYN backlog, socket memory, and conntrack capacity only when the corresponding resource is exhausted.
- Read listen queues with `ss -ltn`; distinguish half-open SYN backlog from accepted/full connection queue.
- Use keepalive settings to discover dead peers according to application recovery needs; shorter probes cost network and CPU.
- Treat SYN cookies as an overload fallback. Size capacity and rate-limit abusive sources separately.
- Evaluate TCP Fast Open only with client/server/proxy compatibility and replay/idempotency considerations.
- For UDP, choose payload size below path MTU to avoid fragmentation; implement loss, ordering, and congestion behavior at the application layer.
- Use `SO_REUSEPORT` for kernel distribution across listeners when flow hashing and deployment topology are suitable.
- Never tune `tcp_fin_timeout` as a generic TIME_WAIT fix. Diagnose connection churn, client/server role, port exhaustion, and kernel semantics first.

## NIC and packet processing

- Observe bandwidth, packets/s, errors, drops, overruns, retransmits, socket overflow, and queue depth.
- Enable and benchmark RSS/multiqueue, IRQ affinity, RPS/RFS, and queue counts. Align application placement with receive processing when cache locality helps.
- Test GRO/LRO for receive aggregation and TSO/GSO/UFO-style transmit offloads as supported by the NIC/kernel. Offloads may complicate packet captures or hurt some latency/forwarding workloads.
- Increase RX/TX ring or qdisc queue sizes only when drops prove they are too small; extra buffering can worsen latency.
- Adjust MTU only end-to-end; inconsistent jumbo-frame paths cause fragmentation or loss.
- Use traffic control for QoS and isolation when multiple traffic classes compete.

Useful observations:

```bash
sar -n DEV,TCP,ETCP 1
ss -s
ss -ltnp
ip -s link show dev eth0
ethtool -S eth0
ethtool -k eth0
ethtool -l eth0
netstat -s
tc -s qdisc show dev eth0
cat /proc/interrupts
cat /proc/softirqs
```

Use `-n`/`-nn` with network tools when name resolution would distort measurements.

## NAT, loss, and extreme packet rates

- For NAT/containers, inspect conntrack count, maximum, bucket sizing, bridge/netfilter overhead, and bandwidth. Increase capacity only with memory and lookup-cost analysis.
- Prefer simpler/stateless forwarding only when connection semantics and security policy permit. `tc`, XDP, or DPDK designs require specialized expertise.
- Localize drops layer by layer: NIC stats and ring overruns; qdisc; IP/transport counters; firewall rules; socket overflow; application queues.
- Use `tcpdump`/Wireshark to validate retransmissions, RTT, windowing, and protocol behavior, while accounting for capture overhead and offloads.
- For very high PPS, first use multiqueue/RSS, batching, affinity, and offloads. Then evaluate XDP (early kernel path) or DPDK (userspace polling, huge pages, dedicated cores, NUMA-aware memory).
- DDoS defense is capacity and security work, not only performance tuning: rate-limit, filter upstream, use SYN protections appropriately, and consider XDP/DPDK or dedicated scrubbing infrastructure.

