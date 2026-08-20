# Bonus - Context-length sweep (prefill cost)

Host `Windows-AMD64` · llama.cpp `b10488` ·
`threads=8` `ngl=99` · RAM 19.7 GB

| Prompt tokens | Prefill (tok/s) | TTFT contribution (ms) | vs linear scaling |
|:--|--:|--:|--:|
| 256 | 1066.1 | 240.1 | 1.00x |
| 1024 | 1117.8 | 916.1 | 0.95x |
| 2048 | 1107.1 | 1849.9 | 0.96x |
| 4096 | 1508.4 | 2715.4 | 0.71x |
| 8192 | 2575.0 | 3181.3 | 0.41x |

At 8192 tokens, prefill costs **3181 ms**, which is
**0.41x** linear scaling -- so on this hardware, over this range, prefill is
still growing **roughly linearly**, not quadratically.

That is the correct finding, not a failed experiment. Attention is O(N^2), but it is only
one term: the per-layer linear projections and MLP are O(N), and on a 2B-class model at
short prompts they dominate. The quadratic term only overtakes them once N gets large
enough. Your prefill cost is currently bounded by throughput, not by sequence length.

To find where it *does* bend, extend the grid:

```bash
.venv/bin/python bonus/sweeps/ctx-len-sweep.py --grid 1024,4096,8192,16384,32768
```

Watch the "vs linear" column: the first row that climbs meaningfully above 1.0 is where
attention starts to matter on your machine. Report that crossover point.

Either way, this is the number to remember when someone proposes stuffing more retrieved
context into a RAG prompt "because the context window allows it". Prefill is paid in full,
on every request, before the first token appears.

## Your finding

No quadratic bend visible in this range — the "vs linear" column goes 1.00x →
0.95x → 0.96x → 0.71x → 0.41x, i.e. prefill throughput *improves* with longer
prompts (1066 → 2575 tok/s at 8192 tokens) rather than degrading. On this
hardware (RTX 3050, GPU-offloaded prefill, `ngl=99`), the matmul-heavy prefill
pass benefits from larger batched matrix multiplies at longer context — GPU
compute utilization goes up faster than the O(N²) attention term can bite,
because at N≤8192 on a 2B-class model the linear per-layer projections still
dominate FLOPs over attention's quadratic term. TTFT still grows in absolute
terms (240ms → 3181ms, a real 13.2x increase in wait time) — that growth is
just sub-linear, not the runaway curve the deck predicts.

What this tells me about RAG chunk budget: on this hardware, the danger isn't
"attention blows up" within an 8192-token window — it's that TTFT is already
paid in full, on every request, before the first token streams. 3.2 seconds
of pure wait at 8192 tokens is a bad user experience regardless of whether the
scaling is linear or quadratic. So the practical budget isn't set by where the
curve bends (I didn't find that point here) but by what TTFT the product can
tolerate — for a chat-latency SLO under ~1s, that caps retrieved context at
roughly 1-2K tokens on this machine, well before quadratic effects would ever
become the limiting factor. To find the actual bend I'd need to extend the
grid past 8192 (16384/32768), where prompt tokens would exceed this model's
practical context and the attention term should start to dominate.
