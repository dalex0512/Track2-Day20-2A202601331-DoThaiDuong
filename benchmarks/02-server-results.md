# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 157 | 2.70 | 2600 | 4300 | 5900 | 7.5 | 0.0% |
| 50 | 165 | 2.78 | 15000 | 19000 | 19000 | 39.6 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.03x** (21% of linear) |
| P95 latency | **4.42x** |
| Effective concurrency at 50 users | 39.6 vs `--parallel 4` slots (occupancy/slot ratio 9.90) |

**Saturated.** Throughput delivered only 1.03x for 5x the offered load, and effective concurrency (39.6) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.03x while P95 moved 4.42x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

Server saturates somewhere at or below 10 users already, and is fully saturated
by 50. The number that convinced me: RPS barely moved (2.70 → 2.78, a 1.03x
change) while offered load went up 5x — throughput is flat, which only happens
when the server is queue-bound, not compute-idle. The `make metrics` run
confirms the mechanism directly: peak `n_busy_slots_per_decode = 3.97 of 4`
(99% of `--parallel 4`) with `requests_deferred` sitting around 40-45
throughout the 60s window. All 4 decode slots were continuously busy and ~40
requests were parked in queue the whole time — that is Little's Law made
visible: effective concurrency 39.6 vs 4 physical slots means the other ~35.6
"in-flight" requests were waiting, not computing. P95 growing 4.42x while
throughput grew 1.03x is exactly what queue time looks like on top of a fixed
compute budget: the extra latency is spent waiting for a slot, not waiting for
tokens to generate.

The knob I would change first: raise `--parallel` (e.g. 4 → 8). Evidence for
that specific knob over others: the bottleneck is *slot count*, not per-token
compute — TPOT stayed ~12-14ms/token throughout the earlier bench regardless
of load, so the GPU has headroom per request, it just has too few concurrent
slots to admit more requests into decode at once. Raising `--parallel` costs
KV-cache VRAM (more slots x `ctx=2048` each), which is the real constraint on
a 6 GB card — so the second knob I'd check is whether VRAM allows it before
raising `LAB_N_CTX` or switching to the smaller Q2_K_XL quant to buy back the
headroom `--parallel 8` would need.
