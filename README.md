# Back of the envelope

Round numbers that are good enough to *start* a system design discussion with.

None of these are benchmarks. Real numbers depend on hardware, payload size and what the request actually does. The point is to get the order of magnitude right (within ~2–10×), say your assumptions out loud, and find which resource is under pressure. Then you go measure.

## How to approach it

1. **Clarify and pick inputs out loud.** Daily active users, actions per user per day, read:write ratio, object size, retention. Round everything: "say 100M DAU, 10 reads each, 1 KB per item".
2. **Traffic.** Requests per day ÷ 100,000 = average QPS. Multiply by 2–5× for peak (10× if very spiky).
3. **Storage and bandwidth.** Writes × size × retention. QPS × response size.
4. **Compare with one node.** Divide by what a single server, database or cache can do (tables below). That tells you whether you need one box, ten, or a different design.
5. **Name the bottleneck.** The interesting output is not the number, it is "storage is the problem, not compute" or "the DB needs a cache in front".

Say "roughly", round to one significant digit, and move on. Nobody cares if RAM is 80 ns or 120 ns.

## The numbers to remember

### Time and traffic

| | |
|---|---|
| 1 day | 86,400 s ≈ **100k s** |
| 1 month | ≈ **2.5M s** |
| 1 year | ≈ **30M s** |
| 1M requests/day | ≈ **12 QPS** |
| 100M requests/day | ≈ **1,200 QPS** |
| 1B requests/day | ≈ **12,000 QPS** |
| Peak | **2–5×** average |

### Latency

Remember the ladder, not the digits: **RAM is ns, SSD is µs, network is ms, another continent is ~100 ms.**

| Operation | Roughly |
|---|---|
| L1 cache | 1 ns |
| L2 cache | 10 ns |
| RAM access | 100 ns |
| SSD random read | 100 µs |
| Round trip in the same datacenter | 0.5 ms |
| Redis read over the network | ~0.2–1 ms (mostly the round trip) |
| Simple indexed DB query | 1–10 ms |
| HDD seek | 10 ms |
| Round trip, same continent | 50 ms |
| Round trip, cross-continent | 100–150 ms |
| Full table scan | 100 ms+ |

If the critical path has a disk seek or a cross-continent call in it, that is the latency.

### What one node can do

| Resource | Starting assumption |
|---|---|
| App server, normal API with business logic | **~1k–10k RPS** (say 5k) |
| App server, very simple/optimized endpoint | 10k–50k RPS |
| Static content / proxy | 50k+ RPS |
| Heavy CPU work, or several downstream calls | 100s–1k RPS |
| Postgres/MySQL, simple indexed reads | **~10k QPS** (5k–50k; 100k+ on a big box with the working set in RAM) |
| Postgres/MySQL, writes / transactions | **~1k–10k/s** |
| Postgres/MySQL, heavy joins or aggregation | 100s–1k QPS |
| Redis | **~100k ops/s** |
| Kafka broker | **~100 MB/s**, 100k–1M small msgs/s |

### Network and disk

| | Roughly |
|---|---|
| 1 Gbps | **~100 MB/s** usable |
| 10 Gbps | ~1 GB/s |
| HDD sequential | 100–200 MB/s |
| SATA SSD sequential | 500 MB/s |
| NVMe SSD sequential | 1–7 GB/s |
| Local NVMe random IOPS | 100k–1M |
| Cloud network volume (e.g. EBS gp3) | 3k IOPS baseline, ~1 ms per I/O |
| Server RAM | 64–256 GB is a normal big box |

### Data sizes

| Item | Roughly |
|---|---|
| Tweet-sized text | 300 B |
| Small JSON object / DB row with a few columns | 1 KB |
| Feed-sized photo, ~1080 px (JPEG/WebP) | 200 KB |
| Thumbnail / typical small web image | 10–20 KB |
| Phone photo | 2–5 MB |
| 1 minute of MP3 | 1 MB |
| 1 minute of HD video | 50–100 MB stored; ~30 MB streamed |

## Formulas

```
QPS_avg       = DAU × actions_per_user / 100,000
QPS_peak      = QPS_avg × 2..5   (10 if spiky)
Storage/year  = writes_per_day × size × 365 × replicas (×1.3 for indexes and overhead)
Bandwidth     = QPS × response_size
Servers       = QPS_peak / per_server_capacity   (then add ~50% headroom)
In flight     = RPS × latency                    (Little's law: 5k RPS × 0.2 s = 1k concurrent)
Cache size    = hot_items × item_size            (10M × 1 KB = 10 GB, fits in RAM)
```

## Things that change the answer

- **API QPS is not DB QPS.** 10k RPS × 3 queries each = 30k DB QPS. With a 90% cache hit rate it is 3k on the DB and 27k on Redis, which is fine.
- **Think in bytes, not messages.** 1M messages/s of 100 B is 100 MB/s. 1M messages/s of 1 MB is 1 TB/s, a different architecture.
- **Don't push big files through app servers.** Hand out a presigned URL and let the client upload straight to object storage. The API only says "you may upload".
- **Connections are not QPS.** 100 app instances × 100 connections = 10k Postgres connections, which it won't like. Pool them (PgBouncer) down to hundreds.
- **Provisioned storage is more than raw data.** Indexes, logs and 3× replication can turn 100 GB into ~400 GB before headroom.
- **"A server" is not a number.** `GET /user/123` from Redis and `POST /payment` with a transaction, Kafka and three external APIs differ by 100× on the same box.

## Worked example: photo sharing

Assume 100M DAU. Each user views 50 photos and 1 in 10 users uploads one photo a day.

- **Reads:** 100M × 50 = 5B/day ÷ 100k ≈ **50k QPS**, peak ~150k.
- **Writes:** 10M/day ÷ 100k ≈ **100 QPS**, peak ~300. Writes are easy.
- **Photo storage:** 10M × 2 MB = 20 TB/day → ~7 PB/year (~20 PB physical if you replicate 3× yourself; S3/GCS bill the logical 7 PB). Object storage, not a database.
- **Metadata:** 10M rows × 1 KB = 10 GB/day → ~4 TB/year raw, ~15 TB with replicas and indexes. One Postgres primary handles 300 writes/s; plan to shard or archive within a year or two.
- **Egress:** viewers get a ~200 KB resized copy, not the 2 MB original: 150k × 200 KB ≈ 30 GB/s at peak. That is ~300 servers' worth of 1 Gbps links, so it goes on a CDN.
- **App servers** (if the CDN didn't exist): 150k / 5k = 30 by CPU, call it 50 with headroom. But 30 GB/s of egress needs ~300 × 1 Gbps (or ~30–50 × 10 Gbps) NICs, so bandwidth, not CPU, sets the count.

Conclusion to say out loud: compute is cheap here; **storage and egress bandwidth drive the design**, so object storage + CDN + presigned uploads, and a cache in front of the metadata DB.

## Resources

- **Latency Numbers Every Programmer Should Know** — Peter Norvig's timing table in [Teach Yourself Programming in Ten Years](https://norvig.com/21-days.html#answers), later extended and popularized by Jeff Dean ([gist](https://gist.github.com/jboner/2841832)).
- **Interactive latency numbers** — [Colin Scott](https://colin-scott.github.io/personal_website/research/interactive_latency.html), adjusted by year; also [samwho's visualization](https://samwho.dev/numbers).
- **System Design Primer** — [Powers of two](https://github.com/donnemartin/system-design-primer#powers-of-two-table) and [latency numbers](https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know) tables.
- **Napkin Math** — [sirupsen/napkin-math](https://github.com/sirupsen/napkin-math). First-principles numbers and techniques for estimating system performance.
