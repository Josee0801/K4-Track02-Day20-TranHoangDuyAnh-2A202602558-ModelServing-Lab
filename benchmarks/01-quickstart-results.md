# 01 - Measure: latency baseline

Model `Gemma 4 E2B` � host `Windows-AMD64` � llama.cpp `b10488`
Settings: `threads=8` `ngl=0` `ctx=2048`
`max_tokens=64` � warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 � `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3118 | 227 / 318 | 43.8 / 57.2 | 2980 / 3921 / 3921 | 22.8 |
| UD-Q2_K_XL | 2.24 | 2546 | 374 / 466 | 39.2 / 41.5 | 2843 / 2926 / 2926 | 25.5 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.12x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

On my machine (i5-12450HX, 8 threads, CPU-only with ngl=0) the 2-bit model
(UD-Q2_K_XL) is 0.73 GB smaller (2.24 vs 2.97 GB, about 25% less) but only 1.12x
faster at decode: TPOT P50 dropped from 43.8 ms to 39.2 ms (22.8 -> 25.5 tok/s). The
gain is small because decode is memory-bandwidth bound, so it scales with bytes read per
token, and the Q2 file is only about 25% smaller. Not all weights shrink: the "UD"
(dynamic) quants keep sensitive layers at higher precision.

The 2-bit model is worse on TTFT: P50 374 ms vs 227 ms (about 1.6x slower). Prefill is
compute bound, and dequantizing 2-bit weights costs extra CPU work per token. Its P95
TTFT (466 ms) is also higher. Q2 does have tighter TPOT tails (P95 41.5 ms vs 57.2 ms),
and its E2E P50 is only slightly better (2843 vs 2980 ms).

Verdict: on this laptop the 2-bit quant is not worth it. It saves under 0.75 GB and
gives about 12% faster decode, but it makes the first token noticeably slower, and
2-bit quantization usually costs answer quality. I would keep Q4 as the default and use
Q2 only if RAM or disk were the binding constraint.
