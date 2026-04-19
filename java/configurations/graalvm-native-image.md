---
title: "GraalVM Native Image for Spring Boot — AOT, Reachability, Cold Starts"
date: 2026-04-19
updated: 2026-04-19
tags: [graalvm, native-image, spring-boot, aot, serverless]
---

# GraalVM Native Image for Spring Boot — AOT, Reachability, Cold Starts

**Date:** 2026-04-19 | **Updated:** 2026-04-19
**Tags:** `graalvm` `native-image` `spring-boot` `aot` `serverless`

## Table of Contents

- [Summary](#summary)
- [Why Native Image](#why-native-image)
- [AOT vs JIT Trade-offs](#aot-vs-jit-trade-offs)
- [Reachability and the Closed World](#reachability-and-the-closed-world)
- [Setting Up Spring Boot Native](#setting-up-spring-boot-native)
- [Reflection, Resources, and Hints](#reflection-resources-and-hints)
- [Build Times and Memory](#build-times-and-memory)
- [When Not to Use Native](#when-not-to-use-native)
- [CRaC as an Alternative](#crac-as-an-alternative)
- [Related](#related)
- [References](#references)

---

## Summary

[GraalVM Native Image](https://www.graalvm.org/latest/reference-manual/native-image/) ahead-of-time compiles a Java app to a standalone native executable — no JVM, no classloader, ~50 MB binary, starts in 30–100 ms, uses 50–80% less memory. Spring Boot 3.x has first-class support via [Spring AOT](https://docs.spring.io/spring-boot/docs/current/reference/html/native-image.html). The trade-off is a **closed-world assumption**: everything the app can reach at runtime must be known at build time. Reflection, resource loading, dynamic proxies, and serialization need hints. Build times go from seconds (JIT JAR) to minutes (native). Peak throughput is usually lower than JIT because the JIT has runtime profiling the AOT compiler doesn't. The sweet spot: AWS Lambda, Cloud Run, CLIs, and containers where fast startup and low memory matter more than peak throughput. For long-running servers, [CRaC](https://docs.azul.com/core/crac/crac-introduction) is often a simpler path to the same goals.

---

## Why Native Image

Three numbers that change: startup, memory, and throughput.

| Metric | JIT (HotSpot) | Native Image |
|--------|---------------|--------------|
| Cold start | 2–10 s | 30–150 ms |
| First request warm | 200–2000 ms | 30–100 ms |
| Memory (RSS at idle) | 250–500 MB | 80–150 MB |
| Peak throughput | 100% (baseline) | 60–90% |
| Build time | 5–30 s | 60–300 s |

For AWS Lambda, Cloud Run, or anywhere you scale to zero, startup latency dominates the user experience. A Spring Boot service that takes 5s to cold-start is a non-starter on Lambda; native image makes it viable.

For long-running K8s pods, peak throughput matters more than startup. Native is net-negative there unless you're memory-constrained.

---

## AOT vs JIT Trade-offs

- **JIT** observes runtime behavior, inlines hot paths, deoptimizes cold ones. Superior peak throughput, slow warmup.
- **AOT** compiles ahead of time with static analysis. Fast startup, predictable behavior, but no speculative optimization.

GraalVM includes both: `graalvm-ce` (the JIT JVM with Graal as C2 replacement) and `native-image` (the AOT compiler). This doc is about the latter.

---

## Reachability and the Closed World

Native image performs **points-to analysis** at build time. Only classes, methods, and fields that are reachable from the entry point are included. Everything else is dead-code-eliminated.

This breaks common JVM patterns:

- **Reflection** — `Class.forName("com.foo.Bar")` with a runtime string. Native can't know `Bar` is reachable.
- **Dynamic proxies** — JDK `Proxy.newProxyInstance` with interfaces passed at runtime.
- **Serialization** — `ObjectInputStream.readObject` materializes arbitrary classes.
- **Resource loading** — `getResource("messages.properties")` paths unknown until runtime.
- **JNI** — native libraries loaded by name.

Fix: provide **reachability metadata** — JSON files telling the native-image compiler "these classes/methods/resources are reachable dynamically". Spring Boot 3 generates most of this automatically via Spring AOT; external libraries ship theirs via the [GraalVM Reachability Metadata repository](https://github.com/oracle/graalvm-reachability-metadata).

---

## Setting Up Spring Boot Native

Dependencies:

```gradle
plugins {
    id 'org.springframework.boot' version '3.2.0'
    id 'org.graalvm.buildtools.native' version '0.10.0'
}
```

Build native executable:

```bash
./gradlew nativeCompile              # produces build/native/nativeCompile/<app>
./build/native/nativeCompile/orders
```

Build native container:

```bash
./gradlew bootBuildImage -PnativeImage=true
docker run -p 8080:8080 orders:latest
```

Under the hood, Spring Boot 3 runs an **AOT processing** step during build:

1. Boot the app in a special AOT mode.
2. Record bean definitions, reflective access, resource paths.
3. Generate Java source for deterministic bean setup (no runtime scanning).
4. Emit reachability metadata JSON.
5. Hand everything to `native-image`.

Result: most Spring features work out of the box — `@RestController`, `@Autowired`, JPA, validation, actuator. Some don't — see caveats below.

---

## Reflection, Resources, and Hints

When Spring AOT misses something, add a hint:

```java
@Configuration
@ImportRuntimeHints(MyHints.class)
public class NativeConfig {

    static class MyHints implements RuntimeHintsRegistrar {
        @Override
        public void registerHints(RuntimeHints hints, ClassLoader cl) {
            hints.reflection()
                .registerType(MyDto.class, MemberCategory.INVOKE_PUBLIC_CONSTRUCTORS,
                                           MemberCategory.DECLARED_FIELDS);
            hints.resources().registerPattern("db/migration/*.sql");
            hints.serialization().registerType(MyDto.class);
            hints.proxies().registerJdkProxy(MyInterface.class);
        }
    }
}
```

For Jackson-deserialized DTOs, `@RegisterReflectionForBinding(MyDto.class)` is shorter.

Detect missing hints at build time by running the [native agent](https://www.graalvm.org/latest/reference-manual/native-image/metadata/AutomaticMetadataCollection/) against the regular JVM app:

```bash
./gradlew bootRun -Pagent
# exercise all code paths
./gradlew metadataCopy
```

The agent records every reflective call, resource load, and proxy and dumps JSON under `src/main/resources/META-INF/native-image/`. Run it against your test suite for best coverage.

---

## Build Times and Memory

Native image build is heavy:

- **Time**: 1–5 min for a small Spring Boot app, 10+ min for large monoliths.
- **Memory**: 4–12 GB during build. CI runners with < 8 GB will OOM.
- **CPU**: usually pins all cores.

Configure:

```gradle
graalvmNative {
    binaries {
        main {
            buildArgs.addAll('-J-Xmx8g', '--no-fallback', '-O2')
        }
    }
}
```

Build once in CI, push the image, deploy the image. Don't rebuild native on every deploy.

---

## When Not to Use Native

- **Throughput-critical services** — the JIT wins at steady state.
- **Heavy reflection / dynamic class loading** — libraries that load plugins at runtime (some app servers, ORMs with lazy discovery) break.
- **Fast iteration dev loop** — 3-minute builds kill productivity. Run JVM in dev, native in prod only.
- **Debugging** — native binaries are harder to profile. `async-profiler` works but heap dumps and JFR are limited.
- **Libraries without reachability metadata** — you become the metadata maintainer.

---

## CRaC as an Alternative

[Coordinated Restore at Checkpoint](https://docs.azul.com/core/crac/crac-introduction) (CRaC) is a JVM feature (Azul Zulu, OpenJDK) that snapshots a running JVM to disk and restores it in ~100 ms. Same cold-start improvement as native, but keeps JIT throughput.

How:

1. Start the app, warm it up with a few requests.
2. `jcmd <pid> JDK.checkpoint` — serializes the JVM to a checkpoint directory.
3. Package the checkpoint with the app.
4. On deploy, `java -XX:CRaCRestoreFrom=<dir>` restores in ~100 ms.

AWS Lambda's [SnapStart](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html) uses a similar mechanism for Java 21 runtimes.

CRaC caveats: checkpointing needs coordination with open file handles, DB connections, and network sockets (resources must be released and reacquired). Spring Boot 3.2+ has [built-in CRaC support](https://docs.spring.io/spring-framework/reference/integration/checkpoint-restore.html).

Rule of thumb: **try CRaC first** for long-running services that need fast startup. Reach for native image when memory footprint or binary size is the hard constraint (Lambda, edge, CLI).

---

## Related

- [Docker and Deployment](docker-and-deployment.md) — image building; native builds produce much smaller images.
- [Kubernetes for Spring Boot](kubernetes-spring-boot.md) — native image shrinks pod startup time.
- [Spring Boot on AWS and GCP](../cloud/spring-boot-aws-gcp.md) — Lambda SnapStart and Cloud Run native use case.
- [Build Tools and JVM](../java-fundamentals/build-tools-and-jvm.md) — JIT basics.
- [GC Collectors](../jvm-gc/collectors.md) — native uses a different GC (Serial GC by default; G1 in Enterprise).
- [Performance Testing](../testing/performance-testing.md) — measure throughput delta before committing to native.

---

## References

- [GraalVM Native Image documentation](https://www.graalvm.org/latest/reference-manual/native-image/)
- [Spring Boot Native Image](https://docs.spring.io/spring-boot/docs/current/reference/html/native-image.html)
- [Spring AOT documentation](https://docs.spring.io/spring-framework/reference/core/aot.html)
- [GraalVM Reachability Metadata repository](https://github.com/oracle/graalvm-reachability-metadata)
- [GraalVM Native Image Gradle plugin](https://graalvm.github.io/native-build-tools/latest/gradle-plugin.html)
- [Coordinated Restore at Checkpoint (CRaC)](https://docs.azul.com/core/crac/crac-introduction)
- [AWS Lambda SnapStart for Java](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html)
- [Spring Boot — Checkpoint Restore](https://docs.spring.io/spring-framework/reference/integration/checkpoint-restore.html)
