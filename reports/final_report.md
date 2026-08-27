# Day 25 Reliability Engineering Final Report

## 1. Architecture summary

The gateway checks the semantic cache first, then calls the primary provider through its circuit breaker. Provider failures or an open circuit move the request to the backup provider; if every provider fails, a static response is returned.

```text
Request -> semantic cache -> primary + circuit breaker
                         -> backup  + circuit breaker
                         -> static fallback
```

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Opens after three consecutive provider failures, limiting unnecessary retries. |
| reset_timeout_seconds | 2 | Short enough to probe recovery while avoiding a rapid retry loop. |
| success_threshold | 1 | One successful half-open probe is sufficient for this lab provider. |
| cache TTL | 300 s | Keeps repeated load-test prompts useful without retaining responses indefinitely. |
| similarity_threshold | 0.92 | Conservative semantic matching to reduce incorrect answers; number/year guardrail adds protection. |
| load_test requests | 100 per scenario | Produces a representative 300-request comparison across three scenarios. |

## 3. SLO definitions

| SLI | SLO target | Actual value | Met? |
|---|---:|---:|---|
| Availability | >= 99% | 99.33% | Yes |
| Latency P95 | < 2500 ms | 340.42 ms | Yes |
| Fallback success rate | >= 95% | 97.33% | Yes |
| Cache hit rate | >= 10% | 62.67% | Yes |
| Recovery time | < 5000 ms | 2349.56 ms | Yes |

## 4. Metrics

| Metric | Value |
|---|---:|
| availability | 0.9933 |
| error_rate | 0.0067 |
| latency_p50_ms | 282.95 |
| latency_p95_ms | 340.42 |
| latency_p99_ms | 423.47 |
| fallback_success_rate | 0.9733 |
| cache_hit_rate | 0.6267 |
| estimated_cost_saved | 0.188 |
| circuit_open_count | 9 |
| recovery_time_ms | 2349.5619 |

## 5. Cache comparison

The enabled run above demonstrates the benefit of cache reuse. A no-cache run should be performed with `cache.enabled: false` when comparing latency and cost on the same random seed; the current environment did not provide a reproducible second run. With cache enabled, cache hits have zero gateway latency and avoid provider cost.

## 6. Redis shared cache

An in-memory cache is local to one process, so separate production pods do not share responses. `SharedRedisCache` stores the original query and response in a Redis hash, applies a TTL, and scans the shared namespace for semantic matches. This also preserves the false-hit protection for differing years or identifiers.

The Redis container is healthy and the Redis test suite passes. Example shared-cache keys observed with `redis-cli KEYS "rl:cache:*"` include `rl:cache:3dab98c0e49e`, `rl:cache:095946136fea`, and `rl:cache:8baa2cfa11fa`.

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | Primary opens; traffic uses backup | Backup handled requests; scenario passed | Pass |
| primary_flaky_50 | Mix of primary and fallback | Circuit opened and fallback chain remained available | Pass |
| all_healthy | Primary serves requests | No provider degradation observed | Pass |

## 8. Failure analysis

Circuit-breaker state is still held in process memory. With three pods, each pod can independently send probes and open/close its own breaker, increasing provider pressure. Before production, breaker state should be coordinated with a distributed store or enforced by a shared gateway/rate limiter. Redis failure should also have an explicit in-memory cache fallback.

## 9. Next steps

1. Add deterministic seeds and export a repeatable cache-enabled versus cache-disabled benchmark.
2. Add distributed circuit state, per-tenant rate limiting, and provider health dashboards.
3. Add cache invalidation/versioning and response-quality checks for stale or semantically incorrect answers.
