# Lab 4: Hub-and-Spoke Topology, Peering & Network Virtual Appliances (NVAs)

Welcome to Lab 4! We've tackled subnets, security policies, and ingress traffic. Now it's time to build the gold standard of enterprise cloud architecture: **The Hub-and-Spoke Topology**.

Almost every medium-to-large organization in Azure organizes its resources into a Hub-and-Spoke layout. Instead of letting every application environment connect to every database or on-prem VPN in a tangled web, we route through a central **Hub VNet**. That hub houses our centralized shared services: VPN/ExpressRoute gateways, shared DNS resolvers, and, most importantly, a central inspection firewall or **Network Virtual Appliance (NVA)** like Azure Firewall, Fortinet, or Palo Alto.

In this lab, we'll uncover the single biggest stumbling block for engineers transitioning to enterprise cloud networking: **non-transitive peering** and how **User-Defined Routes (UDRs)** fix it.

---

### What Are We Simulating?

```
                     ┌──────────────────────────────────────────────┐
                     │            Hub VNet (10.0.0.0/24)            │
                     │                                              │
                     │   ┌──────────────────────────────────────┐   │
                     │   │     Central NVA / Azure Firewall     │   │
                     │   │         (hub-firewall-nva)           │   │
                     │   │                                      │   │
                     │   │   • IP Forwarding Enabled            │   │
                     │   │   • tcpdump Live Packet Inspection   │   │
                     │   └───────▲──────────────────────▲───────┘   │
                     │           │                      │           │
                     └───────────┼──────────────────────┼───────────┘
                                 │                      │
             Peering Link 1      │                      │      Peering Link 2
            (UDR: Next-Hop NVA)  │                      │     (UDR: Next-Hop NVA)
                                 │                      │
       ┌─────────────────────────┴────┐            ┌────┴─────────────────────────┐
       │   Spoke Compute (10.1.0.0/24)│            │    Spoke Data (10.2.0.0/24)  │
       │                              │            │                              │
       │   ┌──────────────────────┐   │            │   ┌──────────────────────┐   │
       │   │   spoke-web-app      │   │     X      │   │   spoke-db-service   │   │
       │   │   (10.1.0.4)         │───┼────────────┼──►│   (10.2.0.4)         │   │
       │   └──────────────────────┘   │            │   └──────────────────────┘   │
       │                              │            │                              │
       │  UDR to 10.2.0.0/24 -> NVA   │            │  UDR to 10.1.0.0/24 -> NVA   │
       └──────────────────────────────┘            └──────────────────────────────┘
                         NO DIRECT SPOKE-TO-SPOKE PEERING

```

---

### Core Concepts Explained

#### 1. Why Hub-and-Spoke?

Instead of peering every spoke directly with every other spoke ($N \times (N-1) / 2$ connections, which quickly becomes unmanageable), we connect each spoke only to the central hub.

* **Centralized Security:** All egress to the internet, cross-spoke traffic, and on-premises traffic passes through a single point of inspection (the NVA).
* **Cost Efficiency:** You only pay for one firewall deployment rather than putting individual firewalls into every application subnet.
* **Separation of Duties:** The platform/networking team manages the Hub, while dev teams independently own their Spokes.

#### 2. The Golden Rule: Azure Peering is Non-Transitive!

If **Spoke A** is peered to the **Hub**, and **Spoke B** is peered to the **Hub**, **Spoke A CANNOT talk to Spoke B by default**.

* Peering does not daisy-chain routing.
* The Hub will not act as a router unless you explicitly place a routing engine inside the Hub and force the spokes to use it.

```
Spoke A ──(Peered)──► Hub ◄──(Peered)── Spoke B
  │                                       ▲
  └───────────── X BLOCKED X ─────────────┘
      (Azure does not bridge them automatically)

```

#### 3. User-Defined Routes (UDRs) & Next Hops

How do we get Spoke A to talk to Spoke B across the hub? With an **Azure Route Table (UDR)**.
By default, Azure system routes say: *"If traffic is destined for another VNet, send it across the peering directly."* But when spokes aren't peered directly, Azure has no route and drops the packet.

We create a **User-Defined Route** attached to the Spoke subnet:

* **Address Prefix:** `10.2.0.0/24` (Spoke Data)
* **Next Hop Type:** `Virtual Appliance`
* **Next Hop IP:** `10.0.0.4` (The Private IP of the central firewall in the Hub)

This forces the packets to leave Spoke Compute, enter the Hub firewall, get inspected, and then get forwarded down into Spoke Data.

---

### Step-by-Step Hands-on Lab

#### Step 1: Set Up Your Project Directory

Fire up your terminal on your Mac mini and create a dedicated workspace:

```bash
mkdir -p ~/azure-labs/lab4-hub-spoke
cd ~/azure-labs/lab4-hub-spoke

```

---

#### Step 2: Create the Lab Definition (`docker-compose.yml`)

Create a `docker-compose.yml` file.

We will define three isolated subnets:

1. `hub-vnet` (`10.0.0.0/24`): The transit hub.
2. `spoke-compute` (`10.1.0.0/24`): Front-facing application workloads.
3. `spoke-data` (`10.2.0.0/24`): Back-end data workloads.

Notice that our central `hub-firewall-nva` container has three network interfaces attached (one in the Hub, and one simulating the peering leg into each spoke), with kernel IP forwarding turned on (`net.ipv4.ip_forward=1`).

```yaml
services:
  # ==========================================================
  # Central Hub: Network Virtual Appliance / Firewall (NVA)
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
      spoke-compute:
        ipv4_address: 10.1.0.254
      spoke-data:
        ipv4_address: 10.2.0.254

  # ==========================================================
  # Spoke 1: Application Web Server (spoke-compute)
  # ==========================================================
  spoke-web-app:
    image: nicolaka/netshoot:latest
    container_name: spoke-web-app
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    networks:
      spoke-compute:
        ipv4_address: 10.1.0.4

  # ==========================================================
  # Spoke 2: Database / Internal Service (spoke-data)
  # ==========================================================
  spoke-db-service:
    image: nicolaka/netshoot:latest
    container_name: spoke-db-service
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    command: >
      sh -c "while true; do echo -e 'HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n[DATA SERVICE] Secure SQL/Data payload accessed successfully via Hub NVA' | nc -l -p 5432 -q 1; done"
    networks:
      spoke-data:
        ipv4_address: 10.2.0.4

networks:
  # Hub Network
  hub-vnet:
    driver: bridge
    ipam:
      config:
        - subnet: 10.0.0.0/24
          gateway: 10.0.0.1

  # Spoke 1: Compute Network
  spoke-compute:
    driver: bridge
    ipam:
      config:
        - subnet: 10.1.0.0/24
          gateway: 10.1.0.1

  # Spoke 2: Data Network
  spoke-data:
    driver: bridge
    ipam:
      config:
        - subnet: 10.2.0.0/24
          gateway: 10.2.0.1

```

---

#### Step 3: Launch the Topology

Spin up the lab:

```bash
docker compose up -d

```

Verify that all three containers are healthy and running:

```bash
docker compose ps

```

---

### Step 4: Verification & Experiments

#### Experiment 1: The Non-Transitive Peering Wall (Default Behavior)

Let's see what happens out of the box when `spoke-web-app` (`10.1.0.4`) tries to talk directly to `spoke-db-service` (`10.2.0.4`).

Try pinging the database from the web container:

```bash
docker exec -it spoke-web-app ping -c 3 -W 2 10.2.0.4

```

*Result:* 100% packet loss!

Check `spoke-web-app`'s local route table:

```bash
docker exec -it spoke-web-app ip route show

```

*Output:*

```text
default via 10.1.0.1 dev eth0 
10.1.0.0/24 dev eth0 proto kernel scope link src 10.1.0.4

```

**Why it failed:** `spoke-web-app` only knows about its local subnet `10.1.0.0/24`. For anything else, it sends packets to the Docker default bridge gateway `10.1.0.1`, which drops them because `spoke-data` is an isolated bridge. The spokes have no idea the Hub NVA exists! This mirrors Azure when spokes are attached to a Hub without route tables.

---

#### Experiment 2: Programming User-Defined Routes (UDRs)

In Azure, you associate a **Route Table** with each spoke subnet, defining a next-hop route to the firewall appliance. Let's configure the equivalent routes on both spokes pointing to our Hub NVA interface:

1. Add the UDR on **Spoke Compute**: Route all traffic for `10.2.0.0/24` to the NVA (`10.1.0.254`):

```bash
docker exec -it spoke-web-app ip route add 10.2.0.0/24 via 10.1.0.254

```

2. Add the return UDR on **Spoke Data**: Route all return traffic for `10.1.0.0/24` to the NVA (`10.2.0.254`):

```bash
docker exec -it spoke-db-service ip route add 10.1.0.0/24 via 10.2.0.254

```

3. Verify the updated route table on `spoke-web-app`:

```bash
docker exec -it spoke-web-app ip route show

```

*Output:*

```text
default via 10.1.0.1 dev eth0 
10.1.0.0/24 dev eth0 proto kernel scope link src 10.1.0.4 
10.2.0.0/24 via 10.1.0.254 dev eth0

```

Notice the explicit next-hop route!

---

#### Experiment 3: Live Packet Sniffing on the Hub NVA (`tcpdump`)

This is the moment that proves your Hub NVA is actively inspecting cross-spoke traffic.

1. Open a **second terminal window** on your Mac mini and attach to `hub-firewall-nva` to run a live packet capture on all interfaces:

```bash
docker exec -it hub-firewall-nva tcpdump -n -i any "host 10.1.0.4 or host 10.2.0.4"

```

2. In your **first terminal window**, send an HTTP request from `spoke-web-app` to `spoke-db-service` on port 5432:

```bash
docker exec -it spoke-web-app curl -s http://10.2.0.4:5432

```

*Expected output in Terminal 1:*

```text
[DATA SERVICE] Secure SQL/Data payload accessed successfully via Hub NVA

```

3. Look at the output in **Terminal 2 (the tcpdump window)**:

```text
IP 10.1.0.4.48392 > 10.2.0.4.5432: Flags [S], seq 12345678, ...
IP 10.2.0.4.5432 > 10.1.0.4.48392: Flags [S.], seq 87654321, ack 12345679, ...
IP 10.1.0.4.48392 > 10.2.0.4.5432: Flags [P.], seq 1:85, ack 1, ...
IP 10.2.0.4.5432 > 10.1.0.4.48392: Flags [P.], seq 1:120, ack 85, ...

```

**What this proves:**

* The request left `spoke-web-app`.
* It arrived at the Hub NVA on interface `eth1` (`10.1.0.254`).
* The Hub NVA inspected the packet, checked its kernel forwarding table, and forwarded it out of interface `eth2` (`10.2.0.254`) down to `spoke-db-service`.
* The return packet took the exact same symmetric path back through the Hub NVA.

Press `Ctrl+C` in Terminal 2 to stop `tcpdump`.

---

#### Experiment 4: Simulating Azure Firewall Layer 7 Application Rules

Now that traffic flows through our Hub NVA, we can enforce centralized security policies.

Let's simulate an Azure Firewall rule that **allows database queries (port 5432)**, but **blocks unauthorized shell access (SSH on port 22)** between spokes.

1. Access the `hub-firewall-nva` shell:

```bash
docker exec -it hub-firewall-nva bash

```

2. Flush any default forwarding and set up a stateful firewall policy:

```bash
iptables -F FORWARD

# Allow returning stateful traffic
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# ALLOW Rule: Database traffic (Port 5432) from Spoke Compute to Spoke Data
iptables -A FORWARD -p tcp -s 10.1.0.0/24 -d 10.2.0.4 --dport 5432 -m conntrack --ctstate NEW -j ACCEPT

# DENY Rule: Drop all other cross-spoke traffic (Azure Default Deny)
iptables -A FORWARD -s 10.1.0.0/24 -d 10.2.0.0/24 -j DROP

```

3. Exit the firewall container:

```bash
exit

```

4. Test allowed traffic: Verify port 5432 still works:

```bash
docker exec -it spoke-web-app curl -s http://10.2.0.4:5432

```

*Result:* Success!

5. Test unauthorized traffic: Try scanning or connecting to SSH port 22 on the database:

```bash
docker exec -it spoke-web-app nc -zv -w 2 10.2.0.4 22

```

*Result:*

```text
nc: connect to 10.2.0.4 port 22 (tcp) timed out: Operation timed out

```

The Hub Firewall intercepted the unauthorized connection attempt and dropped it before it ever touched the database subnet.

---

### Step 5: Clean Up

When you've finished the lab, bring down the containers and isolated bridge networks:

```bash
docker compose down

```

---

### Key Architecture Takeaways for the Team

* **Peering is Never Transitive:** Connecting Spoke A to Hub and Spoke B to Hub will *not* enable Spoke A to talk to Spoke B.
* **UDRs Are Mandatory:** To enable spoke-to-spoke communication through a hub, you must create an Azure Route Table with a route pointing to the NVA's private IP as the `Virtual Appliance` next hop.
* **Symmetric Routing Is Critical:** Both the forward path (Spoke A $\rightarrow$ Hub $\rightarrow$ Spoke B) and the return path (Spoke B $\rightarrow$ Hub $\rightarrow$ Spoke A) must point to the firewall. If one side bypasses the NVA, the stateful inspection engine drops the connection (asymmetric routing).