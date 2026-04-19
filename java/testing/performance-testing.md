---
title: "Performance Testing — Gatling, k6, JMH, and SLO Gates in CI"
date: 2026-04-19
updated: 2026-04-19
tags: [performance, testing, gatling, k6, jmh, load-testing]
---

# Performance Testing — Gatling, k6, JMH, and SLO Gates in CI

**Date:** 2026-04-19 | **Updated:** 2026-04-19
**Tags:** `performance` `testing` `gatling` `k6` `jmh` `load-testing`

## Table of Contents

- [Summary](#summary)
- [Three Scales of Performance Testing](#three-scales-of-performance-testing)
- [Gatling — Scala/Java Load Tests as Code](#gatling--scalajava-load-tests-as-code)
- [k6 — JavaScript Load Tests](#k6--javascript-load-tests)
- [JMH — Microbenchmarks](#jmh--microbenchmarks)
- [Designing Load Test Scenarios](#designing-load-test-scenarios)
- [SLO Gates in CI](#slo-gates-in-ci)
- [Flame Graphs Under Load](#flame-graphs-under-load)
- [Related](#related)
- [References](#references)

---

## Summary

Three tools cover 95% of Java performance testing: [Gatling](https://gatling.io/) for realistic HTTP/WebSocket load scenarios written in Java/Scala, [k6](https://k6.io/) when you prefer JavaScript and want CNCF/cloud-native tooling, and [JMH](https://github.com/openjdk/jmh) (Java Microbenchmark Harness) for method-level throughput measurements. Each targets a different scale — JMH for a single method, Gatling/k6 for an endpoint under realistic traffic. The production habit worth building: **SLO gates in CI** — every PR runs a short Gatling smoke test, and if p99 latency or error rate regresses beyond budget, the PR blocks. This doc shows the minimum viable setup for all three and the playbook for turning them into regression-detection systems.

---

## Three Scales of Performance Testing

| Scale | Tool | Measures |
|-------|------|----------|
| Method (ns – µs) | JMH | "Is this parser 2x slower than before?" |
| Endpoint (ms) | Gatling / k6 | "p99 under 500 RPS?" |
| System (minutes) | Gatling + chaos tools | "Survives 2x peak for an hour?" |

Pick by question. Don't use Gatling to measure a function — overhead dominates. Don't use JMH to verify DB-backed behavior — no real I/O. Each tool's strength is the other's weakness.

---

## Gatling — Scala/Java Load Tests as Code

Gatling 3.10+ has a full Java DSL (Scala still supported). Tests live in the same repo as the code, run in CI, version-controlled.

Gradle dependency:

```gradle
plugins { id 'io.gatling.gradle' version '3.11.5.2' }
```

Scenario (`src/gatling/java/OrdersSimulation.java`):

```java
import io.gatling.javaapi.core.*;
import io.gatling.javaapi.http.*;
import static io.gatling.javaapi.core.CoreDsl.*;
import static io.gatling.javaapi.http.HttpDsl.*;

public class OrdersSimulation extends Simulation {

    HttpProtocolBuilder httpProtocol = http
        .baseUrl("http://localhost:8080")
        .acceptHeader("application/json")
        .userAgentHeader("Gatling");

    ScenarioBuilder placeOrder = scenario("Place order")
        .exec(http("Create order")
            .post("/api/orders")
            .body(StringBody("""
                { "userId": "u1", "items": [{"productId":"p1","qty":2}] }
                """))
            .check(status().is(201))
            .check(jsonPath("$.id").saveAs("orderId")))
        .pause(1)
        .exec(http("Get order")
            .get("/api/orders/#{orderId}")
            .check(status().is(200)));

    {
        setUp(
            placeOrder.injectOpen(
                rampUsersPerSec(10).to(200).during(60),
                constantUsersPerSec(200).during(300)
            )
        ).protocols(httpProtocol)
         .assertions(
            global().responseTime().percentile3().lt(500),   // p95 < 500ms
            global().responseTime().percentile4().lt(1000),  // p99 < 1000ms
            global().failedRequests().percent().lt(1.0)      // < 1% errors
         );
    }
}
```

Run: `./gradlew gatlingRun`. Output is an HTML report with percentile charts, request distribution, error breakdown.

Gatling's model is **open workload** — simulates N users arriving per second regardless of system capacity. This is more realistic than closed-loop "N concurrent users" for internet-facing APIs.

---

## k6 — JavaScript Load Tests

Scenario (`test.js`):

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
    stages: [
        { duration: '1m', target: 100 },
        { duration: '5m', target: 100 },
        { duration: '1m', target: 0 },
    ],
    thresholds: {
        http_req_duration: ['p(95)<500', 'p(99)<1000'],
        http_req_failed: ['rate<0.01'],
    },
};

export default function () {
    const payload = JSON.stringify({ userId: 'u1', items: [{ productId: 'p1', qty: 2 }] });
    const params = { headers: { 'Content-Type': 'application/json' } };

    const res = http.post('http://localhost:8080/api/orders', payload, params);
    check(res, { 'status 201': (r) => r.status === 201 });
    sleep(1);
}
```

Run: `k6 run test.js`. Built-in metrics, Prometheus remote write, Grafana dashboards, Kubernetes operator ([k6-operator](https://github.com/grafana/k6-operator)). Written in Go — very low overhead per VU (virtual user) compared to Gatling.

**Pick k6 when**: your team is polyglot or JS-heavy, you want to run in Kubernetes clusters as jobs, or you want built-in Prometheus output. **Pick Gatling when**: you're Java-only, want deep DSL expressiveness, or need to share domain code with the service under test.

---

## JMH — Microbenchmarks

For measuring method throughput. Gradle:

```gradle
plugins { id 'me.champeau.jmh' version '0.7.2' }

jmh {
    fork = 1
    warmupIterations = 3
    iterations = 5
    benchmarkMode = ['thrpt']
    timeUnit = 'us'
}
```

Benchmark:

```java
@State(Scope.Benchmark)
public class ParserBenchmark {

    String payload;

    @Setup public void setup() { payload = generatePayload(); }

    @Benchmark
    public Order parseWithJackson(Blackhole bh) {
        return mapper.readValue(payload, Order.class);
    }

    @Benchmark
    public Order parseWithProtobuf(Blackhole bh) {
        return Order.parseFrom(payload.getBytes());
    }
}
```

Run: `./gradlew jmh`. Output: ops/sec with confidence intervals.

Allocation-sensitive benchmark (catches GC regressions — see [jvm-gc/pause-diagnosis.md](../jvm-gc/pause-diagnosis.md)):

```bash
./gradlew jmh -Pjmh.profilers=gc
```

Output includes `gc.alloc.rate.norm` — bytes allocated per operation. If this doubles after a refactor, you created an allocation regression.

Common traps:

- **Dead code elimination** — use `Blackhole.consume(x)` or return the result.
- **Constant folding** — use `@State` fields, not method-local constants.
- **Insufficient warmup** — JIT needs 2–3 iterations to specialize.
- **Wrong mode** — `Throughput` for rate, `AverageTime` for latency, `SingleShotTime` for startup.

Always read [Aleksey Shipilëv — JMH samples](https://github.com/openjdk/jmh/tree/master/jmh-samples/src/main/java/org/openjdk/jmh/samples) before your first serious benchmark.

---

## Designing Load Test Scenarios

A load test answers a specific question. Pick one per test:

1. **Smoke** — "Does it run under any load?" 1 user, 10s.
2. **Baseline** — "What's the steady-state behavior at normal load?" 50% of production peak, 10 min.
3. **Peak** — "Can we handle expected peak?" 100% of peak, 30 min.
4. **Stress** — "Where does it break?" Ramp until errors/latency breach SLO.
5. **Soak** — "Is there a leak?" Run peak for 4–24 hours. Watch heap, direct memory, DB pool.
6. **Spike** — "How does a sudden surge behave?" 0 → 5x peak in 30s.

Smoke + baseline run in CI. Peak + stress run nightly. Soak runs weekly. Spike runs before a product launch.

**Data realism matters**: test with realistic payload sizes, realistic distribution of hot vs cold keys, realistic think times. A load test on a pre-warmed cache with uniform keys tells you nothing about production.

---

## SLO Gates in CI

The habit that makes performance testing stick. GitHub Actions example:

```yaml
- name: Start app
  run: ./gradlew bootRun &

- name: Wait for readiness
  run: until curl -f http://localhost:8080/actuator/health/readiness; do sleep 2; done

- name: Gatling smoke
  run: ./gradlew gatlingRun-SmokeSimulation

- name: Check results
  run: |
    if grep -q 'KO' build/reports/gatling/*/simulation.log; then
      echo "Performance regression"; exit 1
    fi
```

The smoke simulation runs 30s at low load and asserts p95 < 500ms, p99 < 1s, errors < 1%. PRs that regress fail CI the same way unit tests do.

For finer-grained gates, integrate with [k6 thresholds](https://k6.io/docs/using-k6/thresholds/) or parse Gatling's `assertions.json`. Don't just run the test — assert on its result and fail the build.

---

## Flame Graphs Under Load

When performance regresses, the next question is *why*. Run [async-profiler](https://github.com/async-profiler/async-profiler) against the app while the load test runs:

```bash
./gradlew bootRun &
./gradlew gatlingRun-PeakSimulation &
sleep 120
java -jar async-profiler.jar -d 60 -f flame.html <pid>
```

Compare flame graphs before/after. Differential flame graphs ([`async-profiler` differential mode](https://github.com/async-profiler/async-profiler/blob/master/docs/Diff.md)) show which methods got *more* expensive. Allocation profiling (`-e alloc`) catches GC regressions specifically.

Hook this into CI for long-term PR: archive the flame graph as an artifact on every run.

---

## Related

- [Scaling Spring MVC Before Virtual Threads](../web-layer/mvc-high-throughput.md) — the tuning knobs performance tests verify.
- [GC Pause Diagnosis Playbook](../jvm-gc/pause-diagnosis.md) — async-profiler and JMH with GC profiler.
- [GC Impact on Reactive, Virtual Threads, and Streaming](../jvm-gc/reactive-impact.md) — reactive-specific allocation hotspots.
- [Spring Boot Testing Fundamentals](spring-boot-test-basics.md) — the correctness counterpart.
- [Distributed Tracing](../observability/distributed-tracing.md) — tracing captured during load tests identifies slow hops.
- [Kubernetes for Spring Boot](../configurations/kubernetes-spring-boot.md) — run k6 as Kubernetes jobs in staging.

---

## References

- [Gatling documentation](https://docs.gatling.io/)
- [Gatling Java DSL](https://docs.gatling.io/reference/script/core/simulation/#java-kotlin)
- [k6 documentation](https://k6.io/docs/)
- [k6-operator for Kubernetes](https://github.com/grafana/k6-operator)
- [JMH — Java Microbenchmark Harness](https://github.com/openjdk/jmh)
- [JMH Samples](https://github.com/openjdk/jmh/tree/master/jmh-samples) — canonical examples.
- [async-profiler](https://github.com/async-profiler/async-profiler)
- [Brendan Gregg — Flame Graphs](https://www.brendangregg.com/flamegraphs.html)
- [Google SRE Book — Load Balancing and Testing](https://sre.google/sre-book/load-balancing-datacenter/)
