# Spring Boot on GCP — End-to-End to a Production HTTPS URL

A practical, copy-pasteable guide taking a Spring Boot service from source code to a
production-ready, custom-domain HTTPS URL on Google Cloud — with CI/CD, secrets,
observability, and hardening.

Two deployment paths are covered:
- **Path A — Cloud Run** (serverless containers): fastest route to a production URL, scales to zero, managed TLS. *Recommended default.*
- **Path B — GKE** (Kubernetes): when you need fine-grained networking, sidecars, stateful workloads, or an existing cluster.

---

## Table of Contents
1. [Architecture & flow](#1-architecture--flow)
2. [Prerequisites](#2-prerequisites)
3. [The Spring Boot application](#3-the-spring-boot-application)
4. [Run & test locally](#4-run--test-locally)
5. [Containerize (Jib — no Dockerfile)](#5-containerize-jib--no-dockerfile)
6. [GCP project bootstrap](#6-gcp-project-bootstrap)
7. [Build & push the image to Artifact Registry](#7-build--push-the-image-to-artifact-registry)
8. [Config & secrets (Secret Manager)](#8-config--secrets-secret-manager)
9. [Path A — Deploy to Cloud Run](#9-path-a--deploy-to-cloud-run)
10. [Path B — Deploy to GKE](#10-path-b--deploy-to-gke)
11. [Production URL: custom domain + managed TLS](#11-production-url-custom-domain--managed-tls)
12. [CI/CD with Cloud Build](#12-cicd-with-cloud-build)
13. [Observability: logs, metrics, health, tracing](#13-observability-logs-metrics-health-tracing)
14. [Production hardening checklist](#14-production-hardening-checklist)
15. [Cleanup](#15-cleanup)

---

## 1. Architecture & flow

```
 Developer → git push → Cloud Build (build, test, image) → Artifact Registry
                                                                  │
                                          ┌───────────────────────┴───────────────────────┐
                                          ▼                                                 ▼
                                  Cloud Run service                                 GKE Deployment + Service
                                          │                                                 │  + Gateway/Ingress
                                          ▼                                                 ▼
                            Global HTTPS LB + managed cert  ←── custom domain (api.example.com) ──→  Managed cert
                                          │
                                          ▼
                                   Secret Manager · Cloud SQL · Pub/Sub
                                          │
                                   Cloud Logging / Monitoring / Trace
```

---

## 2. Prerequisites

```bash
# Tooling
gcloud --version          # Google Cloud CLI
java -version             # 17 or 21 (LTS)
mvn -version              # Maven 3.9+

# Auth
gcloud auth login
gcloud auth application-default login

# Pick your project + defaults
export PROJECT_ID="my-prod-project"
export REGION="us-central1"
gcloud config set project "$PROJECT_ID"
gcloud config set run/region "$REGION"
```

---

## 3. The Spring Boot application

### `pom.xml` (key parts)

```xml
<project>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.4</version>
  </parent>

  <groupId>com.example</groupId>
  <artifactId>orders-api</artifactId>
  <version>1.0.0</version>

  <properties>
    <java.version>21</java.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- /actuator/health, /actuator/info for probes -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
      <!-- Jib: build a container with NO Dockerfile -->
      <plugin>
        <groupId>com.google.cloud.tools</groupId>
        <artifactId>jib-maven-plugin</artifactId>
        <version>3.4.3</version>
      </plugin>
    </plugins>
  </build>
</project>
```

### Application entrypoint + a REST endpoint

```java
// src/main/java/com/example/ordersapi/OrdersApiApplication.java
package com.example.ordersapi;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class OrdersApiApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrdersApiApplication.class, args);
    }
}
```

```java
// src/main/java/com/example/ordersapi/OrderController.java
package com.example.ordersapi;

import org.springframework.web.bind.annotation.*;
import java.util.Map;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping("/{id}")
    public Map<String, Object> get(@PathVariable final String id) {
        return Map.of("id", id, "status", "CONFIRMED");
    }
}
```

### `application.yaml`

```yaml
server:
  port: ${PORT:8080}            # Cloud Run injects PORT; default 8080 locally
spring:
  application:
    name: orders-api
management:
  endpoints:
    web:
      exposure:
        include: health,info    # expose only what you need
  endpoint:
    health:
      probes:
        enabled: true           # /actuator/health/liveness & /readiness
```

> **Why `PORT`:** Cloud Run sets `$PORT` (usually 8080) and routes to it. Binding to it is mandatory.

---

## 4. Run & test locally

```bash
mvn spring-boot:run
# in another shell:
curl localhost:8080/api/orders/123
curl localhost:8080/actuator/health
```

---

## 5. Containerize (Jib — no Dockerfile)

Jib builds an optimized, layered, distroless image straight from Maven — no Docker daemon,
no Dockerfile, reproducible builds. Ideal for Java.

```bash
# Build to a local tarball / daemon for a smoke test (optional)
mvn compile jib:dockerBuild -Dimage=orders-api:local
docker run -p 8080:8080 orders-api:local
```

> Dockerfile alternative (if you must): use a multi-stage build with
> `eclipse-temurin:21-jre` as the runtime base and copy the fat jar. Jib is preferred for Java.

---

## 6. GCP project bootstrap

```bash
# Enable the APIs you'll use
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  secretmanager.googleapis.com \
  compute.googleapis.com \
  container.googleapis.com \
  logging.googleapis.com \
  monitoring.googleapis.com \
  cloudtrace.googleapis.com

# Create an Artifact Registry Docker repo
gcloud artifacts repositories create app-images \
  --repository-format=docker \
  --location="$REGION" \
  --description="App container images"

export IMAGE="${REGION}-docker.pkg.dev/${PROJECT_ID}/app-images/orders-api"
```

---

## 7. Build & push the image to Artifact Registry

```bash
# Let Jib authenticate via gcloud and push directly
gcloud auth configure-docker "${REGION}-docker.pkg.dev"

mvn compile jib:build \
  -Dimage="${IMAGE}:1.0.0" \
  -Djib.to.tags=1.0.0,latest

# Verify
gcloud artifacts docker images list "${REGION}-docker.pkg.dev/${PROJECT_ID}/app-images"
```

---

## 8. Config & secrets (Secret Manager)

Never bake secrets into the image or env in plain text.

```bash
# Create a secret (e.g., a DB password)
printf 'super-secret-value' | gcloud secrets create db-password --data-file=-

# Dedicated runtime service account (least privilege)
gcloud iam service-accounts create orders-api-sa \
  --display-name="orders-api runtime"

export RUNTIME_SA="orders-api-sa@${PROJECT_ID}.iam.gserviceaccount.com"

# Grant access to just this secret
gcloud secrets add-iam-policy-binding db-password \
  --member="serviceAccount:${RUNTIME_SA}" \
  --role="roles/secretmanager.secretAccessor"
```

Consume it at deploy time (Cloud Run shown; GKE uses CSI driver or env-from-secret).

---

## 9. Path A — Deploy to Cloud Run

```bash
gcloud run deploy orders-api \
  --image="${IMAGE}:1.0.0" \
  --region="$REGION" \
  --service-account="$RUNTIME_SA" \
  --port=8080 \
  --cpu=1 --memory=512Mi \
  --min-instances=1 --max-instances=20 \
  --concurrency=80 \
  --set-secrets="DB_PASSWORD=db-password:latest" \
  --set-env-vars="SPRING_PROFILES_ACTIVE=prod" \
  --no-allow-unauthenticated        # lock down; front with LB/IAP (see §11)

# Get the auto-generated URL (https://orders-api-xxxxx-uc.a.run.app)
gcloud run services describe orders-api --region "$REGION" --format='value(status.url)'
```

Health-check config (readiness/liveness) on Cloud Run:

```bash
gcloud run services update orders-api --region "$REGION" \
  --startup-probe="httpGet.path=/actuator/health/readiness,initialDelaySeconds=10,periodSeconds=5" \
  --liveness-probe="httpGet.path=/actuator/health/liveness,periodSeconds=10"
```

> `--min-instances=1` avoids cold starts for a production API. Set to 0 for cost-saving on low-traffic services.

---

## 10. Path B — Deploy to GKE

### Connect to a cluster

```bash
gcloud container clusters get-credentials my-cluster --region "$REGION"
```

### `k8s/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
spec:
  replicas: 3
  selector: { matchLabels: { app: orders-api } }
  template:
    metadata: { labels: { app: orders-api } }
    spec:
      serviceAccountName: orders-api-ksa     # bound to GCP SA via Workload Identity
      containers:
        - name: orders-api
          image: REGION-docker.pkg.dev/PROJECT_ID/app-images/orders-api:1.0.0
          ports: [ { containerPort: 8080 } ]
          resources:
            requests: { cpu: "250m", memory: "256Mi" }
            limits:   { cpu: "1",    memory: "512Mi" }
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            initialDelaySeconds: 10
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            initialDelaySeconds: 20
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: prod
---
apiVersion: v1
kind: Service
metadata:
  name: orders-api
spec:
  selector: { app: orders-api }
  ports: [ { port: 80, targetPort: 8080 } ]
  type: ClusterIP
```

### Workload Identity (pod → GCP SA, no key files)

```bash
gcloud iam service-accounts add-iam-policy-binding "$RUNTIME_SA" \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:${PROJECT_ID}.svc.id.goog[default/orders-api-ksa]"

kubectl create serviceaccount orders-api-ksa
kubectl annotate serviceaccount orders-api-ksa \
  iam.gke.io/gcp-service-account="$RUNTIME_SA"

kubectl apply -f k8s/deployment.yaml
```

### Expose via Gateway API (managed cert, global LB)

```yaml
# k8s/gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: orders-gw
spec:
  gatewayClassName: gke-l7-global-external-managed
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        options:
          networking.gke.io/pre-shared-certs: orders-api-cert   # managed cert (see §11)
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: orders-route
spec:
  parentRefs: [ { name: orders-gw } ]
  hostnames: [ "api.example.com" ]
  rules:
    - backendRefs: [ { name: orders-api, port: 80 } ]
```

---

## 11. Production URL: custom domain + managed TLS

The auto-generated `*.run.app` / LB IP is not a production URL. Map your own domain with a
Google-managed certificate (auto-renewing).

### Cloud Run — domain mapping (simple)

```bash
# Verify domain ownership once in Search Console, then:
gcloud beta run domain-mappings create \
  --service=orders-api --domain=api.example.com --region="$REGION"
# It returns DNS records (A/AAAA or CNAME) — add them at your registrar.
```

### Cloud Run / GKE — global HTTPS Load Balancer (recommended for prod)

A global external Application LB gives you a stable anycast IP, Google-managed TLS, Cloud Armor (WAF), and CDN.

```bash
# 1) Reserve a global static IP
gcloud compute addresses create orders-api-ip --global
gcloud compute addresses describe orders-api-ip --global --format='value(address)'

# 2) Create a Google-managed cert for your domain
gcloud compute ssl-certificates create orders-api-cert \
  --domains=api.example.com --global

# 3) Point DNS A record → the static IP at your registrar.
#    Cert provisioning auto-completes once DNS resolves (can take ~15–60 min).

# 4) For Cloud Run: create a Serverless NEG → backend service → URL map →
#    target-https-proxy → forwarding rule. (Terraform module recommended;
#    see google_compute_region_network_endpoint_group with serverless_deployment.)
```

> Add **Cloud Armor** to the backend service for WAF/rate-limiting, and enable **Cloud CDN** for cacheable responses.

Final production URL: **`https://api.example.com/api/orders/123`** ✅

---

## 12. CI/CD with Cloud Build

### `cloudbuild.yaml`

```yaml
steps:
  # 1. Unit tests
  - name: maven:3.9-eclipse-temurin-21
    entrypoint: mvn
    args: ["test"]

  # 2. Build + push image with Jib (tagged with the commit SHA)
  - name: maven:3.9-eclipse-temurin-21
    entrypoint: mvn
    args:
      - "compile"
      - "jib:build"
      - "-Dimage=${_REGION}-docker.pkg.dev/$PROJECT_ID/app-images/orders-api:$SHORT_SHA"

  # 3. Deploy to Cloud Run
  - name: gcr.io/google.com/cloudsdktool/cloud-sdk
    entrypoint: gcloud
    args:
      - "run"; "deploy"; "orders-api"
      - "--image=${_REGION}-docker.pkg.dev/$PROJECT_ID/app-images/orders-api:$SHORT_SHA"
      - "--region=${_REGION}"
      - "--service-account=orders-api-sa@$PROJECT_ID.iam.gserviceaccount.com"

substitutions:
  _REGION: us-central1
options:
  logging: CLOUD_LOGGING_ONLY
```

> Note: in real `cloudbuild.yaml`, each `args` entry is its own YAML list item (no semicolons) — the line above is condensed for readability.

### Trigger on push to `main`

```bash
gcloud builds triggers create github \
  --repo-name=orders-api --repo-owner=my-org \
  --branch-pattern='^main$' \
  --build-config=cloudbuild.yaml
```

Grant the Cloud Build SA the deploy roles (`roles/run.admin`, `roles/iam.serviceAccountUser`,
`roles/artifactregistry.writer`).

---

## 13. Observability: logs, metrics, health, tracing

- **Structured logs** → add `spring-cloud-gcp-starter-logging` so logs land in Cloud Logging as JSON with severity + trace correlation. On Cloud Run/GKE, stdout is auto-collected.
- **Health** → Actuator `/actuator/health/{liveness,readiness}` wired to probes (done above).
- **Metrics** → add `micrometer-registry-stackdriver` (or scrape `/actuator/prometheus` with Google Managed Prometheus on GKE).
- **Tracing** → `spring-cloud-gcp-starter-trace` exports spans to Cloud Trace; propagates `X-Cloud-Trace-Context`.
- **Dashboards & alerts** → Cloud Monitoring: alert on p99 latency, 5xx rate, instance count, and error-log spikes. Wire alerts to your on-call channel.

```xml
<dependency>
  <groupId>com.google.cloud</groupId>
  <artifactId>spring-cloud-gcp-starter-logging</artifactId>
</dependency>
```

---

## 14. Production hardening checklist

**Security**
- [ ] Dedicated runtime service account, **least-privilege** roles only (no `Editor`).
- [ ] Secrets in **Secret Manager**, never in env/image; rotate regularly.
- [ ] `--no-allow-unauthenticated` + front with LB; add **Cloud Armor** WAF and rate limits.
- [ ] **IAP** or JWT/OAuth auth on the LB if the API isn't fully public.
- [ ] Image scanning enabled in Artifact Registry; pin base images by digest.
- [ ] Workload Identity on GKE (no exported SA keys).

**Reliability**
- [ ] Liveness + readiness probes on `/actuator/health/*`.
- [ ] `min-instances ≥ 1` (Cloud Run) or HPA + PodDisruptionBudget (GKE).
- [ ] Set CPU/memory **requests and limits**; load-test to size them.
- [ ] Graceful shutdown: `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase`.
- [ ] Multi-region or multi-zone for HA; health-checked global LB.

**Delivery**
- [ ] Immutable image tags (commit SHA), not `latest`, in prod.
- [ ] Tests + vulnerability scan gate the pipeline.
- [ ] Gradual rollout: Cloud Run **traffic splitting** / GKE rolling update; easy rollback.
- [ ] Infra as code (Terraform) for LB, certs, IAM, triggers.

**Cost/observability**
- [ ] Dashboards + alerts (latency, error rate, saturation).
- [ ] Log-based metrics for business/error signals; set log retention.
- [ ] Budget alerts on the project.

---

## 15. Cleanup

```bash
gcloud run services delete orders-api --region "$REGION" -q
gcloud artifacts repositories delete app-images --location "$REGION" -q
gcloud compute addresses delete orders-api-ip --global -q
gcloud compute ssl-certificates delete orders-api-cert --global -q
gcloud secrets delete db-password -q
# GKE: kubectl delete -f k8s/
```

---

### Quick reference — the happy path (Cloud Run)

```bash
# code → image → deploy → URL, in 4 commands
mvn compile jib:build -Dimage="${IMAGE}:1.0.0"
gcloud run deploy orders-api --image "${IMAGE}:1.0.0" --region "$REGION" \
  --service-account "$RUNTIME_SA" --min-instances 1
gcloud beta run domain-mappings create --service orders-api \
  --domain api.example.com --region "$REGION"
curl https://api.example.com/api/orders/123
```
