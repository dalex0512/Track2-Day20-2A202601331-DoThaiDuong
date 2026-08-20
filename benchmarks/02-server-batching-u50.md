# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 15 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.97 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 45 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 18148 |

Highest sampled value was **3.97 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak batch width was 3.97 of 4 slots — essentially every decode step during
the load-50 window had all 4 slots occupied. This does NOT match the effective
concurrency figure in `02-server-results.md` (39.6): that number is ~10x
higher than the slot gauge.

I trust both numbers, because they are measuring different things, not
disagreeing. `n_busy_slots_per_decode` is server-side occupancy — how many of
the 4 physical decode slots were doing work at each sampled instant, capped at
4 by definition. Effective concurrency from Little's Law (RPS x avg latency)
counts every request currently "in the system", including the ~40 sitting in
`requests_deferred` waiting for a slot to free up. So 3.97/4 says "the compute
resource is maxed out"; 39.6 says "there is a ~10x backlog behind that maxed-out
resource". Together they are the saturation story: the server isn't dropping
requests or crawling per-token — it is completely compute-saturated at exactly
4 concurrent decodes, and everything past that queues.
