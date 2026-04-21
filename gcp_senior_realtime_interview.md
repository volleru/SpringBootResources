# GCP Senior Engineer — Real Interview Questions & Answers

> These are the type of questions actually asked in senior-level GCP interviews.
> Interviewers expect depth, trade-offs, and real experience — not textbook answers.

---

## Round 1: Deep Technical Dive

---

**Q1. You deployed a new version of your service on GKE. After rollout, p99 latency spiked from 200ms to 2s. Production is affected. Walk me through exactly what you do.**

> This is a war story question. They want to see your debugging instincts, not a checklist.
>
> **Step 1 — Immediate triage (first 2 minutes):**
> ```bash
> kubectl rollout history deployment/my-service
> kubectl rollout undo deployment/my-service   # rollback immediately if spike is confirmed
> ```
> Don't wait to investigate if users are affected — rollback first, investigate after.
>
> **Step 2 — Confirm the rollback fixed it.** Check Cloud Monitoring latency dashboard. If yes, you have a bad deployment. If no, the issue is environmental.
>
> **Step 3 — Investigate the bad deployment:**
> - **Cloud Trace** — Which span is taking 1.8s? Is it a DB call, an upstream API, or internal logic?
> - **Cloud Logging** — Any new errors, GC pauses, connection pool exhaustion?
> - `kubectl top pods` — CPU throttling? (check `limits` vs `requests` — a common mistake is setting CPU limit too low)
> - Check if new version introduced a synchronous call where the old version was async
> - Check if new version changed DB query pattern (missing index on new query)
>
> **Step 4 — Root cause categories:**
> - CPU throttling due to low `cpu limit` → increase limits or remove them
> - N+1 query problem introduced in new code → fix the query
> - New dependency (external API call) in hot path → move to async
> - Connection pool misconfiguration → tune pool size
>
> **Key answer the interviewer wants:** You rolled back first. You used traces to isolate the slow span. You didn't guess.

---

**Q2. Your GKE pods are OOMKilled every few hours. How do you diagnose and fix this permanently?**

> **Immediate check:**
> ```bash
> kubectl describe pod <pod-name>   # shows OOMKilled in Last State
> kubectl top pods --containers     # current memory usage
> ```
>
> **Diagnose root cause — there are 3 different causes with different fixes:**
>
> **Cause 1: Memory limit set too low**
> - `kubectl get pod -o yaml | grep -A4 resources` — check what limit is set
> - Check actual usage trend in Cloud Monitoring → `container/memory/used_bytes`
> - Fix: Increase `memory limit`, or use VPA to auto-tune it
>
> **Cause 2: Memory leak in application**
> - Usage grows slowly over hours then crashes — classic leak pattern
> - Enable JVM heap dumps on OOM: `-XX:+HeapDumpOnOutOfMemoryError`
> - Analyse with Eclipse MAT or IntelliJ — look for accumulating objects
> - Fix: Find the leak (usually a cache with no eviction, or listener not deregistered)
>
> **Cause 3: JVM not aware of container limits (very common in Java)**
> - JVM by default reads host memory, not cgroup limit
> - A pod with `memory limit: 512Mi` on a 64GB node → JVM thinks it has 64GB → allocates huge heap → OOMKilled
> - Fix: Add `-XX:+UseContainerSupport` (Java 10+) or explicitly set `-Xmx400m`
>
> **Permanent fix:**
> - Use VPA in recommendation mode first to get right sizing
> - Set `requests == limits` for memory (prevents overcommit)
> - Add alerting on `container/memory/used_bytes > 80% of limit`

---

**Q3. Explain how GKE networking works when a pod in namespace A calls a service in namespace B. What happens at the network layer?**

> This is a deep question. Interviewers expect you to know the actual packet flow.
>
> **Flow:**
> 1. Pod A makes a DNS call: `service-b.namespace-b.svc.cluster.local`
> 2. DNS resolves to the ClusterIP of Service B (e.g., `10.96.0.5`)
> 3. Pod A sends the packet to `10.96.0.5`
> 4. The packet hits `kube-proxy` rules (iptables or IPVS) on the node
> 5. iptables DNAT rewrites the destination from ClusterIP → one of the pod IPs backing the service (load balanced)
> 6. GKE uses VPC-native networking — pod IPs are real GCP VPC IPs (from the node's alias IP range)
> 7. The packet is routed directly to the target pod's node at the VPC level — no NAT between nodes
>
> **Network Policy check:**
> - If NetworkPolicy exists, the policy is enforced by the CNI (Calico or Cilium in GKE)
> - If namespace B has an ingress policy, it must explicitly allow traffic from namespace A
>
> **Key point interviewers look for:**
> - GKE uses VPC-native (alias IP) — pods have real VPC IPs, no overlay network tunneling between nodes
> - This is different from Flannel/Weave which use encapsulation (VXLAN)
> - Makes GKE pods directly routable from other GCP services, Cloud SQL, etc.

---

**Q4. You have a Terraform config that manages a GKE cluster. A team member manually changed a node pool setting in the console. What happens when someone runs `terraform apply` next? How do you handle config drift?**

> **What happens:**
> `terraform plan` will show a diff — it compares the remote state in GCS with the actual GCP resource. The manual change is detected. `terraform apply` will revert it back to the Terraform-defined state.
>
> **The real problem:** If the manual change was intentional (e.g., emergency scaling), blindly running `terraform apply` reverts it — dangerous.
>
> **How to handle config drift properly:**
>
> **Option 1: Import the manual change back into Terraform state**
> ```bash
> terraform import google_container_node_pool.main projects/my-proj/locations/us-east1/clusters/my-cluster/nodePools/main-pool
> # Then update the .tf file to match
> ```
>
> **Option 2: `terraform refresh` then review**
> ```bash
> terraform refresh   # updates state to match real world
> terraform plan      # shows what TF would now change
> ```
>
> **Prevent drift in the first place:**
> - IAM: Remove direct console access for production. Only Terraform service account can modify infra.
> - Use `terraform plan` in CI on PRs — any unreviewed change is caught
> - Set up Cloud Asset Inventory alerts for resource changes not originating from Cloud Build
> - Org policy: `constraints/gcp.resourceLocations` + custom constraint to prevent node pool changes outside IaC

---

**Q5. Your Cloud SQL instance is hitting 100% CPU during peak hours. The queries are all simple SELECTs on an indexed table. What do you investigate?**

> This is a trick question — "simple SELECTs on indexed table" still has many failure modes.
>
> **Step 1: Check actual query plans**
> ```sql
> EXPLAIN ANALYZE SELECT ...   -- Is the index actually being used?
> -- Check for index scan vs sequential scan
> ```
>
> **Step 2: Check Cloud SQL Insights**
> - Top queries by CPU, by lock wait
> - Check if there are lock contention issues (reads blocked by writes)
>
> **Step 3: Connection pool exhaustion**
> - Too many connections opening/closing → connection overhead, not query overhead
> - Cloud SQL has max_connections limit (depends on instance size)
> - Fix: Add PgBouncer or Cloud SQL Auth Proxy with connection pooling
>
> **Step 4: Missing composite index**
> - Index exists on column A, but query filters on `WHERE A = ? AND B = ?` → needs composite index on (A, B)
>
> **Step 5: Read replica offloading**
> - Move read traffic to read replicas
> - Route reporting/analytics queries to replica
>
> **Step 6: Instance sizing**
> - Check if instance is undersized for workload — upgrade to higher-CPU tier
> - Consider Spanner if you've truly outgrown Cloud SQL

---

## Round 2: System Design

---

**Q6. Design a real-time event processing pipeline on GCP that ingests 1 million events/second from IoT devices, processes them, stores raw and aggregated data, and triggers alerts. What services do you use and why?**

> **Architecture:**
>
> ```
> IoT Devices
>     └── Cloud IoT Core / direct HTTPS/MQTT
>             └── Pub/Sub Topic (raw-events)
>                     ├── Dataflow (streaming pipeline)
>                     │       ├── Windowed aggregation (5-min windows)
>                     │       ├── Anomaly detection logic
>                     │       ├── Write aggregates → BigQuery
>                     │       └── Write raw → Cloud Storage (GCS)
>                     └── Cloud Functions / Pub/Sub filter
>                             └── Alert → Cloud Monitoring custom metric → Alerting policy → PagerDuty
> ```
>
> **Why these choices:**
> - **Pub/Sub** — Handles 1M events/sec natively, decouples ingestion from processing, durable 7-day retention
> - **Dataflow** — Managed Apache Beam, exactly-once processing, auto-scales workers, handles late-arriving data with watermarks
> - **BigQuery** — Aggregated data for analysis, partitioned by event_time, clustered by device_id
> - **Cloud Storage** — Raw event archive, partitioned by date, Coldline after 90 days
> - **NOT Kafka** — Pub/Sub is fully managed, no cluster to operate; use Kafka only if you need consumer group semantics or exactly-once with offset control that Pub/Sub can't provide
>
> **Interviewer follow-up: How do you handle duplicate events?**
> - Dataflow with exactly-once sink handles deduplication via Pub/Sub message IDs
> - Add idempotency key in BigQuery using `MERGE` statement instead of plain `INSERT`

---

**Q7. You need to migrate a monolithic Java application (running on bare metal) to GKE with zero downtime. Walk me through your strategy.**

> **This is a real migration question. They want a phased approach.**
>
> **Phase 1: Containerise without changing behaviour**
> - Dockerize the monolith as-is. Don't refactor yet.
> - Run container locally, verify behaviour matches
> - Push to Artifact Registry
>
> **Phase 2: Deploy to GKE alongside the monolith (shadow mode)**
> - Deploy the containerised monolith on GKE in a new environment
> - Mirror production traffic using Cloud Load Balancer weighted routing (5% to GKE, 95% to old)
> - Compare responses — don't cut over until responses match
>
> **Phase 3: Data layer**
> - This is the hardest part. Options:
>   - Keep DB on-prem, connect from GKE over Cloud Interconnect initially
>   - Migrate DB to Cloud SQL with DMS (Database Migration Service) — continuous replication, cut over with minimal downtime
>
> **Phase 4: Gradual traffic shift**
> - Use weighted backend services on Global LB: 10% → 25% → 50% → 100%
> - Monitor error rate, latency at each step
> - Keep old servers as fallback with fast DNS failover
>
> **Phase 5: Strangle the monolith (optional)**
> - Once stable, extract microservices from the monolith one at a time using Strangler Fig pattern
>
> **Key point:** Zero downtime = gradual traffic shift + data replication running in parallel before cutover. Never big-bang.

---

**Q8. Design a multi-tenant SaaS platform on GCP where each tenant's data must be strictly isolated. What are your options and what do you recommend?**

> **Option 1: Separate GCP project per tenant**
> - Strongest isolation — separate billing, IAM, VPC, quotas
> - Hard to manage at scale (100+ tenants = 100+ projects)
> - Use Terraform modules to provision new tenant projects automatically
> - Shared VPC or VPC peering for shared services (logging, monitoring)
>
> **Option 2: Separate namespace per tenant in shared GKE cluster**
> - Use Kubernetes namespaces + NetworkPolicy for isolation
> - ResourceQuota per namespace to prevent noisy neighbour
> - Workload Identity per tenant namespace
> - Cheaper, but noisy neighbour risk at node level
>
> **Option 3: Separate GKE node pool per tenant (middle ground)**
> - Node taints + pod affinity rules to pin tenant workloads to specific nodes
> - Better isolation than namespace, cheaper than separate project
>
> **Data isolation:**
> - **Cloud SQL** — separate DB per tenant (same Cloud SQL instance, different database)
> - **Firestore** — separate collection hierarchy per tenant ID
> - **BigQuery** — separate dataset per tenant, IAM on dataset
>
> **Recommended for most SaaS:**
> - Namespace isolation on shared GKE + separate database per tenant in shared Cloud SQL
> - If compliance requirement (SOC2, HIPAA) demands it → separate project per tenant
>
> **VPC Service Controls** — wrap BigQuery/GCS with a perimeter so tenant A's service account literally cannot access tenant B's data even if misconfigured.

---

## Round 3: Real Scenario-Based

---

**Q9. Your Cloud Build pipeline is taking 45 minutes. Dev team is complaining. How do you speed it up?**

> **Diagnose first — identify the slow step:**
> ```yaml
> # Cloud Build shows step timing in logs
> # Check which step takes most time
> ```
>
> **Common culprits and fixes:**
>
> **1. Docker build rebuilding from scratch every time**
> - Fix: Use layer caching
> ```yaml
> - name: 'gcr.io/cloud-builders/docker'
>   args: ['build', '--cache-from', 'gcr.io/$PROJECT_ID/myapp:latest', '-t', 'gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA', '.']
> ```
>
> **2. Maven/Gradle downloading dependencies every build**
> - Fix: Cache `.m2` or `.gradle` in Cloud Storage between builds
> ```yaml
> - name: 'gcr.io/cloud-builders/gsutil'
>   args: ['cp', 'gs://my-cache-bucket/m2-cache.tar.gz', '.']
> ```
>
> **3. Running tests sequentially**
> - Fix: Parallelize test execution with `--parallel` flag or split test suites into separate Cloud Build steps running concurrently using `waitFor`
>
> **4. Large Docker image**
> - Fix: Multi-stage builds — compile in a full JDK image, copy artifact to slim JRE base image
> ```dockerfile
> FROM maven:3.9-eclipse-temurin-21 AS builder
> RUN mvn package -DskipTests
> FROM eclipse-temurin:21-jre-alpine
> COPY --from=builder /app/target/app.jar /app.jar
> ```
>
> **5. Machine type too small**
> - Fix: Use higher-tier Cloud Build machine: `options: machineType: E2_HIGHCPU_32`

---

**Q10. A team wants to run a batch job on GKE every night that processes 500GB of data. The job takes 4 hours and uses 16 CPUs and 64GB RAM. After the job, the resources should not be wasted. How do you design this?**

> **Correct answer: GKE Node Auto Provisioning + Spot VMs**
>
> **Design:**
> - Use a Kubernetes `CronJob` to schedule the job at midnight
> - Use a dedicated node pool with Spot VMs (70-80% cheaper)
> - Set node pool `autoscaling: min=0, max=5`
> - Node Auto Provisioner creates nodes when the Job pod is scheduled, scales to 0 after completion
>
> ```yaml
> apiVersion: batch/v1
> kind: CronJob
> spec:
>   schedule: "0 0 * * *"
>   jobTemplate:
>     spec:
>       template:
>         spec:
>           nodeSelector:
>             cloud.google.com/gke-spot: "true"
>           tolerations:
>           - key: "cloud.google.com/gke-spot"
>             operator: "Equal"
>             value: "true"
>             effect: "NoSchedule"
>           containers:
>           - resources:
>               requests:
>                 cpu: "16"
>                 memory: "64Gi"
> ```
>
> **Handle Spot preemption:**
> - Use checkpointing — write progress to Cloud Storage every 30 min
> - On restart, resume from last checkpoint
> - Set `restartPolicy: OnFailure`
>
> **Cost estimate:** Spot n2-standard-16 ≈ $0.20/hr vs on-demand $0.80/hr. 4 hours × 5 nodes = ~$4 vs ~$16.

---

**Q11. You're asked to reduce GCP cloud costs by 40% without reducing performance. What do you look at?**

> **This is a very common question for senior roles. They want a structured approach.**
>
> **1. Right-size compute (usually biggest win)**
> - Enable GCE Recommender — Google automatically suggests undersized/oversized VMs
> - Check GKE node utilization — if nodes are at 20% CPU, you're paying for 80% idle
> - Use VPA recommendations to right-size pod requests
>
> **2. Committed Use Discounts (CUD)**
> - 1-year CUD = 37% discount, 3-year = 55% for Compute Engine
> - Analyze last 3 months of baseline usage → commit to that, pay on-demand for spikes
> - CUDs are machine-type flexible in GCP (unlike AWS Reserved Instances)
>
> **3. Spot/Preemptible VMs for non-critical workloads**
> - Batch jobs, CI/CD workers, dev/staging environments → 60-80% savings
>
> **4. Cloud Storage lifecycle policies**
> - Data older than 30 days → Nearline, 90 days → Coldline, 1 year → Archive
> - Often forgotten — teams leave TBs in Standard class
>
> **5. BigQuery optimisation**
> - Use partitioned + clustered tables → reduce bytes scanned → reduce cost
> - Set slot reservations instead of on-demand for predictable workloads
> - Expire old partitions with `partition_expiration_days`
>
> **6. Network egress**
> - Use Private Google Access to avoid internet egress charges
> - Keep services in the same region to eliminate inter-region egress
>
> **7. Idle resources**
> - Cloud Asset Inventory → find unattached persistent disks, idle static IPs, stopped VMs still incurring storage costs

---

**Q12. How do you manage secrets in GKE? A junior engineer put a DB password in a ConfigMap. What do you do?**

> **Immediate action:**
> 1. Rotate the DB password NOW — assume it's compromised
> 2. Delete the ConfigMap
> 3. Check Cloud Audit Logs to see who/what accessed that ConfigMap
>
> **Correct pattern — Secret Manager + Workload Identity:**
>
> ```bash
> # Store secret in Secret Manager
> echo -n "my-db-password" | gcloud secrets create db-password --data-file=-
>
> # Grant GKE service account access
> gcloud secrets add-iam-policy-binding db-password \
>   --member="serviceAccount:my-ksa@my-project.iam.gserviceaccount.com" \
>   --role="roles/secretmanager.secretAccessor"
> ```
>
> **Mount in pod using External Secrets Operator or Secret Store CSI Driver:**
> ```yaml
> volumes:
> - name: db-password
>   csi:
>     driver: secrets-store.csi.k8s.io
>     readOnly: true
>     volumeAttributes:
>       secretProviderClass: gcp-secrets
> ```
>
> **Or use Secret Manager SDK directly in app** — fetch at startup, no file on disk.
>
> **Prevention:**
> - OPA/Gatekeeper policy: reject any ConfigMap or Pod spec containing values matching secret patterns (regex for passwords, API keys)
> - `kubesec` scan in CI pipeline to detect secrets in manifests before they're applied
> - Enable Secret Manager audit logging

---

**Q13. Explain how you implemented zero-downtime deployments in GKE. What can go wrong and how do you prevent it?**

> **The basics everyone says:** RollingUpdate with `maxSurge` and `maxUnavailable`.
>
> **What they actually want to hear — the failure modes:**
>
> **Problem 1: Readiness probe not tuned correctly**
> - If readiness probe passes before the app is truly warm, LB sends traffic to a cold pod → user errors
> - Fix: Add a warmup endpoint. Only return 200 from readiness after cache is loaded / connections are pooled.
>
> **Problem 2: Termination not graceful**
> - Pod receives SIGTERM but in-flight requests are dropped
> - Fix:
> ```yaml
> terminationGracePeriodSeconds: 60
> lifecycle:
>   preStop:
>     exec:
>       command: ["/bin/sh", "-c", "sleep 10"]  # wait for LB to drain
> ```
> The `preStop` sleep gives the LB time to stop sending new requests before the process exits.
>
> **Problem 3: Database migration runs before old pods are drained**
> - New code expects new DB schema, but old pods still running against new schema fail
> - Fix: Backwards-compatible migrations only — never rename/drop columns in the same deployment. Expand-contract pattern:
>   - Deploy 1: Add new column (nullable) — both old and new code work
>   - Deploy 2: Migrate data, new code writes to new column
>   - Deploy 3: Remove old column
>
> **Problem 4: PodDisruptionBudget not set**
> - Node upgrade can evict all pods of a deployment at once
> - Fix:
> ```yaml
> apiVersion: policy/v1
> kind: PodDisruptionBudget
> spec:
>   minAvailable: 2   # always keep at least 2 pods running
>   selector:
>     matchLabels:
>       app: my-service
> ```

---

## Round 4: Terraform & IaC Deep Dive

---

**Q14. What is Terraform state and what problems have you faced with remote state in GCP?**

> **State is Terraform's source of truth** — maps your HCL config to real GCP resources.
>
> **Remote state on GCS:**
> ```hcl
> terraform {
>   backend "gcs" {
>     bucket = "my-tf-state"
>     prefix = "prod/gke"
>   }
> }
> ```
>
> **Problems I've actually faced:**
>
> **1. State lock conflicts**
> - Two engineers run `terraform apply` simultaneously → one gets a lock error
> - GCS backend uses object locking for state lock
> - Fix: Never run apply locally in prod — only via Cloud Build pipeline where applies are serialized
>
> **2. State drift from manual changes**
> - Someone deletes a GCS bucket manually → next apply fails because TF tries to delete a resource that doesn't exist
> - Fix: `terraform state rm google_storage_bucket.my_bucket` to remove from state, then re-import or recreate
>
> **3. Sensitive values in state**
> - Cloud SQL password ends up in plain text in state file
> - Fix: Use Secret Manager + `data.google_secret_manager_secret_version` to fetch secrets at apply time, never hardcode in state
>
> **4. Large state files slow plan/apply**
> - Monolithic state with 500+ resources → `terraform plan` takes 10 minutes
> - Fix: Split state by component (networking, GKE, databases) using separate state files and `terraform_remote_state` data sources to share outputs

---

**Q15. A `terraform destroy` was accidentally run on a production environment. What do you do?**

> **This has actually happened.** Interviewers want to know your recovery plan.
>
> **Immediate (first 5 min):**
> 1. Stop the destroy if still running: `Ctrl+C` — Terraform stops after current resource. GCP operations may still complete in background.
> 2. Alert the team. This is a P0.
>
> **Recovery:**
> - **Stateful resources (databases):** Cloud SQL has automatic backups. Restore from last backup.
>   ```bash
>   gcloud sql backups restore BACKUP_ID --restore-instance=INSTANCE_NAME
>   ```
> - **GKE cluster:** If cluster is gone, re-apply Terraform to recreate. Nodes are stateless — they come back. Persistent volumes via Persistent Disk — check if `reclaimPolicy: Retain` was set. If yes, PDs still exist and can be re-attached.
> - **Cloud Storage:** Enable soft delete policy (now default in GCP) — buckets and objects can be restored within retention window.
>
> **Prevention:**
> - IAM: Only CI/CD service account has `terraform apply` permissions. No human has it on prod.
> - Sentinel policy (Terraform Cloud) or OPA: Block `terraform destroy` on prod workspaces
> - `lifecycle { prevent_destroy = true }` on critical resources:
> ```hcl
> resource "google_sql_database_instance" "main" {
>   lifecycle {
>     prevent_destroy = true
>   }
> }
> ```
> - Separate state backend for prod with more restrictive IAM

---

## Round 5: Behavioral / Leadership (Senior Level)

---

**Q16. Tell me about a time a production incident happened because of something you deployed. What happened and what did you change?**

> **What they're looking for:** Ownership, systematic fix, process improvement. Not blame.
>
> **Structure your answer (STAR):**
> - **Situation:** Describe the system and what you deployed
> - **What went wrong:** Be specific — error rate, latency, user impact
> - **What you did:** Immediate mitigation (rollback/hotfix), root cause investigation
> - **What you changed:** Not just the code fix but the process — what gate was missing that would have caught this?
>
> **Example answer:**
> "I deployed a config change to our GKE yserv service that updated the Athenz JWT token domain. The change caused all warmup requests to fail with CLIENT_JWT_NULL because the cache key didn't match what the SIA agent was generating. Pods couldn't complete warmup, readiness probes failed, and canary deployment was blocked.
>
> I rolled back immediately, then traced the issue to a domain mismatch between what the config expected and what SIA was generating. Fixed it with the correct prd domain.
>
> What we changed: added a pre-deployment validation step that actually calls the token endpoint and verifies the JWT can be fetched with the configured role, so this mismatch would be caught in staging before reaching prod."

---

**Q17. A junior engineer on your team wrote a Terraform module that works but is not following best practices. How do you handle the code review?**

> **What they're testing:** How you mentor without demotivating. Senior engineers are multipliers.
>
> **What NOT to say:** "I rewrote it" or "I rejected it".
>
> **Good answer:**
> - Approve what works. Acknowledge the effort.
> - Leave specific, educational comments — not "this is wrong" but "here's why `prevent_destroy = true` matters on databases and what happens without it"
> - Distinguish between blockers (security issue, will break prod) vs nice-to-haves (naming convention)
> - Pair program on one of the issues to teach rather than just comment
> - Follow up in next PR — did they apply the feedback?

---

**Q18. Your team is asked to cut cloud costs by 30% in 2 weeks. The manager wants a quick win. What do you prioritize?**

> **Quick wins (can do in 2 weeks):**
> 1. **Enable Committed Use Discounts** — 1-click in console, immediate 37% discount on existing VMs. No code change.
> 2. **Storage lifecycle policies** — Script to apply Nearline/Coldline policies to existing buckets with old data. 1-day effort.
> 3. **Shut down or reduce dev/staging environments on weekends** — CronJob to scale GKE node pools to 0 nights and weekends. 60-70% savings on dev infra.
> 4. **Delete unattached Persistent Disks** — `gcloud compute disks list --filter="users:( )"` — often TBs of orphaned disks from deleted VMs.
>
> **Tell the manager:** These 4 items alone can achieve 30%+ with low risk. Deeper optimisation (right-sizing, architecture changes) takes longer but gives more savings.

---

## Quick Reference: What Senior Interviewers Actually Judge

| They ask about | They're really testing |
|---|---|
| Debugging a production issue | Do you rollback first or investigate first? |
| GKE networking | Do you know VPC-native vs overlay? |
| Terraform state | Have you actually hit state conflicts? |
| Cost optimisation | Do you know CUDs vs Spot vs right-sizing? |
| Zero-downtime deployment | Do you know the preStop hook + PDB pattern? |
| Secret management | Do you know Workload Identity + Secret Manager? |
| System design | Can you explain WHY you chose each service? |
| Incident response | Do you own mistakes and fix the process? |

---

## What to always say in a senior interview

- **"I rolled back first, then investigated"** — shows production discipline
- **"We did a blameless postmortem"** — shows maturity
- **"We automated that so it can't happen again"** — shows engineering mindset
- **"The trade-off was X vs Y, we chose Y because of our specific constraints"** — shows depth
- **Never say "I don't know"** — say "I haven't faced that specific case but here's how I'd approach it"

---

*These questions are based on real senior/staff engineer interview patterns at cloud-focused companies.*
