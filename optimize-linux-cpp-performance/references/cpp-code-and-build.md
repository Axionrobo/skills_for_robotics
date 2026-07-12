# C++ code and build optimization

## Contents

- Build and compiler
- Algorithms and data layout
- Allocation and memory access
- Concurrency and scheduling
- Caching, batching, and system calls
- Architecture
- Review checklist

## Build and compiler

- Benchmark an optimized release build; keep debug symbols in a separate file or unstripped benchmark binary.
- Start with `-O2` or `-O3` only after checking numerical and code-size behavior. Compare generated code and benchmark results.
- Use architecture targeting (`-march=`/`-mtune=`) only when deployment CPU compatibility is controlled.
- Test LTO and profile-guided optimization. PGO must use a representative workload; stale or biased profiles can regress uncommon paths.
- Enable section garbage collection or reduce template/code duplication when instruction-cache pressure or binary size is material.
- Retain usable stacks for profiling (`-g`; consider frame pointers when unwinding quality matters). Measure the frame-pointer cost on the target architecture.
- Turn warnings and sanitizers on in correctness builds, but do not compare their performance against the optimized production build.
- Inspect vectorization reports and hot-loop assembly. Confirm alignment, aliasing, trip counts, and reductions permit SIMD; do not force vectorization that increases shuffles or code size.

## Algorithms and data layout

- Improve algorithmic complexity before micro-optimizing instructions.
- Remove repeated parsing, conversion, sorting, hashing, serialization, and allocation from hot paths.
- Choose containers by access pattern and memory behavior, not only asymptotic complexity. Contiguous storage often wins through locality.
- Prefer compact structures and hot/cold field separation. Consider structure-of-arrays for vectorizable scans and array-of-structures for whole-object access.
- Reduce pointer chasing, indirection, virtual dispatch, and polymorphic allocation in tight loops when profiles show front-end or cache stalls.
- Reserve container capacity; avoid accidental copies; move only when it actually avoids expensive ownership transfer.
- Pass non-owning views (`span`, `string_view`) when lifetimes are safe. Avoid turning lifetime bugs into performance wins.
- Hoist invariants, precompute stable values, and cache only data whose invalidation is correct and bounded.
- Replace unpredictable branches with better data organization or branchless code only after measuring branch misses; branchless work can execute more instructions.
- Batch work to amortize locks, syscalls, timestamps, logging, serialization, and queue operations.

## Allocation and memory access

- Reduce dynamic allocations on the hot path. Reuse objects, arenas, pools, freelists, or monotonic resources when ownership and reset points are clear.
- Compare allocators under the real thread count and allocation-size distribution. Watch retained memory, fragmentation, and tail latency, not only allocation throughput.
- Size pools and caches with hard bounds and pressure behavior. Unbounded caches merely exchange latency for eventual memory failure.
- Pre-fault or warm latency-critical memory only when startup cost and resident footprint are acceptable.
- Align data for SIMD and cache lines when required. Avoid false sharing by separating frequently written per-thread/per-core fields; verify with cache-coherency counters.
- Place compute threads and their memory on the same NUMA node. Use first-touch intentionally; validate with `numastat` and cross-node bandwidth/latency.
- Test THP or explicit huge pages for large, long-lived working sets with TLB pressure. Measure page-fault stalls, compaction, wasted memory, and fragmentation.
- Understand allocator behavior: large allocations may use `mmap`; frequent map/unmap can add page faults and kernel work, while retained heap arenas can increase fragmentation.
- Diagnose leaks with allocation stacks and heap profilers. RSS growth alone can be allocator retention or page-cache effects, not necessarily a leak.

## Concurrency and scheduling

- Use threads to share an address space and reduce process-switch cost when isolation requirements permit, but do not assume more threads mean more throughput.
- Bound worker counts near the measured parallel capacity. Oversubscription increases run-queue delay, cache eviction, and context switches.
- Prefer asynchronous/event-driven handling for many mostly idle connections. Use `epoll` and nonblocking I/O; keep per-connection work bounded.
- Use sharding, per-thread state, read-copy-update patterns, or batched queues to reduce lock contention. Measure contention and queue delay before replacing locks.
- Keep critical sections short and avoid blocking I/O while holding a lock.
- Choose atomics and memory order from correctness requirements. Weaker ordering may help but must be justified by the synchronization design.
- Avoid hot global counters; use per-thread/per-CPU aggregation when exact instantaneous values are unnecessary.
- Use connection pools and resource pools to amortize setup, but bound them and account for peer limits.
- Handle child processes correctly; unreaped children become zombies. Avoid spawning short-lived external commands in hot paths.
- Separate CPU-bound and blocking work pools to prevent I/O stalls from consuming compute capacity.

## Caching, batching, and system calls

- Use application caches, page cache, or external caches to reduce expensive computation and I/O. Track hit rate, eviction, staleness, and memory cost.
- Prefer long-lived connections over repeated TCP/TLS setup when peer and failure semantics allow.
- Cache DNS responses according to TTL; consider prefetching or asynchronous resolution for latency-sensitive paths.
- Reduce system calls and copies with batching, vectored I/O, `sendfile`/zero-copy paths, or `mmap` where semantics and lifetime are appropriate.
- Use larger I/O operations to amortize overhead, but cap batch size to protect tail latency and fairness.
- Make logging asynchronous or sampled under load; preserve error visibility and crash diagnostics.
- For durable writes, coalesce records and call `fsync` at a deliberate commit boundary. Never remove required sync semantics merely to improve a benchmark.
- Append instead of random-write where the storage format permits; add compaction or indexing deliberately.

## Architecture

- Offload slow, retryable work through bounded queues and backpressure.
- Use load balancing and horizontal partitioning when one host has reached verified resource capacity.
- Place static content behind a CDN and cache stable dependency results.
- Instrument dependency call rate, error rate, and latency; a remote bottleneck cannot be fixed by local micro-optimization.
- For extreme packet rates, evaluate XDP or DPDK only after normal socket/NIC tuning is demonstrably insufficient.

## Review checklist

- Is the hot path known from a profile?
- Is the workload representative and warmed appropriately?
- Are correctness and durability unchanged?
- Did instructions/request, cycles/request, allocations/request, syscalls/request, and bytes/request improve?
- Did p99/p999 latency, memory high-water mark, or error rate regress?
- Does the result survive multiple runs and production-like concurrency?

