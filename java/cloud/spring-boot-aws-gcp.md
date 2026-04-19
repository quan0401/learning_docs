---
title: "Spring Boot on AWS and GCP — Spring Cloud AWS, Lambda, IRSA, Cloud Run"
date: 2026-04-19
updated: 2026-04-19
tags: [aws, gcp, spring-cloud, lambda, cloud-run, cloud-native]
---

# Spring Boot on AWS and GCP — Spring Cloud AWS, Lambda, IRSA, Cloud Run

**Date:** 2026-04-19 | **Updated:** 2026-04-19
**Tags:** `aws` `gcp` `spring-cloud` `lambda` `cloud-run` `cloud-native`

## Table of Contents

- [Summary](#summary)
- [Identity First — IAM Roles, Not Access Keys](#identity-first--iam-roles-not-access-keys)
- [Spring Cloud AWS](#spring-cloud-aws)
- [Spring Cloud GCP](#spring-cloud-gcp)
- [Lambda with SnapStart and CRaC](#lambda-with-snapstart-and-crac)
- [Cloud Run](#cloud-run)
- [Secret and Config Injection](#secret-and-config-injection)
- [Observability — CloudWatch and Cloud Ops](#observability--cloudwatch-and-cloud-ops)
- [Related](#related)
- [References](#references)

---

## Summary

Java in AWS or GCP in 2026 is less about "the JVM on EC2" and more about which managed service fits the workload: [Lambda](https://docs.aws.amazon.com/lambda/) with [SnapStart](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html) for event-driven and low-traffic APIs, [EKS](https://docs.aws.amazon.com/eks/) / [GKE](https://cloud.google.com/kubernetes-engine) for long-running services, [Cloud Run](https://cloud.google.com/run) / [App Runner](https://docs.aws.amazon.com/apprunner/) for container-as-a-service simplicity. [Spring Cloud AWS](https://awspring.io/) and [Spring Cloud GCP](https://spring.io/projects/spring-cloud-gcp) give you idiomatic bindings — `@EnableSqsListener`, `@ConfigurationProperties` from Parameter Store, Secret Manager, pub/sub templates — so you don't reach for the raw SDK for common patterns. The non-negotiable piece across both clouds: **identity is an IAM role attached to the compute, not an access key in a file**. This doc covers the patterns that show up across Spring Boot deployments and the Lambda-specific wrinkles around cold starts.

---

## Identity First — IAM Roles, Not Access Keys

Never put AWS access keys or GCP service-account JSON files into a container image, k8s secret, or CI variable. Use workload identity:

| Compute | AWS | GCP |
|---------|-----|-----|
| EC2 | Instance profile | — |
| EKS pod | [IRSA (IAM Roles for Service Accounts)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) | [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity) |
| Lambda | Execution role | — |
| Cloud Run | — | Service account |
| ECS / Fargate | Task role | — |
| App Runner | Access role | — |

The app calls `DefaultCredentialsProvider.create()` (AWS SDK v2) or `GoogleCredentials.getApplicationDefault()` (GCP). The SDK discovers the role via a metadata endpoint; tokens are short-lived and rotated automatically; no secrets anywhere.

IRSA setup:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123:role/orders-role
```

```yaml
spec:
  serviceAccountName: orders-sa
```

IAM role's trust policy allows the SA to assume it. No access keys, no secrets, nothing to rotate.

---

## Spring Cloud AWS

The community-maintained ([awspring](https://awspring.io/)) module covers common AWS services.

```gradle
implementation 'io.awspring.cloud:spring-cloud-aws-starter-sqs:3.2.0'
implementation 'io.awspring.cloud:spring-cloud-aws-starter-parameter-store:3.2.0'
implementation 'io.awspring.cloud:spring-cloud-aws-starter-secrets-manager:3.2.0'
implementation 'io.awspring.cloud:spring-cloud-aws-starter-s3:3.2.0'
```

SQS listener:

```java
@Component
public class OrderListener {
    @SqsListener("orders-queue")
    public void handle(OrderMessage msg) {
        orderService.process(msg);
    }
}
```

S3:

```java
@Autowired S3Template s3;

s3.upload("my-bucket", "key.pdf", inputStream);
byte[] bytes = s3.download("my-bucket", "key.pdf").asBytes();
```

Parameter Store as config:

```yaml
spring:
  config:
    import:
      - aws-parameterstore:/config/orders/
```

Values at `/config/orders/db.password` map to `${db.password}` automatically.

---

## Spring Cloud GCP

```gradle
implementation 'com.google.cloud:spring-cloud-gcp-starter-pubsub:5.7.0'
implementation 'com.google.cloud:spring-cloud-gcp-starter-secretmanager:5.7.0'
implementation 'com.google.cloud:spring-cloud-gcp-starter-storage:5.7.0'
```

Pub/Sub listener:

```java
@Service
public class Listener {
    @PubSubListener("orders-subscription")
    public void handle(BasicAcknowledgeablePubsubMessage msg) {
        // ...
        msg.ack();
    }
}
```

Secret Manager:

```yaml
spring:
  config:
    import:
      - sm://projects/my-project/secrets/db-password/versions/latest
```

Cloud Storage via `Storage` bean (same mental model as S3Template).

---

## Lambda with SnapStart and CRaC

Java on Lambda traditionally suffers from 3–8 second cold starts — Spring Boot makes it worse (context init). Two mitigations:

**1. SnapStart** (Lambda-native): Lambda snapshots an initialized JVM and restores it on each cold start. Available for Java 17+ runtimes. 90%+ cold-start reduction for free.

```yaml
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: java21
      SnapStart:
        ApplyOn: PublishedVersions
```

Only applies to published versions, not `$LATEST`. Deploy via alias that points to the latest published version. Spring Boot 3.2+ has [SnapStart-aware](https://docs.spring.io/spring-framework/reference/integration/checkpoint-restore.html) lifecycle hooks to handle open connections across the snapshot boundary.

**2. Spring Cloud Function + native image**: for sub-100ms cold starts, compile the Lambda handler to a native executable via GraalVM. See [graalvm-native-image.md](../configurations/graalvm-native-image.md).

Lambda handler pattern (Spring Cloud Function):

```gradle
implementation 'org.springframework.cloud:spring-cloud-function-adapter-aws:4.1.0'
```

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) { SpringApplication.run(App.class, args); }

    @Bean
    public Function<Order, Receipt> processOrder() {
        return order -> orderService.process(order);
    }
}
```

The function bean is the Lambda handler — same code runs locally, in tests, or in Lambda.

---

## Cloud Run

GCP's serverless container runtime. Deploy any container, auto-scales 0 → N, pay per request-second.

```bash
gcloud run deploy orders \
  --image gcr.io/my-project/orders:v1 \
  --region us-central1 \
  --service-account orders@my-project.iam.gserviceaccount.com \
  --set-env-vars SPRING_PROFILES_ACTIVE=prod \
  --min-instances 1 \
  --max-instances 20 \
  --cpu-boost                 # burst CPU at startup -> faster Spring boot
  --startup-probe=httpGet.path=/actuator/health/readiness,port=8080
```

Key Spring Boot settings:

- `spring.cloud.gcp.project-id=my-project` (auto-detected usually).
- `server.port=${PORT:8080}` (Cloud Run injects PORT env var).
- `--cpu-boost` halves cold-start time for Spring Boot — usually worth paying for.
- `--min-instances 1` keeps one container warm, eliminating most cold starts at the cost of ~$10/mo/region.

---

## Secret and Config Injection

Goal: no secrets in env vars, no config files in images.

- **AWS**: Parameter Store for config + Secrets Manager for secrets. Spring Cloud AWS bootstraps both.
- **GCP**: Secret Manager for both (GCP has no separate "Parameter Store"). Versioned references in bootstrap.

Rotation: both services rotate via API; Spring Boot doesn't auto-reload. Trigger a pod restart on secret change — use [Spring Cloud Bus](https://spring.io/projects/spring-cloud-bus) + `/actuator/refresh`, or rely on k8s re-pulling via ESO (see [secrets-management.md](../security/secrets-management.md)).

---

## Observability — CloudWatch and Cloud Ops

**AWS**: Micrometer ships metrics to CloudWatch via `micrometer-registry-cloudwatch2`. Logs flow to CloudWatch Logs via Docker log driver. X-Ray or OTel Collector → CloudWatch Logs for traces. Use [aws-otel-collector](https://aws-otel.github.io/) as the sidecar.

**GCP**: Micrometer → Cloud Monitoring via `micrometer-registry-stackdriver`. Logs flow to Cloud Logging with structured JSON. [Cloud Trace](https://cloud.google.com/trace) receives OTel spans.

For vendor-agnostic: always export to OpenTelemetry Collector, let the collector route to CloudWatch/Cloud Ops *and* an independent backend (Grafana Cloud, Honeycomb). Don't couple your app to one vendor's observability. See [distributed-tracing.md](../observability/distributed-tracing.md).

---

## Related

- [Kubernetes for Spring Boot](../configurations/kubernetes-spring-boot.md) — EKS/GKE Spring Boot setup.
- [Secrets Management](../security/secrets-management.md) — KMS, Parameter Store, Secret Manager.
- [GraalVM Native Image](../configurations/graalvm-native-image.md) — sub-100ms Lambda cold starts.
- [Docker and Deployment](../configurations/docker-and-deployment.md) — image building.
- [Distributed Tracing](../observability/distributed-tracing.md) — cross-cloud observability.
- [Messaging — Event-Driven Patterns](../messaging/event-driven-patterns.md) — SQS and Pub/Sub fit this model.

---

## References

- [Spring Cloud AWS (awspring)](https://awspring.io/)
- [Spring Cloud GCP](https://spring.io/projects/spring-cloud-gcp)
- [AWS Lambda SnapStart for Java](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html)
- [IAM Roles for Service Accounts (EKS)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [GKE Workload Identity](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity)
- [Cloud Run documentation](https://cloud.google.com/run/docs)
- [AWS SDK for Java v2](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/)
- [Google Cloud Java SDK](https://cloud.google.com/java/docs/reference)
- [AWS Observability — OTel collector](https://aws-otel.github.io/)
- [Spring Cloud Function](https://docs.spring.io/spring-cloud-function/reference/)
