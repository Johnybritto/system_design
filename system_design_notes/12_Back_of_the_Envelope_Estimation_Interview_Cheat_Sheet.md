# Back-of-the-Envelope Estimation — System Design Interview Cheat Sheet

> Quick-reference numbers for system design interviews. Estimates are rounded; explain assumptions before calculating.

## 1. Time and traffic

| Item | Remember |
|---|---:|
| 1 day | 86,400 seconds ≈ 100K |
| 1 month (30 days) | 2.592 million seconds ≈ 2.6M |
| 1 year (365 days) | 31.536 million seconds ≈ 31.5M |
| 1M requests/day | ≈ 12 RPS |
| 10M requests/day | ≈ 116 RPS |
| 100M requests/day | ≈ 1,200 RPS |
| 1B requests/day | ≈ 12,000 RPS |
| Initial peak-traffic assumption | 3–5× average RPS (state your assumption) |

**Formula:** `Average RPS = requests per day / 86,400`  
**Formula:** `Peak RPS = average RPS × chosen peak multiplier`

## 2. Data and storage

| Item | Approximation |
|---|---:|
| 1 million records × 1 KB each | 1 GB |
| 1 billion records × 1 KB each | 1 TB |
| Three complete replicas | 3× raw data storage |
| Text message | ~1 KB (illustrative assumption) |
| Database metadata record | ~1 KB (illustrative assumption) |
| Image | ~2–5 MB (depends on resolution/compression) |
| HD video bitrate | ~3–8 Mbps (depends on codec/quality) |
| 1-hour video at 5 Mbps | ~2.25 GB |

**Formula:** `Daily storage = events/day × bytes/event`  
**Formula:** `Storage for N days = daily storage × N × replication factor`

Note: These use decimal units (1 GB = 1,000 MB; 1 TB = 1,000 GB). Real storage needs additional space for indexes, metadata, logs, backups and replication overhead.

## 3. Network bandwidth: bits vs bytes (important)

**Yes: 1 Gbps = 1,000 Mbps.**

- Lowercase **b** = **bits**; uppercase **B** = **bytes**.
- **1 byte = 8 bits**.
- **1 Gbps = 1,000 Mbps ÷ 8 = 125 MB/s**.
- **10 Gbps = 10,000 Mbps ÷ 8 = 1,250 MB/s = 1.25 GB/s**.

| Network speed | Megabits per second | Megabytes per second (theoretical) |
|---|---:|---:|
| 100 Mbps | 100 Mbps | 12.5 MB/s |
| 1 Gbps | 1,000 Mbps | 125 MB/s |
| 10 Gbps | 10,000 Mbps | 1,250 MB/s = 1.25 GB/s |

**Formula:** `Bandwidth in bytes/sec = RPS × average bytes per request`; multiply by 8 for bits/sec.  
**Interview tip:** Network links are advertised in **bits/s**, while files and storage are usually discussed in **bytes**. Actual throughput is lower due to overhead and conditions.

## 4. Approximate latency reference points

| Operation | Order-of-magnitude latency |
|---|---:|
| Memory access | ~100 ns |
| SSD random read | ~100 µs |
| Same-region network round trip | ~0.5–2 ms |
| Cross-region network round trip | ~50–200 ms |

These are illustrative, not fixed infrastructure guarantees.

## 5. Availability / allowed downtime

30-day month = 43,200 minutes.

| Availability target | Allowed downtime per 30-day month |
|---|---:|
| 99.9% (three nines) | 43.2 minutes |
| 99.99% (four nines) | 4.32 minutes |
| 99.999% (five nines) | 25.9 seconds |

## 6. Four formulas to memorize

1. **Requests per second:** `RPS = daily requests / 86,400`
2. **Storage:** `bytes/day = events/day × bytes/event`
3. **Bandwidth:** `bytes/sec = RPS × bytes/request`; `bits/sec = bytes/sec × 8`
4. **Concurrency (Little's Law):** `in-flight requests ≈ RPS × average response time in seconds`

## 7. Worked example: messaging service

**Assumptions:** 10M daily active users, 20 messages per user/day, 1 KB stored per message, 5× peak multiplier, three full replicas.

| Metric | Calculation | Result |
|---|---|---:|
| Messages/day | 10M × 20 | 200M |
| Average messages/sec | 200M ÷ 86,400 | ~2,315 |
| Peak messages/sec | 2,315 × 5 | ~11,575 |
| Raw daily storage | 200M × 1 KB | ~200 GB |
| Raw annual storage | 200 GB × 365 | ~73 TB |
| Annual storage with 3 replicas | 73 TB × 3 | ~219 TB |

Excludes media attachments, indexes, protocol overhead, backups, logs and potential additional copies.

## 8. What to prioritize in an interview

Remember **86,400 sec/day**, **RPS conversions**, **1M × 1 KB = 1 GB**, **3–5× peak assumption**, **1 Gbps = 1,000 Mbps = 125 MB/s**, and **99.9% / 99.99% availability downtime**. State assumptions and use round numbers to keep the arithmetic fast.
