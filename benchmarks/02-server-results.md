# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` � llama.cpp `b10488` �
`--parallel 4` � `ctx=2048` � `threads=8` �
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 31 | 0.52 | 15000 | 32000 | 35000 | 8.9 | 0.0% |
| 50 | 46 | 0.79 | 32000 | 53000 | 58000 | 23.5 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.50x** (30% of linear) |
| P95 latency | **1.66x** |
| Effective concurrency at 50 users | 23.5 vs `--parallel 4` slots (occupancy/slot ratio 5.89) |

**Saturated.** Throughput delivered only 1.50x for 5x the offered load, and effective concurrency (23.5) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.50x while P95 moved 1.66x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

The server saturates at or below 10 users on this laptop, and certainly before 50.
Evidence: going from 10 to 50 users (5x offered load) raised throughput only 0.52 -> 0.79
RPS (1.50x, 30% of linear), while P95 rose from 32 s to 53 s. The number that convinced me
is the effective concurrency from Little's Law: 23.5 in flight at 50 users against only
`--parallel 4` decode slots (about 5.9 requests per slot). Even at 10 users it was 8.9,
already more than 2x the slots. The server-side gauges agree: `n_busy_slots_per_decode`
reached 3.93 of 4 and `requests_deferred` stayed around 45, so 4 requests were decoding
while about 45 sat in the queue. The extra users added queue time, not throughput.

SLO example: take P95 <= 30 s. At 10 users P95 was 32 s, so even there I only just miss
it. At 50 users P95 was 53 s and the whole run missed it, so goodput at that SLO is close
to zero for the added load. The goodput that actually counts is what the 4 slots deliver
(about 0.5-0.8 requests/s), however many users connect.

First change: reduce work per request, by cutting `max_tokens` or switching to the
smaller prompt mix, not by adding slots. This is a CPU-only box (ngl=0) and decode is
memory-bandwidth bound, so raising `--parallel` shares the same bandwidth: each stream
decodes more slowly and total tokens/s barely moves, with more KV cache memory used. With
a P95 latency target, shorter outputs shrink every request's service time, which lowers
the queue wait for everyone. Offloading layers to the RTX 3050 (`-ngl`) would be the
biggest single win for raw speed, but it is a bonus-track sweep and wasn't tested here.
