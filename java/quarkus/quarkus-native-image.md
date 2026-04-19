---
title: "Quarkus Native Image — GraalVM, Mandrel, and Serverless Deployment"
date: 2026-04-19
updated: 2026-04-19
tags: [quarkus, graalvm, native-image, mandrel, serverless]
---

# Quarkus Native Image — GraalVM, Mandrel, and Serverless Deployment

**Date:** 2026-04-19 | **Updated:** 2026-04-19
**Tags:** `quarkus` `graalvm` `native-image` `mandrel` `serverless`

## Table of Contents

- [Summary](#summary)
- [Why Quarkus Is Native-Image-Friendly](#why-quarkus-is-native-image-friendly)
- [GraalVM vs Mandrel](#graalvm-vs-mandrel)
- [Building a Native Image](#building-a-native-image)
  - [Local Build](#local-build)
  - [Container Build (No Local GraalVM)](#container-build-no-local-graalvm)
- [Limitations and Workarounds](#limitations-and-workarounds)
- [Testing Native Images](#testing-native-images)
- [Performance Characteristics](#performance-characteristics)
- [Serverless Deployment](#serverless-deployment)
  - [AWS Lambda](#aws-lambda)
  - [Google Cloud Run](#google-cloud-run)
  - [Kubernetes](#kubernetes)
- [JVM Mode vs Native — When to Use Each](#jvm-mode-vs-native--when-to-use-each)
- [Related](#related)
- [References](#references)

---

## Summary

Quarkus was designed from the ground up for [GraalVM native image](https://www.graalvm.org/latest/reference-manual/native-image/) compilation. Its build-time processing (ArC CDI, Jandex indexing, config validation) eliminates most runtime reflection, making native compilation far more reliable than frameworks that depend on runtime reflection. The result: **20–60ms startup, 30–50 MB memory, 40–60 MB disk** — numbers that make Java viable for serverless and scale-to-zero workloads. [Mandrel](https://github.com/graalvm/mandrel), Red Hat's downstream GraalVM distribution, is the recommended build tool for Linux containers.

---

## Why Quarkus Is Native-Image-Friendly

GraalVM native image performs ahead-of-time (AOT) compilation with **closed-world analysis** — it must know every class, method, and field at build time. This conflicts with frameworks that use:

- Runtime reflection (Spring's `@Autowired` scanning)
- Dynamic proxies (CGLIB for AOP, JDK proxies for interfaces)
- Runtime class loading
- Serialization of arbitrary objects

Quarkus avoids all of these by design:

| Challenge | Spring Boot Approach | Quarkus Approach |
|-----------|---------------------|-----------------|
| **Bean discovery** | Runtime classpath scan | Build-time Jandex index |
| **DI wiring** | Reflection at startup | ArC generates bytecode at build time |
| **Proxies** | CGLIB/JDK dynamic proxies | Gizmo-generated classes at build time |
| **Config** | Runtime property binding | Build-time validation and code generation |
| **Reflection hints** | `reflect-config.json` (manual or Spring AOT) | Extensions auto-register — rarely manual |

This means most Quarkus extensions work in native mode out of the box, while Spring Boot requires Spring AOT processing and often manual `@RegisterReflectionForBinding` annotations.

---

## GraalVM vs Mandrel

| Aspect | GraalVM CE | GraalVM Oracle | Mandrel |
|--------|-----------|---------------|---------|
| **Vendor** | Oracle (community) | Oracle (commercial) | Red Hat |
| **Purpose** | General-purpose polyglot VM | Enterprise GraalVM | Quarkus-optimized native-image |
| **Platforms** | Linux, macOS, Windows | Linux, macOS, Windows | Linux (primary), macOS ARM |
| **Polyglot** | Yes (JS, Python, Ruby, etc.) | Yes | No — native-image only |
| **License** | GPLv2 + CE | GFTC (free for dev/prod) | GPLv2 + CE |
| **Quarkus recommendation** | Yes | Yes | Yes (primary for Linux containers) |

**Mandrel** is a GraalVM distribution that strips out everything except `native-image`. It's lighter, faster to download, and purpose-built for building Quarkus applications as Linux container images. Use it for CI/CD pipelines and Docker multi-stage builds.

For local development on macOS (amd64), use GraalVM CE or Oracle GraalVM — Mandrel historically had limited macOS x86 support. On macOS ARM (M-series), Mandrel works well.

---

## Building a Native Image

### Local Build

Prerequisites: GraalVM or Mandrel installed, `GRAALVM_HOME` set.

```bash
# Build native executable
quarkus build --native

# Or via Maven
mvn package -Dnative

# Run it
./target/my-app-1.0.0-runner
# Started in 0.023s
```

Build time: **2–5 minutes** depending on application size and available RAM. Native image builds are CPU and memory intensive — recommend **4+ CPU cores and 4+ GB RAM**.

### Container Build (No Local GraalVM)

Build a native image inside a container — no local GraalVM installation needed:

```bash
# Build native inside container, produce a Linux binary
quarkus build --native -Dquarkus.native.container-build=true

# Build and create a Docker image in one step
quarkus build --native -Dquarkus.container-image.build=true
```

Multi-stage Dockerfile:

```dockerfile
# Stage 1: Build native image
FROM quay.io/quarkus/ubi-quarkus-mandrel-builder-image:jdk-21 AS build
COPY --chown=quarkus:quarkus . /code
WORKDIR /code
RUN ./mvnw package -Dnative -DskipTests

# Stage 2: Runtime image
FROM quay.io/quarkus/quarkus-micro-image:2.0
COPY --from=build /code/target/*-runner /application
EXPOSE 8080
CMD ["./application", "-Dquarkus.http.host=0.0.0.0"]
```

Final image size: **~50–80 MB** (compared to ~300+ MB for a JVM-based image with a full JDK).

---

## Limitations and Workarounds

Native image has fundamental constraints from closed-world analysis:

| Limitation | Impact | Workaround |
|-----------|--------|-----------|
| **Reflection** | Classes accessed via reflection must be declared | Extensions auto-register; manual: `@RegisterForReflection` |
| **Dynamic proxies** | JDK proxies need explicit registration | Quarkus uses build-time Gizmo proxies instead |
| **Serialization** | Java serialization requires class registration | Use JSON (Jackson/JSON-B) instead; register with `@RegisterForReflection(serialization = true)` |
| **Resource bundles** | Must be declared at build time | `quarkus.native.resources.includes=**/*.properties` |
| **Class loading** | No dynamic class loading at runtime | All classes must be known at build time |
| **JNI** | Native methods need explicit configuration | Handled by Quarkus extensions for known libs |
| **Build time** | 2–5 minutes (vs seconds for JVM) | Use JVM mode for dev; native for CI/deploy |
| **Debugging** | Limited debugging support | Use JVM mode for debugging; native for prod |

### When Things Break

```java
// This FAILS in native — Class.forName is runtime reflection
Object obj = Class.forName("com.example.MyClass").getDeclaredConstructor().newInstance();

// Fix: register for reflection
@RegisterForReflection
public class MyClass { ... }

// Or in application.properties
quarkus.native.additional-build-args=\
  -H:ReflectionConfigurationFiles=reflection-config.json
```

Most issues arise from **third-party libraries** that use reflection internally. Quarkus extensions handle these automatically — if a library has a Quarkus extension, native mode should work. If not, you may need manual configuration.

---

## Testing Native Images

Quarkus provides `@QuarkusIntegrationTest` for testing the native binary:

```java
import io.quarkus.test.junit.QuarkusIntegrationTest;
import static io.restassured.RestAssured.given;

@QuarkusIntegrationTest  // tests the actual native binary (or JVM jar)
public class FruitResourceIT {

    @Test
    void testListFruits() {
        given()
            .when().get("/fruits")
            .then()
            .statusCode(200);
    }
}
```

Run with:

```bash
# Build native and run integration tests
mvn verify -Dnative
```

**Important distinction:**
- `@QuarkusTest` — starts the application in JVM mode, uses CDI injection in tests
- `@QuarkusIntegrationTest` — starts the built artifact (native binary or JAR) as a separate process, tests via HTTP only

---

## Performance Characteristics

### Startup Time

| Mode | Startup | Notes |
|------|---------|-------|
| **JVM** | ~0.8–1.2s | Good for long-running services |
| **Native** | ~0.02–0.05s | Essential for serverless, scale-to-zero |
| **Native (complex app)** | ~0.1–0.3s | More extensions = more startup work |

### Memory (RSS)

| Mode | Idle | Under Load | Notes |
|------|------|-----------|-------|
| **JVM** | ~80–130 MB | ~200–400 MB | JIT warms up, GC overhead |
| **Native** | ~30–50 MB | ~60–120 MB | No JIT, no class metadata overhead |

### Throughput

**Important caveat**: JVM mode often has **higher peak throughput** than native mode because the JIT compiler optimizes hot paths at runtime. Native image uses AOT compilation without profile-guided optimization (by default), which produces less optimized machine code for hot loops.

| Scenario | JVM | Native |
|----------|-----|--------|
| **Cold start** | Slow (class loading, JIT warmup) | Fast (pre-compiled) |
| **Warm throughput** | Higher (JIT-optimized) | Lower (AOT-compiled) |
| **Tail latency (p99)** | Higher variance (GC pauses, JIT) | More consistent (no JIT deopt) |
| **Scale-to-zero viability** | No (too slow to cold start) | Yes (20ms startup) |

Choose native for **startup-sensitive** workloads. Choose JVM for **throughput-sensitive** long-running services.

---

## Serverless Deployment

### AWS Lambda

```bash
quarkus extension add amazon-lambda-rest

# Build native Lambda deployment package
quarkus build --native -Dquarkus.native.container-build=true
# Produces target/function.zip

# Deploy
aws lambda create-function \
  --function-name my-quarkus-fn \
  --runtime provided.al2023 \
  --handler not.used \
  --zip-file fileb://target/function.zip
```

Cold start: **~200ms** (native) vs **~3–8s** (JVM). This makes native Quarkus competitive with Go and Rust Lambda functions.

### Google Cloud Run

```bash
# Build container with native image
quarkus build --native \
  -Dquarkus.container-image.build=true \
  -Dquarkus.container-image.push=true \
  -Dquarkus.container-image.registry=gcr.io \
  -Dquarkus.container-image.group=my-project

# Deploy
gcloud run deploy my-service \
  --image gcr.io/my-project/my-app:latest \
  --min-instances 0 \
  --max-instances 10
```

Scale-to-zero with ~50ms cold start — viable because native Quarkus starts fast enough.

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-quarkus-app
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: app
          image: registry/my-app:native
          resources:
            requests:
              memory: "64Mi"    # native needs much less
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /q/health/ready
              port: 8080
            initialDelaySeconds: 1   # native starts in <1s
```

---

## JVM Mode vs Native — When to Use Each

| Factor | JVM Mode | Native Mode |
|--------|---------|-------------|
| **Development** | Always (fast builds, full debugging) | Never |
| **Long-running services** | Preferred (JIT throughput) | Consider if memory-constrained |
| **Serverless / Lambda** | Only if cold start doesn't matter | Preferred (20–200ms cold start) |
| **Scale-to-zero** | Not viable (~1–3s startup) | Ideal |
| **CI build time** | Fast (~30s) | Slow (~2–5 min) |
| **Third-party lib compat** | Everything works | Some libs need configuration |
| **Debugging** | Full support | Limited |
| **Peak throughput** | Higher (JIT) | Lower (AOT) |

**Recommendation**: develop and test in JVM mode. Deploy as native only when startup time or memory footprint justifies the build complexity and throughput tradeoff.

---

## Related

- [Quarkus Overview](quarkus-overview.md) — build-time architecture that enables native compilation
- [Quarkus Extensions](quarkus-extensions.md) — extensions provide native-image support automatically
- [GraalVM Native Image for Spring Boot](../configurations/graalvm-native-image.md) — Spring's native story for comparison
- [Helidon Overview](../helidon/helidon-overview.md) — Helidon SE's lean reflection surface also helps native
- [Spring Boot on AWS and GCP](../cloud/spring-boot-aws-gcp.md) — Spring's serverless deployment for comparison

## References

- [Building a Native Executable — Quarkus](https://quarkus.io/guides/building-native-image) — official guide
- [Mandrel — GitHub](https://github.com/graalvm/mandrel) — Red Hat's GraalVM distribution for Quarkus
- [GraalVM Native Image Reference](https://www.graalvm.org/latest/reference-manual/native-image/) — native-image documentation
- [Quarkus AWS Lambda Guide](https://quarkus.io/guides/amazon-lambda) — serverless deployment
- [Quarkus Container Images Guide](https://quarkus.io/guides/container-image) — building and pushing container images
- [GraalVM Native Image for Spring Boot/Quarkus — Java Code Geeks](https://www.javacodegeeks.com/2025/08/graalvm-native-image-for-spring-boot-quarkus-step-by-step-production-example.html) — step-by-step comparison
