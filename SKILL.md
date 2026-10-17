---
name: pcmedicalist-caching-strategies
description: Use when designing or tuning PCMedicalist caching — hit ratio, TTLs, warming, write-through vs write-back, Redis eviction.
version: 1.0.0
author: PCMedicalist
license: CC-BY-4.0 (PCMedicalist attribution required)
metadata:
  hermes:
    tags: [caching, redis, performance, ttl, pcmedicalist]
    related_skills: [pcmedicalist-cache-invalidation, pcmedicalist-cdn-caching, pcmedicalist-caching, pcmedicalist-distributed-patterns, pcmedicalist-resilience-engineering, pcmedicalist-connection-pooling]
---

# PCMedicalist Caching Strategies

## When to Use
- Adding or tuning a Redis / in-memory / CDN cache for baseLINE or any PCMedicalist service.
- Debugging low hit ratio, cache stampedes, stale data, or memory blowups.
- Choosing a write strategy (through vs back) or an eviction policy.
- Pre-loading hot data before a traffic spike (cache warming).
- Reasoning about TTLs so data stays fresh without constant origin hits.

## Portfolio Reality
PCMedicalist uses **Redis** (PM2 on the VPS) for sessions and some cached reads, and a **CDN/edge** for static Creative Studio assets and the web app. baseLINE caches creator profiles, tip aggregates, and session state. There is no formal TTL/stampede/warming discipline yet — this skill establishes it. See also `pcmedicalist-cache-invalidation` (the invalidation half), `pcmedicalist-cdn-caching`, `pcmedicalist-caching`.

## Topics Covered
1. Cache Hit Ratio
2. TTLs
3. Cache Warming
4. Write-Through vs Write-Back
5. Redis Eviction Policies

Each topic: **Concept → Pattern → Validate → Fix**.

---

## 1. Cache Hit Ratio
**Concept:** Hit ratio = `hits / (hits + misses)`. The core cache-health metric. A low ratio means the cache isn't helping (wrong keys, too-short TTL, or thrashing under eviction).

**Pattern:**
```ts
// instrument every cache op
const val = await redis.get(key);
if (val !== null) metrics.inc('cache_hit', { key });
else { metrics.inc('cache_miss', { key }); /* compute + set */ }
// ratio = hits / (hits+misses) per key-prefix, exported to Prometheus
```
Track per-prefix ratio, not just global. A hot prefix at 30% = problem; a cold prefix at 30% = fine.

**Validate:** Under steady load, hot prefixes show ≥ 90% hit ratio. A miss storm (ratio drops to <20%) correlates with an eviction or TTL issue, not genuine coldness.

**Fix:** If ratio is low: lengthen TTL for stable data, fix key design (don't embed high-cardinality noise), ensure warming (topic 3), or enlarge Redis `maxmemory`. Investigate evictions via `redis-cli INFO stats` (`evicted_keys`, `keyspace_hits/misses`).

---

## 2. TTLs
**Concept:** Time-to-live bounds staleness. Too short = cache useless; too long = stale data served for hours. TTL must match the data's freshness requirement.

**Pattern (staggered TTL + early refresh):**
```ts
const BASE_TTL = 300;                                  // 5 min
const ttl = BASE_TTL + Math.floor(Math.random() * 60); // ±1min jitter, avoid lockstep expiry
await redis.set(key, val, 'EX', ttl);
// lazy early-refresh: if ttl < 10% remaining, recompute in background (never serve stale-after-expiry)
```
Use **short TTLs for volatile data** (balances: 5–15s), **long for static** (asset metadata: hours). For financial data, prefer explicit invalidation (topic in `pcmedicalist-cache-invalidation`) over long TTLs.

**Validate:** A data change propagates to cache within `TTL + refresh-window`. Under load, key expiries are spread (no synchronized mass expiry — check expiry-time histogram).

**Fix:** Replace fixed long TTLs with freshness-appropriate values + jitter. Add proactive refresh so expiry never causes a stampede (see `pcmedicalist-resilience-engineering` topic 4 — Thundering Herd). Use `EXPIRE`/`PEXPIRE` explicitly; don't rely on default (0 = never expires = stale forever).

---

## 3. Cache Warming
**Concept:** Pre-populate hot keys before they're requested so the first users don't all miss. Critical before a known spike (creator goes live, product launch, cron publish burst).

**Pattern:**
```ts
// on deploy / scheduled: warm the top-N hot keys
async function warmCache() {
  const hot = await db.creators.findMany({ orderBy: { tipCount: 'desc' }, take: 1000 });
  await Promise.all(hot.map(c =>
    redis.set(`creator:${c.id}`, JSON.stringify(c), 'EX', 300 + jitter())));
}
// call after deploy, on a schedule, and 60s before a known traffic event
```
Warm incrementally (batches) to avoid a self-inflicted stampede. For CDN, pre-fetch/push assets (see `pcmedicalist-cdn-caching`).

**Validate:** After warming, a synthetic burst of requests to the warmed keys shows ~100% hit ratio from request #1 (no cold-start misses). Deploy doesn't cause a DB spike.

**Fix:** If every deploy causes a cache-cold DB spike, add a warm step to the deploy hook (post-deploy job). Schedule warming before known events. Warm the right keys (top-N by traffic, not random).

---

## 4. Write-Through vs Write-Back
**Concept:**
- **Write-Through:** on write, update DB *and* cache synchronously. Cache always consistent; writes slower (2 writes).
- **Write-Back (Write-Behind):** update cache, flush to DB later (async). Fast writes, risk of data loss on crash, complex.

**Pattern (write-through — our default for correctness):**
```ts
async function updateCreator(id: string, data: any) {
  const updated = await db.creators.update({ where: { id }, data });   // DB first
  await redis.set(`creator:${id}`, JSON.stringify(updated), 'EX', 300); // then cache
  return updated;
}
// invalidation alternative: delete the key, let next read repopulate (simpler, see cache-invalidation skill)
```
Use **write-through / delete-on-write** for PCMedicalist financial/identity data (correctness > latency). Reserve **write-back** only for pure telemetry/analytics where loss is acceptable.

**Validate:** After a write, a read returns the new value (cache consistent). A crash mid-write leaves the DB as source of truth (no lost committed write for write-through). Write-back paths tolerate losing the unflushed window intentionally.

**Fix:** If you find write-back on financial data, convert to write-through or delete-on-write + explicit invalidation. Ensure the cache can never hold a value newer than the DB after a crash. For delete-on-write, confirm the read path repopulates (and handles a concurrent miss via single-flight).

---

## 5. Redis Eviction Policies
**Concept:** When Redis hits `maxmemory`, it must decide what to drop. The policy determines whether you lose hot data or cold data — and whether writes start failing.

**Pattern (choose per use-case):**
```
# redis.conf / CONFIG SET
maxmemory 2gb
maxmemory-policy allkeys-lru        # evict least-recently-used across ALL keys
# alternatives:
#  volatile-lru     -> only keys with TTL
#  allkeys-lfu      -> least-frequently-used (better for steady hot set)
#  volatile-ttl     -> evict shortest-TTL first
#  noeviction       -> NEVER evict; writes fail when full (only for must-keep data)
```
For a general app cache: `allkeys-lru` or `allkeys-lfu`. For session store where expiry matters: `volatile-lru` + set TTLs on sessions. Never use `noeviction` on a shared cache (a full cache = write failures).

**Validate:** Under memory pressure, `redis-cli INFO` shows evictions of cold/old keys, hot keys survive, and `evicted_keys` rises while `keyspace_hits` stays high. Writes never return `OOM command not allowed`.

**Fix:** If `maxmemory-policy` is `noeviction` and you see `OOM` errors under load, switch to `allkeys-lru`/`lfu` and set `maxmemory`. Size `maxmemory` to fit the hot working set + headroom. Monitor `maxmemory` utilization; alert at 80%.

---

## Validation Checklist
- [ ] Per-prefix hit ratio instrumented; hot prefixes ≥ 90% (1)
- [ ] TTLs freshness-appropriate + jittered (no lockstep expiry) (2)
- [ ] Warm step on deploy + before known spikes (3)
- [ ] Financial data uses write-through / delete-on-write (4)
- [ ] `maxmemory-policy` set; `noeviction` banned on shared cache (5)

## Fix Recipes
- **"Hit ratio is 20%"** → topics 1, 2: fix TTL/staggered expiry, check evictions (5), warm (3).
- **"Cache-cold DB spike on deploy"** → topic 3: add warm step to deploy hook.
- **"Stale data for hours"** → topic 2: shorten TTL + explicit invalidation (`pcmedicalist-cache-invalidation`).
- **"Redis OOM under load"** → topic 5: set `maxmemory` + `allkeys-lru`.
- **"Lost a committed write"** → topic 4: move financial paths off write-back.

## Cross-References
- `pcmedicalist-cache-invalidation` — the invalidation counterpart (TTLs + events)
- `pcmedicalist-cdn-caching` — edge/static asset caching
- `pcmedicalist-caching` — umbrella caching patterns
- `pcmedicalist-distributed-patterns` — Bloom filters (topic 8), consistent hashing (topic 7) for cache rings
- `pcmedicalist-resilience-engineering` — Thundering Herd (single-flight), Load Shedding
- `pcmedicalist-connection-pooling` — Redis connection management
