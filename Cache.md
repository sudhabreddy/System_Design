# Caching: Interview Reference Notes

Oct 22, 2026 · @Sudha

## 1. The core idea

A cache is a faster, smaller copy of data kept closer to whoever needs it. You gain speed and protect the source of truth. You pay with possible staleness.

Every caching decision comes back to one question: **how stale can this data be, and what happens if it's wrong?**

**Why it works (approximate orders of magnitude; verify with your own measurements):**

| Source | Rough latency |
| --- | --- |
| In-process memory | Nanoseconds to about a microsecond |
| Redis in the same data centre | Sub-millisecond to about 1 ms |
| Indexed database query | A few ms, often more under load |
| Cross-region or third-party API | Tens to hundreds of ms |

**Staleness tolerance in banking:**

| Data | Tolerance | Cache it? |
| --- | --- | --- |
| Branch locations, currency list, product catalogue | Hours | Yes, aggressively |
| Indicative FX rates on a quote screen | Seconds | Yes, short TTL |
| User profile, preferences | Minutes | Yes, with invalidation on update |
| Balance shown on a dashboard | Seconds | Carefully, show "as of" time |
| Balance used to authorise a payment | None | **Never** |

**When not to cache:** data that changes faster than it's read, data where correctness needs the latest value, and data rarely read twice (low hit ratio, wasted memory).

**Golden rule:** never let a cached value drive a money-moving decision.

## 2. Where caches live

Real systems stack several caches, from the user's browser down to the database.

| Layer | Example | Notes |
| --- | --- | --- |
| Client / browser | HTTP `Cache-Control`, `ETag` | Authenticated banking responses usually `no-store` |
| CDN | Static assets, public content | Never personalised data |
| API gateway / reverse proxy | Public read-heavy endpoints |  |
| In-process (local) | `IMemoryCache` in .NET | Fastest; each instance has its own copy |
| Distributed | Redis, Valkey, Memcached | Shared by all instances; one network hop |
| Database internal | Buffer pool, plan cache | Free; know it exists |

**Local vs distributed:**

|  | Local | Distributed |
| --- | --- | --- |
| Speed | Fastest, no network | One network call |
| Consistency across instances | Each instance can differ | Same data for all |
| Invalidation | Hard (every instance) | One place |
| Best for | Static reference data, config, hot keys | Shared data, sessions, most app caching |

**Two-level cache (common in production):** small local L1 with a very short TTL (a few seconds) → Redis L2 → database. L1 absorbs hot-key traffic; the short TTL limits how long instances disagree.

**Invalidating local caches across instances:** short TTLs as the baseline; for faster propagation, publish an invalidation message (Redis pub/sub or Kafka) that every instance listens to.

## 3. Read and write patterns

Cache-aside with delete-on-write is the default choice for most read-heavy services.

| Pattern | How it works | Strength | Weakness | Banking use |
| --- | --- | --- | --- | --- |
| Cache-aside | App checks cache; on miss reads DB and fills cache; on write updates DB and deletes key | Simple; survives cache failure | First read after a miss is slow; race risk (section 4) | Default for profiles, reference data |
| Read-through | Cache library loads from DB on a miss | App code only talks to cache | Tied to library support | Same uses as cache-aside |
| Write-through | Write to cache and DB together | Cache always warm | Slower writes; caches unread data | Data read right after writing |
| Write-behind (write-back) | Write to cache; flush to DB asynchronously | Very fast writes | **Data loss if cache dies before flush** | Never for financial records; maybe non-critical counters |
| Write-around | Writes go to DB, bypass cache | Cache not polluted by writes | First read after write misses | Write-heavy, rarely read data |

**Cache-aside vs write-around: not the same.**

- Cache-aside answers *how reads work and who fills the cache* (the app, lazily on a miss).
- Write-around answers *what happens to the cache on a write* (it's bypassed).
- They're usually combined, which is why they get blurred.
- Subtle difference: strict write-around doesn't invalidate, so an already-cached value stays stale until TTL. Cache-aside as usually practised deletes the key on write. Definitions vary between sources.
- Read and write strategies are separate choices: write-around can pair with read-through, cache-aside reads can pair with write-through.

**Interview line:** "Cache-aside describes how reads populate the cache; write-around describes the write path. I'd combine them and explicitly delete the key on write, so we don't rely on TTL alone."

## 4. TTL and invalidation

Use explicit invalidation for freshness and a TTL as the safety net. Always set a TTL.

**Three tools, usually combined:**

1. **TTL.** Every key expires eventually, so a missed invalidation has bounded damage.
2. **Explicit invalidation.** Delete the key when data changes. At scale, drive it from change events (Kafka, or change-data-capture from the database).
3. **Versioned keys.** e.g. `product:v42:123`. Bump the version on data or schema change; old keys expire naturally. Useful for safe deployments when the cached object's shape changes.

**TTL jitter:** add a random offset (e.g. 300 s ± 10%) so many keys don't expire at the same moment.

**Why delete instead of update on write?** With two concurrent writers, "update DB, then update cache" can finish out of order. The cache then holds the older value until TTL. Deleting means the next read loads the current value.

**The race that delete doesn't fully fix:**

1. Reader A misses the cache and reads the **old** value from the DB.
2. Writer B updates the DB and deletes the key.
3. Reader A writes the old value into the cache.

**Mitigations:** short TTLs, a delayed second delete, or CDC-driven invalidation. Facebook's *Scaling Memcache at Facebook* (NSDI 2013) describes a "leases" mechanism for this class of problem.

## 5. Eviction

LRU is the usual default; LFU is better when some keys stay consistently popular.

| Policy | Removes | Good when | Watch out |
| --- | --- | --- | --- |
| LRU (least recently used) | Item untouched longest | Most workloads | A one-off scan can push out hot keys |
| LFU (least frequently used) | Item used least often | Steady popular keys | Slower to adapt to new hot keys |
| FIFO / random | Oldest / any item | Simplicity | Rarely the best choice |

**LRU implementation (common coding question):** hash map for O(1) lookup + doubly linked list ordered by recency. On access, move the item to the front; when full, evict from the back. Get and put are both O(1).

**Redis `maxmemory-policy`** (check current Redis docs for the full list):

- `allkeys-lru` / `allkeys-lfu`: good for a pure cache.
- `volatile-lru`: evicts only keys that have a TTL.
- `noeviction`: rejects writes when full. Use it when Redis holds things you must not lose (rate-limit counters, locks), or better, keep those on a separate instance.

## 6. Failure modes

This is where senior candidates stand out: name the failure, its cause, and the fix.

| Failure | What happens | Fixes |
| --- | --- | --- |
| Stampede (thundering herd) | Hot key expires; many requests miss together and hit the DB at once | Single-flight / request coalescing; background refresh before expiry; serve stale while revalidating; TTL jitter |
| Penetration | Requests for keys that don't exist (often malicious) always miss | Cache "not found" with short TTL (negative caching); Bloom filter in front |
| Avalanche | Many keys expire together, or the cache node dies; DB overwhelmed | TTL jitter; HA cache (replicas, failover); circuit breakers and rate limiting in front of DB |
| Hot key | One key overloads a single shard | Local L1 cache in front; replicate key across shards with suffixes (`fx:GBPUSD:1…N`) |
| Big key | Huge value blocks Redis (mostly single-threaded command execution) | Keep values small; split collections; compress |

**Stampede details:**

- Single-flight within one instance = a per-key lock. Across instances = a distributed lock or similar.
- If the lock holder crashes: the lock needs an expiry, and waiters need a timeout with a fallback (serve stale, or go to DB).
- `IMemoryCache.GetOrCreateAsync` in .NET is, as far as I know, **not** atomic, so it doesn't prevent stampedes by itself. Verify in current Microsoft docs.

**When the cache goes down:**

- Treat the cache as an optimisation, never a hard dependency. Fall back to the DB: slower but correct.
- The real danger: the DB was sized assuming most reads hit the cache. Protect it with a circuit breaker on cache calls, rate limiting or load shedding, and a local cache on critical paths.
- Prevent outages with replicas and automatic failover.
- Deliberately test a cold-cache restart before it happens on a busy morning.
- Data that only lives in Redis (sessions, idempotency keys) isn't really a cache. It needs durability and HA, ideally on a separate instance.

## 7. Distributed cache in production

Redis is the industry default; know its scaling, availability and persistence options at a high level.

|  | Memcached | Redis | Valkey |
| --- | --- | --- | --- |
| Model | Simple multi-threaded key-value | Rich data types (hashes, sorted sets, streams), Lua, replication, clustering | Fork of Redis |
| Persistence | None | RDB snapshots, AOF log | Same lineage as Redis |
| Notes | Pure cache | Industry default | Exists because of Redis licence changes |

**Licensing (verify current status):** as far as I know, Redis changed its licence in 2024, the Linux Foundation forked it as Valkey, and Redis later added an AGPL option. Banks' legal teams care about this.

**Scaling and availability:**

- **Sharding** spreads keys across nodes. Redis Cluster splits the keyspace into 16,384 hash slots.
- **Consistent hashing** means adding or removing a node moves only a fraction of keys.
- **Replication + automatic failover** (Sentinel for non-cluster Redis, or the managed service's own failover).

**Persistence:** a pure cache often doesn't need it. If losing the cache would be a disaster, ask whether it's really a cache or actually a database.

**Managed services:** banks often use AWS ElastiCache or Azure Cache for Redis. Check current product names and offerings.

## 8. Banking-specific practices

In a bank, correctness and data protection outrank speed.

1. **No money decisions from the cache.** Authorisation, payment limits and transfer balance checks always read the source of truth.
2. **PII protection.** Cached customer data is still customer data: TLS in transit, encryption at rest, network isolation, cache authentication (Redis ACLs). No raw card or account numbers in keys, since keys appear in logs and monitoring. Use internal IDs or hashes.
3. **Right to erasure (GDPR / UK GDPR).** Deleted customer data must leave caches too. Short TTLs plus an explicit purge path. Defer to compliance on legal details.
4. **Idempotency keys.** Stored in Redis so retried payments aren't processed twice. This is correctness, not performance: plan for what happens if a key is evicted.
5. **Sessions and token revocation.** Revocation must take effect fast, so keep local L1 TTLs short.
6. **Rate limiting.** Redis counters, often updated atomically with Lua scripts.
7. **Honest UX.** Show freshness, e.g. "Balance as of 10:42".

## 9. Observability and key naming

Monitor trends, not magic numbers; what "good" looks like depends on the workload.

| Metric | What it tells you |
| --- | --- |
| Hit ratio | Whether the cache is earning its memory |
| Latency p50 / p99 / p999 | Tail latency, not just averages |
| Eviction rate | High = cache too small or TTLs wrong |
| Memory usage, fragmentation | Capacity headroom |
| Connection count | Client pooling issues |
| DB load on a cold cache | Whether the DB survives a cache restart |

**Key naming convention:** `service:entity:id:version`, e.g. `profile:customer:8812:v3`. Makes debugging and bulk invalidation much easier.

## 10. Interview framework

When you add a cache to a design, cover these eight points without being asked.

1. What are you caching, and how stale can it be?
2. Which layer, and why?
3. Which read and write pattern?
4. TTL and invalidation approach?
5. Eviction policy?
6. Failure modes (stampede, penetration, cache down) and the handling for each?
7. Consistency risks, and why they're acceptable for this data?
8. What will you monitor?

**Answer structure:** point → reason → example → trade-off. Learn the structure, not the wording.

**Common follow-ups and short answers:**

| Question | Short answer |
| --- | --- |
| When would you NOT cache? | Fast-changing data, correctness-critical reads, data rarely read twice |
| How do you invalidate local caches across instances? | Short TTLs, plus invalidation messages via pub/sub or Kafka |
| Which pattern for a customer profile service? | Cache-aside, delete-on-write, TTL with jitter |
| Describe the cache-aside race | Reader gets old value, writer updates and deletes, reader caches old value (section 4) |
| Implement LRU | Hash map + doubly linked list, O(1) get and put |
| Lock holder crashes during stampede protection? | Lock expiry; waiters time out and serve stale or go to DB |
| Data that only lives in Redis? | Not a cache: needs durability, HA, separate instance |

**What to prioritise:** know sections 1–6 well enough to apply on the spot. Sections 7–9 are for showing depth when pushed. Learn each concept as a problem and its fix, not as a definition.

## 11. Things to verify and further reading

These notes are written from general knowledge without looked-up sources; check the items below before relying on them.

- [ ] Latency figures in section 1 (approximate orders of magnitude only)
- [ ] Current Redis / Valkey licensing status
- [ ] Full list of Redis `maxmemory-policy` options
- [ ] Whether `IMemoryCache.GetOrCreateAsync` is atomic (current .NET docs)
- [ ] Current API and status of .NET `HybridCache`
- [ ] Current AWS and Azure managed cache product names
- [ ] GDPR / UK GDPR obligations for cached data (with compliance)

**Further reading:** *Scaling Memcache at Facebook* (NSDI 2013), after the replication chapters of *Designing Data-Intensive Applications*.
