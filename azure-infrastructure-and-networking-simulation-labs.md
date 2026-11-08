# Azure Infrastructure & Networking Simulation Labs: Curriculum Overview

This hands-on curriculum simulates core Azure enterprise infrastructure, software-defined networking, security boundaries, and PaaS architectures entirely on a local development workstation using Docker, `netshoot`, custom routing engines, and official Microsoft container runtimes.

Each module translates abstract cloud primitives—such as Network Security Groups, Route Tables, Load Balancers, Gateways, and Private Endpoints—into tangible, inspectable networking states without requiring live Azure subscriptions or incurring cloud costs.

---

## Curriculum Structure at a Glance

| Phase / Module | Core Azure Focus Area | Primary Technical Mechanism |
| --- | --- | --- |
| **Section 1** | Virtual Networks & Subnetting | CIDR math, VNet boundaries, address conflicts, sub-allocation |
| **Section 2** | Security Policies & NSG Logic | Stateful 5-tuple filtering, Subnet vs. NIC NSGs, ASGs, Service Tags |
| **Section 3** | Ingress & Traffic Distribution | Azure Load Balancer (L4) vs. Application Gateway (L7), Health Probes |
| **Section 4** | Enterprise Topology & Transit | Hub-and-Spoke, Non-transitive Peering, UDRs, Forced Tunneling, IPSec |
| **Section 5** | PaaS Integration & Private Link | Private Endpoints, Split-Horizon DNS, SMB Shares, Queues, Key Vault |

---

## Detailed Lab Breakdown

### Section 1: Subnet Boundaries, CIDR Planning & Overlap Resolution

* **Lab 1.1: VNet Subnet Boundaries & CIDR Math**
* **Objective:** Understand how Azure reserves IPs within subnets and how boundary masks enforce network containment.
* **Key Concept:** Azure automatically reserves 5 IP addresses per subnet (`.0` Network, `.1` Default Gateway, `.2`/`.3` Azure DNS mapping, and `.255` Broadcast).
* **Hands-on Simulation:** Build isolated bridge networks simulating `/24` and `/28` allocations. Verify direct host unreachable states across distinct CIDRs without routing.


* **Lab 1.2: CIDR Overlap Conflict Resolution**
* **Objective:** Experience the operational consequences of overlapping IP spaces when connecting enterprise environments.
* **Key Concept:** Azure strictly forbids VNet peering between address spaces that overlap. Connecting overlapping networks requires intermediate SNAT/DNAT translation pools.
* **Hands-on Simulation:** Deploy duplicate address spaces (`10.0.1.0/24`) connected to a transit router. Implement `iptables` NAT translation rules to allow two identical IP spaces to communicate without IP collisions.



---

### Section 2: Security Policies, Network Security Groups & Firewalls

* **Lab 2.1: NSG Policy Evaluation & Stateful Connection Tracking**
* **Objective:** Master priority-ordered rule processing and understand stateful return flow mechanics.
* **Key Concept:** NSG rules evaluate in numeric priority order (lowest number first). Once a rule matches, evaluation stops. Outbound replies to allowed inbound connections are automatically permitted via stateful connection tracking (`conntrack`).
* **Hands-on Simulation:** Build an intermediate router with `iptables` rules emulating priorities 100 through 4096. Test how high-priority allow rules override low-priority deny rules, and verify that return traffic bypasses outbound rule evaluation.


* **Lab 2.2: The "Double-Door" Trap — Subnet NSGs vs. NIC NSGs**
* **Objective:** Troubleshoot multi-tier firewall filtering and shadowed rule configurations.
* **Key Concept:** Inbound traffic must pass the Subnet NSG first, then the NIC NSG. Outbound traffic hits the NIC NSG first, then the Subnet NSG. Both must independently permit traffic.
* **Hands-on Simulation:** Chain two firewall hops (`nsg-subnet` and `nsg-nic`). Open port `8080` on the NIC firewall while leaving it closed on the Subnet firewall to reproduce shadowed allow failures.


* **Lab 2.3: Application Security Groups (ASGs) — Decoupled Security Policies**
* **Objective:** Group dynamic workloads by business role rather than hardcoded IP addresses or broad CIDR blocks.
* **Key Concept:** ASGs allow writing policies such as `Allow ASG-Web to ASG-Database on Port 5432` across workloads living inside the same subnet.
* **Hands-on Simulation:** Use Linux `ipset` inside a router container to create hashed dynamic groups (`asg_web`, `asg_db`). Add and remove container IPs dynamically to show instant policy application without modifying underlying firewall rules.


* **Lab 2.4: Service Tags & Outbound Internet Lockdown**
* **Objective:** Lock down data exfiltration without breaking core operating system and cloud management dependencies.
* **Key Concept:** A naive `DenyAllOutBound` rule breaks NTP clock sync, package managers, and Azure agent telemetry. Service Tags provide Microsoft-managed IP prefix groups to permit essential platform egress.
* **Hands-on Simulation:** Enforce an outbound `0.0.0.0/0` drop policy. Use `ipset` to build emulated Service Tags (`tag_ntp_servers`, `tag_azure_storage`), proving that platform health can be preserved while blocking general data exfiltration.



---

### Section 3: Ingress Architectures — Load Balancers vs. Application Gateways

* **Lab 3.1: Layer 4 (Azure Load Balancer) vs. Layer 7 (Application Gateway)**
* **Objective:** Contrast transport-layer packet distribution with intelligent application-layer reverse proxying.
* **Key Concept:** Layer 4 balancers distribute raw TCP/UDP streams based on an IP 5-tuple hash without terminating connections. Layer 7 gateways terminate HTTP sessions, inspect URI paths/headers, and proxy upstream requests.
* **Hands-on Simulation:** Deploy HAProxy (TCP mode on port 8080) alongside Nginx (HTTP mode on port 9090). Demonstrate that the L4 balancer cannot route `/api` paths to specific backends, while the L7 gateway dynamically routes `/api/*` to microservices and `/*` to web instances.


* **Lab 3.2: Advanced Ingress — Health Probes, Sticky Sessions & Multi-Site Routing**
* **Objective:** Diagnose silent outages caused by naive health probes and manage stateful user sessions.
* **Key Concept:** L4 TCP health probes consider a node healthy as long as the socket accepts handshakes, even if the application returns `HTTP 500`. L7 probes inspect HTTP response status codes. L7 gateways also support cookie-based affinity and multi-site host header routing on a single IP.
* **Hands-on Simulation:** Run an application returning `HTTP 500`. Observe the L4 balancer continue to direct client traffic to the failed instance while the L7 gateway detects the failure and evicts the node. Test cookie-based affinity and host-header routing (`portal.local` vs. `billing.local`).



---

### Section 4: Enterprise Topologies, Transit Routing & Hybrid Connectivity

* **Lab 4.1: Hub-and-Spoke Topology, Peering & Network Virtual Appliances (NVAs)**
* **Objective:** Build standard enterprise landing zone topologies and overcome non-transitive peering limitations.
* **Key Concept:** VNet peering is strictly non-transitive (Spoke A cannot reach Spoke B through the Hub automatically). Routing between spokes requires a central NVA and User-Defined Routes (UDRs).
* **Hands-on Simulation:** Connect two isolated spoke networks to a central NVA. Apply UDRs inside the spokes pointing to the NVA as the next hop. Use `tcpdump` inside the hub firewall to capture live cross-spoke traffic traversing the inspection engine.


* **Lab 4.2: Forced Tunneling & Centralized Egress via Hub NVA**
* **Objective:** Enforce centralized outbound internet inspection across all spoke environments.
* **Key Concept:** A UDR with `0.0.0.0/0` pointing to a `Virtual Appliance` overrides Azure's default internet system route, forcing all outbound web calls through the firewall for inspection and Source Network Address Translation (SNAT).
* **Hands-on Simulation:** Mark spoke subnets as completely isolated (`internal: true`). Set a `0.0.0.0/0` default route to the NVA, configure `iptables` MASQUERADE on the hub, and verify via packet capture that spoke traffic adopts the firewall's egress IP while unapproved outbound domains are dropped.


* **Lab 4.3: Enterprise Multi-Spoke Landing Zone — Environment Isolation & Shared Services**
* **Objective:** Implement zero-trust network segmentation between Non-Production and Production environments while sharing centralized tooling.
* **Key Concept:** Dev and Prod spokes must never communicate directly. Shared services (CI runners, artifact repos, domain controllers) are placed in a dedicated spoke accessible by both environments through firewall rules.
* **Hands-on Simulation:** Deploy four networks (`hub`, `spoke-nonprod`, `spoke-prod`, `spoke-shared`). Program a central firewall matrix: block lateral movement between Dev and Prod, permit Dev/Prod access to the shared repository on port 8080, and route all outbound traffic through the central hub.


* **Lab 4.4: The Asymmetric Routing Outage — The Silent Cloud Killer**
* **Objective:** Diagnose and resolve the most common failure mode in new Hub-and-Spoke deployments.
* **Key Concept:** Stateful firewalls require symmetric traffic flows. If the outbound packet crosses the firewall but the return packet bypasses the firewall due to a missing return UDR, the firewall drops the connection due to an incomplete TCP handshake.
* **Hands-on Simulation:** Intentionally delete the return route on the destination database subnet. Observe client connection timeouts while watching `SYN` arrivals and unreplied states in `conntrack` and `tcpdump`. Restore the symmetric route to resolve the outage.


* **Lab 4.5: Gateway Transit — Simulating On-Premises Hybrid Connectivity**
* **Objective:** Connect on-premises corporate infrastructure to cloud spokes without deploying redundant VPN gateways in every VNet.
* **Key Concept:** The peering flags **"Allow Gateway Transit"** (on the Hub) and **"Use Remote Gateway"** (on the Spoke) propagate on-premises routes to spoke route tables automatically.
* **Hands-on Simulation:** Connect an on-premises corporate subnet (`192.168.1.0/24`) to a Hub Gateway container. Route traffic between the cloud spoke and the on-prem payroll server through the Hub Gateway.


* **Lab 4.6: Site-to-Site IPSec VPN with StrongSwan — Cryptographic Tunneling**
* **Objective:** Inspect encrypted hybrid tunnels connecting on-premises data centers to Azure.
* **Key Concept:** IPSec uses IKEv2 (UDP 500/4500) for control-plane authentication and parameter negotiation, and ESP (IP Protocol 50) to encapsulate and encrypt all data-plane payloads.
* **Hands-on Simulation:** Deploy StrongSwan daemons on both sides of a simulated public WAN bridge. Perform an IKEv2 Pre-Shared Key handshake, establish Security Associations (SAs), and run `tcpdump` to prove that traffic on the public wire appears strictly as encrypted ESP packets.


* **Lab 4.7: Application-Layer Encryption — End-to-End TLS & The Firewall Blind Spot**
* **Objective:** Understand how zero-trust application encryption impacts middlebox network inspection.
* **Key Concept:** End-to-end TLS (HTTPS) protects payload confidentiality across all network hops, preventing standard Layer 4/7 firewalls from inspecting URL paths, queries, or payloads without dedicated TLS termination and intermediate certificate authority (CA) proxies.
* **Hands-on Simulation:** Deploy an Nginx server terminating TLS 1.3 with an X.509 certificate on port 8443. Route traffic across the Hub NVA and use `tcpdump` to observe that the firewall can only see Server Name Indication (SNI) metadata, remaining blind to encrypted application payloads and URIs.



---

### Section 5: PaaS Integration, Private Endpoints & Secure Cloud Services

* **Lab 5.1: PaaS Integration — Azure Functions, Azurite Storage, Azure SQL Edge & Private Link**
* **Objective:** Connect serverless application runtimes to cloud storage and managed databases over private IP addresses.
* **Key Concept:** Azure Private Link projects a PaaS resource onto a private network interface (NIC) inside a subnet, allowing public access to be completely disabled. Serverless compute reaches these endpoints via Regional VNet Integration.
* **Hands-on Simulation:** Deploy official Microsoft images for **Azurite** and **Azure SQL Edge** with zero host ports exposed. Deploy an Azure Function container that receives HTTP orders, uploads a JSON blob to Azurite, and inserts an audit record into SQL Edge over private IP routes.


* **Lab 5.2: Split-Horizon DNS & Azure Private DNS Zones**
* **Objective:** Solve DNS resolution failures when accessing Private Endpoints using standard Azure FQDNs.
* **Key Concept:** Applications must connect to standard FQDNs (`*.blob.core.windows.net`) to preserve TLS validation and SDK signing. Azure uses a CNAME to `*.privatelink.blob.core.windows.net`. Without an Azure Private DNS Zone linked to the VNet, queries resolve to public IPs and fail with `403 Forbidden`.
* **Hands-on Simulation:** Deploy CoreDNS simulating the Azure recursive resolver (`168.63.129.16`). Demonstrate how an unlinked zone returns a public IP and produces an HTTP 403 error, then link the authoritative private zone file to resolve the CNAME to the Private Endpoint IP (`10.2.0.10`).


* **Lab 5.3: Azure Files, Managed SMB Shares & The Port 445 ISP Block**
* **Objective:** Manage shared multi-mount file storage and understand external ISP protocol restrictions.
* **Key Concept:** Azure Files exposes managed SMB 3.0 shares for concurrent multi-writer workloads (`ReadWriteMany`). Because residential and mobile ISPs block outbound TCP port 445, mounting Azure Files over the public internet fails, making Private Endpoints or VPNs mandatory.
* **Hands-on Simulation:** Deploy a Samba SMB 3.0 server on the private data subnet. Mount the share concurrently across two compute nodes, verify real-time file synchronization and POSIX file locking, and simulate an ISP port 445 block to demonstrate why external connections fail without private connectivity.


* **Lab 5.4: Event-Driven Serverless — Azurite Storage Queues & Poison Message Dead-Lettering**
* **Objective:** Implement asynchronous decoupling and prevent malformed messages from stalling processing pipelines.
* **Key Concept:** Serverless queue workers poll messages using visibility timeouts. If a message repeatedly crashes the consumer, the runtime checks the `DequeueCount` property; once it hits `maxDequeueCount` (default 5), the message is routed to a `<queue>-poison` dead-letter queue.
* **Hands-on Simulation:** Run Azurite Queue Storage. Build an asynchronous worker simulating an Azure Functions `QueueTrigger`. Publish valid orders alongside corrupted JSON payloads, observe the 5-attempt retry cycle, and verify automatic quarantine into the poison queue while the main pipeline continues processing.


* **Lab 5.5: Secret-Free PaaS — Key Vault Emulation & Runtime Credential Injection**
* **Objective:** Eliminate hardcoded credentials from container definitions and application configuration files.
* **Key Concept:** Compute workloads authenticate using Managed Identity tokens to retrieve secrets from Azure Key Vault over Private Link at runtime, enabling seamless secret rotation without redeploying code.
* **Hands-on Simulation:** Deploy an Azure Key Vault REST API emulator on a private IP. Launch an Azure Function with zero database passwords in its environment, fetch credentials into memory on demand, and rotate the secret inside the vault to demonstrate live credential adoption without restarting the container.



---

## Technical Prerequisites & Setup

All labs are designed to run on macOS (Apple Silicon / Intel) or Linux workstations equipped with:

* **Docker Desktop** or **Colima / Docker Engine**
* **Docker Compose v2**
* Standard terminal utilities: `curl`, `jq`, `openssl`, `nc`, `dig` / `drill`

To run any individual lab, navigate to its respective directory and execute:

```bash
docker compose up -d

```