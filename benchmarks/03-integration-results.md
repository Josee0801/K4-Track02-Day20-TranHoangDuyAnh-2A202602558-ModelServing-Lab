# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` � llama.cpp `b10488` �
retrieval backend: **keyword overlap** � 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 4709.6 | 4709.7 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 4569.2 | 4569.3 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 4488.9 | 4489.0 |

Mean per stage (ms): embed **0.0** � retrieve **0.0** �
llm **4589.2** � total **4589.3**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

| Day | Piece | Real or stub |
|:--|:--|:--|
| N16 Cloud/IaC | none; everything runs on my laptop | stub (not used) |
| N17 Data pipeline | the `TOY_DOCS` list built into `pipeline.py` | stub |
| N18 Lakehouse | no tables or storage layer | stub |
| N19 Vector + features | keyword-overlap retrieval (no embedding server was running) | stub |
| N20 Serving | `llama-server` with Gemma 4 E2B, OpenAI-compatible API | **real** |

Only the LLM stage is real. The retrieval stage is a keyword-overlap stub over 3 toy
docs, which is why `embed` and `retrieve` show 0.0 ms.

Is the dominant stage what I expected? Yes. `llm` is 100% of the total (4589 ms mean).
Each query sends about 110-150 prompt tokens and generates about 23 tokens on a CPU-only
run, so prefill (about 1.2-1.5 s) plus decode (about 1.1-1.2 s) dominates, and the stub
retrieval is free. This split is not representative: a real vector search would add tens
of milliseconds, but would still be small next to the LLM call.

To halve the latency I would attack the LLM stage, since nothing else is worth touching.
First, offload layers to the RTX 3050 with `-ngl`, which speeds up both prefill (compute
bound) and decode (higher memory bandwidth). Second, send fewer context tokens by
retrieving 1-2 chunks instead of 3, since prefill is about half of the LLM time. A faster
retriever would change almost nothing.
