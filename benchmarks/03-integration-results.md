# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 1.8 | 3056.2 | 3058.1 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 2708.6 | 2708.8 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 2646.3 | 2646.4 |

Mean per stage (ms): embed **0.0** · retrieve **0.7** ·
llm **2803.7** · total **2804.4**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- **N16 Cloud/IaC** — stub (no cloud infra wired into this pipeline)
- **N17 Data pipeline** — stub (fixed 3-doc corpus baked into `pipeline.py`)
- **N18 Lakehouse** — stub (no storage layer; documents live in memory)
- **N19 Vector + features** — stub (`embed: 0.0 ms` — retrieval is keyword
  overlap, not a real embedding model or vector index)
- **N20 Serving** — real (`llama-server`, actual HTTP round-trip, real
  prefill/decode timings from the server)

The dominant stage (llm, 100% of total, mean 2803.7ms) is exactly what I
expected: embed and retrieve are near-zero because they're stubs doing
in-memory keyword matching over 3 documents, so there is nothing there to cost
time. The only real work in this pipeline is the LLM call, so of course it
dominates.

If I had to halve this pipeline's latency, I'd attack the llm stage, since
it's the only stage with any latency to cut — and specifically the decode
portion (`max_tokens`/answer length), since TPOT here (~12-14ms/token, GPU
offloaded) is already close to the hardware's memory-bandwidth floor; the
faster lever is generating fewer output tokens or truncating earlier once the
retrieved context is short, not further server tuning.
