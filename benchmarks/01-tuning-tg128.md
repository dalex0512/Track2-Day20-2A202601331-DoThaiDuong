# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 12 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 75.0 | 99% |
| 4 | 75.3 | 99% |
| 8 | 75.8 | 100% |
| 12 | 75.8 | 100% |
| 24 | 75.8 | 100% |

**Best**: `-t 12` at 75.8 tok/s
**Slowest tested**: `-t 1` at 75.0 tok/s (1.01x spread)
**Against the physical-core default** (`-t 8`, 75.8 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=12 make bench
```

## Your explanation

No knee at all — the curve is flat (75.0 to 75.8 tok/s, 1.01x spread across
1→24 threads). This contradicts the deck's expected shape (peak at physical
core count, drop from P/E-core barrier sync above it), and the reason is
`ngl=99`: every layer is offloaded to the RTX 3050. Decode happens almost
entirely on the GPU; the CPU threads controlled by `-t` only handle a thin
sliver of orchestration (sampling, KV-cache bookkeeping) that never becomes
the bottleneck. Changing thread count from 1 to 24 barely touches wall-clock
because the thing being timed — `tg128` — is bound by GPU VRAM bandwidth and
CUDA core throughput, not CPU scheduling.

To find where thread count *does* matter on this machine, I re-ran the same
model CPU-only (`-ngl 0`): 22.83 tok/s. That is 3.32x slower than the GPU path
at 75.8 tok/s — a far bigger lever than any thread setting, and it is the
number I used for REFLECTION §5. The mechanism: with `ngl=0` every decode step
re-reads the full ~3 GB of weights from system RAM over a dual-channel DDR
bus; with `ngl=99` the same read happens from GDDR6 VRAM at several times the
bandwidth. TPOT is memory-bandwidth bound either way — GPU offload just moves
you to a much faster memory tier, which is why it dwarfs anything thread
tuning can do once GPU offload is already on.
