# GCP Interview Questions & Answers

---

## Section 1: Core GCP Concepts

**Q1. What is Google Cloud Platform (GCP)?**
> GCP is Google's suite of cloud computing services that runs on the same infrastructure Google uses internally. It offers services across compute, storage, networking, big data, machine learning, and identity management.

---

**Q2. What are the core compute options in GCP?**
> - **Compute Engine** – IaaS, virtual machines (VMs)
> - **Google Kubernetes Engine (GKE)** – Managed Kubernetes
> - **Cloud Run** – Serverless containers
> - **App Engine** – Managed PaaS for applications
> - **Cloud Functions** – Event-driven serverless functions

---

**Q3. What is a GCP Project?**
> A Project is the fundamental organizational unit in GCP. All resources (VMs, buckets, databases) belong to a project. It has a unique project ID, project number, and name. Billing, IAM, and APIs are managed at the project level.

---

**Q4. What is the GCP Resource Hierarchy?**
> ```
> Organization
>     └── Folders
>             └── Projects
>                     └── Resources (VMs, buckets, etc.)
> ```
> IAM policies applied at higher levels are inherited by lower levels.

---

**Q5. What is a Zone and a Region in GCP?**
> - **Zone** – A single deployment area within a region (e.g., `us-east1-b`). Maps to a specific data center.
> - **Region** – A geographic location containing multiple zones (e.g., `us-east1`). Use multi-zone deployments for high availability.

---

## Section 2: IAM & Security

**Q6. What is Cloud IAM?**
> Identity and Access Management (IAM) lets you control who (identity) has what access (role) to which resource. It follows the principle of least privilege.

---

**Q7. What are the types of IAM roles in GCP?**
> - **Primitive roles** – Owner, Editor, Viewer (broad, legacy, avoid in prod)
> - **Predefined roles** – Fine-grained roles defined by Google (e.g., `roles/storage.objectViewer`)
> - **Custom roles** – User-defined roles with specific permissions

---

**Q8. What is a Service Account in GCP?**
> A Service Account is a special identity used by applications and VMs (not humans) to authenticate to GCP APIs. It has an email address and can be granted IAM roles. Keys can be generated for external use, but Workload Identity is preferred in GKE.

---

**Q9. What is Workload Identity in GKE?**
> Workload Identity binds a Kubernetes Service Account (KSA) to a GCP Service Account (GSA). Pods using the KSA automatically inherit the GSA's permissions without needing to manage key files inside containers — the recommended approach for GKE.

---

**Q10. What is VPC in GCP and how does it differ from AWS?**
> GCP VPC is global — a single VPC spans all regions. Subnets are regional. In AWS, a VPC is regional and subnets are per-AZ. GCP also supports Shared VPC for cross-project networking and VPC peering for connecting VPCs.

---

**Q11. What is a Firewall Rule in GCP?**
> GCP firewall rules control ingress/egress traffic to VM instances. Rules are applied at the network level but enforced at the VM. They are stateful and defined by direction, IP ranges, protocols, ports, and target tags or service accounts.

---

**Q12. What is Cloud Armor?**
> Cloud Armor is GCP's DDoS protection and WAF (Web Application Firewall) service. It works with HTTP(S) Load Balancers to protect applications using security policies, IP allowlists/denylists, and pre-configured rules for OWASP top 10.

---

## Section 3: Compute Engine

**Q13. What are the types of VM instances in Compute Engine?**
> - **General purpose** – E2, N2, N2D (balanced)
> - **Compute optimized** – C2, C3 (high CPU)
> - **Memory optimized** – M2, M3 (large memory workloads)
> - **Accelerator optimized** – A2 (GPU/TPU workloads)
> - **Spot VMs** – Preemptible, low-cost, can be interrupted

---

**Q14. What is a Managed Instance Group (MIG)?**
> A MIG is a group of identical VM instances managed as a single entity. It supports:
> - **Auto-scaling** based on CPU, load balancing, or custom metrics
> - **Auto-healing** using health checks to replace unhealthy VMs
> - **Rolling updates** for zero-downtime deployments

---

**Q15. What is the difference between Persistent Disk and Local SSD?**
> | Feature | Persistent Disk | Local SSD |
> |---|---|---|
> | Durability | Survives VM deletion | Lost when VM stops |
> | Performance | Good (up to 120k IOPS) | Very high (up to 2.4M IOPS) |
> | Use case | General storage | Temp caches, scratch data |
> | Scope | Network-attached | Physically attached |

---

## Section 4: Google Kubernetes Engine (GKE)

**Q16. What is GKE and what are its modes?**
> GKE is Google's managed Kubernetes service. Two modes:
> - **Standard mode** – You manage nodes; GKE manages the control plane
> - **Autopilot mode** – Google manages both nodes and control plane; you pay per pod, not per node

---

**Q17. What is a Node Pool in GKE?**
> A Node Pool is a group of nodes within a cluster that share the same configuration (machine type, disk, labels). A cluster can have multiple node pools with different configs — e.g., a CPU pool and a GPU pool.

---

**Q18. How does GKE handle cluster upgrades?**
> GKE supports:
> - **Auto-upgrade** – Control plane and nodes upgraded automatically within a maintenance window
> - **Surge upgrades** – New nodes created before old ones are drained (configurable `maxSurge` and `maxUnavailable`)
> - **Blue/green node pools** – Manually create a new pool and migrate workloads

---

**Q19. What is GKE Ingress and how does it work?**
> GKE Ingress creates a Google Cloud HTTP(S) Load Balancer. When you create an Ingress resource, GKE provisions:
> - A global external IP
> - URL map for path/host-based routing
> - Backend services pointing to NodePort services
> - Health checks auto-configured from readiness probes

---

**Q20. What is Horizontal Pod Autoscaler (HPA) vs Vertical Pod Autoscaler (VPA)?**
> - **HPA** – Scales the number of pod replicas based on CPU/memory utilization or custom metrics
> - **VPA** – Adjusts CPU/memory requests and limits of individual pods based on actual usage
> - **Node Auto Provisioner (NAP)** – Automatically adds/removes node pools based on pending pod requirements (GKE Autopilot does this natively)

---

**Q21. What is Workload Identity Federation and why is it preferred over Service Account keys?**
> Workload Identity Federation eliminates the need to manage long-lived service account JSON key files. In GKE, pods assume a GCP IAM identity via a Kubernetes service account binding — no key rotation needed, no risk of key leakage, auditable via Cloud Audit Logs.

---

## Section 5: Storage & Databases

**Q22. What are the storage options in GCP?**
> | Service | Type | Use Case |
> |---|---|---|
> | Cloud Storage | Object store | Blobs, backups, static assets |
> | Persistent Disk | Block | VM attached disks |
> | Filestore | NFS file system | Shared file storage |
> | Cloud SQL | Relational (MySQL/PG/SQL Server) | Transactional workloads |
> | Cloud Spanner | Globally distributed relational | Planet-scale ACID transactions |
> | Firestore | NoSQL document | Mobile/web apps |
> | Bigtable | NoSQL wide-column | Time-series, IoT, analytics |
> | BigQuery | Data warehouse | Analytics at petabyte scale |

---

**Q23. What are the storage classes in Cloud Storage?**
> - **Standard** – Frequent access (no minimum storage duration)
> - **Nearline** – Access less than once a month (30-day minimum)
> - **Coldline** – Access less than once a quarter (90-day minimum)
> - **Archive** – Access less than once a year (365-day minimum, lowest cost)

---

**Q24. What is Cloud Spanner and when would you use it over Cloud SQL?**
> Cloud Spanner is a fully managed, globally distributed relational database with strong consistency and horizontal scaling. Use Spanner when:
> - You need > 10TB or multi-region write availability
> - Strong consistency across regions is required
> - Scaling beyond a single Cloud SQL instance
>
> Cloud SQL is sufficient for standard single-region transactional workloads.

---

**Q25. What is BigQuery and what makes it fast?**
> BigQuery is a serverless, highly scalable data warehouse. It's fast because:
> - **Columnar storage** – Reads only required columns
> - **Dremel execution engine** – Massively parallel query execution across thousands of nodes
> - **Automatic partitioning & clustering** – Reduces bytes scanned
> - **Separation of compute and storage** – Independent scaling

---

## Section 6: Networking

**Q26. What is Cloud Load Balancing in GCP?**
> GCP offers multiple load balancer types:
> - **Global HTTP(S) LB** – Layer 7, anycast IP, SSL termination, URL routing
> - **Global SSL Proxy / TCP Proxy** – Layer 4, non-HTTP
> - **Regional External TCP/UDP LB** – Pass-through, preserves client IP
> - **Internal HTTP(S) LB** – Layer 7, for internal services
> - **Internal TCP/UDP LB** – Layer 4 for internal traffic

---

**Q27. What is Cloud CDN?**
> Cloud CDN uses Google's global edge network to cache HTTP(S) Load Balancer responses close to users, reducing latency and origin load. It integrates with Cloud Storage backends and supports cache invalidation, signed URLs, and cache keys.

---

**Q28. What is Cloud Interconnect vs Cloud VPN?**
> - **Cloud VPN** – Encrypted IPsec tunnel over public internet; up to 3 Gbps per tunnel; lower cost
> - **Dedicated Interconnect** – Direct physical connection to Google's network; 10/100 Gbps; low latency, no public internet
> - **Partner Interconnect** – Via a service provider; flexible bandwidth; for locations without direct Interconnect

---

**Q29. What is Shared VPC?**
> Shared VPC allows multiple GCP projects to share a common VPC network hosted in a central **host project**. Service projects attach to it. This centralizes network administration (firewall rules, subnets) while allowing teams to manage their own resources independently.

---

**Q30. What is Private Google Access?**
> Private Google Access allows VM instances without external IP addresses to reach Google APIs and services (like Cloud Storage, BigQuery) using internal IPs, without going through the public internet. Enabled at the subnet level.

---

## Section 7: CI/CD & DevOps

**Q31. What is Cloud Build?**
> Cloud Build is GCP's fully managed CI/CD platform. It executes builds as a series of steps defined in a `cloudbuild.yaml`. Each step runs in a Docker container. It integrates with Cloud Source Repositories, GitHub, and Artifact Registry.

---

**Q32. What is Artifact Registry?**
> Artifact Registry is GCP's managed repository for storing build artifacts — Docker images, Maven/Gradle packages, npm packages, Python packages. It replaces Container Registry and supports fine-grained IAM, vulnerability scanning via Container Analysis.

---

**Q33. What is Cloud Deploy?**
> Cloud Deploy is a managed continuous delivery service for GKE, Cloud Run, and Anthos. It defines a delivery pipeline with ordered targets (dev → staging → prod). Supports approval gates, rollbacks, and deployment metrics.

---

**Q34. What is Terraform and how does it work with GCP?**
> Terraform is an IaC tool by HashiCorp. The `google` provider allows you to define GCP resources (VMs, GKE clusters, VPCs) in HCL. Terraform maintains state in a backend (e.g., GCS bucket), plans changes with `terraform plan`, and applies with `terraform apply`. GCP recommends using Terraform with remote state in Cloud Storage.

---

## Section 8: Monitoring & Operations

**Q35. What is Cloud Monitoring (formerly Stackdriver)?**
> Cloud Monitoring collects metrics, events, and metadata from GCP resources and applications. It supports:
> - Dashboards and custom charts
> - Alerting policies with notification channels (email, PagerDuty, Slack)
> - Uptime checks
> - Integration with Prometheus via Managed Service for Prometheus

---

**Q36. What is Cloud Logging?**
> Cloud Logging is a fully managed log management service. It ingests logs from GCP services, VMs, GKE, and custom applications. Supports:
> - Log-based metrics for alerting
> - Log sinks to export to BigQuery, Cloud Storage, or Pub/Sub
> - Log Explorer for real-time querying using the Logging query language

---

**Q37. What is Cloud Trace?**
> Cloud Trace is a distributed tracing system that collects latency data from applications. It auto-instruments App Engine and integrates with OpenTelemetry for GKE/Cloud Run. Helps identify bottlenecks in microservice call chains.

---

**Q38. What is Error Reporting in GCP?**
> Error Reporting automatically aggregates and displays errors from Cloud Logging. It groups similar errors, shows occurrence frequency and first/last seen timestamps, and can notify developers via email or PagerDuty when new errors are detected.

---

## Section 9: Messaging & Serverless

**Q39. What is Pub/Sub and how does it work?**
> Cloud Pub/Sub is a fully managed asynchronous messaging service. Publishers send messages to a **topic**. Subscribers receive messages via **subscriptions** (pull or push). Messages are durably stored until acknowledged. It guarantees at-least-once delivery and supports message ordering and filtering.

---

**Q40. What is Cloud Run?**
> Cloud Run is a fully managed serverless platform that runs stateless containers. It automatically scales from zero to thousands of instances based on HTTP traffic. You pay only for the CPU and memory used during request processing. Supports any language/runtime packaged as a container.

---

**Q41. What is the difference between Cloud Functions and Cloud Run?**
> | | Cloud Functions | Cloud Run |
> |---|---|---|
> | Package | Source code | Container image |
> | Trigger | Events (Pub/Sub, HTTP, GCS) | HTTP requests, events |
> | Runtime | Fixed runtimes (Node, Python, Go, Java) | Any runtime in a container |
> | Timeout | Up to 60 min (2nd gen) | Up to 60 min |
> | Use case | Simple event handlers | Complex apps, custom runtimes |

---

## Section 10: Architecture & Advanced

**Q42. What is a multi-region vs dual-region bucket in Cloud Storage?**
> - **Multi-region** – Data replicated across multiple regions in a continent (e.g., `US`, `EU`). Higher availability, geo-redundant.
> - **Dual-region** – Data replicated between exactly two specific regions you choose. Offers turbo replication (RPO < 15 min) and predictable latency.
> - **Region** – Single region. Lowest cost, lowest latency within region.

---

**Q43. How do you design a highly available architecture in GCP?**
> - Deploy VMs across multiple zones using MIGs with auto-healing
> - Use regional persistent disks (replicated across zones)
> - Use Global HTTP(S) Load Balancer for anycast traffic routing
> - Use Cloud SQL with HA (primary + standby in different zones)
> - Use multi-region Cloud Storage for static assets
> - Use GKE regional clusters (control plane and nodes spread across zones)

---

**Q44. What is Anthos?**
> Anthos is Google's hybrid and multi-cloud platform. It extends GKE to on-premises (via GKE on-prem / Bare Metal) and other clouds (AWS, Azure). It provides a consistent control plane for policy, configuration, and service mesh (Anthos Service Mesh / Istio) across environments.

---

**Q45. What is Cloud Endpoints / Apigee?**
> - **Cloud Endpoints** – Lightweight API management for Cloud Run, GKE, and Compute Engine. Handles auth, monitoring, and logging via an Extensible Service Proxy (ESP).
> - **Apigee** – Full enterprise API management platform with developer portals, monetization, advanced analytics, and full lifecycle API management. Used for external API programs.

---

**Q46. What is the difference between horizontal and vertical scaling in GCP?**
> - **Vertical scaling** – Increase machine type (more CPU/RAM) of a single VM. Has limits. Requires restart.
> - **Horizontal scaling** – Add more instances (MIG auto-scaling, GKE HPA). No downtime, effectively unlimited.
>
> GCP best practice: design stateless services that scale horizontally behind a load balancer.

---

**Q47. What is VPC Service Controls?**
> VPC Service Controls creates a security perimeter around GCP services (like BigQuery, Cloud Storage) to prevent data exfiltration. Even if a user has IAM access, requests from outside the perimeter are denied. Used in regulated industries (finance, healthcare).

---

**Q48. How does GCP handle disaster recovery?**
> GCP DR strategies by RTO/RPO:
> - **Cold standby** – Resources provisioned only on disaster (high RTO, low cost)
> - **Warm standby** – Scaled-down replica running in another region
> - **Hot standby** – Full replica running (low RTO, high cost) using multi-region Spanner, GKE regional clusters, global LB
> - Cloud Storage multi-region + Spanner + global LB = near-zero RPO/RTO

---

**Q49. What is the difference between Cloud NAT and a NAT instance?**
> - **Cloud NAT** – Fully managed, software-defined NAT. No single point of failure, auto-scales, no VMs to manage. Allows private VMs to access the internet for outbound traffic without exposing them.
> - **NAT instance** – A VM configured as NAT gateway. Must be managed, patched, scaled manually. Prone to single point of failure.

---

**Q50. A microservice on GKE is experiencing high latency. How would you diagnose it?**
> Systematic approach:
> 1. **Cloud Monitoring** – Check CPU, memory, network metrics on pods and nodes
> 2. **Cloud Trace** – Identify which service in the call chain is slow
> 3. **Cloud Logging** – Check application logs for errors or slow queries
> 4. **kubectl top pods/nodes** – Check resource exhaustion
> 5. **HPA status** – Check if pods are being throttled due to CPU limits
> 6. **Check GKE network policy** – Ensure no unnecessary hops
> 7. **Cloud SQL Insights / Bigtable monitoring** – Check if DB is the bottleneck
> 8. **Error Reporting** – Look for retry storms causing latency spikes

---

## Quick Reference Cheat Sheet

| Service | Category | Purpose |
|---|---|---|
| Compute Engine | Compute | IaaS VMs |
| GKE | Compute | Managed Kubernetes |
| Cloud Run | Compute | Serverless containers |
| Cloud Functions | Compute | Event-driven functions |
| Cloud Storage | Storage | Object storage |
| Cloud SQL | Database | Managed relational DB |
| Cloud Spanner | Database | Global distributed SQL |
| BigQuery | Analytics | Serverless data warehouse |
| Pub/Sub | Messaging | Async message queue |
| Cloud Build | DevOps | CI/CD pipelines |
| Terraform | IaC | Infrastructure provisioning |
| Cloud IAM | Security | Access management |
| Cloud Armor | Security | DDoS / WAF |
| VPC | Networking | Virtual private network |
| Cloud Load Balancing | Networking | Traffic distribution |
| Cloud Monitoring | Operations | Metrics & alerting |
| Cloud Logging | Operations | Log management |
| Cloud Trace | Operations | Distributed tracing |
| Anthos | Hybrid | Multi-cloud Kubernetes |

---

*Good luck with your GCP interview!*
