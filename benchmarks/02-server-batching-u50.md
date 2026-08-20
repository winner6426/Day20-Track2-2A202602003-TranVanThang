# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 6 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.40 of 4 slots (85%) |
| `requests_processing` | 4 |
| `requests_deferred` | 6 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 1067 |

Highest sampled value was **3.40 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

The peak sampled average decode batch width was 3.40 of 4 slots (85%). This is
direct evidence of continuous batching: the server decoded several requests
together rather than serving one request at a time. The server also reported
4 requests processing and 6 deferred requests, showing queueing when requests
exceeded immediately available slots.

This is higher than the effective concurrency of 2.0 in the load report. I trust
the server-side busy-slot gauge more because it measures active decode slots
directly, while Little's Law used only two completed requests and underestimates
requests still queued or in flight.
