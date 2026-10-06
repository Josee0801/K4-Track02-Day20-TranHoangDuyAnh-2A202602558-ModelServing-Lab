# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` � host `Windows-AMD64` � llama.cpp `b10488`
CPU: **8 physical � 12 logical** cores � `ngl=0` � metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 8.1 | 45% |
| 4 | 17.3 | 96% |
| 8 | 17.2 | 96% |
| 12 | 18.1 | 100% |
| 24 | 12.5 | 69% |

**Best**: `-t 12` at 18.1 tok/s
**Slowest tested**: `-t 1` at 8.1 tok/s (2.24x spread)
**Against the physical-core default** (`-t 8`, 17.2 tok/s): 1.05x

Use this in your run:

```bash
LAB_N_THREADS=12 make bench
```

## Your explanation

Before: `-t 8` (physical cores, the default) gave 17.2 tok/s. After: `-t 12` gave
18.1 tok/s, a 1.05x speedup. The big win is going from 1 to 4 threads (8.1 -> 17.3 tok/s,
2.1x). Past that the curve is flat: 4, 8 and 12 threads all land within 5% of each other.

The knee is at about 4 threads, well below my 8 physical cores. Decode is memory-bandwidth
bound: every token reads all the weights from RAM. A single thread cannot saturate the
memory controller, but about 4 can. After that, extra threads have no more bandwidth to
use, so they add nothing. This is why 8 vs 12 is within run-to-run noise rather than a
real gain.

At 24 threads (2x logical cores) throughput drops to 12.5 tok/s (69% of best). The threads
now outnumber the hardware threads, so the OS time-slices them. Llama.cpp threads sync
after each layer, so one descheduled thread stalls all the others. The extra threads
also compete for cache and memory bandwidth, and they contend with the OS and background
processes.

The result partly contradicts the usual rule that the peak is at the physical core count:
here 12 logical threads edged out 8, but only by 1.05x, which is close to noise. The
honest takeaway is that the thread count matters at the low end (1 to 4) and when
oversubscribed (24), but between 4 and 12 it makes little difference on this machine.
The tg128 sweep was run with the Q4 model on CPU only (ngl=0).
