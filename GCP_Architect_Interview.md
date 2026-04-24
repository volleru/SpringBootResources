# GCP Cloud Architect Interview Prep — TCS

> Covers core architect-level topics: design, compute, networking, data, security, migration, and cost.

---

## Category Index

1. [Core GCP Concepts](#1-core-gcp-concepts) — Q1–Q5
2. [Compute & Containers](#2-compute--containers) — Q6–Q12
3. [Networking & Load Balancing](#3-networking--load-balancing) — Q13–Q18
4. [Storage & Databases](#4-storage--databases) — Q19–Q24
5. [Data & Analytics](#5-data--analytics) — Q25–Q29
6. [Security & IAM](#6-security--iam) — Q30–Q35
7. [Architecture & Design](#7-architecture--design) — Q36–Q41
8. [Migration & Hybrid Cloud](#8-migration--hybrid-cloud) — Q42–Q46
9. [Cost Optimization](#9-cost-optimization) — Q47–Q50

---

## 1. Core GCP Concepts

---

### Q1. What is the GCP resource hierarchy and why does it matter for an architect?

**Answer:**

```
Organization
    └── Folders (optional — for departments / teams)
            └── Projects
                    └── Resources (VMs, buckets, DBs...)
```

| Level | Purpose |
|---|---|
| **Organization** | Root node tied to a Google Workspace/Cloud Identity domain |
| **Folder** | Group projects by department, environment (prod/dev), or team |
| **Project** | Unit of billing, API enablement, and resource isolation |
| **Resource** | Actual GCP service (VM, bucket, GKE cluster, etc.) |

**Why it matters for architects:**
- IAM policies, org policies, and billing are all inherited down this hierarchy
- You can grant a role at folder level — it applies to all projects inside
- Org policies (like "deny public IPs") can be enforced at org or folder level and cascade down
- Architects design this hierarchy to match the company's team/environment structure

---

### Q2. What is a GCP Project? What happens when you delete one?

**Answer:** A project is the base unit of GCP. It:
- Has a unique Project ID (immutable) and Project Number
- Is the billing unit — all resource costs are attributed to a project
- Controls which APIs are enabled
- Defines the IAM boundary

**When deleted:**
- Enters a **30-day shutdown period** — resources stop but aren't permanently deleted
- Can be restored within 30 days
- After 30 days, all resources are permanently deleted and billing stops

**Senior tip:** Always use **Folders** to group projects per environment. Don't put prod and dev in the same project — shared IAM, shared quotas, and one mistake affects both.

---

### Q3. What is the difference between a Region, Zone, and Multi-Region in GCP?

**Answer:**

| Concept | Description | Example |
|---|---|---|
| **Zone** | Single data center within a region. Smallest failure domain. | `us-central1-a` |
| **Region** | Cluster of 3+ zones in geographic proximity. Low latency between zones (~1ms). | `us-central1` |
| **Multi-Region** | Geo-redundant grouping of regions. Used for globally distributed storage. | `US`, `EU`, `ASIA` |

**Architect decision guide:**

| Need | Deploy where |
|---|---|
| Single VM / non-critical workload | Single zone |
| High availability, survive zone failure | Multi-zone (e.g., GKE regional cluster) |
| Survive region failure / DR | Multi-region |
| Globally distributed static content | Cloud CDN + multi-region bucket |

---

### Q4. What are Google-managed vs customer-managed vs customer-supplied encryption keys?

**Answer:**

| Type | Who manages the key | Use case |
|---|---|---|
| **Google-managed (GMEK)** | Google rotates and stores keys automatically | Default — good for most workloads |
| **Customer-managed (CMEK)** | You create key in Cloud KMS; Google uses it | Compliance — you control rotation/revocation |
| **Customer-supplied (CSEK)** | You provide key with every API call; GCP never stores it | Highest sensitivity — you own everything |

**When to use CMEK:** PCI-DSS, HIPAA, or any regulation requiring you to demonstrate key control. If a key is deleted in Cloud KMS, data encrypted with it is irrecoverable.

---

### Q5. What is the GCP Shared Responsibility Model?

**Answer:**

| Layer | Google's Responsibility | Customer's Responsibility |
|---|---|---|
| Physical infrastructure | Hardware, data centers, power | Nothing |
| Network | Global network, DDoS infrastructure | VPC design, firewall rules |
| Compute (managed) | Patching managed services (Cloud SQL, GKE nodes in Autopilot) | App-level security, data |
| Compute (IaaS - GCE) | Hypervisor | OS patching, VM hardening |
| Data | Storage durability | Classifying, encrypting, access control |
| Identity | Google identity platform | IAM policies, service accounts |
| Application | Nothing | Code security, secrets management |

---

## 2. Compute & Containers

---

### Q6. When would you choose Compute Engine vs GKE vs Cloud Run vs App Engine vs Cloud Functions?

**Answer:**

| Service | Best For | Key Characteristic |
|---|---|---|
| **Compute Engine (GCE)** | Legacy app lift-and-shift, full OS control, custom runtimes | Maximum control, maximum ops overhead |
| **GKE Standard** | Microservices needing fine-grained cluster control | You manage node pools, autoscaling config |
| **GKE Autopilot** | Microservices where you just want to deploy pods | Google manages nodes, billing per pod |
| **Cloud Run** | Stateless HTTP containers, event-driven workloads | Scales to zero, per-request billing |
| **App Engine Standard** | Simple web apps in supported runtimes (Java, Python, Go) | Scales to zero, opinionated framework |
| **Cloud Functions** | Single-purpose event handlers, lightweight triggers | Scales to zero, simplest deployment unit |

**Decision tree:**
```
Need full OS control?           → Compute Engine
Stateful microservices / K8s?   → GKE Autopilot (default) / Standard (if tuning needed)
Stateless HTTP container?       → Cloud Run
Simple event handler?           → Cloud Functions (gen2)
Legacy web app, no containers?  → App Engine
```

---

### Q7. What is the difference between GKE Standard and GKE Autopilot?

**Answer:**

| | GKE Standard | GKE Autopilot |
|---|---|---|
| Node management | You manage node pools, sizes, autoscalers | Google manages all nodes |
| Billing | Per node (VM) — even idle nodes cost money | Per pod resource request (vCPU + memory) |
| Customization | Full — choose machine type, GPU, taints | Limited — Google enforces security best practices |
| Scale to zero | No (node pool always has min nodes) | Yes — no nodes when no pods |
| Use case | When you need GPUs, DaemonSets, node-level tuning | Default choice for most workloads |

**Senior tip:** Default to Autopilot for new workloads. Move to Standard only when you need DaemonSets, specific GPU types, or custom node images.

---

### Q8. What is a Managed Instance Group (MIG)? How does it differ from an unmanaged IG?

**Answer:** A **Managed Instance Group** creates and manages identical VMs from an **instance template**.

| | Managed (MIG) | Unmanaged |
|---|---|---|
| Instances | Identical — created from a template | Can be different VMs |
| Autoscaling | Yes | No |
| Auto-healing | Yes — replaces unhealthy VMs | No |
| Rolling updates | Yes | No |
| Use with LB | Yes (backend service) | Yes (but limited) |

**MIG types:**
- **Zonal MIG** — all VMs in one zone; simpler, lower latency between VMs
- **Regional MIG** — VMs spread across zones; survives zone failure → use for production

---

### Q9. What is Preemptible / Spot VM? When should an architect use them?

**Answer:** Spot VMs (formerly Preemptible) are spare compute capacity sold at up to **91% discount**. Google can reclaim them with a 30-second notice at any time.

**Use:**
- Batch jobs, data processing pipelines (Dataflow workers, BigQuery BI Engine)
- Fault-tolerant, stateless workloads
- ML/AI training jobs (checkpointing to GCS)
- GKE node pools for non-critical pods

**Never use for:**
- Databases (stateful, can't handle sudden termination)
- Real-time transaction processing
- Anything requiring guaranteed availability SLA

**Architecture pattern:** Use a **mixed node pool** in GKE — on-demand nodes for critical pods, Spot nodes for batch/worker pods with `tolerations`.

---

### Q10. What is Cloud Run? How does it handle concurrency?

**Answer:** Cloud Run runs stateless containers triggered by HTTP requests or Pub/Sub events. It automatically scales up (and down to zero) based on traffic.

**Key settings:**

| Setting | Default | What it controls |
|---|---|---|
| `--concurrency` | 80 | Max simultaneous requests per container instance |
| `--max-instances` | 1000 | Cap on how many instances scale out |
| `--min-instances` | 0 | Set > 0 to eliminate cold starts |
| `--cpu` | 1 | Can set CPU to always-on (not just during request) |

**Concurrency model:**
- Each container instance handles multiple requests simultaneously (unlike Lambda's 1 per instance)
- If all instances are at max concurrency → new instance starts (cold start ~200ms for JVM, ~50ms for Go)

**Senior tip:** For Java/Spring Boot on Cloud Run, set `--min-instances=1` to avoid cold start penalties in production. For truly serverless cost optimization, keep it at 0.

---

### Q11. What is Anthos? When would you recommend it?

**Answer:** Anthos is Google's **hybrid and multi-cloud platform** built on GKE. It lets you run and manage Kubernetes workloads consistently across:
- GCP (GKE)
- On-premises (Anthos clusters on VMware or bare metal)
- AWS / Azure (Anthos attached clusters)

**Key components:**
| Component | What it does |
|---|---|
| Anthos Config Management | GitOps — apply policies and configs across all clusters from a Git repo |
| Anthos Service Mesh (ASM) | Managed Istio — mTLS, traffic management, observability across clusters |
| Cloud Run for Anthos | Run Cloud Run workloads on-prem |

**When to recommend:**
- Enterprise migrating to cloud gradually — some workloads stay on-prem for compliance
- Multi-cloud strategy required by the business
- Consistent security policy across on-prem + cloud

---

### Q12. What is the difference between vertical scaling and horizontal scaling on GCP? What are the limits?

**Answer:**

| | Vertical Scaling (Scale Up) | Horizontal Scaling (Scale Out) |
|---|---|---|
| How | Increase VM size (more CPU/RAM) | Add more VM instances |
| GCP mechanism | Resize GCE instance (requires stop/start) | MIG autoscaling, GKE HPA |
| Limit | Max 416 vCPU, 11.7 TB RAM per GCE VM | Virtually unlimited (quota-bound) |
| Downtime | Yes (for GCE resize) | No |
| Best for | Stateful DBs, legacy apps | Stateless microservices, web servers |

**Architect rule:** Design for horizontal scaling from day one. Vertical scaling is a short-term fix, not an architecture.

---

## 3. Networking & Load Balancing

---

### Q13. What is a VPC in GCP? How is it different from AWS VPC?

**Answer:** A GCP VPC is a **global resource** — one VPC spans all regions. Subnets are regional.

| | GCP VPC | AWS VPC |
|---|---|---|
| Scope | **Global** — spans all regions | Regional — one VPC per region |
| Subnets | Regional — span all zones in a region | Availability Zone specific |
| Peering | VPC peering (non-transitive) | VPC peering (non-transitive) |
| Private connectivity | Private Service Connect, VPC peering | PrivateLink, VPC endpoints |
| Default VPC | One per project (auto mode) | One per region per account |

**Subnet types:**
- **Auto mode VPC** — automatically creates one subnet per region (`10.128.0.0/9`)
- **Custom mode VPC** — you define every subnet — recommended for production

**Senior tip:** Never use auto mode VPC in production. The IP ranges overlap with common on-prem ranges, causing VPN/interconnect issues.

---

### Q14. What are the GCP Load Balancer types? When do you use each?

**Answer:**

| Load Balancer | Scope | Protocol | Use Case |
|---|---|---|---|
| **Global External HTTP(S) LB** | Global | HTTP/HTTPS, HTTP/2, gRPC | Public web apps, APIs — single anycast IP worldwide |
| **Regional External HTTP(S) LB** | Regional | HTTP/HTTPS | Single-region public web app |
| **External TCP/UDP Network LB** | Regional | TCP/UDP | Non-HTTP traffic, gaming servers |
| **Internal HTTP(S) LB** | Regional | HTTP/HTTPS | Private microservice-to-microservice traffic inside VPC |
| **Internal TCP/UDP LB** | Regional | TCP/UDP | Internal DB proxies, private services |
| **SSL Proxy LB** | Global | SSL/TLS (non-HTTP) | SSL termination for TCP apps |

**Most common architect choice:** Global External HTTP(S) LB for public-facing apps — it terminates SSL, integrates with Cloud Armor (WAF), Cloud CDN, and routes to backends in any region.

---

### Q15. What is Cloud CDN? How does caching work?

**Answer:** Cloud CDN is Google's content delivery network. It caches HTTP(S) responses at Google's **edge PoPs** (Points of Presence) globally — close to end users.

**How it works:**
1. User request hits nearest Google edge PoP
2. Cache HIT → response served from edge (no backend call)
3. Cache MISS → request forwarded to backend, response cached at edge

**Cache key defaults:** full URL (method + host + path + query). Can customize to exclude certain query params.

**Cache modes:**

| Mode | Behavior |
|---|---|
| `CACHE_ALL_STATIC` | Cache static content (images, CSS, JS) based on content type |
| `USE_ORIGIN_HEADERS` | Respect `Cache-Control` headers from backend |
| `FORCE_CACHE_ALL` | Cache everything regardless of headers (override origin) |

**Architect use case:** Attach Cloud CDN to Global LB → dramatically reduces backend load and latency for static assets and cacheable API responses.

---

### Q16. What is VPC Peering vs Shared VPC vs Cloud VPN vs Cloud Interconnect?

**Answer:**

| Option | What it is | Use case |
|---|---|---|
| **VPC Peering** | Direct private connectivity between two VPCs (same or different projects/orgs) | Two teams' projects need to talk privately |
| **Shared VPC** | One host project owns the VPC; service projects use its subnets | Centralized network team manages one VPC; app teams deploy into it |
| **Cloud VPN** | Encrypted IPsec tunnel over public internet to on-prem | Small/medium on-prem traffic, lower cost, tolerates some latency |
| **Cloud Interconnect** | Dedicated or partner private physical connection to GCP | High-bandwidth, low-latency, guaranteed on-prem connection |

**Key VPC Peering limitation:** Non-transitive — if A peers with B and B peers with C, A cannot reach C. Use Shared VPC or a hub-and-spoke with NVA for transitive routing.

---

### Q17. What is Cloud Armor? What problems does it solve?

**Answer:** Cloud Armor is GCP's **WAF (Web Application Firewall)** and **DDoS protection** service, attached to the Global External HTTP(S) Load Balancer.

**What it provides:**
| Feature | Description |
|---|---|
| **DDoS protection** | Always-on volumetric DDoS mitigation at Google's edge |
| **WAF rules** | Pre-built OWASP Top 10 rule sets (SQL injection, XSS, etc.) |
| **IP allow/deny lists** | Block or allow specific IPs or CIDR ranges |
| **Geo-based rules** | Block traffic from specific countries |
| **Rate limiting** | Throttle requests per IP to prevent abuse |
| **Adaptive protection** | ML-based anomaly detection for Layer 7 DDoS |

**Architect tip:** Always attach Cloud Armor to public-facing Global LBs in production. Enable the OWASP managed rule set and start in preview mode before switching to deny to avoid false positives.

---

### Q18. What is Private Google Access? Why is it important?

**Answer:** Private Google Access allows VM instances **without external IP addresses** to reach Google APIs and services (BigQuery, GCS, etc.) using internal IP addresses — traffic stays on Google's network.

**Without Private Google Access:**
```
VM (no external IP) → cannot reach storage.googleapis.com → API calls fail
```

**With Private Google Access enabled on subnet:**
```
VM (no external IP) → routes via 199.36.153.4/30 (restricted.googleapis.com) → GCS/BQ/etc.
Traffic never leaves Google's network
```

**Why architects care:**
- Security best practice: VMs in private subnets should never need external IPs
- Compliance: data never traverses public internet
- Enable via: subnet setting + DNS pointing `*.googleapis.com` to `199.36.153.4`

---

## 4. Storage & Databases

---

### Q19. What are the GCP storage options and when do you use each?

**Answer:**

| Service | Type | Best For |
|---|---|---|
| **Cloud Storage (GCS)** | Object storage | Backups, data lake, static assets, ML datasets |
| **Persistent Disk (PD)** | Block storage | GCE VM boot/data disks |
| **Filestore** | Managed NFS | Shared file system for GKE / GCE workloads |
| **Cloud SQL** | Managed relational (MySQL, PostgreSQL, SQL Server) | OLTP apps needing SQL |
| **Cloud Spanner** | Globally distributed relational | Global OLTP requiring horizontal scale + strong consistency |
| **Firestore** | Serverless NoSQL document DB | Mobile/web apps, real-time sync |
| **Bigtable** | Wide-column NoSQL | High-throughput time-series, IoT, analytics |
| **Memorystore** | Managed Redis / Memcached | Session cache, rate limiting, leaderboards |
| **AlloyDB** | PostgreSQL-compatible, in-memory acceleration | High-performance PostgreSQL workloads, analytics |

---

### Q20. What is the difference between Cloud SQL and Cloud Spanner?

**Answer:**

| | Cloud SQL | Cloud Spanner |
|---|---|---|
| Engine | MySQL, PostgreSQL, SQL Server | Google-proprietary (SQL-compatible) |
| Scale | Vertical (max ~96 vCPU, 624 GB) | Horizontal (unlimited nodes) |
| Global distribution | Read replicas in other regions | True multi-region active-active writes |
| Consistency | Strong (single region) | External consistency (globally) |
| Availability SLA | 99.95% (HA config) | 99.999% (multi-region) |
| Cost | Low–Medium | High (min ~$0.90/node/hour) |
| Use case | Standard OLTP (e-commerce, SaaS) | Global apps needing horizontal write scale |

**When to use Spanner:** Financial systems (global ledger), gaming (global leaderboard), anything needing 5-nines SLA with multi-region writes. If Cloud SQL is fast enough — use Cloud SQL.

---

### Q21. What are GCS storage classes? How do you choose?

**Answer:**

| Storage Class | Min storage duration | Access | Cost vs Standard |
|---|---|---|---|
| **Standard** | None | Frequent | Baseline |
| **Nearline** | 30 days | Once/month or less | ~50% cheaper storage, retrieval fee |
| **Coldline** | 90 days | Once/quarter or less | ~75% cheaper storage, higher retrieval fee |
| **Archive** | 365 days | Less than once/year | ~95% cheaper storage, highest retrieval fee |

**Architect tip — Object Lifecycle Management:**
```
Day 0:    Upload → Standard
Day 30:   → auto-transition to Nearline
Day 90:   → auto-transition to Coldline
Day 365:  → auto-transition to Archive
Day 2555: → auto-delete (7-year compliance window)
```
Set this with GCS Lifecycle rules — no code needed.

---

### Q22. What is Cloud Bigtable? When would you recommend it over Firestore or BigQuery?

**Answer:** Bigtable is a petabyte-scale, low-latency, wide-column NoSQL database. Originally Google's internal DB for Search, Maps, and Gmail.

| Choose Bigtable when... | Choose Firestore when... | Choose BigQuery when... |
|---|---|---|
| >1TB of data | Mobile/web app with real-time sync | Analytics / reporting (not OLTP) |
| >10K reads/writes per second | Hierarchical document data | SQL queries over large datasets |
| Time-series, IoT, logs | Need offline client sync | Batch processing |
| ML feature store | Smaller dataset (<1TB) | |

**Key design principle:** In Bigtable, **the row key IS the index** — all reads must start from a row key or range. Design row keys carefully to avoid hotspots (don't use sequential IDs or timestamps as prefix).

---

### Q23. What is the difference between Memorystore Redis and Memcached?

**Answer:**

| | Memorystore for Redis | Memorystore for Memcached |
|---|---|---|
| Data structures | Strings, hashes, lists, sets, sorted sets, streams | Strings only |
| Persistence | Yes (RDB snapshots, AOF) | No |
| Replication | Yes (primary + read replicas) | No (sharded, stateless) |
| Use cases | Sessions, rate limiting, pub/sub, leaderboards, queues | Simple key-value cache, high-throughput read cache |

**Senior tip:** Default to Redis. Use Memcached only if you need simple caching at extremely high throughput with horizontal sharding and don't need persistence.

---

### Q24. How would you design a disaster recovery strategy for Cloud SQL?

**Answer:** Design based on **RTO** (Recovery Time Objective) and **RPO** (Recovery Point Objective):

| Strategy | RTO | RPO | Cost |
|---|---|---|---|
| **Automated backups only** | Hours | Up to 24h | Low |
| **HA (same-region standby)** | ~60 seconds | Near zero (synchronous) | Medium (2x instance) |
| **Cross-region read replica** | Minutes (manual promote) | Seconds (async lag) | Medium–High |
| **HA + cross-region replica** | ~60s (regional), minutes (cross-region) | Near zero | High |

```
Production setup recommendation:
  Primary (us-central1) — HA enabled (auto-failover within region)
      └── Read Replica (us-east1) — for DR + offload reporting queries
  Point-in-time recovery: 7 days
  Daily backups: retained 30 days
```

---

## 5. Data & Analytics

---

### Q25. What is BigQuery? What makes its architecture unique?

**Answer:** BigQuery is a fully managed, serverless data warehouse that separates **storage and compute**.

**Architecture:**
- **Dremel execution engine** — massively parallel SQL query engine
- **Colossus storage** — columnar storage; reads only columns needed by the query
- **Jupiter network** — Google's petabit internal network connecting storage and compute
- **Capacitor** — columnar file format optimized for analytical queries

**Why it's fast:** Reads only the columns queried (columnar), distributes work across thousands of slots, no indexing needed.

**Key features:**

| Feature | Description |
|---|---|
| **Slot-based billing** | On-demand: bill per TB scanned. Flat-rate: reserve slots for predictable cost |
| **Partitioning** | Partition tables by date/column — queries only scan relevant partitions |
| **Clustering** | Sort data within partitions by column — further reduces data scanned |
| **Materialized views** | Pre-computed query results updated incrementally |
| **BigQuery ML** | Train ML models using SQL inside BigQuery |
| **Omni** | Run BigQuery on AWS/Azure data without moving it |

---

### Q26. What is Cloud Pub/Sub? What patterns does it enable?

**Answer:** Pub/Sub is a fully managed, global, real-time messaging service. Decouples publishers from subscribers.

**Core concepts:**
- **Topic** — named resource publishers send messages to
- **Subscription** — named resource that receives messages from a topic
- **Push subscription** — Pub/Sub pushes messages to an HTTPS endpoint
- **Pull subscription** — subscriber polls Pub/Sub for messages

**Patterns it enables:**

| Pattern | How |
|---|---|
| **Fan-out** | One topic, multiple subscriptions — each subscriber gets all messages (e.g., audit + processing) |
| **Event-driven pipeline** | GCS upload → Pub/Sub → Cloud Function → process |
| **Work queue** | One subscription, multiple subscribers — each message delivered to only one |
| **Buffering** | Absorb traffic spikes between producer and slow consumer |
| **Dead letter topic** | Messages that fail N times are forwarded to a dead-letter topic for inspection |

**Retention:** Messages retained up to 7 days (or 31 days with message retention). Replay from any point in retention window.

---

### Q27. What is Dataflow? How is it different from Dataproc?

**Answer:**

| | Dataflow | Dataproc |
|---|---|---|
| Model | Managed Apache Beam pipelines | Managed Hadoop / Spark cluster |
| Server management | Fully serverless — no cluster to manage | You manage cluster lifecycle (can autoscale) |
| Scaling | Auto-scales workers mid-job | Pre-provision or autoscale node pool |
| Programming model | Apache Beam (Java, Python, Go) | Spark, Hadoop, Hive, Pig, Flink |
| Streaming | First-class (unified batch + stream) | Possible but more complex |
| Cost | Pay per vCPU/hour used | Pay for cluster uptime (even if idle) |
| Use case | New pipelines, ETL, streaming analytics | Existing Spark/Hadoop code, lift-and-shift |

**Architect rule:** New pipelines → Dataflow. Existing Spark jobs → Dataproc. Dataproc is also cheaper for long-running, predictable batch workloads.

---

### Q28. What is the difference between Looker, Looker Studio, and Data Studio?

**Answer:**

| | Looker | Looker Studio (formerly Data Studio) |
|---|---|---|
| Type | Enterprise BI platform | Free self-service reporting tool |
| Modeling layer | LookML — semantic model defines metrics centrally | No modeling layer — connect directly to source |
| Governance | Strong — one definition of "revenue" for everyone | Weak — each report can define differently |
| Target user | Enterprise BI teams, data governance focus | Analysts, developers, quick dashboards |
| Cost | Expensive (enterprise license) | Free |
| Embedding | Strong API + embedding capabilities | Basic embedding |

**Senior tip:** For large enterprises, Looker ensures "single source of truth" for metrics. For quick dashboards on BigQuery, Looker Studio is free and fast.

---

### Q29. How would you design a data pipeline from raw ingestion to reporting?

**Answer:** Classic **Lambda / Medallion architecture** on GCP:

```
[Sources]          [Ingestion]       [Storage]         [Processing]      [Serving]
App events    →    Pub/Sub       →   GCS (raw)     →   Dataflow      →   BigQuery
DB changes    →    Datastream    →   GCS (raw)     →   Dataflow      →   BigQuery
Batch files   →    Transfer Svc  →   GCS (raw)     →   Dataproc      →   BigQuery
                                                                      →   Looker Studio
```

**Storage layers (Medallion):**
| Layer | Location | Content |
|---|---|---|
| **Bronze (raw)** | GCS bucket (raw/) | Unmodified source data, immutable |
| **Silver (cleaned)** | GCS bucket (processed/) or BQ staging | Validated, deduplicated, typed |
| **Gold (curated)** | BigQuery production dataset | Business-ready aggregated tables |

---

## 6. Security & IAM

---

### Q30. What is the difference between a Service Account and a User Account?

**Answer:**

| | User Account | Service Account |
|---|---|---|
| Represents | A human | An application or workload |
| Credentials | Google account password + MFA | JSON key file OR metadata server token |
| Use in code | Never — use service accounts | Yes — applications assume SA identity |
| Best practice | MFA, context-aware access | No keys if possible — use Workload Identity |

**Key principle — Workload Identity (on GKE):**
Instead of mounting a JSON key file into a pod (which can be leaked), bind a Kubernetes Service Account to a GCP Service Account. The pod gets a token from the metadata server automatically — no key files anywhere.

```yaml
# Bind k8s SA to GCP SA — no keys needed
kubectl annotate serviceaccount my-ksa \
  iam.gke.io/gcp-service-account=my-gsa@project.iam.gserviceaccount.com
```

---

### Q31. What is the Principle of Least Privilege? How do you apply it in GCP?

**Answer:** Grant only the minimum permissions needed to perform a task — nothing more.

**GCP implementation:**

| Practice | How |
|---|---|
| Use predefined roles over basic roles | `roles/storage.objectViewer` instead of `roles/editor` |
| Avoid basic roles (Owner/Editor/Viewer) in production | They grant too broadly |
| Use custom roles when no predefined role fits | Fine-grained permissions |
| Grant at lowest resource level | Grant at bucket level, not project level |
| Use Conditions | Time-bound access — `request.time < timestamp("2026-06-01T00:00:00Z")` |
| Regularly audit with IAM Recommender | Google ML suggests overly permissive roles to remove |

---

### Q32. What is VPC Service Controls? When do you need it?

**Answer:** VPC Service Controls creates a **security perimeter** around GCP services (BigQuery, GCS, etc.) that prevents data exfiltration even if credentials are compromised.

**Problem it solves:**
```
Without VPC SC: Attacker steals service account key → copies your BQ data to their own project
With VPC SC: Even with valid credentials, API calls from outside the perimeter are blocked
```

**Use cases:**
- Regulated industries (financial, healthcare) — prevent insider data exfiltration
- Projects handling PII or sensitive data
- When you need "data can only leave through approved paths"

**Access Levels:** Can allow specific identities, IP ranges, or devices to cross the perimeter (e.g., your corp network can access, personal devices cannot).

---

### Q33. What is Cloud KMS? What is Secret Manager? How are they different?

**Answer:**

| | Cloud KMS | Secret Manager |
|---|---|---|
| Stores | Encryption **keys** (AES, RSA, EC) | Application **secrets** (API keys, passwords, certs) |
| Use | Encrypt/decrypt data; sign/verify | Inject secrets into apps at runtime |
| Rotation | Manual or auto key rotation | Versions — new version on rotation |
| Access | Encrypt/decrypt API calls | `gcloud secrets versions access` or SDK call |

**When to use what:**
- **Cloud KMS:** CMEK for BigQuery/GCS/Pub/Sub, envelope encryption in your app, signing JWTs
- **Secret Manager:** DB passwords, API keys, OAuth client secrets — anything your app needs to read at startup

**Senior tip:** Never put secrets in environment variables in Cloud Run / GKE directly. Mount them from Secret Manager. This way secrets are auditable, rotatable, and never appear in deployment configs.

---

### Q34. What is Binary Authorization?

**Answer:** Binary Authorization is a deploy-time security control for GKE and Cloud Run that ensures only **trusted, signed container images** can be deployed.

**How it works:**
1. CI/CD pipeline builds image and pushes to Artifact Registry
2. CI/CD pipeline uses Cloud KMS to **sign the image** with an Attestor
3. Binary Authorization policy requires the signature before allowing deployment
4. If image is unsigned or signature is invalid → deployment is blocked

**Why it matters:** Prevents supply chain attacks — even if an attacker pushes a malicious image to your registry, they can't deploy it without the signing key.

---

### Q35. What is Organization Policy? Give 3 practical examples.

**Answer:** Org policies are **guardrails** applied at Org/Folder/Project level that prevent specific actions regardless of IAM permissions.

| Policy Constraint | What it enforces |
|---|---|
| `compute.vmExternalIpAccess` | Block all VMs from having external IPs (force private-only) |
| `iam.disableServiceAccountKeyCreation` | Prevent creating SA JSON keys (force Workload Identity) |
| `compute.requireOsLogin` | Require OS Login for SSH — ties SSH access to IAM |
| `storage.uniformBucketLevelAccess` | Disable per-object ACLs; use only IAM for bucket access |
| `gcp.resourceLocations` | Restrict all resources to specific regions (data residency) |
| `iam.allowedPolicyMemberDomains` | Only allow IAM bindings from your org's domain |

**Senior tip:** Apply `compute.vmExternalIpAccess = deny` at the org level with exceptions for specific projects. This single policy eliminates an entire class of security exposure.

---

## 7. Architecture & Design

---

### Q36. How would you design a highly available, global web application on GCP?

**Answer:**

```
Users (global)
    │
    ▼
Cloud DNS (Geo-routing)
    │
    ▼
Global External HTTP(S) LB  ←── Cloud Armor (WAF/DDoS)
    │                    └────── Cloud CDN (edge cache)
    │
    ├── Backend: Cloud Run (us-central1)
    ├── Backend: Cloud Run (europe-west1)
    └── Backend: Cloud Run (asia-east1)
             │
             ▼
     Cloud Spanner (multi-region: nam-eur-asia1)
     or Cloud SQL (regional HA) + cross-region read replica
```

**Key decisions:**
| Decision | Choice | Reason |
|---|---|---|
| Compute | Cloud Run | Auto-scales to zero, pay per request, no idle cost |
| LB | Global External HTTP(S) LB | Single anycast IP, routes to nearest healthy region |
| DB | Cloud Spanner (global) | Multi-region writes with strong consistency |
| Cache | Memorystore Redis per region | Session storage, rate limiting |
| CDN | Cloud CDN on LB | Reduce backend hits for static/cacheable content |
| Security | Cloud Armor + VPC SC | WAF + data exfiltration prevention |

---

### Q37. What is the Strangler Fig pattern in the context of GCP migration?

**Answer:** Incrementally replace a monolith by routing traffic to new microservices one feature at a time — both run simultaneously until migration is complete.

**On GCP:**
```
Phase 1: All traffic → Monolith on GCE
Phase 2: Add Global LB → route /api/orders/* → Cloud Run (new service)
                       → everything else → Monolith on GCE
Phase 3: Migrate /api/users/* → Cloud Run
Phase 4: Migrate remaining → decommission GCE monolith
```

**Tools:**
- **Global LB URL map** — route by path prefix to different backends
- **Traffic Director** — more advanced traffic splitting (canary %)
- **Cloud Endpoints / Apigee** — API gateway to abstract routing from clients

---

### Q38. What is the difference between horizontal pod autoscaling (HPA) and vertical pod autoscaling (VPA) in GKE?

**Answer:**

| | HPA | VPA |
|---|---|---|
| Scales | Number of pod replicas | CPU/memory requests of each pod |
| Trigger | CPU%, memory%, custom metrics | Actual resource usage over time |
| Downtime | No (adds/removes pods) | Yes — pods are recreated with new requests |
| Use case | Stateless services with variable load | Right-sizing pods, stateful services |

**Can you use both?** Not on the same metric (e.g., both on CPU). But you can use VPA in recommendation mode to right-size requests, then use HPA on CPU% for scaling.

**KEDA (Kubernetes Event-Driven Autoscaling):** Extends HPA to scale on Pub/Sub queue depth, Cloud Tasks queue length, etc. — scale to zero based on message backlog.

---

### Q39. How do you implement blue/green deployments and canary releases on GCP?

**Answer:**

**Blue/Green (on Cloud Run):**
```bash
# Deploy new version (gets no traffic yet)
gcloud run deploy my-service --image gcr.io/project/app:v2 --no-traffic

# Instantly switch 100% traffic to new version
gcloud run services update-traffic my-service --to-revisions=v2=100

# Rollback instantly if needed
gcloud run services update-traffic my-service --to-revisions=v1=100
```

**Canary (on Cloud Run):**
```bash
# Send 5% to new version, 95% to old
gcloud run services update-traffic my-service \
  --to-revisions=v1=95,v2=5

# Gradually shift: 5% → 20% → 50% → 100%
```

**On GKE:** Use Istio/Anthos Service Mesh `VirtualService` for traffic splitting at the sidecar level — no LB config changes needed.

---

### Q40. What is Event-Driven Architecture on GCP? Walk through a practical example.

**Answer:**

**Example: E-commerce order processing**

```
Customer places order
        │
        ▼
Cloud Run (Order API) → saves to Cloud Spanner → publishes to Pub/Sub topic: "order-placed"
                                                              │
                  ┌───────────────────────┬──────────────────┘
                  ▼                       ▼                   ▼
         Cloud Function             Cloud Run            Dataflow
       (send confirm email)     (reserve inventory)   (update analytics)
                                         │
                                         ▼
                                   Pub/Sub: "inventory-updated"
                                         │
                                         ▼
                                  Cloud Function
                                 (notify warehouse)
```

**Benefits:**
- Services are fully decoupled — Order API doesn't know about email, inventory, or analytics
- Each service scales independently
- Failed events go to dead-letter topic for retry/inspection
- Easy to add new consumers (just add a new subscription) without changing publishers

---

### Q41. What is a multi-tenant architecture? How do you isolate tenants on GCP?

**Answer:** Multi-tenancy means multiple customers share the same infrastructure. Isolation levels:

| Isolation Level | Implementation | Trade-off |
|---|---|---|
| **Database row-level** | `tenant_id` column in shared DB | Simplest, cheapest; one bug leaks all data |
| **Schema-level** | Separate schema per tenant in shared DB | Medium isolation; schema migrations complex |
| **Database-level** | Separate Cloud SQL instance per tenant | Strong isolation; expensive at scale |
| **Project-level** | Separate GCP project per tenant | Full isolation; highest cost and ops overhead |

**GCP tools for tenant isolation:**
- **VPC Service Controls** per tenant project
- **Org policy** applied per tenant folder
- **BigQuery row-level security** with row access policies
- **AlloyDB / Cloud Spanner** — logical DB isolation within one cluster

---

## 8. Migration & Hybrid Cloud

---

### Q42. What is the Google Cloud Adoption Framework?

**Answer:** GCP's structured approach to cloud adoption — divided into 4 themes and 3 phases:

**4 Themes:**
| Theme | Focus |
|---|---|
| **Learn** | Training, skills, cloud literacy |
| **Lead** | Executive alignment, cloud governance |
| **Scale** | Center of Excellence, automation, IaC |
| **Secure** | Identity, data protection, compliance |

**3 Phases:**
| Phase | Characteristics |
|---|---|
| **Tactical** | Individual projects, manual processes, learning |
| **Strategic** | Defined processes, some automation, growing team |
| **Transformational** | Cloud-native, fully automated, data-driven decisions |

---

### Q43. What is the 6R migration strategy? Map each to a GCP tool.

**Answer:**

| Strategy | Description | GCP Tool / Approach |
|---|---|---|
| **Rehost** (Lift & Shift) | Move VMs as-is | Migrate for Compute Engine (M4CE) |
| **Replatform** (Lift & Reshape) | Minor optimizations without refactoring | Move to Cloud SQL, GKE (containerize app, keep code) |
| **Repurchase** (Drop & Shop) | Replace with SaaS | Move CRM to Salesforce, email to Google Workspace |
| **Refactor** (Re-architect) | Rebuild cloud-native | Redesign as microservices on Cloud Run + Pub/Sub |
| **Retire** | Decommission unused apps | Identify via inventory assessment |
| **Retain** | Keep on-prem for now | Anthos for hybrid management |

---

### Q44. What is Database Migration Service (DMS)?

**Answer:** Google's managed service for migrating databases to GCP with minimal downtime using **continuous data replication (CDC)**.

**Supported migrations:**
- MySQL → Cloud SQL for MySQL
- PostgreSQL → Cloud SQL for PostgreSQL / AlloyDB
- SQL Server → Cloud SQL for SQL Server
- Oracle → Cloud SQL for PostgreSQL (via conversion)

**How it works (minimal downtime migration):**
```
1. Full dump: copy all existing data to Cloud SQL destination
2. CDC: continuously replicate ongoing changes (binlog / WAL)
3. Lag reaches near-zero → cutover window (seconds of downtime)
4. Update app connection string → done
```

**Senior tip:** Always run DMS in parallel for at least 1 week before cutover to validate data consistency. Use Cloud SQL replicas for read-only validation before switching production.

---

### Q45. What is Transfer Appliance? When would you use it over gsutil?

**Answer:**

| | `gsutil` / Storage Transfer Service | Transfer Appliance |
|---|---|---|
| Method | Network transfer | Physical hardware appliance shipped to you |
| Best for | < 20 TB or fast network | > 20 TB or slow/expensive network |
| Speed limit | Your network bandwidth | Appliance capacity (up to 1 PB) |
| Cost | Network egress fees | Appliance rental fee |

**Rule of thumb:** If transferring data over your network would take more than **a week**, use Transfer Appliance. The formula:

```
Time to transfer = Data size / Network bandwidth
100 TB / 1 Gbps = ~222 hours ≈ 9 days → use Transfer Appliance
```

---

### Q46. What is Migrate for Anthos (now Google Cloud Migrate)? How does it work?

**Answer:** Migrate for Anthos (now called Migrate to Containers) automatically converts VMs into containers — without manually rewriting the app.

**How it works:**
1. Install migration controller in GKE
2. Point it at source (VMware, AWS, Azure, GCE VM)
3. Tool analyzes VM, extracts the application layer
4. Generates `Dockerfile`, Kubernetes `Deployment` YAML, and `PersistentVolume` configs
5. Container runs in GKE without code changes

**What it does NOT do:** Create a cloud-native microservice. It containerizes the monolith as-is — first step in modernization, not the final step.

---

## 9. Cost Optimization

---

### Q47. What are Committed Use Discounts (CUDs) and Sustained Use Discounts (SUDs)?

**Answer:**

| | Sustained Use Discount (SUD) | Committed Use Discount (CUD) |
|---|---|---|
| How to get | Automatic — no action needed | Commit to 1 or 3 year usage upfront |
| Discount | Up to 30% for running all month | Up to 57% (1 yr) or 70% (3 yr) |
| Applies to | GCE VMs, GKE nodes (Standard) | GCE VMs, Cloud SQL, GKE Standard |
| Risk | None | You pay even if you don't use it |

**Strategy:** Use CUDs for your **baseline** predictable load (always-on VMs). Use on-demand for burst. Use Spot for batch.

---

### Q48. What are the key cost optimization strategies for BigQuery?

**Answer:**

| Strategy | Savings |
|---|---|
| **Partition tables** by date | Queries scan only relevant partitions — reduce TB scanned |
| **Cluster tables** by common filter columns | Further reduces scan within partitions |
| **Use SELECT col1, col2** not SELECT * | BigQuery is columnar — each column is an extra scan |
| **Materialized views** for repeated heavy queries | Pre-computed; incremental updates only |
| **On-demand vs flat-rate slots** | Flat-rate cheaper if >~$3K/month of on-demand queries |
| **Set project-level spend limits** | Prevent runaway query costs |
| **Use BI Engine** | In-memory acceleration for Looker Studio — reduce query count |

---

### Q49. How do you right-size GCE VMs? What GCP tools help?

**Answer:**

| Tool | What it does |
|---|---|
| **VM Recommender** | Analyzes CPU/memory usage over 8 days; recommends smaller machine type |
| **Active Assist** | Aggregates all recommendations (VM resize, idle resources, CUD recommendations) |
| **Cloud Monitoring** | Custom dashboards to track actual CPU/memory utilization |
| **Committed Use Discount Recommender** | Suggests CUDs based on your usage patterns |

**Process:**
1. Enable VM Recommender (automatic — just check Recommendations panel in console)
2. For any VM at <40% avg CPU → candidate for downsizing
3. Test on staging, then resize production during low-traffic window
4. Re-evaluate every quarter

---

### Q50. What is the GCP pricing calculator and how should an architect use it?

**Answer:** The GCP Pricing Calculator (`cloud.google.com/products/calculator`) estimates monthly costs before deploying.

**Architect usage:**
1. **Architecture proposal stage** — estimate cost of proposed design; compare alternatives (e.g., Cloud SQL vs Spanner)
2. **Committed Use planning** — model baseline vs burst to decide CUD commitment level
3. **Client presentations** — show TCO (Total Cost of Ownership) vs on-prem
4. **Budget alerting** — cross-check calculator estimate with actual Cloud Billing budget alerts

**Senior tip:** Always add 20–30% buffer to calculator estimates. Real-world costs include network egress, operations/management overhead, and unexpected usage spikes that the calculator doesn't capture from load tests alone.

---

## Quick Reference — GCP Architect Cheat Sheet

| Scenario | GCP Service |
|---|---|
| Run containers without managing K8s | Cloud Run |
| Managed Kubernetes | GKE Autopilot |
| Global relational DB with strong consistency | Cloud Spanner |
| Managed MySQL/PostgreSQL | Cloud SQL |
| Real-time messaging / event bus | Pub/Sub |
| ETL pipelines (batch + streaming) | Dataflow |
| Big data analytics / SQL on petabytes | BigQuery |
| WAF + DDoS protection | Cloud Armor |
| Secrets management | Secret Manager |
| Encryption key management | Cloud KMS |
| Object storage / data lake | Cloud Storage (GCS) |
| Globally distributed cache | Memorystore (Redis) |
| On-prem to GCP private connectivity | Cloud Interconnect |
| Encrypt VPN tunnel over internet | Cloud VPN |
| Centralize network for multiple projects | Shared VPC |
| Prevent data exfiltration at API level | VPC Service Controls |
| Auto-scale VMs | Managed Instance Group (MIG) |
| Migrate VMs to cloud | Migrate for Compute Engine |
| Hybrid / multi-cloud Kubernetes | Anthos |
