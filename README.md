# Kcompressd-Unofficial
Kcompressd-Unofficial is a patch designed to improve the Linux kernel's memory reclamation mechanism, enhancing system responsiveness and throughput under high memory pressure. It achieves this by alleviating the burden on the existing kswapd daemon and enabling asynchronous swapout processing.
This project builds upon and significantly extends the concepts introduced by the original **Kcompressd** patch by Qun-Wei Lin from MediaTek.

## The Problem
In the current Linux kernel, the kswapd daemon is responsible for both scanning LRU pages and handling memory compression/swapout tasks (e.g., for ZSWAP/ZRAM). This centralized responsibility in a single thread can lead to significant performance bottlenecks, particularly under high memory pressure. When kswapd gets blocked by compression or I/O operations, the entire memory reclamation process stalls, leading to increased page allocation stalls and a noticeable degradation in system responsiveness.

## The Solution
Kcompressd-Unofficial introduces a dedicated mechanism to asynchronously offload compression and swapout operations from kswapd. This allows kswapd to focus on its primary task of LRU scanning and memory reclamation decisions, leading to a substantial improvement in overall system responsiveness.

## Key Improvements
While Kcompressd-Unofficial inherits the fundamental idea of offloading from the original Kcompressd, it introduces significant enhancements in the following key areas:

*   **Dynamic Queue Depth Tuning**
Kcompressd-Unofficial introduces sysctl interfaces (/proc/sys/vm/kcompressd).  
This interface allows for runtime adjustment of the maximum depth of the FIFO queue. This flexibility enables optimal performance tuning tailored to specific system characteristics and workloads.

*   **Advanced FIFO Queue Management with Order Preservation**
Kcompressd could lead to out-of-order processing when its FIFO overflowed (new requests were handled synchronously while older requests remained in the queue).  
Kcompressd-Unofficial implements a sophisticated mechanism: when the queue is nearing full, it synchronously processes the oldest folio currently in the queue, before enqueuing the new folio.  
This mechanism tries to ensure that requests submitted earlier are always processed first, preventing unnecessary latency and the stagnation of old data, thereby maintaining the freshness and fairness of memory reclamation.

*   **Ability to Turn Kcompressd Off**
    Kcompressd-Unofficial provides the flexibility to be completely disabled at runtime. This is achieved by setting the `vm.kcompressd` sysctl tunable to 0:
    ```bash
    sysctl -w vm.kcompressd=0
    ```
    When `vm.kcompressd` is set to 0, Kcompressd-Unofficial's asynchronous processing is bypassed, and memory compression/swapout tasks revert to being handled synchronously by kswapd, similar to the behavior of an unpatched kernel. This feature is valuable for:
    *   Troubleshooting system behavior or isolating issues (e.g., when investigating the known Intel graphics driver problem).
    *   Performance comparison against the default kernel's memory management.
    *   Temporarily disabling the feature if a specific workload does not benefit from it.

## Sysctl Tunable
Swapout parallelism can be tuned using sysctl.
- `vm.kcompressd` (range: 0-256, default: 6)
Sets the maximum FIFO queue depth.

## Known Problem(s)
As well as Kcompressd does, also Kcompressd-Unofficial currently suffers from system hangup cases caused by **Intel graphics driver**. (confirmed on Linux 6.15.0, Kaby Lake processor)

## Contributions
This project was built upon the ideas of Kcompressd by Qun-Wei Lin from MediaTek.

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/Y8Y5NHO2I)

