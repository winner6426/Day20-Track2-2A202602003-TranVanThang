# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` � llama.cpp `b10488` �
retrieval backend: **keyword overlap** � 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.3 | 22497.1 | 22497.5 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 13344.8 | 13345.0 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 21776.8 | 21777.0 |

Mean per stage (ms): embed **0.0** � retrieve **0.2** �
llm **19206.2** � total **19206.5**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **goodput** is more useful than raw throughput because it specifically accounts for the overhead of **SLOs** (Service Level Objectives).

While raw throughput measures the total requests per second, goodput only counts requests that met the specific targets (TTFT and TPOT). This means goodput provides a more accurate and realistic measure of the system's capacity to 

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** within GPU memory.

By storing the KV cache in non-contiguous pages, it avoids the wasted space that would occur if all KV data were packed into a single contiguous block of memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when the **prefill operation is compute-bound** (requires significant CPU/GPU processing) and the **decode operation is memory-bandwidth-bound** (requires significant memory throughput).

By separating these steps, the model can:
1.  **Prefill efficiently**: Use the compute-bound prefill to prepare the model state and compute the attention keys efficiently.
2.  *


## Which N16-N19 pieces are real

- **N16 Cloud/IaC:** stubbed — this lab runs locally on `localhost`; no cloud or IaC deployment was used.
- **N17 Data pipeline:** stubbed — the documents come from an in-memory toy corpus, not a batch or streaming pipeline.
- **N18 Lakehouse:** stubbed — no Delta/Iceberg/database lakehouse was used; toy documents stand in for stored data.
- **N19 Vector + features:** stubbed — no embedding endpoint or vector database was used; retrieval used keyword overlap.
- **N20 Serving:** real — the pipeline sent requests to the local `llama-server` OpenAI-compatible endpoint.

The dominant LLM stage was expected: it averaged 19,206.2 ms of the 19,206.5 ms total, while keyword retrieval averaged only 0.2 ms and embeddings were not used. To halve pipeline latency, I would optimize generation/decode first by reducing the output-token budget or using a faster serving configuration. Optimizing retrieval would have almost no effect on end-to-end latency.
