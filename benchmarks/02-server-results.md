# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` � llama.cpp `b10488` �
`--parallel 4` � `ctx=2048` � `threads=4` �
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1 | 0.02 | 57000 | 57000 | 57000 | 1.0 | 0.0% |
| 50 | 2 | 0.02 | 94000 | 94000 | 94000 | 2.0 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.20x** (24% of linear) |
| P95 latency | **1.65x** |
| Effective concurrency at 50 users | 2.0 vs `--parallel 4` slots (occupancy/slot ratio 0.50) |

**Saturated.** Throughput stopped scaling (1.20x delivered for 5x offered, 24% of linear) even though effective concurrency (2.0) sits below 4 slots. Something other than decode-slot count is the limit -- look at memory bandwidth, or context/KV pressure.

Throughput moved 1.20x while P95 moved 1.65x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

> **Small sample.** Only 1 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

This result is provisional because the 10-user run completed only one request, but the saturation signal is still clear. Raising offered load from 10 to 50 users (5×) increased delivered throughput by only 1.20×, from about 0.02 to 0.024 RPS. At the same time, P95 latency rose from 57 s to 94 s (1.65×). The number that convinced me is therefore the 24% of linear throughput scaling: most additional users increased waiting time rather than useful completed work.

For a P95 SLO of 60 seconds, the 10-user result is already at the boundary and the 50-user result misses the SLO. I would first reduce the output-token budget for this workload, because decode time dominates these very long requests and each additional generated token keeps a request in a serving slot longer. I would not increase `--parallel` first: observed effective concurrency is only 2.0 while the server has 4 slots, so simply adding slots is unlikely to solve the current bottleneck and could add KV-cache and memory-pressure costs.

I would rerun both tests for three minutes before treating the percentile values as final, since the completed-request sample is still very small.
