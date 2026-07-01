# Day 10 Reliability Report

## 1. Architecture summary

The system is a production-style LLM gateway that routes incoming queries through an intelligent pipeline:
1. **Semantic Cache**: Checks memory or Redis for a semantically similar previous query (using n-gram cosine similarity) with privacy and false-hit guardrails.
2. **Circuit Breakers**: Protects downstream providers from retry storms using a 3-state machine (CLOSED → OPEN → HALF_OPEN).
3. **Provider Fallback Chain**: If the primary provider fails (or its circuit is OPEN), traffic falls back seamlessly to backup providers.
4. **Static Fallback**: If all providers fail, a graceful degraded response is returned to the user.

```
User Request
    |
    v
[Gateway] ---> [Cache check] ---> HIT? return cached
    |                                 |
    v                                 v MISS
[Circuit Breaker: Primary] -------> Provider A
    |  (OPEN? skip)
    v
[Circuit Breaker: Backup] --------> Provider B
    |  (OPEN? skip)
    v
[Static fallback message]
```

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Prevents tripping on 1-2 random network blips, but fails fast on hard outages. |
| reset_timeout_seconds | 1.0 | Short enough for testing/lab, but in production this should be 30s - 60s. |
| success_threshold | 1 | A single successful probe is enough to close the circuit in this lab. |
| cache TTL | 60 | Keeps data fresh while avoiding stale false hits. |
| similarity_threshold | 0.85 | High enough to avoid semantic collisions, low enough to catch rephrased questions. |
| load_test requests | 100/scenario | Generates enough traffic to see circuit transitions and cache hits without taking too long. |

## 3. SLO definitions

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 98.67% | Almost (due to simulated hard failures without enough backups) |
| Latency P95 | < 2500 ms | 317.81 ms | Yes |
| Fallback success rate | >= 95% | 94.29% | Almost |
| Cache hit rate | >= 10% | 63.33% | Yes |
| Recovery time | < 5000 ms | 2280 ms | Yes |

## 4. Metrics

Here is a summary of the metrics generated from `reports/metrics.json`.

| Metric | Value |
|---|---:|
| availability | 0.9867 |
| error_rate | 0.0133 |
| latency_p50_ms | 273.09 |
| latency_p95_ms | 317.81 |
| latency_p99_ms | 320.36 |
| fallback_success_rate | 0.9429 |
| cache_hit_rate | 0.6333 |
| estimated_cost_saved | 0.19 |
| circuit_open_count | 8 |
| recovery_time_ms | 2280.0 |

## 5. Cache comparison

(Metrics here reflect the difference caching makes in cost and latency)

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| latency_p50_ms | ~800.0 | 273.09 | -66% latency |
| latency_p95_ms | ~950.0 | 317.81 | -66% latency |
| cache_hit_rate | 0 | 0.6333 | +63.33% |

## 6. Redis shared cache

Why in-memory cache is insufficient for multi-instance deployments:
- Each load-balanced instance of the gateway has an isolated memory space. 
- A request hitting Instance A is cached there, but if the same user asks the same question and hits Instance B, it's a cache miss, resulting in redundant LLM costs.

How `SharedRedisCache` solves this:
- Extracts cache state out of the application processes into a centralized Redis server.
- All gateway instances read/write to the same Redis keyspace. A cache hit on Instance A immediately benefits Instance B.

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | All traffic fallback to backup, circuit opens | Primary circuit opened, backups handled traffic. | pass |
| primary_flaky_50 | Circuit oscillates, mix of primary and fallback | Circuit transitioned OPEN/HALF_OPEN repeatedly. | pass |
| all_healthy | All requests via primary, no circuit opens | Traffic stayed on primary, 0 circuit opens. | pass |

## 8. Failure analysis

**Weakness**:
If Redis goes down, the current `SharedRedisCache` implementation raises connection errors or silently fails, dropping all caching benefits and potentially causing errors in the gateway.

**Fix before production**:
Implement a graceful degradation mechanism where if `SharedRedisCache.ping()` fails or raises a Redis connection error, the gateway seamlessly falls back to `ResponseCache` (in-memory) so caching continues locally until Redis recovers.

## 9. Next steps

1. Add graceful fallback from Redis to In-memory cache.
2. Implement cost-aware routing (route to cheaper models when nearing the budget limit).
3. Persist the Circuit Breaker states in Redis so that multiple gateway instances share the same OPEN/CLOSED state.