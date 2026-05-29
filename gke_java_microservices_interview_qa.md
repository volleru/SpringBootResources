# Top 20 Interview Questions & Answers

**Topic:** Cloud-native Java microservices on GKE with auto-scaling and zero-downtime rolling deployments.

---

## 1. What does "cloud-native" mean for a Java microservice?

Cloud-native means the service is designed to run on dynamic, distributed infrastructure. Key traits:
- **Stateless** processes (state externalized to DB/cache)
- **Configuration via environment variables** (12-factor app)
- **Container-packaged** (Docker)
- **Orchestrated** (Kubernetes)
- **Observable** (metrics, logs, traces)
- **Resilient** (retries, circuit breakers, graceful shutdown)
- **Horizontally scalable**

In Java, this usually means Spring Boot + actuator + a slim base image (e.g., `eclipse-temurin:21-jre-alpine` or distroless).

---

## 2. Why deploy Java microservices on GKE specifically (over self-managed K8s or other managed offerings)?

- **Managed control plane** — Google operates the masters, etcd, upgrades.
- **Autopilot mode** — Google manages nodes; you pay per pod resources.
- **Tight GCP integration** — Cloud Load Balancing, IAM, Workload Identity, Cloud SQL, Pub/Sub, Cloud Logging/Monitoring out of the box.
- **Regional clusters** — multi-zone HA with no extra work.
- **Cluster Autoscaler & Node Auto-Provisioning** — scales nodes to match pod demand.
- **Release channels** (Rapid/Regular/Stable) — predictable upgrade cadence.

---

## 3. How do you containerize a Spring Boot service for GKE?

Use a multi-stage Dockerfile:

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY . .
RUN ./mvnw -q -DskipTests package

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-XX:+UseContainerSupport","-jar","app.jar"]
```

Better alternatives:
- **Jib** (`mvn jib:build`) — no Dockerfile, reproducible layers, faster pushes.
- **Spring Boot buildpacks** (`mvn spring-boot:build-image`) — CNB-based, layered JAR.
- **Distroless base** — smaller attack surface.

---

## 4. What is a Kubernetes Deployment and why use it for a Java service?

A `Deployment` is a controller that manages a `ReplicaSet`, which manages `Pods`. It provides:
- **Declarative state** (`replicas: 3`)
- **Rolling updates** with configurable surge/unavailability
- **Rollback** to a previous revision
- **Self-healing** (replaces crashed pods)

For a Java service, Deployment is the right primitive (vs StatefulSet) because instances are interchangeable and stateless.

---

## 5. How does Horizontal Pod Autoscaler (HPA) work?

HPA queries the metrics server every 15s (default) and scales replicas based on observed vs target metric:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    kind: Deployment
    name: order-svc
  minReplicas: 3
  maxReplicas: 30
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: { type: Utilization, averageUtilization: 70 }
```

Formula: `desiredReplicas = ceil(currentReplicas × currentMetric / targetMetric)`.

For Java, CPU alone is often misleading because of JIT warmup and GC. Combine with custom metrics (RPS, queue depth) via Prometheus Adapter or Stackdriver.

---

## 6. HPA vs VPA vs Cluster Autoscaler — when to use which?

| Component | Scales | Use case |
|---|---|---|
| **HPA** | Pod count | Stateless services with variable traffic |
| **VPA** | Pod CPU/memory requests | Right-sizing batch jobs, single-replica workloads |
| **Cluster Autoscaler** | Node count | When pods are Pending due to resource shortage |

HPA + VPA together on the same metric is conflicting — VPA should run in `recommendation-only` mode if HPA is on CPU.

---

## 7. What is a zero-downtime rolling deployment?

Kubernetes replaces pods incrementally. Config:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%        # extra pods during rollout
    maxUnavailable: 0    # never drop below desired replicas
```

For true zero-downtime, you also need:
1. **Readiness probes** — new pods don't get traffic until healthy.
2. **`preStop` hook + graceful shutdown** — drain in-flight requests.
3. **`terminationGracePeriodSeconds`** — enough time to finish.
4. **PodDisruptionBudget** — protects against voluntary disruptions during node upgrades.

---

## 8. Difference between `readinessProbe`, `livenessProbe`, and `startupProbe`?

- **Liveness** — "Is the JVM hung?" Failing → pod is **killed and restarted**.
- **Readiness** — "Can this pod serve traffic right now?" Failing → pod **removed from Service endpoints** (no kill).
- **Startup** — "Has the app finished initializing?" Disables the other probes until it passes. Critical for slow-starting Java apps.

Spring Boot example:
```yaml
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 8080 }
  periodSeconds: 10
readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 8080 }
  periodSeconds: 5
startupProbe:
  httpGet: { path: /actuator/health/liveness, port: 8080 }
  failureThreshold: 30
  periodSeconds: 10
```

---

## 9. How do you implement graceful shutdown in a Spring Boot service?

```yaml
# application.yml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

Sequence on `SIGTERM`:
1. Kubernetes sends `SIGTERM` and removes pod from Service endpoints.
2. `preStop` hook sleeps (e.g., 5s) to let endpoint propagation finish — avoids race where kube-proxy still routes traffic.
3. Spring stops accepting new requests, waits for in-flight to complete.
4. After `terminationGracePeriodSeconds`, `SIGKILL`.

```yaml
lifecycle:
  preStop:
    exec: { command: ["sh","-c","sleep 5"] }
terminationGracePeriodSeconds: 45
```

---

## 10. How do you size JVM memory inside a Kubernetes pod?

Java 10+ honors container limits via `-XX:+UseContainerSupport` (default since 11). Best practices:

```yaml
resources:
  requests: { cpu: "500m", memory: "1Gi" }
  limits:   { cpu: "2",    memory: "2Gi" }
```

JVM flags:
- `-XX:MaxRAMPercentage=75.0` — heap = 75% of pod memory, leaving room for metaspace, threads, direct buffers.
- `-XX:InitialRAMPercentage=50.0`
- Avoid `-Xmx` hardcoded — use percentage flags so the same image works at different sizes.

**Never omit `memory.limits`** — without it, OOMKill behavior is unpredictable and node pressure cascades.

---

## 11. CPU requests vs limits in Java — what's the gotcha?

- **Requests** = guaranteed CPU; used by scheduler.
- **Limits** = throttle ceiling via CFS quota.

For Java, CPU limits cause **CFS throttling** that hurts GC and JIT. Two schools:
1. Set `requests` only, no `limits` (best perf, risk of noisy neighbor).
2. Set generous `limits` (e.g., 2× requests).

`-XX:ActiveProcessorCount=N` should match limit, or GC/ForkJoinPool may over-thread.

---

## 12. How does GKE expose your service to the internet?

Layered:
1. **Pod** has an IP, ephemeral.
2. **Service (ClusterIP)** — stable virtual IP, load-balances across pods.
3. **Ingress** (GKE Ingress controller) — provisions a **Google Cloud HTTP(S) Load Balancer**, with managed certs via `ManagedCertificate` CRD.
4. **Gateway API** — newer, more expressive replacement for Ingress.

For internal-only: `Service type=ClusterIP` + Internal LB annotation.

---

## 13. How do you do blue/green or canary on GKE?

**Rolling** is the default but doesn't isolate traffic.

**Canary** options:
- Two Deployments (`v1` and `v2`) behind one Service with label selectors weighted, or
- **Istio / Anthos Service Mesh** with `VirtualService` weighted routing:
```yaml
http:
- route:
  - destination: { host: orders, subset: v1 }
    weight: 90
  - destination: { host: orders, subset: v2 }
    weight: 10
```
- **Argo Rollouts** — declarative canary/blue-green with analysis steps (Prometheus checks between weight bumps).

---

## 14. What is a PodDisruptionBudget and why does it matter for zero-downtime?

A PDB caps how many pods of a workload can be voluntarily disrupted at once:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 2
  selector:
    matchLabels: { app: orders }
```

During GKE node upgrades, node drains respect the PDB — protects you from losing all replicas simultaneously when a node pool rolls.

---

## 15. How do you handle configuration and secrets in GKE?

- **ConfigMap** — non-sensitive config (mounted as env or files).
- **Secret** — base64-encoded, encrypted at rest (with CMEK on GKE).
- **External Secrets Operator / Secret Manager CSI driver** — pulls from Google Secret Manager, no secrets in git.
- **Workload Identity** — pods authenticate as a GCP service account without static keys (preferred over JSON keys).

Spring Boot picks up env vars natively (`spring.datasource.url` ← `SPRING_DATASOURCE_URL`).

---

## 16. How do you make a Java microservice resilient to downstream failures?

- **Resilience4j** — circuit breaker, retry, bulkhead, rate limiter, time limiter (annotations or programmatic).
- **Timeouts everywhere** — never call without one.
- **Idempotency keys** for retried writes.
- **Exponential backoff with jitter** — avoid retry storms.
- **Bulkheads** — separate thread pools per dependency so one slow downstream doesn't exhaust all threads.

```java
@CircuitBreaker(name = "inventory", fallbackMethod = "fallback")
@Retry(name = "inventory")
public Stock getStock(String sku) { … }
```

---

## 17. How do you observe a Java microservice on GKE?

Three pillars:
- **Metrics** — Micrometer → Prometheus → Cloud Monitoring (or Managed Prometheus on GKE). Track RED (Rate/Errors/Duration) and USE (Utilization/Saturation/Errors).
- **Logs** — structured JSON to stdout → Cloud Logging via the node agent. Include `traceId`/`spanId`.
- **Traces** — OpenTelemetry SDK → Cloud Trace or Jaeger. Auto-instrument with the Java agent.

Key SLIs: p50/p95/p99 latency, error rate, saturation (CPU, heap, thread pool).

---

## 18. How do you do service-to-service communication and discovery?

- **In-cluster DNS** — `http://orders.default.svc.cluster.local:8080` resolves via CoreDNS.
- **REST** for simple sync — Spring's `WebClient` (non-blocking) over `RestTemplate`.
- **gRPC** for high-throughput, contract-first (use `grpc-java` + protobuf). Better latency, streaming, code-gen.
- **Async messaging** — Kafka / Pub/Sub for decoupling, event-driven flows.
- **Service mesh** (Istio/ASM) — mTLS, retries, traffic policies, observability without app code changes.

---

## 19. How do you secure a Java microservice on GKE?

- **Workload Identity** — pods get GCP IAM identity, no service-account JSON keys.
- **Network Policies** — default-deny ingress, explicit allow per namespace.
- **mTLS** between services (via Istio or app-level).
- **RBAC** — least-privilege ServiceAccounts; never use `cluster-admin` in workloads.
- **Image security** — Binary Authorization to enforce signed images; scan with Artifact Registry vulnerability scanning.
- **Pod Security Standards** — `restricted` profile: non-root, read-only root FS, drop all capabilities.
- **Secrets** — encrypted via CMEK; rotate via Secret Manager.

---

## 20. Describe an end-to-end zero-downtime release of a Java microservice on GKE.

1. **CI** — Maven builds, runs tests, publishes image to Artifact Registry (immutable tag = git SHA).
2. **CD** — Helm/Kustomize manifest updates image tag; ArgoCD or Cloud Build applies it.
3. **Pre-deploy** — DB migrations run as a Kubernetes `Job` using **expand-contract** pattern (add column first, deploy code, then remove old column in a later release).
4. **Deployment apply** — `maxSurge=25%`, `maxUnavailable=0`.
5. **New pods boot** — startup probe gates traffic until JVM is warm.
6. **Readiness flips** — new pod added to Service endpoints; old pod stays serving until its replacement is ready.
7. **Old pod terminates** — removed from endpoints → `preStop` sleep → Spring graceful shutdown → in-flight requests complete.
8. **PDB enforced** — guarantees ≥N replicas serving throughout.
9. **HPA reacts** — if traffic spikes mid-rollout, more replicas spin up.
10. **Post-deploy verification** — smoke tests + Prometheus checks; automated rollback (`kubectl rollout undo`) if SLOs breach.
