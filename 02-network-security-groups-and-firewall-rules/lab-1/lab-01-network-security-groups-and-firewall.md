# Lab 2: Network Security Groups (NSGs) & Firewall Rules

Welcome to Lab 2! In our previous labs, we discovered that separate subnets are isolated by default unless traffic has a way to cross between them. Now we’re going to open the door between subnets, but with guard rails: **Network Security Groups (NSGs)** and **Firewalls**.

In Azure, an NSG is your primary tool for securing traffic inside a VNet. It sits at either the **Subnet** level or directly on a Virtual Machine's **Network Interface (NIC)**. If you've ever spent three hours troubleshooting why an app server can't talk to an API, only to discover someone put an outbound deny rule with priority 200 that overshadowed their allow rule at 500—this lab will show you why that happened.

We're going to build a realistic traffic inspection scenario right on your Mac mini using a dedicated middleman router container armed with Linux `iptables` and connection tracking (`conntrack`).

---

### What Are We Simulating?

```
 [Frontend Subnet: 10.10.1.0/24]
                │
                │  fe-client (10.10.1.4)
                │
                ▼
 ┌──────────────────────────────────────────────────────────────┐
 │             nsg-firewall Router (10.10.1.254 / 10.10.2.254)   │
 │                                                              │
 │   Stateful Engine (conntrack):                               │
 │   - Established/Related? ───────► ALLOW (Automatic return)   │
 │                                                              │
 │   Simulated NSG Rules (Priority 100 - 4096):                 │
 │   - Rule 100: TCP 80/443 (HTTP/S) ──► ALLOW                  │
 │   - Rule 200: ICMP (Ping)        ──► DROP                    │
 │   - Rule 4096: Default Inbound   ──► DROP ALL                │
 └──────────────────────────────┬───────────────────────────────┘
                                │
                                ▼
                   [Backend Subnet: 10.10.2.0/24]
                                │
                                ├── be-web-api (10.10.2.4) [TCP 80]
                                └── be-db-mock (10.10.2.5) [TCP 5432]

```

---

### Core Concepts Explained

#### 1. The 5-Tuple Match

Every NSG rule in Azure evaluates traffic against 5 distinct attributes known as a **5-tuple**:

1. **Source IP / CIDR** (e.g., `10.10.1.4` or `10.10.1.0/24`)
2. **Source Port** (almost always ephemeral/wildcard `*` like `1024-65535`)
3. **Destination IP / CIDR** (e.g., `10.10.2.4`)
4. **Destination Port** (e.g., `80`, `443`, `22`, `3389`)
5. **Protocol** (`TCP`, `UDP`, or `ICMP`)

If a packet matches all 5 conditions, the rule triggers, and the action (`Allow` or `Deny`) is applied immediately.

#### 2. Rule Priority (100 to 4096)

Rules are processed in strict numerical order:

* **Lower numbers have higher priority** (evaluated first).
* As soon as a packet matches a rule, processing stops—no further rules are evaluated.
* If a packet reaches priority `65000` or `4096` without a match, Azure's built-in default rules kick in:
* `AllowVNetInBound` (Priority 65000)
* `AllowAzureLoadBalancerInBound` (Priority 65001)
* `DenyAllInBound` (Priority 65500)



In this lab, we'll emulate `DenyAllInBound` as our final fallback drop rule.

#### 3. Stateful Filtering vs. Stateless Access Lists

Azure NSGs are **stateful**. This is one of the most important concepts to understand:

* When an inbound packet is allowed (e.g., a client connects to port 80), the firewall records the connection in a state table (`conntrack`).
* When the server sends response data back to the client, the firewall checks the state table, recognizes the packet as `ESTABLISHED`, and automatically lets it pass—even if there is no explicit outbound rule permitting it!
* You do **not** need to open high-numbered ephemeral return ports on an NSG.

---

### Hands-on Step-by-Step Lab

#### Step 1: Set Up Your Project Directory

Open Terminal on your Mac mini and create a dedicated workspace:

```bash
mkdir -p ~/azure-labs/lab2-nsg-firewall
cd ~/azure-labs/lab2-nsg-firewall

```

---

#### Step 2: Create the Lab Definition (`docker-compose.yml`)

Create a `docker-compose.yml` file. We will define:

* Two isolated subnets: `vnet-frontend` and `vnet-backend`.
* A middleman router container (`nsg-firewall`) attached to both subnets with Linux `NET_ADMIN` privileges.
* A client container in the frontend subnet.
* Two service containers in the backend subnet: one running an HTTP web server, and one running a raw TCP mock database.

```yaml
services:
  # The Network Security Group / Firewall Router
  nsg-firewall:
    image: nicolaka/netshoot:latest
    container_name: nsg-firewall
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

  # Frontend Client (e.g., User or Web tier making backend API calls)
  fe-client:
    image: nicolaka/netshoot:latest
    container_name: fe-client
    stdin_open: true
    tty: true
    cap_add:
      - NET_ADMIN
    networks:
      vnet-frontend:
        ipv4_address: 10.10.1.4

  # Backend Web API (Listens on TCP Port 80)
  be-web-api:
    image: nicolaka/netshoot:latest
    container_name: be-web-api
    stdin_open: true
    tty: true
    cap_add:
      - NET_ADMIN
    command: >
      sh -c "while true; do echo -e 'HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\nContent-Length: 25\r\n\r\nHello from Backend API 80' | nc -l -p 80 -q 1; done"
    networks:
      vnet-backend:
        ipv4_address: 10.10.2.4

  # Backend Mock Database (Listens on TCP Port 5432)
  be-db-mock:
    image: nicolaka/netshoot:latest
    container_name: be-db-mock
    stdin_open: true
    tty: true
    cap_add:
      - NET_ADMIN
    command: >
      sh -c "while true; do echo 'DB-CONNECTION-ESTABLISHED' | nc -l -p 5432 -q 1; done"
    networks:
      vnet-backend:
        ipv4_address: 10.10.2.5

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

#### Step 3: Spin Up the Stack

Launch the environment in detached mode:

```bash
docker compose up -d

```

Verify that all 4 containers are running:

```bash
docker compose ps

```

---

#### Step 4: Configure Subnet Routing to Point to the Firewall

Before we write NSG rules, our frontend and backend containers need to know that `nsg-firewall` is their gateway to the opposite subnet.

1. Tell `fe-client` that traffic for `10.10.2.0/24` goes through `10.10.1.254`:

```bash
docker exec -it fe-client ip route add 10.10.2.0/24 via 10.10.1.254

```

2. Tell `be-web-api` and `be-db-mock` that return traffic for `10.10.1.0/24` goes through `10.10.2.254`:

```bash
docker exec -it be-web-api ip route add 10.10.1.0/24 via 10.10.2.254
docker exec -it be-db-mock ip route add 10.10.1.0/24 via 10.10.2.254

```

---

#### Step 5: Implement NSG Rules on the Firewall Router

Now we program the `nsg-firewall` container to mimic an Azure NSG rule table.

1. Access the firewall container:

```bash
docker exec -it nsg-firewall bash

```

2. Flush any default Docker forwarding rules so we start with a clean slate:

```bash
iptables -F FORWARD

```

3. Configure the stateful core and our simulated NSG rules:

```bash
# ------------------------------------------------------------------------------
# Azure Stateful Engine: Automatically allow return packets for active sessions
# ------------------------------------------------------------------------------
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# ------------------------------------------------------------------------------
# Priority 100: Allow HTTP (TCP Port 80) from Frontend to Backend Web API
# ------------------------------------------------------------------------------
iptables -A FORWARD -p tcp -s 10.10.1.0/24 -d 10.10.2.4 --dport 80 -m conntrack --ctstate NEW -j ACCEPT

# ------------------------------------------------------------------------------
# Priority 200: Deny ICMP (Ping)
# ------------------------------------------------------------------------------
iptables -A FORWARD -p icmp -j DROP

# ------------------------------------------------------------------------------
# Priority 4096: Azure Default Inbound Deny (Drop everything else)
# ------------------------------------------------------------------------------
iptables -A FORWARD -j DROP

```

4. View the active rule set:

```bash
iptables -L FORWARD -v -n --line-numbers

```

*Notice how rules are structured in priority order from top to bottom.*

Exit the firewall container:

```bash
exit

```

---

### Step 6: Verification & Experiments

#### Experiment 1: The Classic Cloud Illusion — App Works, Ping Fails

In Azure, junior engineers often ping an internal VM or App Service private IP, see a timeout, and assume the deployment is broken. Let's see why this happens.

1. From `fe-client`, try to ping the backend web API (`10.10.2.4`):

```bash
docker exec -it fe-client ping -c 3 -W 2 10.10.2.4

```

*Result:* 100% packet loss! Priority rule 200 explicitly dropped the ICMP Echo Requests.

2. Now send an HTTP request to the exact same host on port 80:

```bash
docker exec -it fe-client curl -s --connect-timeout 2 http://10.10.2.4

```

*Result:*

```text
Hello from Backend API 80

```

**Takeaway for the team:** Ping tests Layer 3 (ICMP). Web apps live on Layer 7 over Layer 4 (TCP). Never rely on `ping` alone to diagnose cloud application reachability!

---

#### Experiment 2: Testing Azure's Default Inbound Deny Rule

Now try to reach the mock database on port 5432 (`be-db-mock` at `10.10.2.5`):

```bash
docker exec -it fe-client nc -zv -w 3 10.10.2.5 5432

```

*Result:*

```text
nc: connect to 10.10.2.5 port 5432 (tcp) timed out: Operation timed out

```

**Why did it fail?**

* The packet didn't match Rule 100 (wrong destination IP and port).
* It didn't match Rule 200 (not ICMP).
* It fell through to our Priority 4096 catch-all: `DROP`.

---

#### Experiment 3: Inspecting Stateful Connection Tracking (`conntrack`)

Let's look under the hood to see how stateful firewalls remember active streams.

1. Open a second terminal window on your Mac mini and run the connection monitor on `nsg-firewall`:

```bash
docker exec -it nsg-firewall conntrack -E

```

2. In your primary terminal, execute a continuous curl from `fe-client`:

```bash
docker exec -it fe-client curl -s http://10.10.2.4

```

3. Look at the output in the second terminal. You will see an entry similar to:

```text
[NEW] tcp  6 120 SYN_SENT src=10.10.1.4 dst=10.10.2.4 sport=48920 dport=80 ...
[UPDATE] tcp  6 60 SYN_RECV src=10.10.1.4 dst=10.10.2.4 ...
[UPDATE] tcp  6 432000 ESTABLISHED src=10.10.1.4 dst=10.10.2.4 ...

```

**Takeaway for the team:** Notice that the return packets from `10.10.2.4:80` back to `10.10.1.4:48920` were never explicitly permitted by an outbound rule. The firewall permitted them automatically because the connection was flagged as `ESTABLISHED`.

---

#### Experiment 4: Rule Priority Order Matters (The Production Outage Test)

Let's demonstrate how adding an accidental high-priority deny rule can bring down a service.

1. Jump back into `nsg-firewall`:

```bash
docker exec -it nsg-firewall bash

```

2. Insert a broad block rule at the very top (Priority 50), before Rule 100:

```bash
iptables -I FORWARD 2 -s 10.10.1.0/24 -d 10.10.2.0/24 -p tcp -j DROP

```

3. Review the updated table:

```bash
iptables -L FORWARD -v -n --line-numbers

```

*Notice that line 2 is now a blanket TCP DROP, appearing before the port 80 allow rule.*

Exit the container:

```bash
exit

```

4. Now re-run the curl command from `fe-client`:

```bash
docker exec -it fe-client curl -s --connect-timeout 2 http://10.10.2.4

```

*Result:* Timeout!

Even though an allow rule exists, the higher-priority deny rule at line 2 evaluated first, dropping the packet immediately.

---

### Step 7: Clean Up

When you're finished with the lab, shut down the containers and isolated bridge networks:

```bash
docker compose down

```

---

### Quick Cheat-Sheet for the Team

* **Stateful Flow:** NSGs only evaluate the *initial packet* of a new connection. Return traffic is automatically tracked and allowed.
* **First Match Wins:** Always check rule priorities. If a broad rule with priority 200 conflicts with a specific rule with priority 500, the rule with priority 200 takes precedence.
* **Ping $\neq$ Reachability:** ICMP being blocked does not mean your application or database is down. Always test with `curl` or `nc -zv` against the actual application port.