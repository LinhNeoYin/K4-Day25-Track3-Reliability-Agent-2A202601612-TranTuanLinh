# Day 10 Reliability Report
**Họ và tên:** Trần Tuấn Linh
**Mã sinh viên:** 2A202601612
## 1. Architecture summary

The gateway (`ReliabilityGateway.complete()`) routes every request through three layers, in order: a semantic cache, a circuit-breaker-protected provider chain, and a static degraded-mode fallback.

```
User Request
    |
    v
[Gateway] ---> [Cache check] ---> HIT? return cached (route=cache_hit:<score>)
    |                                 |
    v                                 v MISS
[Circuit Breaker: primary] -------> Provider "primary" (route=primary)
    |  (OPEN? skip immediately)
    v
[Circuit Breaker: backup] --------> Provider "backup" (route=fallback)
    |  (OPEN? skip immediately)
    v
[Static fallback message] (route=static_fallback, error=<last provider error>)
```

- **Cache**: `ResponseCache` (in-memory) or `SharedRedisCache` (Redis-backed), selected by `cache.backend` in config. Both share the same n-gram cosine similarity function, the same privacy guardrail (blocks caching/reading queries with balance/password/SSN/account-number patterns), and the same false-hit guardrail (rejects a cache hit if the query and the cached key contain different 4-digit numbers, e.g. different years).
- **Circuit breaker**: one `CircuitBreaker` instance per provider, 3-state machine (CLOSED → OPEN → HALF_OPEN → CLOSED). CLOSED allows all calls; after `failure_threshold` consecutive failures it OPENs and fails fast; after `reset_timeout_seconds` it moves to HALF_OPEN and allows a single probe request; a successful probe closes the circuit again, a failed probe re-opens it immediately.
- **Fallback chain**: providers are tried in the order listed in config (`primary`, then `backup`). The first provider to succeed serves the request; its result is written back into the cache for future hits.
- **Static fallback**: only reached when every provider in the chain fails or has its circuit open — the gateway returns a degraded static message instead of raising an exception, so the caller never sees an unhandled error.

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Opens the circuit after 3 consecutive failures — tolerates transient blips (1-2 failures) without tripping, but reacts quickly to a real outage instead of waiting for many failed requests to pile up. |
| reset_timeout_seconds | 2 | Short enough that a recovered provider is probed again quickly (kept load off it for ~2s), long enough to avoid hammering a still-broken provider with repeated probes. |
| success_threshold | 1 | A single successful probe is enough to trust the provider again in this lab; in a stricter production setting this could be raised to 2-3 for more confidence before fully re-opening traffic. |
| cache TTL | 300s (5 min) | Matches how often the underlying "facts" (refund policy, FAQ answers) are expected to change in this lab's synthetic dataset — long enough to get real hit-rate benefit, short enough that stale answers don't linger. |
| similarity_threshold | 0.92 | Set high on purpose: with n-gram cosine similarity, unrelated queries can still share some short 3-gram tokens, so a high threshold together with the false-hit guardrail (different years/numbers) is what keeps the cache from serving a wrong answer for a "close but different" question. |
| load_test requests | 100 per scenario (300 total across 3 scenarios) | Enough requests per scenario to see the circuit breaker open/close more than once and get a stable P95/P99 latency estimate, while keeping each `make run-chaos` run fast (a few seconds). |

## 3. SLO definitions

Values below are from the in-memory-cache run, `reports/metrics.json` (300 requests across the 3 default scenarios).

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 99.33% | Met |
| Latency P95 | < 2500 ms | 315.1 ms | Met |
| Fallback success rate | >= 95% | 97.3% | Met |
| Cache hit rate | >= 10% | 63.0% | Met |
| Recovery time | < 5000 ms | 2232.1 ms | Met |

## 4. Metrics

From `reports/metrics.json` (cache backend = memory):

| Metric | Value |
|---|---:|
| total_requests | 300 |
| availability | 0.9933 |
| error_rate | 0.0067 |
| latency_p50_ms | 275.51 |
| latency_p95_ms | 315.1 |
| latency_p99_ms | 317.49 |
| fallback_success_rate | 0.973 |
| cache_hit_rate | 0.63 |
| circuit_open_count | 8 |
| recovery_time_ms | 2232.10 |
| estimated_cost | 0.046052 |
| estimated_cost_saved | 0.189 |

## 5. Cache comparison

Same config and same 300-request load test, run twice — once with `cache.enabled: false` (`reports/metrics_no_cache.json`) and once with `cache.enabled: true` (`reports/metrics.json`).

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| availability | 0.9667 | 0.9933 | +0.0266 (cache absorbs some load that would otherwise hit flaky/open providers) |
| latency_p50_ms | 283.26 | 275.51 | -7.75 ms |
| latency_p95_ms | 316.6 | 315.1 | -1.5 ms |
| estimated_cost | 0.118586 | 0.046052 | -0.072534 (about 61% lower) |
| cache_hit_rate | 0 | 0.63 | +0.63 |
| circuit_open_count | 27 | 8 | -19 (fewer requests reach providers at all with cache on, so fewer failures accumulate) |

**Takeaway:** enabling the cache cut estimated cost by roughly 61% and reduced how often the circuit breaker had to open (27 to 8), because a large share of repeat/near-duplicate queries never reach the unreliable `primary` provider at all. The latency improvement at P50/P95 is smaller than expected, since `FakeLLMProvider` latency here is dominated by the fixed `base_latency_ms` + jitter rather than by cache misses — the main win from caching in this lab is on **cost and provider load**, not raw latency.

## 6. Redis shared cache

- **Why in-memory cache is insufficient for multi-instance deployments**: `ResponseCache` keeps its entries in a Python list inside one process. If the gateway is scaled to multiple instances (e.g. behind a load balancer, or multiple Docker/K8s replicas), each instance has its own private cache — a query cached by instance A is invisible to instance B, so the effective hit rate drops and identical requests can be re-sent to the (possibly degraded) provider from every instance independently.
- **How `SharedRedisCache` solves this**: it stores every cache entry as a Redis hash (`query`, `response`) keyed by `rl:cache:<hash-of-query>`, with `EXPIRE` handling TTL automatically. Because all gateway instances point at the same Redis server, a cache write from any instance is immediately visible as a cache hit to every other instance — this was confirmed both by the automated test `test_shared_state_across_instances` (two separate `SharedRedisCache` objects on the same Redis, one writes, the other reads it back) and by the manual chaos run below.

### Evidence of shared state

Ran the full 300-request chaos simulation with `cache.backend: redis` (`reports/metrics_redis.json`), then inspected Redis directly right after:

```
availability: 0.9933
cache_hit_rate: 0.6767
estimated_cost_saved: 0.203
```

### Redis CLI output

```
PS> docker compose exec redis redis-cli KEYS "rl:cache:*"
 1) "rl:cache:9e413fd814eb"
 2) "rl:cache:3dab98c0e49e"
 3) "rl:cache:4fc3c69b9376"
 4) "rl:cache:3936614ac4c2"
 5) "rl:cache:844ef0143a5c"
 6) "rl:cache:fff10da1c72c"
 7) "rl:cache:98332d0d1c9c"
 8) "rl:cache:734852f3cf4a"
 9) "rl:cache:da61fb49b4f6"
10) "rl:cache:dacb2b833659"
11) "rl:cache:0bc3b1acf73d"
12) "rl:cache:d354658dc020"
13) "rl:cache:095946136fea"

PS> docker compose exec redis redis-cli HGETALL "rl:cache:095946136fea"
1) "response"
2) "[backup] reliable answer for: Explain circuit breaker states in one paragraph."
3) "query"
4) "Explain circuit breaker states in one paragraph."
```

13 distinct queries were persisted in Redis (one hash per unique query out of the 20 sample queries — the rest were served as cache hits rather than new writes), each with both the original `query` text and the cached `response`, confirming the data survives outside any single Python process and would be visible to any other gateway instance connected to the same Redis.

### In-memory vs Redis latency comparison

| Metric | In-memory cache | Redis cache | Notes |
|---|---:|---:|---|
| latency_p50_ms | 275.51 | 271.54 | Comparable — dominated by `FakeLLMProvider`'s simulated latency, not by cache backend |
| latency_p95_ms | 315.1 | 312.31 | Comparable |
| cache_hit_rate | 0.63 | 0.6767 | Slightly higher on Redis run (run-to-run variance from `random.choice` query selection, not a backend effect) |

## 7. Chaos scenarios

All three default scenarios passed in the in-memory-cache run (`reports/metrics.json`):

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | Primary fails 100% of the time; all traffic should fall back to backup, primary's circuit opens quickly | Primary's circuit breaker opened after 3 consecutive failures and stayed open; almost all requests in this scenario were served by `backup` (route=fallback) or cache | pass |
| primary_flaky_50 | Primary fails ~50% of the time; circuit should oscillate between CLOSED/OPEN/HALF_OPEN as it keeps failing the probe and recovering | `circuit_open_count` shows multiple open transitions across the run (8 total across all 3 scenarios combined), consistent with the circuit tripping and re-probing repeatedly under flaky conditions | pass |
| all_healthy | Both providers healthy (primary fail_rate stays at its baseline 0.25 from config since no override is applied); most traffic served by primary, backup used only for primary's baseline failures | availability stayed at 99.33%, fallback_success_rate 97.3% — most requests succeeded either via cache, primary, or a quick fallback to backup | pass |

Note: `all_healthy` overrides no fail rates, so it still runs with `primary`'s baseline 25% fail rate from `configs/default.yaml` — it is a "no injected chaos" baseline, not a 0%-failure baseline. This is worth calling out explicitly since the scenario name is a bit misleading.

## 8. Failure analysis

**Weakness: `success_threshold: 1` lets a single lucky probe fully re-open traffic to a still-unhealthy provider.**

Under `primary_flaky_50`, `primary` fails roughly half the time. When its circuit is OPEN and moves to HALF_OPEN after `reset_timeout_seconds`, the gateway sends exactly one probe request. With `success_threshold: 1`, if that one probe happens to succeed — which has about a 50% chance purely by luck under this scenario — the circuit closes immediately and every subsequent request is routed back to `primary` at full volume, even though the underlying provider is still failing half the time. The very next request has a coin-flip chance of failing again, so the breaker can oscillate CLOSED → OPEN → HALF_OPEN → CLOSED → OPEN in rapid succession. This matches what `circuit_open_count` shows in the metrics (8 open transitions across the run) — the breaker is doing a lot of flapping rather than settling into a stable state. Each flap sends at least one live request to a still-degraded provider before the breaker reacts again, which is exactly the kind of instability a circuit breaker is meant to prevent.

**Weakness: circuit breaker state is per-process, not shared, even though the cache is shared via Redis.**

`CircuitBreaker` objects live in the Python process's memory (`self.breakers` in `ReliabilityGateway`). If this gateway were scaled to multiple instances behind a load balancer, each instance would independently track failures for `primary` and `backup` and could disagree about circuit state at the same moment — e.g. instance A's circuit for `primary` could be OPEN (it just saw 3 failures) while instance B's circuit for the same provider is still CLOSED (it happened to get luckier requests) and keeps sending live traffic to a provider that is, in aggregate across the fleet, clearly unhealthy. This defeats part of the purpose of a circuit breaker in a multi-instance deployment: the "3 consecutive failures" threshold is only ever consecutive **from one instance's point of view**, so a provider that is actually failing 25% of all requests fleet-wide might never trip any single instance's breaker if the failures happen to be spread across instances.

**What I would change before production:**

1. Raise `success_threshold` for `primary` (e.g. to 2-3) so a single lucky probe is not enough to fully re-open traffic — this directly reduces the flapping seen in `primary_flaky_50`.
2. Move circuit breaker counters into Redis (`INCR`/`EXPIRE` on a per-provider key, with a small sliding window instead of an in-memory counter) so that all gateway instances share one view of each provider's health, the same way the cache is already shared. This turns "3 consecutive failures" into "3 failures fleet-wide within the last N seconds," which is a much more meaningful signal when there is more than one instance.
3. Add a minimum sample size before letting the breaker act on a percentage-based signal (e.g. don't open on the first 3 failures if fewer than 5 total requests have been made recently) — this would specifically help the `primary_flaky_50` scenario, where the "correct" long-term signal is a stable ~50% failure rate, not a short burst of 3 failures that could just be noise.

## 9. Next steps

1. **Share circuit breaker state via Redis.** Store `failure_count`/`state`/`opened_at` per provider as Redis keys with `INCR` and `EXPIRE`, alongside the existing shared cache. This makes the breaker's decisions consistent across every gateway instance instead of each instance reacting to its own local view of provider health, closing the gap identified in the failure analysis above.
2. **Add cost-budget-aware routing (the bonus TODO in `gateway.py`).** Track cumulative `estimated_cost` per time window; once it crosses ~80% of a configured budget, skip the more expensive provider and prefer cache-only or the cheaper provider; at 100% of budget, serve cache hits or the static fallback only. This would turn `estimated_cost_saved` from a passive metric into something the gateway actively optimizes for.
3. **Tune `success_threshold` per provider instead of one global default.** `primary` (fail_rate 0.25, flaky under chaos) and `backup` (fail_rate 0.05, generally healthy) do not need the same recovery confidence — `primary` should probably require 2-3 consecutive successful probes before being trusted again, while `backup` can safely keep `success_threshold: 1`. This is a small config change but should meaningfully reduce the flapping seen under `primary_flaky_50` without slowing down recovery for genuinely healthy providers.