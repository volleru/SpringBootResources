# GCP Network Engineer Interview Prep — TCS

> Focused on hands-on networking: VPC, firewall, routing, load balancing, hybrid connectivity, DNS, NAT, and troubleshooting.

---

## Category Index

1. [VPC & Subnets](#1-vpc--subnets) — Q1–Q7
2. [Firewall Rules & Policies](#2-firewall-rules--policies) — Q8–Q12
3. [Routing & Cloud Router](#3-routing--cloud-router) — Q13–Q17
4. [Load Balancing (Deep Dive)](#4-load-balancing-deep-dive) — Q18–Q24
5. [Cloud DNS](#5-cloud-dns) — Q25–Q28
6. [Cloud NAT](#6-cloud-nat) — Q29–Q31
7. [Hybrid Connectivity — VPN & Interconnect](#7-hybrid-connectivity--vpn--interconnect) — Q32–Q38
8. [Private Connectivity](#8-private-connectivity) — Q39–Q43
9. [Network Security](#9-network-security) — Q44–Q47
10. [Monitoring & Troubleshooting](#10-monitoring--troubleshooting) — Q48–Q52

---

## 1. VPC & Subnets

---

### Q1. What is a GCP VPC and how is it different from traditional networking and AWS VPC?

**Answer:** A GCP VPC (Virtual Private Cloud) is a **global, software-defined network**. Unlike traditional networks or AWS VPCs, it is not tied to a region.

| Feature | GCP VPC | AWS VPC | Traditional Network |
|---|---|---|---|
| Scope | **Global** — one VPC spans all regions | Regional — separate VPC per region | Physical — per data center |
| Subnets | Regional — span all zones in a region | Tied to one Availability Zone | Per VLAN |
| Inter-zone routing | Automatic within VPC (no peering needed) | Automatic within VPC | Needs routing config |
| Inter-region routing | Automatic within VPC (via Google backbone) | Needs VPC Peering or Transit Gateway | Needs WAN |
| Default route | `0.0.0.0/0` via default internet gateway | `0.0.0.0/0` via IGW | Default gateway |

**Key implication for network engineers:** A VM in `us-central1` can talk to a VM in `europe-west1` in the **same VPC** without any extra peering or routing — traffic flows through Google's private backbone automatically.

---

### Q2. What is the difference between Auto Mode and Custom Mode VPC? Which do you use in production?

**Answer:**

| | Auto Mode VPC | Custom Mode VPC |
|---|---|---|
| Subnet creation | GCP automatically creates one subnet per region using `10.128.0.0/9` | You create subnets manually with your own CIDR |
| CIDR control | None — GCP decides | Full control |
| New region support | New subnets added automatically when GCP adds regions | You must create manually |
| Conversion | Can convert auto → custom (one-way, irreversible) | Cannot convert to auto |
| Production use | **Never** — CIDR overlaps with common on-prem ranges | **Always** |

**Why auto mode is dangerous in production:**
- `10.128.0.0/9` overlaps with many corporate on-prem ranges
- When you set up Cloud VPN / Interconnect, the overlapping routes cause connectivity failures
- No ability to plan IP addressing for future subnets

```bash
# Convert auto mode to custom (one-way, do this immediately after project creation)
gcloud compute networks update my-vpc --switch-to-custom-subnet-mode
```

---

### Q3. What is a subnet's primary and secondary IP range? When do you need secondary ranges?

**Answer:**

- **Primary range:** Used for VM internal IP addresses
- **Secondary range:** Used for **alias IP ranges** — most commonly for GKE pods and services

```
Subnet: 10.0.0.0/24  (primary — VMs get IPs from here)
  └── Secondary range: 10.1.0.0/16  (pods — each GKE node gets a /24 block from here)
  └── Secondary range: 10.2.0.0/20  (services — Cluster IP of K8s services)
```

**Why GKE needs secondary ranges:**
- Each GKE node gets a **slice of the secondary pod range** (e.g., `/24` = 256 pod IPs per node)
- Without secondary ranges, pods would use primary range IPs — conflicts with VM addressing

```bash
# Create subnet with secondary ranges for GKE
gcloud compute networks subnets create gke-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.0.0/24 \
  --secondary-range=pods=10.1.0.0/16,services=10.2.0.0/20
```

---

### Q4. What is an Alias IP? How is it different from a secondary IP on a traditional VM?

**Answer:** An Alias IP range lets a single VM NIC hold a **block of IP addresses** beyond its primary IP. Every IP in the range is routable to that VM.

```
VM primary NIC: 10.0.0.5 (primary IP)
  ├── Alias range: 10.1.0.0/24  → all 256 IPs routed to this VM
  └── Additional alias: 10.1.1.0/24 → another block on same NIC
```

**Use cases:**
- GKE nodes — each pod on the node gets one IP from the alias range
- Running multiple containers on one VM with isolated IPs
- Software that expects to bind to multiple IPs

**Key difference from traditional secondary IP:** Traditional secondary IPs are individual IPs. GCP alias IPs are **ranges** — up to 256 IPs per alias entry.

---

### Q5. What is VPC Flow Logs? What information do they capture and what are their limitations?

**Answer:** VPC Flow Logs records a **sample** of network flows sent and received by VM NICs (and GKE nodes).

**What's captured per record:**
| Field | Example |
|---|---|
| Source/dest IP and port | `10.0.0.5:443 → 10.0.1.3:54321` |
| Protocol | TCP, UDP, ICMP |
| Packets / bytes | `42 packets, 18420 bytes` |
| RTT (for TCP) | `12ms` |
| Direction | Ingress / Egress |
| VM metadata | Instance name, zone, VPC, subnet |
| Geographic info | Country of external IPs |

**Limitations:**
- **Sampling** — by default samples ~1 in 8 packets (12.5%). Can increase to 100% but cost rises.
- No **packet payload** — only headers/metadata (not a packet capture tool)
- Logs go to Cloud Logging — high volume at 100% sampling gets expensive
- **Not real-time** — ~5-15 second aggregation window per log entry

```bash
# Enable Flow Logs on a subnet
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-flow-logs \
  --logging-aggregation-interval=interval-5-sec \
  --logging-flow-sampling=0.5   # 50% sampling
```

---

### Q6. What is the maximum MTU in GCP VPC and why does it matter?

**Answer:**

| Network type | MTU |
|---|---|
| GCP VPC default | **1460 bytes** |
| GCP VPC (jumbo frames enabled) | **8896 bytes** |
| Standard Ethernet | 1500 bytes |
| Cloud VPN tunnel | ~1350 bytes (encapsulation overhead) |

**Why it matters:**
- GCP's default MTU is **1460** (not 1500) because GCP's internal encapsulation uses 40 bytes overhead
- Mismatched MTU between on-prem (1500) and GCP VPN (1350) causes **packet fragmentation** or silent drops for large packets
- TCP usually self-corrects via Path MTU Discovery (PMTUD), but UDP (DNS, QUIC, VoIP) does not

**Fix for VPN MTU issues:**
```bash
# Clamp TCP MSS on Cloud VPN gateway to avoid fragmentation
# Set MSS = MTU - 40 bytes (TCP/IP headers)
# On Linux VMs: add iptables rule
iptables -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

---

### Q7. What is a Shared VPC? Walk through the setup and explain when to use it.

**Answer:** Shared VPC lets one **host project** own the VPC network while **service projects** use its subnets to deploy resources. The network team controls one VPC; app teams deploy into it.

```
Organization
  ├── Host Project  (owns VPC, subnets, firewall rules)
  │     └── VPC: corp-network
  │           ├── subnet: prod-us  10.0.0.0/24
  │           └── subnet: prod-eu  10.1.0.0/24
  ├── Service Project A (app team 1 — uses prod-us subnet)
  └── Service Project B (app team 2 — uses prod-eu subnet)
```

**Setup steps:**
```bash
# 1. Enable Shared VPC on host project
gcloud compute shared-vpc enable HOST_PROJECT_ID

# 2. Attach service project to host project
gcloud compute shared-vpc associated-projects add SERVICE_PROJECT_A \
  --host-project=HOST_PROJECT_ID

# 3. Grant service project's service account permission to use specific subnet
gcloud compute networks subnets add-iam-policy-binding prod-us \
  --region=us-central1 \
  --member="serviceAccount:SERVICE_ACCOUNT@SERVICE_PROJECT_A.iam.gserviceaccount.com" \
  --role="roles/compute.networkUser"
```

**Use Shared VPC when:**
- Central network team needs control over IP addressing and firewall rules
- Multiple app teams share the same network but need project-level billing separation
- Regulatory requirement for centralized network oversight

**Vs VPC Peering:** Shared VPC is simpler for same-org multi-project. VPC Peering connects two independently managed VPCs (different orgs or when you need VPC autonomy).

---

## 2. Firewall Rules & Policies

---

### Q8. How do GCP firewall rules work? What are the key parameters?

**Answer:** GCP firewall rules are **stateful** — if you allow ingress traffic, the return egress traffic is automatically allowed (no need to add a separate egress rule).

**Key parameters:**

| Parameter | Options | Description |
|---|---|---|
| **Direction** | INGRESS / EGRESS | Traffic entering or leaving a VM |
| **Action** | ALLOW / DENY | What to do with matching traffic |
| **Priority** | 0 – 65535 (lower = higher priority) | Which rule wins when multiple match |
| **Target** | All instances / Tags / Service Account | Which VMs this rule applies to |
| **Source/Dest** | IP ranges / Tags / Service Accounts | Who the traffic comes from (ingress) or goes to (egress) |
| **Protocol/Port** | tcp:80, udp:53, icmp, all | What traffic to match |

**Default rules (cannot be deleted):**

| Rule | Priority | Action | Traffic |
|---|---|---|---|
| `default-allow-internal` | 65534 | Allow | Ingress from `10.128.0.0/9` (auto-mode ranges) |
| `default-allow-ssh` | 65534 | Allow | Ingress TCP:22 from `0.0.0.0/0` |
| `default-allow-rdp` | 65534 | Allow | Ingress TCP:3389 from `0.0.0.0/0` |
| `default-allow-icmp` | 65534 | Allow | Ingress ICMP from `0.0.0.0/0` |
| Implied deny all ingress | 65535 | Deny | All ingress |
| Implied allow all egress | 65535 | Allow | All egress |

**Senior tip:** In production, always **delete** or **override** `default-allow-ssh` with a higher priority rule that restricts source to your corporate IP range or IAP proxy CIDR (`35.235.240.0/20`).

---

### Q9. What is the difference between Network Tags and Service Account-based firewall rules?

**Answer:**

| | Network Tags | Service Account |
|---|---|---|
| What it is | A string label applied to a VM | A GCP identity attached to a VM |
| Who can assign | Anyone with `compute.instances.setTags` | Only users with `iam.serviceAccounts.actAs` |
| Security | Lower — any editor can add/remove tags | Higher — controlled by IAM |
| Use case | Dev/test, general grouping | Production, when strict access control needed |

```bash
# Tag-based rule: allow HTTP to VMs tagged "web-server"
gcloud compute firewall-rules create allow-http \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:80 \
  --target-tags=web-server \
  --source-ranges=0.0.0.0/0

# Service account-based rule: allow DB traffic only from app SA to DB SA
gcloud compute firewall-rules create allow-app-to-db \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:5432 \
  --target-service-accounts=db-sa@project.iam.gserviceaccount.com \
  --source-service-accounts=app-sa@project.iam.gserviceaccount.com
```

**Senior tip:** Prefer service account-based rules in production — tags are easy to accidentally add to the wrong VM.

---

### Q10. What are Hierarchical Firewall Policies? How are they different from VPC firewall rules?

**Answer:**

| | VPC Firewall Rules | Hierarchical Firewall Policies |
|---|---|---|
| Scope | Single VPC | Applied at Org or Folder level — affect all VPCs below |
| Priority evaluation | Within VPC rules only | Evaluated before VPC rules |
| `goto_next` action | Not available | Allows lower-level rules to take effect |
| Use case | Per-VPC application rules | Org-wide security baseline (e.g., block SSH from internet everywhere) |

**Evaluation order:**
```
1. Org-level Hierarchical Policy
2. Folder-level Hierarchical Policy
3. VPC Firewall Rules (network-level)
4. Implied deny
```

**Practical use:**
```
Org Policy: DENY ingress TCP:22 from 0.0.0.0/0  (blocks all SSH from internet globally)
Folder Policy: ALLOW ingress TCP:22 from 35.235.240.0/20 (allow IAP SSH for dev folder)
VPC Rules: Application-specific allow rules
```

---

### Q11. What is Identity-Aware Proxy (IAP) TCP Tunneling? How does it replace a bastion host?

**Answer:** IAP TCP Tunneling allows you to SSH/RDP into VMs **without external IPs** or a bastion host. IAP acts as an authenticated proxy.

**Traditional approach (bastion host):**
```
Internet → Bastion Host (has external IP) → SSH → Private VMs
Problem: Bastion is another surface to secure, patch, and manage
```

**With IAP TCP Tunneling:**
```
Developer's laptop → IAP (authenticated by Google Identity + IAM) → SSH → Private VMs
No external IP needed on any VM. No bastion to manage.
```

```bash
# Allow SSH only from IAP's IP range (not public internet)
gcloud compute firewall-rules create allow-iap-ssh \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --target-tags=private-vm

# SSH through IAP tunnel
gcloud compute ssh my-vm --tunnel-through-iap --zone=us-central1-a
```

**Required IAM role:** `roles/iap.tunnelResourceAccessor` on the VM or project.

---

### Q12. What happens when two firewall rules with the same priority match the same traffic?

**Answer:** When two rules have the **same priority**:
- **DENY takes precedence over ALLOW** at the same priority level
- GCP documentation specifies: at equal priority, DENY wins

**Best practice — never rely on this.** Design firewall rules so priorities are explicit:

```
Priority 100: DENY all from known bad IPs (blocklist)
Priority 500: ALLOW tcp:443 from corporate IP range
Priority 1000: ALLOW tcp:80 from load balancer health check ranges
Priority 65535: Implied DENY all (default)
```

Leave gaps between priority values (use 100, 200, 500 not 1, 2, 3) so you can insert rules between existing ones without renumbering.

---

## 3. Routing & Cloud Router

---

### Q13. How does routing work in a GCP VPC? What types of routes exist?

**Answer:** GCP uses a **distributed software-defined routing** system — no physical routers. Every VM has a routing table populated by GCP.

**Route types:**

| Type | Created by | Example |
|---|---|---|
| **Subnet routes** | Auto-created when subnet is created | `10.0.0.0/24 → next hop: subnet` |
| **Default route** | Auto-created (can be deleted) | `0.0.0.0/0 → next hop: default internet gateway` |
| **Static routes** | You create manually | `192.168.1.0/24 → next hop: VPN tunnel` |
| **Dynamic routes** | Cloud Router learns via BGP | `172.16.0.0/12 → next hop: Cloud Router (VPN/Interconnect)` |
| **Peering routes** | From VPC peering | Routes from the peered VPC |

**Dynamic routing modes:**

| Mode | BGP routes distributed to |
|---|---|
| **Regional** | Only VMs in the same region as the Cloud Router |
| **Global** | All VMs in all regions of the VPC |

```bash
# Set dynamic routing mode to global (recommended for multi-region VPCs with VPN)
gcloud compute networks update my-vpc --bgp-routing-mode=global
```

---

### Q14. What is Cloud Router? How does it work with BGP?

**Answer:** Cloud Router is a **fully managed, software-defined BGP router** that enables dynamic route exchange between your GCP VPC and on-prem networks via Cloud VPN or Cloud Interconnect.

**How it works:**
```
On-prem Router (BGP peer) ←── BGP session ──→ Cloud Router
        │                                            │
        │ advertises: 192.168.0.0/16                 │ advertises: 10.0.0.0/8 (GCP subnets)
        │                                            │
   GCP learns on-prem routes              On-prem learns GCP subnet routes
   and programs them in VPC routing table
```

**Key Cloud Router settings:**
| Setting | Description |
|---|---|
| **ASN** | Your Cloud Router's BGP Autonomous System Number (64512–65534 for private) |
| **BGP peer ASN** | Your on-prem router's ASN |
| **Advertised routes** | What GCP subnets to advertise to on-prem (all subnets or custom) |
| **Custom advertised ranges** | Advertise a summary route instead of individual subnets |

**Without Cloud Router (static VPN):** You manually add routes on both sides. Every new subnet requires a manual route update on-prem.

**With Cloud Router (dynamic VPN/Interconnect):** New GCP subnets are automatically advertised to on-prem. Zero-touch route management.

---

### Q15. What is policy-based routing vs route-based VPN in GCP?

**Answer:**

| | Policy-Based VPN | Route-Based VPN |
|---|---|---|
| Traffic selection | Based on local/remote traffic selectors (IP ranges) | Based on routes in routing table; uses `0.0.0.0/0` selectors |
| Cloud Router support | **No** — cannot use dynamic routing | **Yes** — supports Cloud Router + BGP |
| Supported tunnels | 1 tunnel per gateway | Multiple tunnels per gateway |
| HA VPN support | **No** | **Yes** |
| Use case | Legacy VPN, simple static setup | All new VPN deployments |

**Rule:** Always use **route-based (dynamic) VPN** with Cloud Router for new setups. Policy-based is legacy and limits scalability.

---

### Q16. What is a next hop in GCP routing? What are the valid next hop types?

**Answer:** The next hop tells GCP where to forward packets that match a route.

| Next Hop Type | What it means | Example use |
|---|---|---|
| `default-internet-gateway` | Send to internet via GCP's internet gateway | Default `0.0.0.0/0` route |
| `IP address` | Forward to a specific VM's internal IP | VM acting as NVA/firewall |
| `Instance` | Forward to a specific VM (by name) | Same as IP but by instance reference |
| `VPN tunnel` | Forward through a Cloud VPN tunnel | On-prem traffic |
| `Interconnect attachment (VLAN)` | Forward via Dedicated/Partner Interconnect | High-bandwidth on-prem |
| `Internal LB` | Forward to an Internal TCP/UDP LB (for NVA clusters) | HA firewall/NVA appliance |
| `VPC Peering` | Forward to peered VPC | Automatically created by peering |

**NVA (Network Virtual Appliance) pattern:**
Use an Internal LB as next hop for a cluster of third-party firewalls (Palo Alto, Fortinet) — provides HA without changing route tables when an NVA fails.

---

### Q17. What is a Black Hole Route? When would you use one?

**Answer:** A black hole route (also called a null route) drops traffic matching the route silently — the next hop is non-existent or unreachable.

**How to create in GCP:**
```bash
# Route to a non-existent next hop IP = traffic is silently dropped
gcloud compute routes create blackhole-malicious-ip \
  --network=my-vpc \
  --destination-range=203.0.113.0/24 \
  --next-hop-instance=non-existent \
  --priority=100
```

**Use cases:**
- Block known malicious IPs without a firewall rule (routes are evaluated before firewall)
- Null-route traffic to avoid accidental routing to wrong subnets
- Temporarily drop traffic to a decommissioned subnet

**Better alternative:** For security blocking, use Cloud Armor or a high-priority DENY firewall rule — they're more visible, auditable, and easier to manage than black hole routes.

---

## 4. Load Balancing (Deep Dive)

---

### Q18. Explain the full GCP load balancer portfolio. How does a network engineer decide which to use?

**Answer:**

```
External (internet-facing)                    Internal (private traffic)
│                                             │
├── Global External HTTP(S) LB (Layer 7)      ├── Internal HTTP(S) LB (Layer 7)
│   • HTTP/HTTPS/HTTP2/gRPC                   │   • Envoy-based, regional
│   • Single anycast IP                       │   • Microservice-to-microservice
│   • Cloud CDN + Cloud Armor                 │
│                                             ├── Internal TCP/UDP LB (Layer 4)
├── Regional External HTTP(S) LB (Layer 7)    │   • Regional passthrough
│   • HTTP/HTTPS in one region                │   • DB proxies, internal services
│                                             │
├── External TCP/UDP Network LB (Layer 4)     └── Internal LB as Next Hop
│   • Passthrough (preserves client IP)           • Route traffic through NVA/firewall cluster
│   • No SSL termination
│   • UDP support (gaming, VoIP)
│
├── SSL Proxy LB (Layer 4 — SSL only)
│   • Global SSL termination for non-HTTP
│
└── TCP Proxy LB (Layer 4 — TCP only)
    • Global TCP for non-HTTP apps
```

**Decision flow:**
```
Public traffic?
  HTTP/HTTPS → Global External HTTP(S) LB
  TCP/UDP (non-HTTP) → External TCP/UDP Network LB
  SSL non-HTTP → SSL Proxy LB

Private traffic?
  HTTP/HTTPS → Internal HTTP(S) LB
  TCP/UDP → Internal TCP/UDP LB
```

---

### Q19. What is the difference between a passthrough load balancer and a proxy load balancer?

**Answer:**

| | Passthrough (L4) | Proxy (L7) |
|---|---|---|
| How traffic flows | LB forwards packets directly to backend — backend sees **client's real IP** | LB terminates connection, opens new connection to backend — backend sees **LB's IP** |
| SSL termination | No — backend must handle SSL | Yes — LB decrypts, backend gets plain HTTP |
| Session persistence | Based on IP/port (ECMP or 5-tuple hash) | Cookie-based, header-based |
| Protocol support | Any TCP/UDP | HTTP, HTTPS, HTTP/2, gRPC |
| Examples | External TCP/UDP Network LB, Internal TCP/UDP LB | Global HTTP(S) LB, Internal HTTP(S) LB |

**Why passthrough matters for network engineers:**
- Backend receives actual client IP — important for logging, geofencing, IP-based rate limiting
- No SSL overhead on LB — backend handles its own certs
- Lower latency (no proxy overhead)

**Getting client IP behind proxy LB:** Use `X-Forwarded-For` header — the LB inserts the real client IP here.

---

### Q20. What are backend services, backend buckets, health checks, and instance groups — how do they connect?

**Answer:**

```
Load Balancer
  │
  └── URL Map (routes /api/* to backend service, /static/* to backend bucket)
        │
        ├── Backend Service
        │     ├── Health Check (defines how to probe backend health)
        │     ├── Backend: Managed Instance Group (MIG) — zone us-central1-a
        │     ├── Backend: MIG — zone us-central1-b
        │     └── Session affinity, timeout, connection draining settings
        │
        └── Backend Bucket (serves static files directly from GCS)
              └── GCS Bucket (with Cloud CDN if needed)
```

**Health check types:**
| Type | Use when |
|---|---|
| HTTP | Backend serves HTTP — checks for 200 response |
| HTTPS | Backend serves HTTPS |
| TCP | Non-HTTP backend — checks TCP connection established |
| gRPC | gRPC backends — checks gRPC health protocol |
| SSL | SSL (non-HTTP) backends |

**Health check source IPs (must be allowed in firewall):**
- `35.191.0.0/16` and `130.211.0.0/22` for most LBs
- `209.85.152.0/22` and `209.85.204.0/22` for some legacy

```bash
# Firewall rule to allow health checks
gcloud compute firewall-rules create allow-health-checks \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:80 \
  --source-ranges=35.191.0.0/16,130.211.0.0/22 \
  --target-tags=web-server
```

---

### Q21. What is connection draining? Why is it important?

**Answer:** Connection draining (deregistration delay) ensures that when a backend is removed from the LB (scale-in, rolling update, or health failure), **existing in-flight connections are given time to complete** before traffic is cut off.

**Without draining:**
```
VM is removed from LB → existing connections get TCP RST → client sees errors
```

**With draining (e.g., 60 seconds):**
```
VM marked for removal → LB stops sending NEW requests → existing requests get 60s to finish → VM removed cleanly
```

```bash
# Set connection draining timeout on backend service
gcloud compute backend-services update my-backend \
  --connection-draining-timeout=60 \
  --global
```

**Network engineer tip:** Set draining timeout higher than your longest expected request (e.g., if file uploads can take 30s, set draining to 90s). For microservices with sub-second requests, 30s is usually sufficient.

---

### Q22. What is session affinity in GCP Load Balancing? What are the options?

**Answer:** Session affinity (sticky sessions) routes requests from the same client to the **same backend instance** for the duration of a session.

| Affinity Type | Based on | Use case |
|---|---|---|
| **None** (default) | Round-robin / weighted | Stateless backends |
| **Client IP** | Source IP | When backend caches per-client data |
| **Generated cookie** | Cookie inserted by LB | Stateful web apps (shopping carts) |
| **Header field** | Custom HTTP header | Route based on app-defined header |
| **HTTP cookie** | Existing app cookie | Preserve existing app session cookie |

**Important caveat:** Session affinity is **best-effort** — if the backend fails or scales down, the client is routed to a different instance. Design backends to be stateless and use distributed session storage (Memorystore) instead of relying on affinity.

---

### Q23. What is the difference between Global vs Regional External HTTP(S) Load Balancer?

**Answer:**

| | Global External HTTP(S) LB | Regional External HTTP(S) LB |
|---|---|---|
| Anycast IP | **Yes** — single IP routes to nearest region | No — regional IP |
| Backend locations | Multiple regions | One region only |
| Cloud CDN | Yes | No |
| Cloud Armor | Yes | Yes (but limited features) |
| Backends supported | MIG, GKE, Cloud Run, serverless NEG | MIG, GKE, Cloud Run |
| Use case | Multi-region global apps | Single-region, simpler setup |

**When Global LB routes to nearest region:**
- Uses **anycast** — one IP, but Googles edge PoPs direct each user to the nearest healthy backend region
- No DNS TTL tricks needed — routing happens at the network layer

---

### Q24. What is a Network Endpoint Group (NEG)? What types exist?

**Answer:** A NEG is a configuration object that specifies a group of backend endpoints — more flexible than MIGs for modern architectures.

| NEG Type | Endpoints | Use case |
|---|---|---|
| **Zonal NEG** | VM IPs / ports in a zone | Fine-grained load balancing to specific ports |
| **Internet NEG** | External hostname/IP | Route to external backend (on-prem or third-party) |
| **Serverless NEG** | Cloud Run, App Engine, Cloud Functions | LB in front of serverless services |
| **Private Service Connect NEG** | PSC endpoint | Access published services via Private Service Connect |
| **Hybrid NEG** | On-prem endpoints via Interconnect/VPN | Extend LB to on-prem servers |

**Serverless NEG example (Cloud Run behind Global LB):**
```bash
# Create serverless NEG pointing to Cloud Run service
gcloud compute network-endpoint-groups create my-neg \
  --region=us-central1 \
  --network-endpoint-type=serverless \
  --cloud-run-service=my-cloud-run-service
```

---

## 5. Cloud DNS

---

### Q25. What is Cloud DNS? What is the difference between public and private zones?

**Answer:** Cloud DNS is Google's managed, authoritative DNS service with 100% SLA.

| | Public Zone | Private Zone |
|---|---|---|
| Accessible from | Anywhere on the internet | Only from specified VPCs |
| Use case | Resolve your domain publicly | Internal service discovery, resolve GCP resources privately |
| Example | `api.mycompany.com → 34.x.x.x` | `db.internal → 10.0.0.5` (only VMs in VPC can resolve) |

**Private DNS resolution flow:**
```
VM in VPC queries 10.0.0.2 (metadata resolver)
  → Cloud DNS checks private zones for VPC
  → If match: return private record
  → If no match: forward to public DNS (or on-prem DNS via forwarding)
```

```bash
# Create private zone visible only to my-vpc
gcloud dns managed-zones create internal-zone \
  --dns-name=internal.mycompany.com. \
  --visibility=private \
  --networks=my-vpc \
  --description="Internal service discovery"
```

---

### Q26. What is DNS peering and DNS forwarding in Cloud DNS?

**Answer:**

**DNS Forwarding:** Send DNS queries for specific domains to an **external DNS server** (on-prem or third-party).

```
GCP VM queries on-prem.mycompany.com
  → Cloud DNS has no record
  → Forwarding policy: send queries for *.mycompany.com to on-prem DNS 192.168.1.10
  → On-prem DNS responds
```

**DNS Peering:** Let one VPC's private zone answer queries **from another VPC**.

```
VPC-A has private zone: db.internal
VPC-B is peered with VPC-A via DNS peering
  → VMs in VPC-B can resolve db.internal records from VPC-A's zone
  → No need to duplicate records
```

**Common pattern — Hybrid DNS architecture:**
```
On-prem DNS (192.168.1.10)
  ├── Forwarder: *.googleapis.com → Cloud DNS inbound endpoint
  └── Authoritative: *.on-prem.mycompany.com

Cloud DNS
  ├── Forwarding policy: *.on-prem.mycompany.com → 192.168.1.10
  └── Private zone: *.cloud.mycompany.com → GCP internal IPs
```

---

### Q27. What is a Cloud DNS inbound/outbound endpoint? When do you need them?

**Answer:** Needed for **hybrid DNS** where on-prem servers need to resolve GCP private zones (inbound) or GCP needs to resolve on-prem zones (outbound).

| Endpoint Type | Direction | Purpose |
|---|---|---|
| **Inbound endpoint** | On-prem → GCP | On-prem DNS can forward queries to GCP private zones |
| **Outbound endpoint** | GCP → On-prem | Cloud DNS forwards queries to on-prem DNS server |

**Why inbound endpoint?**
On-prem DNS can't directly query `10.0.0.2` (GCP's metadata resolver) — it's only accessible from inside GCP. An inbound endpoint creates a **reachable IP** (via VPN/Interconnect) that on-prem can forward to.

```bash
# Create inbound DNS endpoint (on-prem can forward to this IP via VPN)
gcloud dns policies create hybrid-dns-policy \
  --networks=my-vpc \
  --enable-inbound-forwarding
```

---

### Q28. What is Split-Horizon DNS?

**Answer:** Split-horizon (split-brain) DNS returns **different DNS answers** to the same domain based on where the query comes from.

**Example:**
```
api.mycompany.com queried from internet     → returns 34.102.x.x (public LB IP)
api.mycompany.com queried from inside VPC   → returns 10.0.0.100 (internal LB IP)
```

**GCP implementation:**
```
Public zone:  api.mycompany.com → A record: 34.102.x.x
Private zone: api.mycompany.com → A record: 10.0.0.100  (visible only inside VPC)
```

Cloud DNS private zone **shadows** the public zone for VMs inside the VPC. Internal traffic stays on the internal network — no hairpin through the internet or LB.

---

## 6. Cloud NAT

---

### Q29. What is Cloud NAT? How does it differ from a NAT gateway VM?

**Answer:** Cloud NAT (Network Address Translation) allows VMs **without external IPs** to initiate outbound connections to the internet.

**How it works:**
- Cloud NAT is a **software-defined, distributed** service — not a VM or appliance
- VMs with only internal IPs send traffic to `0.0.0.0/0` → Cloud NAT translates their internal IP to a NAT IP
- Inbound connections from the internet are **blocked** — NAT only works for outbound

| | Cloud NAT | NAT Gateway VM |
|---|---|---|
| Management | Fully managed — no VMs to patch | You manage the VM (patching, HA, sizing) |
| Scalability | Auto-scales | Limited by VM size |
| HA | Built-in | You must set up redundancy |
| Cost | Pay per hour + data processed | Pay for VM + egress |
| Bandwidth | Up to 100 Gbps+ | Limited by VM NIC |

---

### Q30. What are NAT IP allocation modes? What is port exhaustion and how do you fix it?

**Answer:**

**NAT IP allocation modes:**
| Mode | Description |
|---|---|
| **Auto (recommended)** | GCP automatically allocates NAT IPs and ports; scales up automatically |
| **Manual** | You specify the external IPs to use (needed when on-prem allowlists specific IPs) |

**Port allocation:**
Each NAT IP provides **64,512 ports**. Each concurrent outbound connection uses one port.

**Port exhaustion** happens when a VM makes more concurrent outbound connections than its port allocation allows — new connections fail with `Connection timed out`.

**Fix options:**
| Fix | How |
|---|---|
| **Increase min ports per VM** | `--min-ports-per-vm=128` → `--min-ports-per-vm=1024` |
| **Add more NAT IPs** | More IPs = more total ports available |
| **Enable Dynamic Port Allocation** | Cloud NAT dynamically allocates more ports to busy VMs (up to max-ports-per-vm) |

```bash
# Enable Dynamic Port Allocation (best practice)
gcloud compute routers nats update my-nat \
  --router=my-router \
  --region=us-central1 \
  --enable-dynamic-port-allocation \
  --min-ports-per-vm=32 \
  --max-ports-per-vm=65536
```

---

### Q31. What is Private NAT? How is it different from regular Cloud NAT?

**Answer:** Private NAT translates **internal IP ranges** when routing between VPCs or between VPC and on-prem — solves overlapping IP range issues without renumbering.

| | Cloud NAT | Private NAT |
|---|---|---|
| Translates | Internal IP → Public NAT IP | Internal IP → Another internal IP |
| Use case | Private VMs accessing internet | Overlapping CIDRs between VPCs or on-prem |
| Direction | Outbound to internet only | VPC-to-VPC or VPC-to-on-prem |

**Example scenario:**
```
VPC-A: 10.0.0.0/8 (your GCP network)
On-prem: 10.0.0.0/8 (same range — overlap!)
Solution: Private NAT translates on-prem traffic to 172.16.0.0/12 range before entering VPC-A
```

---

## 7. Hybrid Connectivity — VPN & Interconnect

---

### Q32. What is Cloud VPN? What are the two types?

**Answer:** Cloud VPN creates an **IPsec-encrypted tunnel** over the public internet between GCP VPC and an on-prem network.

| | Classic VPN | HA VPN |
|---|---|---|
| Gateways per tunnel | 1 (single interface) | 2 (redundant interfaces) |
| SLA | 99.9% | **99.99%** |
| Tunnels supported | 1 per gateway | 2 tunnels (one per interface) |
| Dynamic routing (BGP) | Yes (route-based only) | Yes |
| Max bandwidth | ~3 Gbps | ~3 Gbps per tunnel pair |
| Use case | Dev/test, legacy setups | **All production VPNs** |

**HA VPN topology:**
```
GCP HA VPN Gateway
  ├── Interface 0 → Tunnel 0 → On-prem Router 1
  └── Interface 1 → Tunnel 1 → On-prem Router 2
Both tunnels active-active via BGP ECMP
If one tunnel fails → 99.99% — traffic shifts to surviving tunnel
```

```bash
# Create HA VPN gateway
gcloud compute vpn-gateways create ha-vpn-gw \
  --network=my-vpc \
  --region=us-central1

# Create Cloud Router for BGP
gcloud compute routers create vpn-router \
  --network=my-vpc \
  --region=us-central1 \
  --asn=65001
```

---

### Q33. What is Cloud Interconnect? What is the difference between Dedicated and Partner?

**Answer:** Cloud Interconnect provides a **private, high-bandwidth physical connection** between on-prem and GCP — traffic does NOT traverse the public internet.

| | Dedicated Interconnect | Partner Interconnect |
|---|---|---|
| Connection | Physical fiber to Google PoP | Through a service provider |
| Bandwidth | 10 Gbps or 100 Gbps circuits | 50 Mbps to 50 Gbps |
| Requirement | Your facility at a Google PoP (or colocation) | Any location via provider |
| SLA | 99.99% (2 metro + 2 regions) | 99.99% (depending on topology) |
| Cost | Higher | Lower (but provider adds cost) |
| Use case | Large enterprises, >10 Gbps needs | Enterprises not near a Google PoP |

**Why Interconnect over VPN:**
- No encryption overhead — raw bandwidth (though you CAN add MACsec/encryption)
- Consistent low latency — not subject to internet routing changes
- Data transfer pricing is cheaper than internet egress
- Required for SLA-backed connectivity

---

### Q34. What is a VLAN attachment (Interconnectattachment)? How does it relate to Cloud Router?

**Answer:** A VLAN attachment is the **logical endpoint** of a Cloud Interconnect circuit inside your VPC. It's the link between the physical circuit and your VPC routing.

```
Physical Interconnect circuit (Layer 1/2)
  │
  └── VLAN Attachment (Layer 3 — has a VLAN ID + BGP peer IPs)
        │
        └── Cloud Router (runs BGP session with on-prem)
              │
              └── Learned routes distributed to VPC
```

**Key settings on VLAN attachment:**
| Setting | Description |
|---|---|
| VLAN ID | 802.1Q VLAN tag (from Google or you choose) |
| Cloud Router IP | GCP side of BGP link IP (e.g., `169.254.0.1/30`) |
| On-prem Router IP | On-prem side of BGP link IP (e.g., `169.254.0.2/30`) |
| Bandwidth | Reserved bandwidth (50 Mbps to 50 Gbps) |

---

### Q35. How do you achieve 99.99% SLA with Cloud Interconnect?

**Answer:** A single Interconnect circuit only gives **99.9% SLA**. To achieve 99.99%, you need redundant circuits in two different **metro areas** connected to two different **GCP regions**.

**99.99% topology:**
```
On-prem Location 1 (Metro A)
  └── Dedicated Interconnect → Google PoP (Metro A) → GCP Region 1 (us-central1)
        VLAN Attachment 1 → Cloud Router 1

On-prem Location 2 (Metro B — different city)
  └── Dedicated Interconnect → Google PoP (Metro B) → GCP Region 2 (us-east1)
        VLAN Attachment 2 → Cloud Router 2
```

Both Cloud Routers advertise and receive the same routes — active-active via BGP ECMP.

**For 99.9% SLA:** Two circuits in same metro. **For 99.99% SLA:** Two circuits in two different metros.

---

### Q36. What is the difference between Cloud VPN and Interconnect? When do you choose each?

**Answer:**

| Factor | Cloud VPN | Cloud Interconnect |
|---|---|---|
| Connection | Over public internet (encrypted) | Private physical circuit |
| Bandwidth | Up to ~3 Gbps per tunnel pair | 10 Gbps or 100 Gbps per circuit |
| Latency | Variable (internet-dependent) | Consistent, low latency |
| Cost | Low | High (circuit + colocation fees) |
| Setup time | Hours | Weeks–months (physical provisioning) |
| Encryption | Built-in (IPsec) | None by default (add MACsec) |
| SLA | 99.99% (HA VPN) | 99.99% (redundant topology) |

**Choose Cloud VPN when:** Bandwidth < 3 Gbps, temporary connection, cost is priority, or you need quick setup.

**Choose Interconnect when:** > 3 Gbps bandwidth, consistent low latency required, large data transfer volumes (cheaper per GB than internet egress), regulated industries requiring no-internet path.

---

### Q37. What is BGP MED and Local Preference? How are they used for traffic engineering with Cloud Router?

**Answer:**

**BGP attributes for influencing route selection:**

| Attribute | Scope | Influences | Higher is... |
|---|---|---|---|
| **Local Preference** | Internal (within your AS) | Which outbound path GCP prefers (egress from GCP) | Preferred |
| **MED (Multi-Exit Discriminator)** | Sent to BGP peer | Which inbound path on-prem prefers (ingress to GCP) | Less preferred |

**GCP Cloud Router context:**
```bash
# Advertise a route with higher MED → on-prem prefers OTHER tunnel for inbound
gcloud compute routers update-bgp-peer vpn-router \
  --peer-name=on-prem-peer \
  --region=us-central1 \
  --advertised-route-priority=100   # Lower number = higher priority MED in GCP
                                     # 0 = best, 65535 = worst
```

**Use case:** Active-passive failover — primary tunnel has priority 0, backup has priority 100. On-prem sends traffic via primary; Cloud Router fails over to backup automatically.

---

### Q38. What is Network Connectivity Center (NCC)?

**Answer:** NCC is a hub-and-spoke network management service that centralizes connectivity between on-prem, other cloud networks, and GCP VPCs.

```
                    NCC Hub (GCP)
                         │
        ┌────────────────┼────────────────┐
        │                │                │
  VPN Spoke         Interconnect      Router Appliance
  (on-prem 1)       Spoke             Spoke (SD-WAN)
                    (on-prem 2)       (branch offices)
```

**Key capability:** **Spokes can communicate with each other through the hub** — you don't need direct VPN tunnels between every on-prem site. Hub-and-spoke replaces full mesh topology.

**Use case:** Enterprise with 50 branch offices — instead of 50×49/2 = 1225 VPN tunnels, use NCC hub with 50 spokes.

---

## 8. Private Connectivity

---

### Q39. What is Private Service Connect (PSC)? How is it different from VPC Peering?

**Answer:** Private Service Connect allows you to access **Google-managed services or published services** using a **private IP in your VPC** — no public IP, no internet traffic.

| | VPC Peering | Private Service Connect |
|---|---|---|
| What it connects | Two VPCs (peer-to-peer) | Your VPC to a service endpoint |
| Transitive routing | No | N/A (point-to-point) |
| Overlapping IPs | Cannot peer if CIDRs overlap | No issue — uses a single internal IP |
| Direction | Bidirectional route exchange | One-directional — you access the service |
| Use case | Two VPCs need full connectivity | Access Google APIs or published services privately |

**Accessing Google APIs via PSC:**
```bash
# Create PSC endpoint for accessing all Google APIs privately
gcloud compute addresses create psc-google-apis \
  --global \
  --purpose=PRIVATE_SERVICE_CONNECT \
  --addresses=10.0.0.100

gcloud compute forwarding-rules create psc-google-apis-rule \
  --global \
  --address=psc-google-apis \
  --target-google-apis-bundle=all-apis
```

Now `storage.googleapis.com` resolves to `10.0.0.100` inside your VPC — all traffic stays private.

---

### Q40. What is Private Google Access vs Private Service Connect vs VPC Service Controls? How do they differ?

**Answer:**

| Feature | Protects | How |
|---|---|---|
| **Private Google Access** | Traffic path | VMs without external IPs can reach Google APIs; traffic stays on Google network |
| **Private Service Connect** | Traffic path + access point | Creates a private IP endpoint in your VPC for Google APIs or published services |
| **VPC Service Controls** | Data perimeter | Blocks API calls even from valid credentials if they come from outside the perimeter |

**Layered security model:**
```
Private Google Access: Traffic never hits internet ✓
  + Private Service Connect: Traffic goes to your VPC IP, not googleapis.com ✓✓
    + VPC Service Controls: Even if credentials are stolen, calls from outside perimeter are blocked ✓✓✓
```

---

### Q41. What is serverless VPC access? When is it needed?

**Answer:** Serverless VPC Access lets **serverless services** (Cloud Run, Cloud Functions, App Engine) connect to resources **inside a VPC** (e.g., Memorystore, Cloud SQL private IP, internal LB).

**Problem without it:**
```
Cloud Run (serverless) → tries to reach Memorystore Redis at 10.0.0.5 → fails
Because: Cloud Run runs outside your VPC
```

**With Serverless VPC Access connector:**
```
Cloud Run → VPC connector (a small managed VM or set of VMs in your VPC) → Memorystore at 10.0.0.5 ✓
```

```bash
# Create VPC access connector
gcloud compute networks vpc-access connectors create my-connector \
  --region=us-central1 \
  --network=my-vpc \
  --range=10.8.0.0/28   # connector uses IPs from this range

# Reference in Cloud Run deployment
gcloud run deploy my-service \
  --vpc-connector=my-connector \
  --vpc-egress=all-traffic   # route all traffic through VPC (not just RFC1918)
```

---

### Q42. What is VPC Peering? What are its key limitations?

**Answer:** VPC Peering creates private connectivity between two VPCs (same or different projects/organizations) using internal IPs.

**Setup:**
```bash
# Peer VPC-A to VPC-B (must be done from both sides)
gcloud compute networks peerings create peer-a-to-b \
  --network=vpc-a \
  --peer-project=project-b \
  --peer-network=vpc-b

# From project-b side:
gcloud compute networks peerings create peer-b-to-a \
  --network=vpc-b \
  --peer-project=project-a \
  --peer-network=vpc-a
```

**Key limitations:**
| Limitation | Detail |
|---|---|
| **Non-transitive** | A↔B and B↔C does NOT give A↔C. Need direct peering A↔C. |
| **No overlapping CIDRs** | VPCs cannot peer if their subnets have overlapping IP ranges |
| **No tag/SA firewall rules across peers** | Cannot use network tags or service accounts as firewall sources across peered VPCs |
| **Max peerings** | 25 peerings per VPC |
| **No Shared VPC** | Cannot peer a Shared VPC host project VPC with another VPC while subnets are in use |

---

### Q43. What is a Service Attachment? How does PSC service publishing work?

**Answer:** Service Attachment lets you **publish your own service** via PSC — consumers access it via a private IP in their VPC without exposing your internal network.

**Producer side (you publish the service):**
```bash
# Your service is behind an Internal LB
# Create a service attachment pointing to your ILB
gcloud compute service-attachments create my-service \
  --region=us-central1 \
  --producer-forwarding-rule=my-internal-lb \
  --connection-preference=ACCEPT_AUTOMATIC \
  --nat-subnets=psc-nat-subnet
```

**Consumer side (they access your service):**
```bash
# Create a PSC endpoint in their VPC pointing to your service attachment
gcloud compute forwarding-rules create psc-endpoint \
  --region=us-central1 \
  --address=10.0.0.200 \
  --target-service-attachment=projects/my-project/regions/us-central1/serviceAttachments/my-service
```

**Result:** Consumer's VM connects to `10.0.0.200` → goes through PSC → reaches your Internal LB privately. Consumer never sees your VPC's IP space.

---

## 9. Network Security

---

### Q44. What is Cloud Armor? Walk through setting up a WAF policy.

**Answer:** Cloud Armor is GCP's WAF and DDoS protection attached to the Global External HTTP(S) LB.

**Setup:**
```bash
# 1. Create security policy
gcloud compute security-policies create my-waf-policy \
  --description="WAF policy for production"

# 2. Enable OWASP Top 10 managed rules (preview mode first — don't block yet)
gcloud compute security-policies rules add-preconfig-waf-exclusion my-waf-policy \
  --security-policy=my-waf-policy

gcloud compute security-policies rules create 1000 \
  --security-policy=my-waf-policy \
  --expression="evaluatePreconfiguredExpr('sqli-v33-stable')" \
  --action=deny-403 \
  --description="Block SQL injection"

# 3. Block specific country
gcloud compute security-policies rules create 2000 \
  --security-policy=my-waf-policy \
  --expression="origin.region_code == 'CN'" \
  --action=deny-403

# 4. Rate limit: max 100 req/min per IP
gcloud compute security-policies rules create 3000 \
  --security-policy=my-waf-policy \
  --expression="true" \
  --action=rate-based-ban \
  --rate-limit-threshold-count=100 \
  --rate-limit-threshold-interval-sec=60 \
  --ban-duration-sec=600

# 5. Attach to backend service
gcloud compute backend-services update my-backend \
  --security-policy=my-waf-policy \
  --global
```

**Priority evaluation:** Lower number = evaluated first. Default rule is 2147483647 (allow all).

---

### Q45. What is Google Cloud Armor Adaptive Protection? How does it work?

**Answer:** Adaptive Protection is an **ML-based Layer 7 DDoS detection** system built into Cloud Armor. It automatically detects and mitigates application-layer DDoS attacks.

**How it works:**
1. Continuously learns your application's **normal traffic baseline** (req/s, geographic distribution, URL patterns)
2. Detects sudden anomalies (e.g., massive spike from a bot network)
3. Generates an **alert in Cloud Armor dashboard** with a suggested WAF rule
4. You can deploy the suggested rule with one click (or automate with Cloud Functions + pub/sub alert)

**Key metric:** Adaptive Protection scores 0–1 for attack confidence. Score > 0.5 typically warrants action.

**Auto-deploy suggested rules (advanced):**
```
Adaptive Protection Alert → Pub/Sub → Cloud Function → deploy rule to security policy
```

---

### Q46. What is mTLS (mutual TLS)? How is it configured in GCP?

**Answer:** mTLS means both client and server authenticate each other with TLS certificates — not just the server authenticating to the client.

**Standard TLS:**
```
Client verifies server cert → Server trusts all clients with valid connection
```

**mTLS:**
```
Client verifies server cert AND server verifies client cert → Only authorized clients can connect
```

**GCP implementations:**

| Where | How |
|---|---|
| **Cloud Load Balancing** | Configure frontend SSL policy with client authentication (client cert required) |
| **Anthos Service Mesh (Istio)** | PeerAuthentication policy with `STRICT` mTLS between all pods |
| **API Gateway / Cloud Endpoints** | Client cert validation in API config |

```yaml
# Anthos Service Mesh — enforce mTLS across entire namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT   # reject all non-mTLS traffic between services
```

---

### Q47. What is Network Intelligence Center? What tools does it include?

**Answer:** Network Intelligence Center is GCP's suite of **network observability and diagnostics tools**.

| Tool | What it does |
|---|---|
| **Topology** | Visual map of your VPC resources and connections |
| **Connectivity Tests** | Simulate packet flow between two endpoints — tells you if traffic is allowed or blocked and why (which firewall rule blocks it) |
| **Performance Dashboard** | Google-wide latency and packet loss metrics between GCP regions |
| **Firewall Insights** | Identify unused firewall rules, overly permissive rules, and shadowed rules |
| **Network Analyzer** | Continuously monitors for configuration issues and misconfigurations |

**Most useful for network engineers:**

```bash
# Run connectivity test: check if VM A can reach VM B on port 5432
gcloud network-management connectivity-tests create db-test \
  --source-instance=projects/myproject/zones/us-central1-a/instances/app-vm \
  --destination-instance=projects/myproject/zones/us-central1-a/instances/db-vm \
  --destination-port=5432 \
  --protocol=TCP
```

Output tells you exactly: which hops were analyzed, which firewall rule allowed/blocked, whether routing is correct.

---

## 10. Monitoring & Troubleshooting

---

### Q48. How do you troubleshoot a VM that can't reach a Cloud SQL instance?

**Answer:** Systematic network path analysis:

```
Step 1: Verify Cloud SQL is private IP (not public)
  gcloud sql instances describe my-db | grep ipAddresses

Step 2: Check VPC — is VM in same VPC as Cloud SQL private IP?
  Cloud SQL uses Service Networking (VPC peering with google-managed-services-*)
  gcloud compute networks peerings list --network=my-vpc

Step 3: Check VM has no external IP + Private Google Access is enabled on subnet
  gcloud compute networks subnets describe my-subnet --region=us-central1

Step 4: Run Connectivity Test
  gcloud network-management connectivity-tests create sql-test \
    --source-instance=.../app-vm \
    --destination-ip=10.0.1.5 \   # Cloud SQL private IP
    --destination-port=5432 \
    --protocol=TCP

Step 5: Check firewall — Cloud SQL requires ingress TCP:3306 (MySQL) or 5432 (PostgreSQL)
  gcloud compute firewall-rules list --filter="network:my-vpc"

Step 6: Check from inside VM
  telnet 10.0.1.5 5432
  curl -v telnet://10.0.1.5:5432
```

---

### Q49. What is Packet Mirroring? How does it differ from VPC Flow Logs?

**Answer:**

| | VPC Flow Logs | Packet Mirroring |
|---|---|---|
| Data captured | Flow metadata (IPs, ports, bytes, RTT) — no payload | Full **packet capture** including payload |
| Sampling | Yes (default 12.5%) | All packets (no sampling) |
| Use case | Traffic analysis, anomaly detection, billing | Deep packet inspection, IDS/IPS, forensics |
| Destination | Cloud Logging / BigQuery | A collector VM (running Wireshark, Zeek, etc.) |
| Performance impact | Minimal | Higher — copies all traffic |
| Cost | Logging ingestion costs | Collector VM costs + data |

```bash
# Mirror traffic from VMs tagged "web-server" to a collector instance group
gcloud compute packet-mirrorings create web-mirror \
  --region=us-central1 \
  --network=my-vpc \
  --collector-ilb=packet-collector-ilb \
  --mirrored-tags=web-server \
  --filter-cidr-ranges=0.0.0.0/0 \
  --filter-protocols=tcp
```

**Use case:** Attach a third-party IDS (Intrusion Detection System) to the collector — it analyzes full packets for threats without being in the traffic path.

---

### Q50. How do you diagnose a BGP session that won't come up on Cloud VPN?

**Answer:** Systematic BGP troubleshooting:

```
Step 1: Verify VPN tunnel is UP (Phase 1 + Phase 2 IKE)
  gcloud compute vpn-tunnels describe my-tunnel --region=us-central1
  Status should be: ESTABLISHED

Step 2: Check Cloud Router BGP session status
  gcloud compute routers get-status my-router --region=us-central1
  Look for bgpPeerStatus → status: UP / DOWN

Step 3: Common BGP issues:
  ┌─────────────────────────────┬──────────────────────────────────────────┐
  │ Symptom                     │ Cause / Fix                              │
  ├─────────────────────────────┼──────────────────────────────────────────┤
  │ Tunnel UP, BGP DOWN         │ ASN mismatch — verify peer ASN on both   │
  │                             │ sides matches Cloud Router config         │
  ├─────────────────────────────┼──────────────────────────────────────────┤
  │ BGP UP, no routes received  │ On-prem not advertising routes           │
  │                             │ Check on-prem BGP export policy          │
  ├─────────────────────────────┼──────────────────────────────────────────┤
  │ BGP UP, routes not in VPC   │ Dynamic routing mode is REGIONAL         │
  │                             │ Switch VPC to GLOBAL routing mode        │
  ├─────────────────────────────┼──────────────────────────────────────────┤
  │ BGP link IPs unreachable    │ MTU mismatch or firewall blocking 179    │
  │                             │ BGP uses TCP:179 — allow in firewall     │
  └─────────────────────────────┴──────────────────────────────────────────┘

Step 4: Verify learned routes
  gcloud compute routers get-status my-router \
    --region=us-central1 \
    --format="json(result.bgpPeerStatus)"
```

---

### Q51. What is the difference between network latency, jitter, and packet loss? How do you measure each in GCP?

**Answer:**

| Metric | Definition | Acceptable threshold | GCP measurement tool |
|---|---|---|---|
| **Latency** | Round-trip time for a packet | < 5ms intra-region, < 100ms cross-region | Performance Dashboard, `ping`, Connectivity Tests |
| **Jitter** | Variation in latency between packets | < 5ms for real-time apps (VoIP, video) | iperf3 UDP mode from GCE VM |
| **Packet loss** | % of packets that never arrive | < 0.1% for TCP apps, 0% for critical | Performance Dashboard, `ping -c 100` |
| **Bandwidth** | Max throughput | Depends on VM NIC size | iperf3 TCP mode between VMs |

**GCP Performance Dashboard:**
- Shows Google-measured inter-region latency and packet loss at `console.cloud.google.com/networking/networkanalytics`
- Data is from Google's own backbone probes — useful for baseline comparison

```bash
# Test bandwidth between two VMs
# On VM-B (receiver):
iperf3 -s

# On VM-A (sender):
iperf3 -c VM_B_IP -t 30 -P 4   # 30 seconds, 4 parallel streams
```

---

### Q52. What are the GCP VM network performance limits? What affects them?

**Answer:**

| VM family | Max egress bandwidth | Max ingress |
|---|---|---|
| n2-standard-2 | 10 Gbps | ~10 Gbps |
| n2-standard-32 | 32 Gbps | ~32 Gbps |
| n2-standard-80 | 32 Gbps | ~32 Gbps |
| c2-standard-60 | 32 Gbps | ~32 Gbps |
| a2-highgpu-8g | 100 Gbps (RDMA + NIC) | ~100 Gbps |

**Factors that limit network performance:**
| Factor | Impact |
|---|---|
| **vCPU count** | More vCPUs → more bandwidth (up to machine type max) |
| **MTU size** | Default 1460 causes more packets for same data vs 8896 jumbo MTU |
| **Single flow limit** | One TCP flow is limited by single-core CPU speed; use multi-stream iperf for full bandwidth |
| **Tier 1 networking** | Premium tier VMs (`--network-performance-configs=total-egress-bandwidth-tier=TIER_1`) get higher bandwidth |
| **Traffic type** | Inter-zone within region = full bandwidth; Egress to internet = limited + charged |

```bash
# Enable Tier 1 networking (for high-bandwidth VMs)
gcloud compute instances create high-bw-vm \
  --machine-type=n2-standard-32 \
  --network-performance-configs=total-egress-bandwidth-tier=TIER_1
```

---

## Quick Reference — Network Engineer Cheat Sheet

| Problem / Need | GCP Tool / Service |
|---|---|
| Private VMs access internet (outbound only) | Cloud NAT |
| SSH to VM without external IP | IAP TCP Tunneling |
| DNS for internal services | Cloud DNS private zone |
| Resolve on-prem DNS from GCP | Cloud DNS forwarding policy + outbound endpoint |
| Resolve GCP DNS from on-prem | Cloud DNS inbound endpoint |
| Encrypted tunnel to on-prem | Cloud VPN (HA VPN for production) |
| High-bandwidth private link to on-prem | Cloud Interconnect |
| Centralize connectivity for many on-prem sites | Network Connectivity Center (NCC) |
| WAF + DDoS protection | Cloud Armor |
| Distribute global traffic to nearest region | Global External HTTP(S) LB + Anycast |
| Load balance private microservices | Internal HTTP(S) LB |
| Centralized network for multiple projects | Shared VPC |
| Connect two VPCs privately | VPC Peering |
| Access Google APIs with private IP | Private Service Connect |
| Prevent data exfiltration even with stolen creds | VPC Service Controls |
| Analyze full packet captures | Packet Mirroring |
| Debug network path (firewall, routing) | Network Intelligence Center → Connectivity Tests |
| Monitor inter-region latency | Network Intelligence Center → Performance Dashboard |
| Identify unused firewall rules | Network Intelligence Center → Firewall Insights |
| Traffic sampling for flow analysis | VPC Flow Logs |
| Overlapping CIDR between VPCs / on-prem | Private NAT |
| Publish your service privately to consumers | Private Service Connect (Service Attachment) |
