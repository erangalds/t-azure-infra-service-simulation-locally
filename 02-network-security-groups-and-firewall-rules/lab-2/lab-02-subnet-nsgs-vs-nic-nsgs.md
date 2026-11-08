# Lab 2.2: The "Double-Door" Trap — Subnet NSGs vs. NIC NSGs

Welcome to Lab 2.2! In Lab 2, we looked at how an individual Network Security Group (NSG) evaluates 5-tuples, prioritizes rules, and tracks connections statefully.

Now, let's tackle one of the most common production support tickets in Azure: **the two-tier NSG evaluation**.

In Azure, an NSG can be bound to an entire **Subnet**, directly to a specific Virtual Machine’s **Network Interface (NIC)**, or **both**. When both exist, traffic doesn't get to pick which one to follow—it must pass through **both doors**. If *either* door slams shut, the packet is dropped.

To make things even more interesting, the order flips depending on whether traffic is coming in or heading out:

```
INBOUND TRAFFIC (Client -> Workload):
  Packet ──► [ Door 1: Subnet NSG ] ──► [ Door 2: NIC NSG ] ──► Target Workload
             (Must be ALLOWED)          (Must be ALLOWED)

OUTBOUND TRAFFIC (Workload -> Internet / Database):
  Packet ──► [ Door 1: NIC NSG ]    ──► [ Door 2: Subnet NSG ] ──► Destination
             (Must be ALLOWED)          (Must be ALLOWED)

```

In this lab, we’ll build a chained two-tier firewall on your Mac mini to see why a permit rule on a workload's NIC means nothing if the surrounding subnet security policy blocks it.

---

### What Are We Simulating?

```
 [Frontend Subnet: 10.10.1.0/24]
               │
               │  fe-client (10.10.1.4)
               │
               ▼
 ┌──────────────────────────────────────────────────────────────┐
 │ DOOR 1: Subnet-Level NSG (nsg-subnet-router)                 │
 │ Subnet Policy: Only standard web ports allowed (80, 443)     │
 │                                                              │
 │ - Rule 100: TCP 80, 443 ───────────────► ALLOW               │
 │ - Rule 4096: Deny All Other Inbound ───► DROP                │
 └──────────────────────────────┬───────────────────────────────┘
                                │
                                ▼  [Transit Network: 10.10.100.0/24]
                                │
 ┌──────────────────────────────┴───────────────────────────────┐
 │ DOOR 2: NIC-Level NSG (nsg-nic-firewall)                     │
 │ Host Policy: Custom application ports permitted              │
 │                                                              │
 │ - Rule 100: TCP 80 (Public App) ───────► ALLOW               │
 │ - Rule 110: TCP 8080 (Admin API) ──────► ALLOW               │
 │ - Rule 4096: Deny All Other Inbound ───► DROP                │
 └──────────────────────────────┬───────────────────────────────┘
                                │
                                ▼
                   [Backend Subnet: 10.10.2.0/24]
                                │
                                ├── be-workload (10.10.2.4)
                                │   ├── Port 80   (Public Web UI)
                                │   └── Port 8080 (Admin API)

```

---

### Step-by-Step Hands-on Lab

#### Step 1: Project Setup

Open Terminal on your Mac mini and create a dedicated folder:

```bash
mkdir -p ~/azure-labs/lab2-2-dual-nsg
cd ~/azure-labs/lab2-2-dual-nsg

```

---

#### Step 2: Create the Lab Definition (`docker-compose.yml`)

Create a `docker-compose.yml` file.

To create a genuine two-stage evaluation path without complex Linux namespaces, we use an intermediate transit bridge (`vnet-transit`):

1. `fe-client` sends traffic into `nsg-subnet` (Door 1).
2. `nsg-subnet` forwards traffic across the transit network to `nsg-nic` (Door 2).
3. `nsg-nic` forwards traffic to the final `be-workload`.

```yaml
services:
  # Door 1: Subnet-level NSG
  nsg-subnet:
    image: nicolaka/netshoot:latest
    container_name: nsg-subnet
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    sysctls:
      - net.ipv4.ip_forward=1
    networks:
      vnet-frontend:
        ipv4_address: 10.10.1.254
      vnet-transit:
        ipv4_address: 10.10.100.1

  # Door 2: NIC-level NSG
  nsg-nic:
    image: nicolaka/netshoot:latest
    container_name: nsg-nic
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    sysctls:
      - net.ipv4.ip_forward=1
    networks:
      vnet-transit:
        ipv4_address: 10.10.100.2
      vnet-backend:
        ipv4_address: 10.10.2.254

  # Frontend Client
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

  # Target Workload running both Port 80 and Port 8080 listeners
  be-workload:
    image: nicolaka/netshoot:latest
    container_name: be-workload
    stdin_open: true
    tty: true
    cap_add:
      - NET_ADMIN
    command: >
      sh -c "
        while true; do echo -e 'HTTP/1.1 200 OK\r\n\r\nWEB-UI-PORT-80-OK' | nc -l -p 80 -q 1; done &
        while true; do echo -e 'HTTP/1.1 200 OK\r\n\r\nADMIN-API-PORT-8080-OK' | nc -l -p 8080 -q 1; done
      "
    networks:
      vnet-backend:
        ipv4_address: 10.10.2.4

networks:
  vnet-frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 10.10.1.0/24
          gateway: 10.10.1.1
  vnet-transit:
    driver: bridge
    ipam:
      config:
        - subnet: 10.10.100.0/24
          gateway: 10.10.100.254
  vnet-backend:
    driver: bridge
    ipam:
      config:
        - subnet: 10.10.2.0/24
          gateway: 10.10.2.1

```

---

#### Step 3: Spin Up the Stack

Launch the lab:

```bash
docker compose up -d

```

Verify that all 4 containers are up:

```bash
docker compose ps

```

---

#### Step 4: Configure the End-to-End Hop Routes

We need each hop along the path to know where to pass packets:

1. `fe-client` routes backend traffic to `nsg-subnet` (`10.10.1.254`):

```bash
docker exec -it fe-client ip route add 10.10.2.0/24 via 10.10.1.254

```

2. `nsg-subnet` routes backend traffic across the transit bridge to `nsg-nic` (`10.10.100.2`):

```bash
docker exec -it nsg-subnet ip route add 10.10.2.0/24 via 10.10.100.2

```

3. `nsg-nic` routes return traffic for the frontend back across the transit bridge to `nsg-subnet` (`10.10.100.1`):

```bash
docker exec -it nsg-nic ip route add 10.10.1.0/24 via 10.10.100.1

```

4. `be-workload` routes all frontend return packets to `nsg-nic` (`10.10.2.254`):

```bash
docker exec -it be-workload ip route add 10.10.1.0/24 via 10.10.2.254

```

---

#### Step 5: Program the Two-Tier NSG Rules

Now we configure each firewall tier.

##### 1. Configure Door 1: Subnet-Level NSG (`nsg-subnet`)

The platform team manages the subnet. Their governance policy says: *Only standard web traffic (ports 80 and 443) may enter this subnet.*

```bash
docker exec -it nsg-subnet bash -c "
  iptables -F FORWARD
  # Stateful return traffic
  iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
  # Rule 100: Allow HTTP & HTTPS only
  iptables -A FORWARD -p tcp -m multiport --dports 80,443 -m conntrack --ctstate NEW -j ACCEPT
  # Rule 4096: Deny all other inbound
  iptables -A FORWARD -j DROP
"

```

##### 2. Configure Door 2: NIC-Level NSG (`nsg-nic`)

The application team configures their VM's network interface directly. They want users to reach the Web UI (80) and need their administrators to hit the Admin API (8080):

```bash
docker exec -it nsg-nic bash -c "
  iptables -F FORWARD
  # Stateful return traffic
  iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
  # Rule 100: Allow Web App on Port 80
  iptables -A FORWARD -p tcp --dport 80 -m conntrack --ctstate NEW -j ACCEPT
  # Rule 110: Allow Admin API on Port 8080
  iptables -A FORWARD -p tcp --dport 8080 -m conntrack --ctstate NEW -j ACCEPT
  # Rule 4096: Deny all other inbound
  iptables -A FORWARD -j DROP
"

```

---

### Step 6: Verification & Experiments

#### Experiment 1: The Passing Flow (Door 1 Allows, Door 2 Allows)

Test access to the Web UI on Port 80:

```bash
docker exec -it fe-client curl -s --connect-timeout 2 http://10.10.2.4:80

```

*Expected Output:*

```text
WEB-UI-PORT-80-OK

```

**Why it worked:**

* Subnet NSG checked Port 80 $\rightarrow$ Matched Rule 100 $\rightarrow$ **ALLOWED**.
* NIC NSG checked Port 80 $\rightarrow$ Matched Rule 100 $\rightarrow$ **ALLOWED**.
* Both doors opened, connection succeeded.

---

#### Experiment 2: The "Shadowed Allow" Trap (Door 2 Allows, Door 1 Blocks)

Now test access to the Admin API on Port 8080:

```bash
docker exec -it fe-client curl -s --connect-timeout 2 http://10.10.2.4:8080

```

*Expected Output:*

```text
curl: (28) Failed to connect to 10.10.2.4 port 8080 after ... ms: Couldn't connect to server

```

**What happened here?**

* The developer looks at the NIC NSG and says: *"I have an explicit ALLOW rule for port 8080 right here!"*
* But inspect the packet counters on the Subnet NSG:

```bash
docker exec -it nsg-subnet iptables -L FORWARD -v -n

```

Notice the dropped packets counter on the final `DROP` rule.

The packet arrived at **Door 1 (Subnet NSG)** first. Because Port 8080 wasn't permitted in the Subnet NSG, the Subnet NSG dropped it on the floor before the NIC NSG ever had a chance to evaluate the packet.

---

#### Experiment 3: Resolving the Shadowed Rule

To fix this in Azure, you must update the Subnet NSG policy to open the door at the subnet boundary as well.

Add Port 8080 to the Subnet NSG:

```bash
docker exec -it nsg-subnet bash -c "
  # Insert an allow rule for 8080 right before the DROP rule
  iptables -I FORWARD 3 -p tcp --dport 8080 -m conntrack --ctstate NEW -j ACCEPT
"

```

Now try the curl from `fe-client` again:

```bash
docker exec -it fe-client curl -s --connect-timeout 2 http://10.10.2.4:8080

```

*Expected Output:*

```text
ADMIN-API-PORT-8080-OK

```

Both doors are now aligned, and traffic flows smoothly.

---

#### Experiment 4: The Outbound Flip

Let's see what happens when the workload initiates outbound traffic. In Azure, outbound traffic hits the **NIC NSG first**, and then the **Subnet NSG second**.

Let's simulate a strict data exfiltration rule on the **NIC NSG** that forbids the backend workload from making outbound connections to the frontend subnet:

1. Add a blocking outbound rule on `nsg-nic`:

```bash
docker exec -it nsg-nic bash -c "
  # Insert a drop rule for outbound connections originating from the backend to 10.10.1.0/24
  iptables -I FORWARD 1 -s 10.10.2.0/24 -d 10.10.1.0/24 -m conntrack --ctstate NEW -j DROP
"

```

2. From `be-workload`, try to initiate a new connection to `fe-client`:

```bash
docker exec -it be-workload nc -zv -w 2 10.10.1.4 80

```

*Result:* Blocked at Door 1 (NIC NSG).

3. Now, test if inbound requests from `fe-client` to `be-workload:80` still work:

```bash
docker exec -it fe-client curl -s --connect-timeout 2 http://10.10.2.4:80

```

*Result:*

```text
WEB-UI-PORT-80-OK

```

**Why did inbound still work?**
Because of stateful connection tracking (`ESTABLISHED,RELATED`). The outbound rule only matches `NEW` connections initiated by the backend. The return packets for an inbound request are treated as part of the existing session and are permitted automatically.

---

### Step 7: Clean Up

Tear down the lab environment when finished:

```bash
docker compose down

```

---

### Key Takeaways for the Team

1. **Both Must Agree (Logical AND):** For inbound traffic, `Subnet NSG == Allow` **AND** `NIC NSG == Allow`. A permit on one does not override a deny on the other.
2. **Order of Operations Matters:**
* **Inbound:** Subnet NSG processes first $\rightarrow$ NIC NSG processes second.
* **Outbound:** NIC NSG processes first $\rightarrow$ Subnet NSG processes second.


3. **Azure Best Practice:** Whenever possible, attach NSGs at the **Subnet level** and avoid NIC-level NSGs entirely. Subnet-only NSGs provide uniform governance, avoid shadowed rules, and dramatically simplify troubleshooting.