# Lab 1.2: CIDR Overlap & Address Space Conflicts

Welcome to Part 2 of our IP addressing and subnetting series! In the first lab, we carved out clean, isolated subnets. Now, let’s explore what happens when IP planning goes wrong—specifically, **CIDR overlap**.

In Azure, this is one of the most frustrating architectural traps a team can fall into. You spend hours setting up Virtual Networks across multiple subscriptions or regions, and the moment you click **"Add Peering"** or try to hook up an on-premises **Site-to-Site VPN Gateway**, Azure slaps you with an error:

> *"Failed to add peering. Address space 10.10.0.0/16 overlaps with existing peered virtual network address space 10.10.1.0/24."*

In this lab, we’ll see firsthand how IP address spaces collide, why routers and software-defined networks throw their hands up, and how to spot supernet/subnet overlaps before they hit production.

---

### What Exactly Is a CIDR Overlap?

Think of an IP address like a physical house address:

```
VNet A Address Space: 10.10.0.0/23  (Covers 10.10.0.0  to 10.10.1.255)
               ┌────────────────────────────────────────────────────────┐
               │ 10.10.0.0/24 Subnet      │ 10.10.1.0/24 Subnet         │
               └──────────────────────────┴─────────────────────────────┘
                                                      ▲
                                                      │  COLLISION!
                                                      ▼
               ┌────────────────────────────────────────────────────────┐
               │ 10.10.1.0/24 Subnet                                    │
               └────────────────────────────────────────────────────────┘
VNet B Address Space: 10.10.1.0/24  (Covers 10.10.1.0  to 10.10.1.255)

```

If **VNet A** owns `10.10.0.0/23`, its address range spans from `10.10.0.0` all the way through `10.10.1.255`.

If someone else provisions **VNet B** with `10.10.1.0/24`, VNet B’s addresses exist entirely *inside* VNet A’s yard. If a server at `10.10.0.5` tries to send a packet to `10.10.1.4`, who should get it? A server in VNet A, or the server in VNet B?

Because routers use **longest prefix match**, overlapping networks create routing ambiguity. Azure’s SDN blocks this at creation time. Docker's network IPAM engine does the exact same thing on your Mac mini.

---

### Hands-on Lab Overview

```
                               Mac mini (Local Docker Daemon)
                               
   Scenario A: The Hard Block                    Scenario B: The Non-Overlapping Fix
   ──────────────────────────                    ───────────────────────────────────
   Try creating two overlapping subnets:         Carve out contiguous, non-overlapping blocks:
   
   Bridge 1: 10.10.0.0/23 (512 IPs)              Bridge 1: 10.10.0.0/24 (Subnet 1)
       │                                             │
       ├── Covers 10.10.0.0 - 10.10.1.255            ├── Covers 10.10.0.0 - 10.10.0.255
       │                                             │
   Bridge 2: 10.10.1.0/24 (256 IPs)              Bridge 2: 10.10.1.0/24 (Subnet 2)
       │                                             │
       └── Overlaps with Bridge 1!                   └── Clean boundary! Starts at .1.0
           [Docker IPAM Rejection]                       [Clean Creation & Isolated Routing]

```

---

### Step-by-Step Instructions

#### Step 1: Create a Clean Lab Directory

Jump into your terminal on your Mac mini:

```bash
mkdir -p ~/azure-labs/lab1-cidr-overlap
cd ~/azure-labs/lab1-cidr-overlap

```

---

#### Step 2: Triggering the Overlap Conflict (The "Azure Peering Rejection" Simulation)

Let's test what happens when we ask our system to create two overlapping networks.

1. Create the first network, representing an enterprise corporate VNet (`corp-vnet`) carved as a `/23` supernet:

```bash
docker network create \
  --driver bridge \
  --subnet 10.10.0.0/23 \
  --gateway 10.10.0.1 \
  corp-vnet

```

*Expected output:* You’ll get a long network hash indicating it was successfully created.

2. Inspect the range it allocated:

```bash
docker network inspect corp-vnet --format '{{range .IPAM.Config}}{{.Subnet}} (Gateway: {{.Gateway}}){{end}}'

```

*Output:* `10.10.0.0/23 (Gateway: 10.10.0.1)`

> **Math check for the team:** A `/23` mask leaves $32 - 23 = 9$ host bits. $2^9 = 512$ total addresses. This block spans from `10.10.0.0` to `10.10.1.255`.

3. Now, pretend another team in another department spins up their own app VNet (`app-vnet`) using `10.10.1.0/24`:

```bash
docker network create \
  --driver bridge \
  --subnet 10.10.1.0/24 \
  --gateway 10.10.1.1 \
  app-vnet

```

*Expected output:*

```text
Error response from daemon: Pool overlaps with other one on this address space

```

**What just happened?**
The Docker IP Address Management (IPAM) driver threw an error matching the behavior of Azure Resource Manager. Because `10.10.1.0/24` is mathematically encompassed by `10.10.0.0/23`, the host kernel cannot determine which bridge interface should own routing for the `10.10.1.x` address space.

---

#### Step 3: Triggering a Compose-Time CIDR Failure

Let's see how this looks in Infrastructure as Code (IaC) like Docker Compose or Azure Bicep/Terraform.

Create a broken `docker-compose.yml` file:

```yaml
services:
  app-corp:
    image: nicolaka/netshoot:latest
    container_name: app-corp
    stdin_open: true
    tty: true
    networks:
      corp-net:
        ipv4_address: 10.10.0.4

  app-finance:
    image: nicolaka/netshoot:latest
    container_name: app-finance
    stdin_open: true
    tty: true
    networks:
      finance-net:
        ipv4_address: 10.10.1.4

networks:
  corp-net:
    driver: bridge
    ipam:
      config:
        - subnet: 10.10.0.0/23
  finance-net:
    driver: bridge
    ipam:
      config:
        - subnet: 10.10.1.0/24

```

Clean up our manual network from Step 2 first:

```bash
docker network rm corp-vnet

```

Now try spinning up the Compose stack:

```bash
docker compose up -d

```

*Expected output:*

```text
[!] Network lab1-cidr-overlap_finance-net  Error
failed to create network lab1-cidr-overlap_finance-net: Error response from daemon: Pool overlaps with other one on this address space

```

**Teaching point:** When teams write Terraform or Bicep modules in silos, this exact error happens when team A and team B try to merge their VNets into a central Hub.

---

#### Step 4: The Fix — Subnet Planning & Boundary Calculation

To fix this, we need contiguous, non-overlapping address allocations. Let’s divide our address space cleanly using Variable Length Subnet Masking (VLSM).

Suppose our overall parent organization pool is `10.10.0.0/23` (512 addresses). Instead of using the whole `/23` as one flat block, we divide it into two non-overlapping `/24` subnets:

| Subnet Name | CIDR Block | Usable IP Range (Traditional) | Azure Usable Range (First host is .4) |
| --- | --- | --- | --- |
| **Corp Subnet** | `10.10.0.0/24` | `10.10.0.1` - `10.10.0.254` | `10.10.0.4` - `10.10.0.254` |
| **Finance Subnet** | `10.10.1.0/24` | `10.10.1.1` - `10.10.1.254` | `10.10.1.4` - `10.10.1.254` |

Update your `docker-compose.yml` with the resolved subnets:

```yaml
services:
  app-corp:
    image: nicolaka/netshoot:latest
    container_name: app-corp
    stdin_open: true
    tty: true
    networks:
      corp-net:
        ipv4_address: 10.10.0.4

  app-finance:
    image: nicolaka/netshoot:latest
    container_name: app-finance
    stdin_open: true
    tty: true
    networks:
      finance-net:
        ipv4_address: 10.10.1.4

networks:
  corp-net:
    driver: bridge
    ipam:
      config:
        - subnet: 10.10.0.0/24
          gateway: 10.10.0.1
  finance-net:
    driver: bridge
    ipam:
      config:
        - subnet: 10.10.1.0/24
          gateway: 10.10.1.1

```

Now deploy the fixed configuration:

```bash
docker compose up -d

```

*Expected output:*

```text
 ✔ Network lab1-cidr-overlap_corp-net     Created
 ✔ Network lab1-cidr-overlap_finance-net  Created
 ✔ Container app-corp                     Started
 ✔ Container app-finance                  Started

```

---

#### Step 5: Verify the Isolation & Route Independence

Check the routing table of `app-corp`:

```bash
docker exec -it app-corp ip route show

```

*Output:*

```text
default via 10.10.0.1 dev eth0 
10.10.0.0/24 dev eth0 proto kernel scope link src 10.10.0.4

```

Check the routing table of `app-finance`:

```bash
docker exec -it app-finance ip route show

```

*Output:*

```text
default via 10.10.1.1 dev eth0 
10.10.1.0/24 dev eth0 proto kernel scope link src 10.10.1.4

```

Each container has an unambiguous, mathematically distinct subnet, meaning each subnet can now cleanly route, attach to firewalls, or participate in peering without any address collision.

---

### Step 6: Lab Clean-up

Once everyone on the team has run through the conflict and the fix, tear down the lab:

```bash
docker compose down

```

---

### Quick Cheat-Sheet for the Team

* **Overlap Rule:** You can never peer two Azure VNets if any part of their address prefixes overlap, even by a single IP.
* **On-Prem / VPN Rule:** If your Azure VNet address space overlaps with your on-premises office network, an IPSec VPN tunnel will cause routing drops or require complex Network Address Translation (NAT).
* **The Golden Rule of Cloud Subnetting:** Always carve from largest block to smallest block, and document the binary boundaries before deploying any infrastructure.