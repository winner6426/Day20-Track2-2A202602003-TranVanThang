# Bonus C8 - Offline semantic-cache diagnostic

## Setup

I ran the offline semantic-cache demo with its synthetic bag-of-words embedder
and a threshold sweep from 0.70 to 0.95. A cache miss simulates 250 ms of
decode time; a cache hit returns immediately.

## Result

At threshold 0.80, the cache produced 3 hits out of 8 prompts (38%) and skipped
about 750 ms of simulated decode time. Prompts #3 and #6 were true paraphrases
of the first goodput question, and prompt #8 was a true paraphrase of the
PagedAttention question. All three had similarity 1.00 and were cache hits.

Prompt #4, "What does time to first token mean?", is a true paraphrase of
prompt #2, "Explain TTFT and TPOT.", but it had similarity 0.00 and missed.
This is a false miss caused by lexical mismatch: the bag-of-words stub cannot
learn that TTFT and "time to first token" mean the same thing. The distinct
prefix-caching question (#7) also had similarity 0.00 and correctly missed.
No false hit occurred in this short offline stream.

The sweep was flat: every threshold from 0.70 through 0.95 produced 3/8 hits.
That is not evidence that threshold selection is unimportant. It is an artifact
of the stub embedder, whose similarities are almost entirely 0.00 or 1.00. This
offline result therefore demonstrates the cache control flow and the false-miss
failure mode, but cannot establish a production threshold or a false-hit versus
false-miss trade-off. A real sentence embedding model should be evaluated before
using semantic caching in production.

Semantic and prefix caches also need tenant isolation: cache keys should be
salted or partitioned by tenant to prevent cross-user response or timing leaks.
