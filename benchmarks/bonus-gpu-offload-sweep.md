# Bonus - GPU offload sweep

Host `Windows-AMD64` · backend(s) `vulkan` ·
llama.cpp `b10488` · `threads=1` · metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 9.5 | 1.00x | 75% |
| 8 | 7.8 | 0.82x | 62% |
| 16 | 8.1 | 0.85x | 64% |
| 24 | 8.7 | 0.92x | 69% |
| 32 | 12.6 | 1.33x | 100% |
| 99 | 12.4 | 1.30x | 98% |

Best: `-ngl 32` at 12.6 tok/s
-- 1.33x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

`-ngl 32` was the best measured setting at 12.6 tok/s, a 1.33x improvement
over CPU-only decoding at `-ngl 0` (9.5 tok/s). `-ngl 99` reached 12.4 tok/s,
or 98% of the best result, so the two settings are effectively a plateau within
normal benchmark variation rather than strong evidence that full offload failed.

Small partial offloads were worse than CPU-only: `-ngl 8` to `-ngl 24` measured
7.8-8.7 tok/s. They leave substantial work on the CPU while adding Vulkan
synchronization and host-device transfer overhead. Moving enough layers to the
Vulkan device finally outweighs that overhead. The 0.2 tok/s gap between `-ngl
32` and `-ngl 99` is too small to claim that VRAM ran out; bandwidth and launch
overhead are the more cautious explanation for the flat top of this curve.
