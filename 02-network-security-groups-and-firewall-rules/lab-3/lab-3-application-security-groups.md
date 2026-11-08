# Lab 2.3: Application Security Groups (ASGs) — Decoupling Security Policies from IP Addresses

Welcome to Lab 2.3! In our previous security labs, we wrote Network Security Group rules by hardcoding IP addresses (`10.10.1.4`) or entire CIDR blocks (`10.10.1.0/24`).

While this works for simple static topologies, in real cloud environments with autoscaling, container orchestration, and continuous deployments, **hardcoding IP addresses into security policies is an operational nightmare**:

* If a VM scales out, you have to remember to update every NSG rule.
* If a subnet hosts multiple roles (e.g., both public web nodes and administrative workers living inside the same subnet), CIDR-based rules either grant too much access or require hundreds of individual host IP rules.

To solve this, Azure provides **Application Security Groups (ASGs)**.

An ASG is an abstraction tag you attach to a workload’s Network Interface (NIC). Instead of writing rules like:

> *"Allow 10.10.1.14 to 10.10.2.25 on Port 5432"*

You write rules based on workload functions:

> *"Allow **ASG-WebTier** to **ASG-DatabaseTier** on Port 5432"*

In this lab, we'll build this behavior on your Mac mini. We'll use the Linux kernel’s native high-performance IP grouping engine—**`ipset`**—inside our router container to simulate Azure Application Security Groups, showing how workloads gain or lose access instantly without touching a single firewall rule.

---

### What Are We Simulating?

```
                      [Shared Subnet: 10.10.1.0/24]
 ┌──────────────────────────────────────────────────────────────────────┐
 │                                                                      │
 │  ┌──────────────────────┐              ┌──────────────────────┐      │
 │  │      fe-web-01       │              │      fe-web-02       │      │
 │  │     (10.10.1.11)     │              │     (10.10.1.12)     │      │
 │  │  Member of: asg_web  │              │  Member of: asg_web  │      │
 │  └──────────┬───────────┘              └──────────┬───────────┘      │
 │             │                                     │                  │
 │             └──────────────────┬──────────────────┘                  │
 │                                │ (Port 5432 Traffic)                 │
 │                                ▼                                     │
 │                     ┌──────────────────────┐                         │
 │                     │     rogue-worker     │                         │
 │                     │     (10.10.1.99)     │                         │
 │                     │  NOT IN asg_web!     │                         │
 │                     └──────────┬───────────┘                         │
 └────────────────────────────────┼─────────────────────────────────────┘
                                  │
                                  ▼
 ┌──────────────────────────────────────────────────────────────────────┐
 │              Central Router / Firewall (nsg-asg-router)              │
 │                                                                      │
 │   IPSet Definitions:                                                 │
 │   • asg_web ──────► [ 10.10.1.11, 10.10.1.12 ]                      │
 │   • asg_db  ──────► [ 10.10.2.50 ]                                  │
 │                                                                      │
 │   Static Rule:                                                       │
 │   • match-set asg_web SRC + match-set asg_db DST on 5432 ──► ALLOW   │
 │   • Everything else                                      ──► DROP    │
 └────────────────────────────────┬─────────────────────────────────────┘
                                  │
                                  ▼
                      [Database Subnet: 10.10.2.0/24]
 ┌──────────────────────────────────────────────────────────────────────┐
 │                                                                      │
 │                     ┌──────────────────────┐                         │
 │                     │      be-db-prod      │                         │
 │                     │     (10.10.2.50)     │                         │
 │                     │  Member of: asg_db   │                         │
 │                     │  Port 5432 (SQL DB)  │                         │
 │                     └──────────────────────┘                         │
 └──────────────────────────────────────────────────────────────────────┘

```

---

### Core Concepts Explained

#### 1. The Architectural Value of ASGs

* **Subnet Independence:** Workloads inside the same subnet can have completely different security permissions without needing to carve out tiny subnets for every single microservice.
* **Autoscaling Friendly:** When Virtual Machine Scale Sets (VMSS) spin up 10 new instances, Azure simply stamps the new NICs with the target ASG tag. The NSG rule automatically applies to the new IPs immediately.
* **Human-Readable Policies:** Rules reflect business architecture (`ASG-PaymentProcessors` $\rightarrow$ `ASG-PaymentDatabase`), dramatically reducing configuration errors during audits.

#### 2. The Linux Underlying Engine: `ipset`

In Linux, standard `iptables` rules evaluate sequentially. If you have 500 IPs, the kernel evaluates up to 500 lines for every single packet.

`ipset` stores IP addresses and subnets in hashed indexing data structures:

* Lookup time is **$O(1)$** (constant time), regardless of whether the set contains 2 IPs or 100,000 IPs.
* An `iptables` rule points to the set by name using the `-m set --match-set <set_name>` flag.
* You can add or remove members from an `ipset` dynamically while traffic is flowing, with zero downtime and without modifying the `iptables` rule list.

---

### Step-by-Step Hands-on Lab

#### Step 1: Set Up Your Project Directory

Open Terminal on your Mac mini and create a dedicated workspace:

```bash
mkdir -p ~/azure-labs/lab2-3-asg
cd ~/azure-labs/lab2-3-asg

```

---

#### Step 2: Create the Lab Definition (`docker-compose.yml`)

Create a `docker-compose.yml` file.

Notice that in the frontend subnet (`10.10.1.0/24`), we deploy three workloads side-by-side:

* Two approved web servers: `fe-web-01` (`10.10.1.11`) and `fe-web-02` (`10.10.1.12`).
* An unapproved rogue worker: `rogue-worker` (`10.10.1.99`).

All three share the same subnet and default gateway. In standard CIDR-based NSGs, if you allowed `10.10.1.0/24` to the database, the rogue worker would have access too. With ASGs, we'll selectively allow only the web servers.

```yaml
services:
  # ==========================================================
  # Central Firewall / Router simulating Azure NSG + ASG Engine
  # ==========================================================
  nsg-asg-router:
    image: nicolaka/netshoot:latest
    container_name: nsg-asg-router
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    sysctls:
      - net.ipv4.ip_forward=1
    networks:
      vnet-frontend:
        ipv4_address: 10.10.1.254
      vnet-backend:
        ipv4_address: 10.10.2.254

  # ==========================================================
  # Frontend Workload 1 (Web Tier Instance 1)
  # ==========================================================
  fe-web-01:
    image: nicolaka/netshoot:latest
    container_name: fe-web-01
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    networks:
      vnet-frontend:
        ipv4_address: 10.10.1.11

  # ==========================================================
  # Frontend Workload 2 (Web Tier Instance 2 - Autoscaled)
  # ==========================================================
  fe-web-02:
    image: nicolaka/netshoot:latest
    container_name: fe-web-02
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    networks:
      vnet-frontend:
        ipv4_address: 10.10.1.12

  # ==========================================================
  # Frontend Workload 3 (Unapproved Worker in same subnet)
  # ==========================================================
  rogue-worker:
    image: nicolaka/netshoot:latest
    container_name: rogue-worker
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    networks:
      vnet-frontend:
        ipv4_address: 10.10.1.99

  # ==========================================================
  # Backend Database Workload (Target Database Service)
  # ==========================================================
  be-db-prod:
    image: nicolaka/netshoot:latest
    container_name: be-db-prod
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    command: >
      sh -c "while true; do echo -e 'HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n[SQL DATABASE] Query processed successfully for ASG Member!' | nc -l -p 5432 -q 1; done"
    networks:
      vnet-backend:
        ipv4_address: 10.10.2.50

networks:
  vnet-frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 10.10.1.0/24
          gateway: 10.10.1.1
  vnet-backend:
    driver: bridge
    ipam:
      config:
        - subnet: 10.10.2.0/24
          gateway: 10.10.2.1

```

---

#### Step 3: Launch the Stack

Spin up the lab:

```bash
docker compose up -d

```

Verify that all five containers are up:

```bash
docker compose ps

```

---

#### Step 4: Configure Gateway Routes Pointing to the Router

Tell the workloads on both subnets to route cross-subnet traffic through `nsg-asg-router`:

1. Frontend workloads route `10.10.2.0/24` to `10.10.1.254`:

```bash
docker exec -it fe-web-01 ip route add 10.10.2.0/24 via 10.10.1.254
docker exec -it fe-web-02 ip route add 10.10.2.0/24 via 10.10.1.254
docker exec -it rogue-worker ip route add 10.10.2.0/24 via 10.10.1.254

```

2. Database workload routes return traffic for `10.10.1.0/24` to `10.10.2.254`:

```bash
docker exec -it be-db-prod ip route add 10.10.1.0/24 via 10.10.2.254

```

---

#### Step 5: Program the ASGs & Security Policy on the Router

Now we enter `nsg-asg-router` to set up our Application Security Groups using `ipset` and create our single, role-based firewall rule.

1. Access the router container:

```bash
docker exec -it nsg-asg-router bash

```

2. Flush any default forwarding rules:

```bash
iptables -F FORWARD

```

3. Create two **Application Security Groups** using `ipset`:

```bash
# Create ASG for Web Tier
ipset create asg_web hash:ip

# Create ASG for Database Tier
ipset create asg_db hash:ip

```

4. Populate the initial memberships:

* Tag `fe-web-01` (`10.10.1.11`) as a member of `asg_web`.
* Tag `be-db-prod` (`10.10.2.50`) as a member of `asg_db`.
* *(Note: We deliberately leave `fe-web-02` and `rogue-worker` out of the group for now).*

```bash
ipset add asg_web 10.10.1.11
ipset add asg_db 10.10.2.50

```

5. Write the **Single ASG-Based Security Rule**:

```bash
# Allow established/related return traffic
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# ASG Rule: Allow any member of asg_web to reach any member of asg_db on Port 5432
iptables -A FORWARD -m set --match-set asg_web src -m set --match-set asg_db dst -p tcp --dport 5432 -m conntrack --ctstate NEW -j ACCEPT

# Azure Default Deny (Drop all other cross-subnet traffic)
iptables -A FORWARD -j DROP

```

6. Inspect the configured `ipset` groups and firewall rules:

```bash
ipset list
iptables -L FORWARD -v -n --line-numbers

```

*Notice line 2 of the FORWARD chain:*
`match-set asg_web src match-set asg_db dst tcp dpt:5432`

There are **zero hardcoded IP addresses** in that firewall rule!

Exit the router:

```bash
exit

```

---

### Step 6: Verification & Experiments

#### Experiment 1: The Tagged Workload Succeeds

Test connectivity from `fe-web-01` (`10.10.1.11`), which is tagged in `asg_web`:

```bash
docker exec -it fe-web-01 curl -s --connect-timeout 2 http://10.10.2.50:5432

```

*Expected Output:*

```text
[SQL DATABASE] Query processed successfully for ASG Member!

```

The packet matched `asg_web` $\rightarrow$ `asg_db` on port 5432 and passed through.

---

#### Experiment 2: The Rogue Workload in the Same Subnet is Blocked

Now, run the exact same query from `rogue-worker` (`10.10.1.99`), which lives inside the same frontend subnet:

```bash
docker exec -it rogue-worker curl -s --connect-timeout 2 http://10.10.2.50:5432

```

*Expected Output:*

```text
curl: (28) Failed to connect to 10.10.2.50 port 5432 after 2000 ms: Couldn't connect to server

```

**Why did this fail?**
Even though `rogue-worker` is in the same CIDR subnet (`10.10.1.0/24`) and uses the same default gateway, its IP address (`10.10.1.99`) is **not** present in the `asg_web` set. It hits the catch-all `DROP` rule.

This demonstrates how ASGs enforce least privilege *within* a shared subnet.

---

#### Experiment 3: Simulating Autoscaling (Dynamic Membership Update)

Now imagine traffic spikes and an Azure Virtual Machine Scale Set provisions `fe-web-02` (`10.10.1.12`).

1. Before tagging, verify `fe-web-02` cannot reach the database:

```bash
docker exec -it fe-web-02 curl -s --connect-timeout 2 http://10.10.2.50:5432

```

*Result:* Connection timed out.

2. Now, simulate Azure tagging the new NIC with the `asg_web` group:

```bash
docker exec -it nsg-asg-router ipset add asg_web 10.10.1.12

```

3. Re-run the query from `fe-web-02` **without touching or reloading any firewall rules**:

```bash
docker exec -it fe-web-02 curl -s --connect-timeout 2 http://10.10.2.50:5432

```

*Expected Output:*

```text
[SQL DATABASE] Query processed successfully for ASG Member!

```

**The big takeaway:** In production, you don't submit a change ticket to add an IP to 10 different NSGs across multiple resource groups. You assign the VM’s NIC to the ASG, and every firewall rule referencing that group immediately recognizes the new instance.

---

#### Experiment 4: Revoking Access via ASG Removal

What happens if an instance is decommissioned or quarantined? You simply detach the ASG.

1. Remove `fe-web-01` from `asg_web`:

```bash
docker exec -it nsg-asg-router ipset del asg_web 10.10.1.11

```

2. Immediately test access from `fe-web-01`:

```bash
docker exec -it fe-web-01 curl -s --connect-timeout 2 http://10.10.2.50:5432

```

*Result:* Connection timed out immediately. Access is revoked in real time.

---

### Step 7: Clean Up

When you've finished the exercises, shut down the lab environment:

```bash
docker compose down

```

---

### Key Azure Best Practices for the Team

* **Role-Based Over IP-Based:** Always prefer Application Security Groups over hardcoded IP addresses when writing NSG rules for web tiers, business logic tiers, and database tiers.
* **VNet Scope Restriction:** In Azure, all NICs assigned to an ASG must reside within the **same Virtual Network** as the NSG evaluating the rule. ASGs cannot span across peered VNets (cross-VNet rules still require IP addresses or Service Tags).
* **Combining with Subnet NSGs:** You can use ASGs inside an NSG attached to a subnet. This allows a broad subnet (e.g., `10.10.1.0/24`) to host web servers, background workers, and caching instances while maintaining micro-segmentation between them.