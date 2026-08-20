# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 10007 | 74 / 264 | 14.0 / 14.5 | 963 / 1176 / 1176 | 71.2 |
| UD-Q2_K_XL | 2.24 | 4647 | 73 / 207 | 12.2 / 12.6 | 839 / 974 / 974 | 82.1 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.15x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

`UD-Q2_K_XL` decodes 1.15x faster (82.1 vs 71.2 tok/s) and loads 2.2x faster (4647ms
vs 10007ms cold start), for 0.73 GB less on disk. Both numbers are GPU-bound here
(`ngl=99`, RTX 3050) — the gap is smaller than the CPU-only case in the deck because
VRAM bandwidth is already high, so shaving 24% off the weight size buys less relative
headroom than it would on a CPU decode path.

I asked the same question ("Explain in 2 sentences why TPOT is memory-bandwidth
bound, not compute bound.") to both via `make serve` (Q4, port 8080) and
`serve.py --compare --port 8090` (Q2). Both answers were coherent, factually correct,
and near-identical in structure — Q4: "...spends most of its time waiting for data
to be fetched..."; Q2: "...increasing computational power won't significantly improve
performance if memory bandwidth remains a bottleneck." Neither hallucinated or
degraded noticeably on this factual, single-turn prompt.

**Verdict — worth it, conditionally.** For this 2B-class model on a 6 GB GPU, Q2_K_XL
is a reasonable default when VRAM is the binding constraint (it frees ~0.73 GB that
could raise `--parallel` or `LAB_N_CTX`) and the task is short factual Q&A. I would
NOT use it for anything requiring multi-step reasoning or precise numeric output,
where 2-bit quantization's larger rounding error per weight is more likely to
compound across a longer decode.
