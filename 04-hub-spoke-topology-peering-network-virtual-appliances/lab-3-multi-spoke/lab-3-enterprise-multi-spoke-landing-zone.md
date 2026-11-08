# Lab 4.3: Enterprise Multi-Spoke Landing Zone — Environment Isolation & Shared Services

Welcome to Lab 4.3! In our previous routing labs, we connected two spokes across a central firewall. Now, we’re scaling this up to a pattern found in real-world Azure enterprise landing zones: **Multi-Environment Segmentation with Shared Services**.

In large organizations, workloads aren't lumped into one giant network. They are carved into strict operational boundaries:

* **Non-Prod Spoke (`10.1.0.0/24`)**: Development and testing environments where developers have higher privileges.
* **Prod Spoke (`10.2.0.0/24`)**: Mission-critical production systems holding customer data.
* **Shared-Services Spoke (`10.3.0.0/24`)**: Centralized internal tooling that *both* environments must consume (e.g., self-hosted CI/CD runners, package registries, artifact stores, monitoring agents).
* **Hub VNet (`10.0.0.0/24`)**: The core transit network housing the **Azure Firewall / NVA**.

If a developer's test container in Non-Prod gets compromised, it must **never** be able to pivot laterally into Production. However, both Non-Prod and Prod need to pull artifacts from the Shared Services spoke.

In this lab, we’ll build this complete multi-spoke topology on your Mac mini and program the central NVA with an enterprise security policy matrix.

---

### What Are We Simulating?

```
                                  [Public Internet]
                                          ▲
                                          │ Central Egress via SNAT
                 ┌────────────────────────┴────────────────────────┐
                 │                     HUB VNET                    │
                 │                Subnet: 10.0.0.0/24              │
                 │                                                 │
                 │   ┌─────────────────────────────────────────┐   │
                 │   │      Central NVA / Azure Firewall       │   │
                 │   │           (hub-firewall-nva)            │   │
                 │   │                                         │   │
                 │   │   Policy Matrix:                        │   │
                 │   │   • Non-Prod <─X─> Prod      (DROP)     │   │
                 │   │   • Non-Prod ────► Shared    (ALLOW)    │   │
                 │   │   • Prod     ────► Shared    (ALLOW)    │   │
                 │   │   • All Spokes ──► Internet  (SNAT)     │   │
                 │   └───────▲─────────────▲────────────▲──────┘   │
                 └───────────┼─────────────┼────────────┼──────────┘
                             │             │            │
            ┌────────────────┘             │            └────────────────┐
            │                              │                             │
┌───────────┴─────────────┐   ┌────────────┴───────────┐   ┌─────────────┴───────────┐
│     SPOKE: NON-PROD     │   │      SPOKE: PROD       │   │  SPOKE: SHARED-SERVICES │
│   Subnet: 10.1.0.0/24   │   │   Subnet: 10.2.0.0/24  │   │   Subnet: 10.3.0.0/24   │
│                         │   │                        │   │                         │
│  ┌───────────────────┐  │   │  ┌──────────────────┐  │   │  ┌───────────────────┐  │
│  │ dev-workload      │  │   │  │ prod-db-service  │  │   │  │ shared-artifact-repo│  │
│  │ (10.1.0.4)        │  │   │  │ (10.2.0.4)       │  │   │  │ (10.3.0.4)        │  │
│  └───────────────────┘  │   │  └──────────────────┘  │   │  └───────────────────┘  │
│                         │   │                        │   │                         │
│ UDR: 0.0.0.0/0 ──► NVA  │   │ UDR: 0.0.0.0/0 ──► NVA │   │ UDR: 0.0.0.0/0 ──► NVA  │
└─────────────────────────┘   └────────────────────────┘   └─────────────────────────┘

```

---

### Core Concepts Explained

#### 1. Zero-Trust Lateral Movement Prevention

In legacy data centers, once an attacker breached the perimeter, the internal network was often flat, allowing trivial lateral movement between development and production.

In Azure:

* Spokes are **not peered directly to each other**.
* Every spoke connects strictly to the Hub.
* The Hub Firewall enforces an explicit isolation rule: **Block all traffic between Non-Prod (`10.1.0.0/24`) and Prod (`10.2.0.0/24`)**.

#### 2. The Shared Services Hub-and-Spoke Pattern

You don't want to pay for or maintain duplicate build agents, monitoring endpoints, or internal artifact caches inside every single spoke.

* Placing them in a **Shared Services Spoke** allows centralized administration.
* The Hub NVA acts as the gatekeeper, permitting Dev and Prod to access the artifact server on specific ports while preventing them from seeing each other.

#### 3. Enterprise Firewall Policy Matrix

Instead of ad-hoc firewall rules, enterprise landing zones operate on a strict matrix:

| Source | Destination | Protocol / Port | Action | Purpose |
| --- | --- | --- | --- | --- |
| **Non-Prod (`10.1.0.0/24`)** | **Prod (`10.2.0.0/24`)** | ANY | **DENY** | Zero lateral movement. |
| **Prod (`10.2.0.0/24`)** | **Non-Prod (`10.1.0.0/24`)** | ANY | **DENY** | Zero lateral movement. |
| **Non-Prod (`10.1.0.0/24`)** | **Shared (`10.3.0.4`)** | TCP 8080 | **ALLOW** | Pull build packages/artifacts. |
| **Prod (`10.2.0.0/24`)** | **Shared (`10.3.0.4`)** | TCP 8080 | **ALLOW** | Pull deployment releases. |
| **All Spokes** | **Internet (`0.0.0.0/0`)** | TCP 80, 443 | **ALLOW (SNAT)** | Centralized filtered internet egress. |

---

### Step-by-Step Hands-on Lab

#### Step 1: Set Up Your Project Directory

Open Terminal on your Mac mini and create a dedicated workspace:

```bash
mkdir -p ~/azure-labs/lab4-3-enterprise-spokes
cd ~/azure-labs/lab4-3-enterprise-spokes

```

---

#### Step 2: Create the Lab Definition (`docker-compose.yml`)

Create a `docker-compose.yml` file.

Notice key configuration choices:

* All three spoke networks (`spoke-nonprod`, `spoke-prod`, and `spoke-shared`) are set to **`internal: true`**. None of them have direct internet access or default routes to the Mac host.
* `hub-vnet` is standard (has internet egress via Docker host NAT).
* `hub-firewall-nva` bridges all 4 networks, running with `net.ipv4.ip_forward=1` and `cap_add: NET_ADMIN`.

```yaml
services:
  # ==========================================================
  # Central Hub: Azure Firewall / NVA
  # ==========================================================
  hub-firewall-nva:
    image: nicolaka/netshoot:latest
    container_name: hub-firewall-nva
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    sysctls:
      - net.ipv4.ip_forward=1
    networks:
      hub-vnet:
        ipv4_address: 10.0.0.4
      spoke-nonprod:
        ipv4_address: 10.1.0.254
      spoke-prod:
        ipv4_address: 10.2.0.254
      spoke-shared:
        ipv4_address: 10.3.0.254

  # ==========================================================
  # Spoke 1: Non-Prod Workload (Dev / Test)
  # ==========================================================
  dev-workload:
    image: nicolaka/netshoot:latest
    container_name: dev-workload
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    networks:
      spoke-nonprod:
        ipv4_address: 10.1.0.4

  # ==========================================================
  # Spoke 2: Production Workload (Mission Critical DB)
  # ==========================================================
  prod-db-service:
    image: nicolaka/netshoot:latest
    container_name: prod-db-service
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    command: >
      sh -c "while true; do echo -e 'HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n[PROD DATABASE] Classified customer records' | nc -l -p 5432 -q 1; done"
    networks:
      spoke-prod:
        ipv4_address: 10.2.0.4

  # ==========================================================
  # Spoke 3: Shared Services (Artifact Repo / CI Runner)
  # ==========================================================
  shared-artifact-repo:
    image: nicolaka/netshoot:latest
    container_name: shared-artifact-repo
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    command: >
      sh -c "while true; do echo -e 'HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n[SHARED ARTIFACTS] v2.4.0 Release package ready' | nc -l -p 8080 -q 1; done"
    networks:
      spoke-shared:
        ipv4_address: 10.3.0.4

networks:
  hub-vnet:
    driver: bridge
    ipam:
      config:
        - subnet: 10.0.0.0/24
          gateway: 10.0.0.1

  spoke-nonprod:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 10.1.0.0/24

  spoke-prod:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 10.2.0.0/24

  spoke-shared:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 10.3.0.0/24

```

---

#### Step 3: Launch the Stack

Start up the topology:

```bash
docker compose up -d

```

Verify all 4 containers are running:

```bash
docker compose ps

```

---

#### Step 4: Program Spoke User-Defined Routes (UDRs)

In Azure, you attach a Route Table to each spoke subnet with a default route (`0.0.0.0/0`) pointing to the NVA's private IP.

Configure the default routes inside all three spoke containers:

1. **Non-Prod Spoke:** Route everything to `10.1.0.254`:

```bash
docker exec -it dev-workload ip route add default via 10.1.0.254

```

2. **Prod Spoke:** Route everything to `10.2.0.254`:

```bash
docker exec -it prod-db-service ip route add default via 10.2.0.254

```

3. **Shared Services Spoke:** Route everything to `10.3.0.254`:

```bash
docker exec -it shared-artifact-repo ip route add default via 10.3.0.254

```

Verify that `dev-workload` has its default route set:

```bash
docker exec -it dev-workload ip route show

```

*Expected Output:*

```text
default via 10.1.0.254 dev eth0 
10.1.0.0/24 dev eth0 proto kernel scope link src 10.1.0.4

```

---

#### Step 5: Program the Central Enterprise Policy Matrix on the NVA

Now we program the `hub-firewall-nva` with our security policy matrix.

1. Access the NVA container:

```bash
docker exec -it hub-firewall-nva bash

```

2. Flush all existing rules:

```bash
iptables -F FORWARD
iptables -t nat -F

```

3. Configure the stateful core and the policy rules:

```bash
# ------------------------------------------------------------------------------
# 1. Stateful Connection Tracking (Return packets automatically allowed)
# ------------------------------------------------------------------------------
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# ------------------------------------------------------------------------------
# 2. Strict Environment Isolation: Non-Prod <-> Prod is FORBIDDEN
# ------------------------------------------------------------------------------
iptables -A FORWARD -s 10.1.0.0/24 -d 10.2.0.0/24 -j DROP
iptables -A FORWARD -s 10.2.0.0/24 -d 10.1.0.0/24 -j DROP

# ------------------------------------------------------------------------------
# 3. Shared Services Access: Non-Prod & Prod can access Shared Repo on Port 8080
# ------------------------------------------------------------------------------
iptables -A FORWARD -s 10.1.0.0/24 -d 10.3.0.4 -p tcp --dport 8080 -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -s 10.2.0.0/24 -d 10.3.0.4 -p tcp --dport 8080 -m conntrack --ctstate NEW -j ACCEPT

# ------------------------------------------------------------------------------
# 4. Central Egress: Allow spokes outbound to Internet (via Hub eth0) with SNAT
# ------------------------------------------------------------------------------
iptables -A FORWARD -s 10.0.0.0/8 -o eth0 -j ACCEPT
iptables -t nat -A POSTROUTING -s 10.0.0.0/8 -o eth0 -j MASQUERADE

# ------------------------------------------------------------------------------
# 5. Default Catch-All Deny (Azure Firewall Default Deny)
# ------------------------------------------------------------------------------
iptables -A FORWARD -j DROP

```

4. View the active table in priority order:

```bash
iptables -L FORWARD -v -n --line-numbers

```

Exit the firewall container:

```bash
exit

```

---

### Step 6: Verification & Experiments

#### Experiment 1: Verifying Non-Prod to Prod Isolation (Lateral Movement Blocked)

Imagine an attacker breaches `dev-workload` (`10.1.0.4`). They run a network scan and try to connect directly to the Production database (`10.2.0.4:5432`).

Attempt the connection from `dev-workload`:

```bash
docker exec -it dev-workload nc -zv -w 2 10.2.0.4 5432

```

*Expected Output:*

```text
nc: connect to 10.2.0.4 port 5432 (tcp) timed out: Operation timed out

```

Check the packet counters on the Hub Firewall:

```bash
docker exec -it hub-firewall-nva iptables -L FORWARD -v -n | grep "10.1.0.0/24.*10.2.0.0/24"

```

Notice the packet hit counter incrementing! The central firewall dropped the lateral connection attempt dead in its tracks.

---

#### Experiment 2: Verifying Shared Services Access from Non-Prod

Now test legitimate developer activity: developers in `dev-workload` pulling a build dependency or container image from `shared-artifact-repo` (`10.3.0.4:8080`):

```bash
docker exec -it dev-workload curl -s http://10.3.0.4:8080

```

*Expected Output:*

```text
[SHARED ARTIFACTS] v2.4.0 Release package ready

```

*Result:* Allowed cleanly through the Hub NVA!

---

#### Experiment 3: Verifying Shared Services Access from Production

Production also needs to pull releases from `shared-artifact-repo` (`10.3.0.4:8080`):

```bash
docker exec -it prod-db-service curl -s http://10.3.0.4:8080

```

*Expected Output:*

```text
[SHARED ARTIFACTS] v2.4.0 Release package ready

```

*Result:* Success! Both environments share the common utility without ever being connected to one another.

---

#### Experiment 4: Verifying Unauthorized Port Access to Shared Services

What if someone in Non-Prod tries to probe SSH (Port 22) or another unapproved service on the Shared Services container?

```bash
docker exec -it dev-workload nc -zv -w 2 10.3.0.4 22

```

*Result:*

```text
nc: connect to 10.3.0.4 port 22 (tcp) timed out: Operation timed out

```

**Takeaway:** The firewall rule only permitted `TCP 8080`. Any other port falls through to the final `DROP` rule.

---

#### Experiment 5: Centralized Internet Egress Verification

Finally, verify that all three spokes share the Hub's central egress path to reach the internet.

1. Open a **second terminal window** and run a live capture on the Hub's internet-facing interface (`eth0`):

```bash
docker exec -it hub-firewall-nva tcpdump -n -i eth0 "host 1.1.1.1"

```

2. In your **first terminal window**, ping `1.1.1.1` from `dev-workload`:

```bash
docker exec -it dev-workload ping -c 2 1.1.1.1

```

3. Then ping `1.1.1.1` from `prod-db-service`:

```bash
docker exec -it prod-db-service ping -c 2 1.1.1.1

```

4. Look at the output in **Terminal 2 (tcpdump)**:

```text
IP 10.0.0.4 > 1.1.1.1: ICMP echo request ...
IP 1.1.1.1 > 10.0.0.4: ICMP echo reply ...
IP 10.0.0.4 > 1.1.1.1: ICMP echo request ...
IP 1.1.1.1 > 10.0.0.4: ICMP echo reply ...

```

Both Dev and Prod successfully reached the outside world, but to the outside world, **100% of the traffic originated from `10.0.0.4` (the Hub Firewall)**.

Press `Ctrl+C` in Terminal 2 to stop `tcpdump`.

---

### Step 7: Clean Up

Once the team completes the lab, tear down the environment:

```bash
docker compose down

```

---

### Key Landing Zone Takeaways for the Team

* **Segmentation by Environment:** Never mix Non-Prod and Prod within the same VNet or peered mesh. Isolate them into distinct spoke VNets.
* **The Shared Services Pattern:** Centralize shared tooling (runners, registries, monitoring) in a dedicated Shared Services spoke to avoid configuration duplication and reduce licensing costs.
* **Firewall Policy Matrices Over Ad-Hoc Rules:** Real-world enterprise landing zones manage firewall rules in structured tables matching source/destination environment tiers, enforcing least-privilege access across the entire cloud estate.