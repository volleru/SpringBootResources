# Google Cloud Platform — Networking Interview Q&A
### Senior Position | TCS Interview Preparation

---

## Section 1: VPC & Subnets

---

**Q1. What is a VPC in Google Cloud and how does it differ from traditional networks?**

**Answer:**
A **VPC (Virtual Private Cloud)** in GCP is a globally distributed, software-defined network that spans all GCP regions. Key differences from traditional networks:

- **Global by default** — A single VPC spans all regions; you do not need separate VPCs per region.
- **Subnets are regional** — Subnets exist within a single region but resources in different zones of that region share the same subnet.
- **No physical hardware** — It is entirely virtualized using Google's SDN (Software-Defined Networking) infrastructure (Andromeda).
- **Auto mode vs Custom mode** — Auto mode creates subnets in each region automatically; custom mode gives full control.
- **No broadcast/multicast** — GCP VPCs do not support broadcast or multicast traffic.

---

**Q2. What is the difference between Auto Mode and Custom Mode VPCs?**

**Answer:**

| Feature | Auto Mode | Custom Mode |
|---|---|---|
| Subnet creation | Automatic, one per region | Manual, full control |
| IP ranges | Pre-defined (`10.128.0.0/9`) | User-defined |
| Expandable | Yes | Yes |
| Convert to custom | Yes (one-way) | N/A |
| Use case | Dev/test, quick setup | Production, enterprise |

**Best practice:** Always use **Custom Mode** in production to avoid overlapping IP ranges and maintain control over network design.

---

**Q3. What is a Shared VPC and when would you use it?**

**Answer:**
**Shared VPC** allows an organization to connect resources from multiple projects to a common VPC network hosted in a **host project**. Other projects are called **service projects**.

**Use cases:**
- Centralized network management by a networking team while allowing individual teams to manage their own resources.
- Enforce consistent firewall rules and routing across the organization.
- Reduce the number of VPNs/interconnects by sharing connectivity.

**Key roles:**
- `compute.xpnAdmin` — manages Shared VPC at organization/folder level.
- `compute.networkUser` — granted to service project service accounts to use subnets.

**Important:** Resources in service projects can only use subnets from the host project that they've been explicitly granted access to.

---

**Q4. Explain VPC Peering and its limitations.**

**Answer:**
**VPC Peering** allows private RFC 1918 connectivity between two VPC networks, even across different projects or organizations, without using external IPs, VPNs, or Interconnects.

**How it works:** Both VPCs must independently create a peering connection to each other (bidirectional request required).

**Limitations:**
1. **Non-transitive** — If VPC-A peers with VPC-B and VPC-B peers with VPC-C, VPC-A cannot reach VPC-C through VPC-B.
2. **No overlapping CIDRs** — Peered VPCs must have non-overlapping IP ranges.
3. **No tag/service account sharing** — Firewall rules based on network tags or service accounts from a peered VPC do not apply.
4. **Subnet routes only** — Only subnet routes are exchanged by default (custom static/dynamic routes are not exported unless explicitly configured).
5. **Quota limits** — Maximum 25 peering connections per VPC.

---

**Q5. What is Private Google Access and why is it important?**

**Answer:**
**Private Google Access** allows VM instances with **only internal (private) IP addresses** to reach Google APIs and services (e.g., Cloud Storage, BigQuery, Pub/Sub) without needing an external IP or NAT gateway.

**How to enable:** Set `privateIpGoogleAccess: true` on the subnet.

**Why it matters:**
- Improves security by keeping VMs off the public internet.
- Reduces egress costs (traffic stays on Google's network).
- Required for VMs in private subnets to access GCP services.

**Private Google Access for on-premises:** Uses `private.googleapis.com` (199.36.153.8/30) or `restricted.googleapis.com` (199.36.153.4/30) via VPN or Interconnect to access Google APIs from on-prem without traversing the internet.

---

## Section 2: Firewall Rules & Security

---

**Q6. How do GCP Firewall Rules work? Explain priority and direction.**

**Answer:**
GCP firewall rules are **stateful** and applied at the VM instance level (not at the subnet or VPC boundary).

**Key attributes:**
- **Direction:** `INGRESS` (incoming) or `EGRESS` (outgoing)
- **Priority:** 0–65535 (lower number = higher priority). Default implied rules are 65534 (allow all egress) and 65535 (deny all ingress).
- **Action:** `ALLOW` or `DENY`
- **Target:** All instances, specific network tags, or specific service accounts
- **Source/Destination:** IP ranges, tags, or service accounts

**Evaluation order:**
1. Rules are evaluated from lowest to highest priority number.
2. The first matching rule wins.
3. If no rule matches, the implied default rule applies.

**Best practice:** Use **service account-based** firewall rules over network tags in production — service accounts cannot be spoofed by users adding tags to VMs.

---

**Q7. What is Cloud Armor and how does it differ from VPC Firewall Rules?**

**Answer:**

| Feature | VPC Firewall Rules | Cloud Armor |
|---|---|---|
| Layer | L3/L4 (IP, port, protocol) | L7 (HTTP/HTTPS) |
| Scope | VM-level traffic | External HTTP(S) Load Balancer |
| DDoS protection | No | Yes (adaptive protection) |
| WAF rules | No | Yes (OWASP rules) |
| Geo-blocking | No | Yes |
| Rate limiting | No | Yes |

**Cloud Armor** is a WAF (Web Application Firewall) and DDoS mitigation service that sits in front of the **External HTTP(S) Load Balancer**. It uses security policies with rules to allow/deny/throttle traffic based on IP, geography, request headers, or OWASP rule sets.

---

**Q8. What is the difference between Ingress and Egress firewall rules in GCP?**

**Answer:**
- **Ingress rules** — Control traffic **entering** a VM from a source. The source can be an IP range, another instance's tag, or a service account.
- **Egress rules** — Control traffic **leaving** a VM to a destination. Destination can be an IP range.

**Default behavior (implied rules):**
- Default **deny all ingress** (priority 65535)
- Default **allow all egress** (priority 65534)

**Stateful nature:** If an ingress connection is allowed, the return traffic (response) is automatically allowed regardless of egress rules. GCP tracks connection state.

**Important:** Firewall rules in GCP cannot be applied to Cloud SQL, Memorystore, or other managed services — those have their own access controls.

---

## Section 3: Load Balancing

---

**Q9. Explain the different types of Load Balancers available in GCP.**

**Answer:**

**External Load Balancers (internet-facing):**

| Type | Layer | Scope | Protocol |
|---|---|---|---|
| External HTTP(S) LB | L7 | Global | HTTP, HTTPS |
| External SSL Proxy | L4 | Global | SSL/TLS |
| External TCP Proxy | L4 | Global | TCP |
| External Network TCP/UDP | L4 | Regional | TCP, UDP |

**Internal Load Balancers (internal VPC traffic):**

| Type | Layer | Protocol |
|---|---|---|
| Internal HTTP(S) LB | L7 | HTTP, HTTPS |
| Internal TCP/UDP LB | L4 | TCP, UDP |

**Key differentiators:**
- **Global LBs** (HTTP(S), SSL Proxy, TCP Proxy) use **Google's global network** and can route to the nearest healthy backend.
- **Regional LBs** (Network LB, Internal LB) stay within a single region.
- **HTTP(S) LB** supports URL-based routing, content-based routing, and integrates with Cloud Armor and Cloud CDN.

---

**Q10. What is the difference between a Backend Service and a Backend Bucket in GCP Load Balancing?**

**Answer:**
- **Backend Service** — Defines how traffic is distributed to backends (instance groups, NEGs). Supports health checks, session affinity, connection draining, and CDN policies. Used for dynamic content served by compute resources.
- **Backend Bucket** — Serves static content directly from **Cloud Storage buckets**. Integrates with Cloud CDN. Used for static assets (images, JS, CSS).

A single HTTP(S) Load Balancer can have **multiple backend services and backend buckets** differentiated by URL map path rules.

---

**Q11. What is a Network Endpoint Group (NEG) and what are its types?**

**Answer:**
A **NEG** is a configuration object that specifies a group of backend endpoints (IP:port combinations), providing more granular traffic control than managed instance groups.

**Types:**
1. **Zonal NEG** — VM instances or containers (GKE pods) within a single zone. Supports `GCE_VM_IP_PORT` and `NON_GCP_PRIVATE_IP_PORT`.
2. **Internet NEG** — External backend endpoints (FQDN or IP) outside GCP. Used to LB traffic to third-party services.
3. **Serverless NEG** — Points to Cloud Run, App Engine, or Cloud Functions. No IP/port — identified by region and service name.
4. **Hybrid Connectivity NEG** — On-premises or other cloud endpoints reachable via VPN/Interconnect.
5. **Private Service Connect NEG** — Endpoints accessed via PSC.

**Use case:** Serverless NEGs are essential for routing traffic to Cloud Run services behind an HTTP(S) LB with Cloud Armor.

---

**Q12. Explain Health Checks in GCP Load Balancing. What happens when a backend fails?**

**Answer:**
**Health checks** probe backend instances at regular intervals to determine liveness. GCP load balancers only send traffic to **healthy backends**.

**Types:** HTTP, HTTPS, HTTP/2, TCP, SSL, gRPC

**Key parameters:**
- `checkIntervalSec` — How often to probe (default: 5s)
- `timeoutSec` — How long to wait for a response (default: 5s)
- `healthyThreshold` — Consecutive successes to mark healthy (default: 2)
- `unhealthyThreshold` — Consecutive failures to mark unhealthy (default: 2)

**Health check source IPs:**
- For global LBs: `35.191.0.0/16` and `130.211.0.0/22`
- For regional LBs: `35.191.0.0/16` and `209.85.152.0/22`

**Firewall rule required:** You must create an ingress allow rule for these ranges on your backends, otherwise all instances appear unhealthy.

**When backend fails:** Traffic is redistributed to remaining healthy backends. If all backends in a group fail, traffic may spill over to other backend services based on failover configuration.

---

## Section 4: Cloud DNS

---

**Q13. What is Cloud DNS and what are the differences between Public and Private Zones?**

**Answer:**
**Cloud DNS** is Google's scalable, reliable managed DNS service with 100% SLA uptime.

**Public Zones:**
- Accessible from the internet.
- Used for domains that need to be resolved by external clients.
- Requires domain ownership verification for DNSSEC.

**Private Zones:**
- Only accessible from authorized VPC networks.
- Used for internal service discovery and private hostnames.
- Can be peered across VPCs using **DNS peering**.
- Supports **forwarding zones** to send DNS queries to on-premises resolvers.

**Forwarding Zones:** Used to resolve on-premises DNS names from GCP. DNS queries for the specified domain are forwarded to on-premises DNS servers via VPN/Interconnect using forwarding targets.

**Response Policy Zones (RPZ):** Override DNS responses for specific domains — used for internal traffic steering or blocking malicious domains.

---

**Q14. How does DNS work in a Shared VPC setup?**

**Answer:**
In Shared VPC:
- Cloud DNS **private zones** are authorized per-VPC, not per-project.
- A private zone created in the **host project** must be explicitly authorized for each **service project's** VPC network.
- Alternatively, **DNS peering** can be used so that service project VPCs resolve names by querying the host project's DNS.

**Key command:** When creating or updating a private zone, use `--networks` flag to list all authorized VPC networks (including service project VPCs).

**DNS peering direction matters:** A peering zone on VPC-A that peers to VPC-B means VPC-A's DNS queries for that zone are resolved by VPC-B's Cloud DNS. This is unidirectional.

---

## Section 5: Hybrid Connectivity

---

**Q15. What are the options for connecting on-premises networks to GCP? Compare them.**

**Answer:**

| Feature | Cloud VPN | Dedicated Interconnect | Partner Interconnect |
|---|---|---|---|
| Bandwidth | Up to 3 Gbps per tunnel | 10 Gbps or 100 Gbps | 50 Mbps to 10 Gbps |
| Latency | Higher (internet path) | Low (dedicated fiber) | Medium |
| SLA | 99.99% (HA VPN) | 99.99% (with redundancy) | 99.99% (with redundancy) |
| Setup time | Minutes | Weeks–months | Days–weeks |
| Cost | Low | High | Medium |
| Encryption | Yes (IPsec) | No (must add MACsec/VPN) | No (Layer 2 handoff) |
| Use case | Dev/test, backup | Enterprise, high-throughput | Moderate needs, no direct POP |

**HA VPN requirements for 99.99% SLA:**
- Two VPN gateways, each with two interfaces
- Must connect to two separate peer gateways (or a HA peer gateway)
- All four tunnels must be active

---

**Q16. Explain Cloud Interconnect — Dedicated vs Partner. What is a VLAN attachment?**

**Answer:**
**Dedicated Interconnect:**
- Direct physical connection between your on-premises network and Google's network at a **colocation facility**.
- Circuits: 10 Gbps or 100 Gbps.
- You manage the physical cross-connect at the colocation.

**Partner Interconnect:**
- Connection through a **service provider** (partner) that already has Dedicated Interconnect to Google.
- Useful if you can't colocate at a Google facility.
- Bandwidth: 50 Mbps to 10 Gbps.

**VLAN Attachment (Interconnect Attachment):**
- A logical connection that associates an Interconnect circuit with a specific VPC via a **Cloud Router**.
- Each VLAN attachment has a VLAN ID and connects to a single region's Cloud Router.
- Multiple VLAN attachments can share one Dedicated Interconnect circuit.
- Provides a BGP session between Cloud Router and on-premises router.

---

**Q17. What is Cloud Router and how does it work with BGP?**

**Answer:**
**Cloud Router** is a fully distributed, managed BGP (Border Gateway Protocol) routing service that enables dynamic route exchange between GCP VPCs and on-premises networks (via VPN or Interconnect).

**How it works:**
1. Cloud Router establishes a **BGP session** with the on-premises router (or peer VPN gateway).
2. On-premises routes are **advertised to Cloud Router** and programmed as dynamic routes in the VPC routing table.
3. VPC subnet routes are **advertised to on-premises** by Cloud Router.
4. Routes are automatically updated when subnets are added/removed.

**Key features:**
- **Custom route advertisements** — Advertise specific IP ranges to peers.
- **Route filtering** — Import/export route filters using route policies.
- **Graceful restart** — Maintains forwarding during control plane restarts.
- **Regional but affects global routing** — A Cloud Router in one region can manage routes for the entire VPC if dynamic routing mode is set to **global** (vs regional).

---

## Section 6: Cloud NAT

---

**Q18. What is Cloud NAT and when would you use it?**

**Answer:**
**Cloud NAT (Network Address Translation)** is a fully managed, software-defined NAT service that allows **VM instances without external IPs** to initiate outbound connections to the internet.

**Key characteristics:**
- **No proxy VMs** — Implemented in Google's network stack (Andromeda), not on VMs.
- **Per-region** — One NAT gateway per region per VPC.
- **No inbound connections** — Cloud NAT is outbound-only; it does not allow internet-initiated connections.
- **Port allocation:** Dynamic (default) or manual. Dynamic allocates ports as needed; manual pre-allocates ports per VM.

**Use cases:**
- Private VMs that need to download packages (apt, yum) or call external APIs.
- GKE nodes in private clusters.

**Cloud NAT + Private Google Access:** These are complementary. Private Google Access routes traffic to Google APIs internally; Cloud NAT routes other internet traffic.

**Logging:** Cloud NAT supports logging of NAT translations and errors via Cloud Logging.

---

## Section 7: Routing

---

**Q19. Explain the different types of routes in GCP VPC.**

**Answer:**

**1. System-generated routes:**
- **Default route** (`0.0.0.0/0`) — Created automatically, points to the default internet gateway. Can be deleted to block internet access.
- **Subnet routes** — Automatically created for each subnet. Define paths to subnet IP ranges within the VPC.

**2. Custom routes:**
- **Static routes** — Manually created. Next hop can be an instance, instance group, VPN tunnel, internal LB, or IP address.
- **Dynamic routes** — Learned via BGP through Cloud Router. Scope depends on VPC dynamic routing mode (regional or global).

**Route evaluation order:**
1. Most specific prefix match wins (longest prefix match).
2. If equal prefix length, the route with the **lowest priority** number wins.
3. System-generated subnet routes always take precedence over custom routes for the same prefix.

**Policy-based routing (PBR):** Routes traffic based on source IP or other metadata, bypassing destination-based routing. Used for traffic steering to NFV appliances.

---

**Q20. What is the difference between Regional and Global dynamic routing mode in a VPC?**

**Answer:**

| Feature | Regional | Global |
|---|---|---|
| Cloud Router scope | Routes learned/advertised only in the same region | Routes learned/advertised across all regions |
| Route propagation | VMs in other regions don't see on-prem routes via this router | All VMs in the VPC see on-prem routes from any region's Cloud Router |
| Use case | Region-isolated connectivity | Multi-region hybrid networking |

**Example:**
- VPN in `us-central1` with **regional** mode → only VMs in `us-central1` can reach on-prem via this VPN.
- VPN in `us-central1` with **global** mode → VMs in `europe-west1` can also reach on-prem via the `us-central1` VPN (with cross-region traffic on Google's backbone).

**Default:** Regional mode. Use global mode for enterprise multi-region hybrid deployments.

---

## Section 8: Network Service Tiers

---

**Q21. What are Network Service Tiers in GCP?**

**Answer:**
GCP offers two network service tiers that determine how traffic travels between GCP and the internet:

**Premium Tier (default):**
- Traffic enters and exits Google's **global high-quality private network** at the point closest to the user.
- Lower latency, higher reliability.
- Uses Google's global backbone for most of the path.
- Required for **global load balancers**.
- Higher cost.

**Standard Tier:**
- Traffic uses the **public internet** for most of the path; only enters Google's network near the destination region.
- Higher latency, comparable to typical cloud providers.
- Lower cost (~25–30% cheaper).
- Only supports **regional** load balancers.

**Use case guidance:**
- Mission-critical, latency-sensitive applications → Premium Tier.
- Batch workloads, internal tools, cost-sensitive workloads → Standard Tier.

---

## Section 9: Cloud CDN & Load Balancer Integration

---

**Q22. How does Cloud CDN work and how is it integrated with GCP Load Balancing?**

**Answer:**
**Cloud CDN** caches content at Google's globally distributed **edge PoPs (Points of Presence)** to reduce latency and origin load.

**Integration:** Cloud CDN is enabled at the **backend service or backend bucket** level on the **External HTTP(S) Load Balancer**.

**Cache modes:**
- `USE_ORIGIN_HEADERS` — Respects cache-control headers from the origin.
- `CACHE_ALL_STATIC` — Caches static content (images, CSS, JS) even without cache-control headers.
- `FORCE_CACHE_ALL` — Caches all cacheable responses, ignoring cache-control headers.

**Cache invalidation:** Use the `gcloud compute url-maps invalidate-cdn-cache` command to purge cached content by URL or prefix.

**Signed URLs/Cookies:** Control access to cached content with time-limited, cryptographically signed URLs. Used to protect premium content.

**Cloud CDN + Cloud Armor:** Requests intercepted by Cloud Armor before reaching CDN — security policies are enforced at the edge.

---

## Section 10: Network Monitoring & Operations

---

**Q23. What is VPC Flow Logs and what are its use cases?**

**Answer:**
**VPC Flow Logs** capture samples of network flows (TCP, UDP, ICMP, ESP, GRE) sent and received by **VM instances** (including GKE nodes). Logs are sent to **Cloud Logging** and can be exported to BigQuery or Cloud Storage.

**Enabled at the subnet level.**

**Log record fields include:** Source/destination IP, port, protocol, bytes, packets, start/end time, latency (RTT for TCP), and metadata (VM names, geographic info).

**Use cases:**
- **Network troubleshooting** — Diagnose connectivity issues.
- **Security analysis** — Detect port scanning, unusual traffic patterns.
- **Compliance auditing** — Network access audit trails.
- **Cost analysis** — Identify top talkers and optimize traffic routing.

**Sampling rate:** Default 50%. Can be adjusted from 0.0 (off) to 1.0 (100%) — higher sampling = higher cost.

**Important:** Flow logs only capture traffic from/to VMs. Managed services (Cloud SQL, Memorystore) are not captured.

---

**Q24. What is Network Intelligence Center and what tools does it offer?**

**Answer:**
**Network Intelligence Center** is a centralized console for network monitoring, verification, and optimization. It includes:

1. **Network Topology** — Visual graph of your GCP network showing VPCs, subnets, VMs, and traffic flows. Shows live traffic metrics (bytes, packets, latency).

2. **Connectivity Tests** — Logical simulation of packet path between a source and destination. Checks firewall rules, routing, and configuration without sending real traffic. Useful for diagnosing "why can't VM-A reach VM-B."

3. **Performance Dashboard** — Shows packet loss and latency metrics between Google Cloud regions and from Google Cloud to end users.

4. **Firewall Insights** — Analyzes firewall rules to identify:
   - Shadowed rules (rules overridden by higher-priority rules)
   - Overly permissive rules (allow rules that are never hit)
   - Rules not used in the past 90 days

5. **Network Analyzer** — Automatically detects network misconfigurations and provides recommended fixes.

---

**Q25. How would you troubleshoot a connectivity issue between two VMs in different subnets within the same VPC?**

**Answer:**
**Systematic approach:**

1. **Check firewall rules:**
   - Ingress rule on destination VM allowing the source IP/tag and port.
   - Egress rule on source VM allowing the destination IP and port (default allows all egress).
   - Use `gcloud compute firewall-rules list --filter="network=<vpc-name>"`.

2. **Check routing:**
   - Both subnets should have auto-created subnet routes within the same VPC.
   - Use `gcloud compute routes list`.

3. **Check OS-level firewall:** `iptables -L` or `ufw status` — GCP firewalls and OS firewalls are independent.

4. **Check VM network interface:** Ensure both VMs are on the same VPC (not different VPCs without peering).

5. **Use Connectivity Tests (Network Intelligence Center):**
   - Specify source VM, destination VM, protocol/port.
   - The tool traces the logical path and identifies where packets are dropped.

6. **Check VPC Flow Logs:** Enable on both subnets and look for dropped packets.

7. **Check if destination VM has a public IP vs private IP only:** If using internal IP, ensure you're targeting the correct IP.

---

## Section 11: Advanced Networking

---

**Q26. What is Private Service Connect (PSC) and how does it differ from VPC Peering?**

**Answer:**
**Private Service Connect** allows consumers to access **managed services** (Google APIs, third-party services, or your own published services) through a **private endpoint** in their VPC using an internal IP address.

**How it works:**
- A **PSC endpoint** (forwarding rule with `--load-balancing-scheme=INTERNAL`) is created in the consumer's VPC.
- Traffic to the endpoint stays entirely within Google's network — no internet exposure.
- The service provider publishes a **PSC service attachment** (backed by an internal LB).

**PSC vs VPC Peering:**

| Feature | VPC Peering | Private Service Connect |
|---|---|---|
| Transitive routing | No | Yes (service-level) |
| Overlapping CIDRs | Not allowed | Allowed |
| Exposed resources | All routes | Only specific service endpoints |
| Use case | Peer VPCs for full connectivity | Access specific services privately |
| Direction | Bidirectional | Consumer-to-producer only |

**PSC for Google APIs:** Access `googleapis.com` services (GCS, BigQuery) via a PSC endpoint with a custom internal IP in your VPC — stricter than Private Google Access.

---

**Q27. What is a Multi-NIC VM in GCP and when is it used?**

**Answer:**
A **Multi-NIC VM** has multiple network interface cards, each attached to a **different VPC network**. Traffic routing between NICs must be handled at the OS level.

**Constraints:**
- Maximum NICs = min(8, number of vCPUs / 2) — up to 8 NICs.
- Each NIC must be in a **different VPC network** (not just different subnets).
- NICs are configured at VM creation time and cannot be added/removed afterward.
- Each NIC can optionally have an external IP.

**Use cases:**
1. **Network Virtual Appliances (NVAs)** — Firewalls, IDS/IPS, or packet inspection appliances that need to sit between two VPCs.
2. **Transit VPC pattern** — A hub VPC with NVAs that inspect traffic between spoke VPCs.
3. **Traffic mirroring targets** — Receive mirrored traffic from production VPC while connected to a management VPC.

**Routing note:** The default route (`0.0.0.0/0`) only applies to NIC0 by default. For other NICs, you must add policy-based routing rules at the OS level.

---

**Q28. What is Packet Mirroring in GCP and how is it used for security?**

**Answer:**
**Packet Mirroring** clones traffic from selected VM instances and forwards the copy to a collector (IDS/IPS tool or packet analyzer) without affecting the original traffic flow.

**Components:**
- **Mirrored source:** VM instances, subnets, or instance tags in a specific region.
- **Collector destination:** An **Internal TCP/UDP Load Balancer** fronting the IDS/packet analyzer VMs.
- **Packet mirroring policy:** Filters (CIDR, protocol, direction) to define what traffic is mirrored.

**Use cases:**
- Intrusion Detection System (IDS) — Tools like Palo Alto, Suricata, or Zeek receive full packet copies.
- Security forensics.
- Network performance analysis.

**Key points:**
- Mirroring is at the **packet level** (not just metadata like Flow Logs).
- Mirrored traffic and collector must be in the **same region**.
- Only affects VMs — not traffic between managed services.
- Works across Shared VPC (mirror in service project, collect in host project).

---

**Q29. Explain the concept of Hierarchical Firewall Policies.**

**Answer:**
**Hierarchical Firewall Policies** allow firewall rules to be defined at the **organization or folder level** in the resource hierarchy, and inherited by all child projects and VPCs.

**How it works:**
- Policies contain rules with `ALLOW`, `DENY`, or `GOTO_NEXT` actions.
- `GOTO_NEXT` — Rule does not match; evaluation proceeds to the next policy layer (child policy or VPC firewall rules).
- Policies are evaluated **top-down**: organization → folder → project VPC firewall rules.

**Evaluation order:**
1. Organization-level hierarchical firewall policy
2. Folder-level hierarchical firewall policy (from outer to inner folder)
3. VPC network firewall rules (legacy or network firewall policy)

**Advantages over VPC firewall rules:**
- Centrally enforce security baselines (e.g., always deny port 23/Telnet) across all projects.
- Cannot be overridden by project admins if the policy rule is `ALLOW` or `DENY` (not `GOTO_NEXT`).
- Delegated administration — assign `compute.orgFirewallPolicyAdmin` role to network teams.

---

**Q30. How would you design a high-availability hybrid network architecture connecting on-premises to GCP for an enterprise?**

**Answer:**
**Architecture: HA VPN + Dedicated Interconnect with redundancy**

**Design principles:**
1. **Dual redundancy at every layer** — No single point of failure.
2. **Separate physical paths** — Different routers, different circuits.

**Reference Architecture:**

```
On-Premises                        GCP
┌─────────────────┐        ┌─────────────────────────┐
│  Router A ──────┼──VPN───┼─ Cloud VPN GW (region1) │
│  Router A ──────┼──IXC───┼─ Interconnect (region1) │
│                 │        │         │                │
│  Router B ──────┼──VPN───┼─ Cloud VPN GW (region2) │
│  Router B ──────┼──IXC───┼─ Interconnect (region2) │
└─────────────────┘        │                         │
                           │  Cloud Router (global)  │
                           │         │               │
                           │       VPC               │
                           └─────────────────────────┘
```

**Key design decisions:**

1. **Primary path:** Dedicated Interconnect (low latency, high bandwidth).
2. **Backup path:** HA VPN (automatic failover if Interconnect fails).
3. **Cloud Router in global dynamic routing mode** — All regions share on-prem routes.
4. **BGP MED/local preference** — Set MED values to prefer Interconnect over VPN.
5. **Two Cloud Routers** (one per region) each with BGP sessions to both on-prem routers.
6. **Monitoring:** Cloud Monitoring alerts on BGP session status and tunnel uptime.
7. **Shared VPC** — Centralize the hybrid connectivity in a host project; service projects attach via Shared VPC.

**For 99.99% SLA:** Use two VLAN attachments in different metropolitan areas for Interconnect, and HA VPN (4 tunnels) as tertiary backup.

---

## Quick Reference: Key GCP Networking Concepts

| Concept | Key Point |
|---|---|
| VPC scope | Global (subnets are regional) |
| Firewall rules | Stateful, applied at VM level |
| HA VPN SLA | 99.99% with 4 tunnels (2 gateways × 2 tunnels each) |
| Interconnect encryption | Not encrypted by default — add MACsec or VPN overlay |
| Cloud Router BGP | ASN range 16550, 64512–65534 (private ASNs) |
| DNS resolution order | Private zone > Forwarding zone > Public zone |
| LB health check IPs | 35.191.0.0/16 and 130.211.0.0/22 |
| Max VPC peerings | 25 per VPC |
| Flow logs sampling | Default 50%, configurable 0–100% |
| PSC vs Peering | PSC allows overlapping CIDRs; Peering does not |

---

*Prepared for TCS Senior GCP Networking Interview — 2026*
