# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 37 | 0.65 | 12000 | 21000 | 21000 | 8.1 | 0.0% |
| 50 | 29 | 0.50 | 37000 | 56000 | 57000 | 16.2 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.78x** (16% of linear) |
| P95 latency | **2.67x** |
| Effective concurrency at 50 users | 16.2 vs `--parallel 4` slots (occupancy/slot ratio 4.05) |

**Saturated.** Throughput delivered only 0.78x for 5x the offered load, and effective concurrency (16.2) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.78x while P95 moved 2.67x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading (required -- replace this line)

_Where does your server saturate, and what is the evidence? Name the number that
convinced you. Then say what you would change first to raise goodput at your SLO --
and why that knob and not another._
