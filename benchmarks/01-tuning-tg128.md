# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` � host `Windows-AMD64` � llama.cpp `b10488`
CPU: **4 physical � 8 logical** cores � `ngl=99` � metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 8.7 | 100% |
| 2 | 8.5 | 98% |
| 4 | 8.1 | 94% |
| 8 | 8.0 | 92% |
| 16 | 8.1 | 93% |

**Best**: `-t 1` at 8.7 tok/s
**Slowest tested**: `-t 8` at 8.0 tok/s (1.09x spread)
**Against the physical-core default** (`-t 4`, 8.1 tok/s): 1.07x

Use this in your run:

```bash
LAB_N_THREADS=1 make bench
```

## Your explanation

My curve is flat rather than peaking at the physical-core count. The best result is `-t 1` at 8.7 tok/s, but every tested setting remains close: 8.0–8.5 tok/s from 2 to 16 threads. The physical-core default (`-t 4`) is only 7% slower than the best setting, so thread count is not the dominant bottleneck for this workload.

There is no useful knee at 4 physical cores. Adding threads does not increase throughput; instead, performance gradually falls after one thread. This is consistent with a small model and short decode workload where synchronization, scheduling overhead, cache/memory contention, and possibly the GPU/offload path cost more than the parallel work available to the CPU. Hyperthreads and oversubscription (`-t 8` and `-t 16`) add contention without adding useful compute capacity.

I would use `LAB_N_THREADS=1` for the best measured throughput, while noting that `-t 4` is a reasonable default if I need the machine to remain responsive or want a more conservative configuration.
