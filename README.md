# parallel-swapper
parallel-swapper is a patch designed to improve the Linux kernel's memory reclamation mechanism, enhancing system responsiveness and throughput under high memory pressure. It achieves this by alleviating the burden on the existing kswapd daemon and enabling asynchronous, parallel swapout processing.
This project builds upon and significantly extends the concepts introduced by the **kcompressd** patch by Qun-Wei Lin from MediaTek.

## The Problem
In the current Linux kernel, the kswapd daemon is responsible for both scanning LRU pages and handling memory compression/swapout tasks (e.g., for ZSWAP/ZRAM). This centralized responsibility in a single thread can lead to significant performance bottlenecks, particularly under high memory pressure. When kswapd gets blocked by compression or I/O operations, the entire memory reclamation process stalls, leading to increased page allocation stalls and a noticeable degradation in system responsiveness.

## The Solution
parallel-swapper introduces a dedicated mechanism to asynchronously and in parallel offload compression and swapout operations from kswapd. This allows kswapd to focus on its primary task of LRU scanning and memory reclamation decisions, leading to a substantial improvement in overall system responsiveness.

## Key Improvements
While parallel-swapper inherits the fundamental idea of offloading from kcompressd, it introduces significant enhancements in the following key areas:

*   **True Parallel Processing**
Unlike kcompressd which employed a single-threaded approach with one dedicated thread per NUMA node, parallel-swapper introduces a dedicated workqueues for each NUMA node.  
These workqueues can execute multiple workers in parallel, achieving true parallelization of compression and write operations within a single NUMA node. This design may potentially lead to a significant increase in throughput.

*   **Dynamic Parallelism Adjustment and Flexible Tuning**
parallel-swapper introduces sysctl interfaces (/proc/sys/vm/parallel_swap, /proc/sys/vm/parallel_swap_factor).  
These interfaces allow for runtime adjustment of the maximum number of workers and the maximum depth of the FIFO queue. This flexibility enables optimal performance tuning tailored to specific system characteristics and workloads.

*   **Advanced FIFO Queue Management with Order Preservation**
kcompressd could lead to out-of-order processing when its FIFO overflowed (new requests were handled synchronously while older requests remained in the queue).  
parallel-swapper implements a sophisticated mechanism: when the queue is nearing full, it synchronously processes the oldest folio currently in the queue, before enqueuing the new folio.  
This mechanism tries to ensure that requests submitted earlier are always processed first, preventing unnecessary latency and the stagnation of old data, thereby maintaining the freshness and fairness of memory reclamation.

*   **Thundering Herd Problem Avoidance**
parallel-swapper cleverly avoids the "Thundering Herd Problem" that could arise from simple parallelization (where numerous workers simultaneously contend for a resource). This is achieved through intelligent worker management by the workqueue, the introduction of a Mempool, and the cooperative locking strategy mentioned above. It minimizes unnecessary worker awakenings and resource contention, ensuring efficient parallel execution.

*   **kswapd-Prioritized Locking Strategy**
Workers detect when kswapd is waiting to acquire the shared mutex and explicitly yield their lock to kswapd.  
This minimizes kswapd's blocking time, allowing it to swiftly fulfill its role as the "command center" of memory reclamation. This is critical for maintaining overall system responsiveness.

*   **Efficient Resource Management**
parallel-swapper utilizes a mempool for allocating work_struct objects that are submitted to the workqueue. This minimizes the overhead and latency of dynamic memory allocations during high-pressure scenarios, enhancing robustness.

## Sysctl Tunables
Swapout parallelism can be tuned using sysctl.
- `vm.sysctl_parallel_swap` (range: 0-INT_MAX, default: 1)
Sets the maximum number of parallel swapout workers.
- `vm.sysctl_parallel_swap_factor` (range: 1-INT_MAX, default: 6)
Sets the factor multiplied by the number of workers to determine the maximum FIFO queue depth.

### Example:
To configure 2 workers with a maximum queue depth (e.g., 2 * 6 * X, considering SWAP_ASYNC_FIFO_SIZE cap of 256)
```
# sysctl vm.parallel_swap=2
# sysctl vm.parallel_swap_factor=6
```

## Known Problem(s)
As well as Kcompressd does, also parallel-swapper currently suffers from system hangup cases caused by **Intel graphics driver**. (confirmed on Linux 6.15.0, Kaby Lake processor)

## Contributions
This project was built upon the ideas of kcompressd by Qun-Wei Lin from MediaTek.

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/Y8Y5NHO2I)

