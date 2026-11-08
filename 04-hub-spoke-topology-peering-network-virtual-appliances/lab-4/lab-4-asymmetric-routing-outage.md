# Lab 4.4: The Asymmetric Routing Outage — The Silent Cloud Killer

Welcome to Lab 4.4! If there is one networking issue that consumes days of debugging time during an enterprise Azure migration, it is **Asymmetric Routing**.

Here is the setup: You spin up two spokes, peer them with a central Hub, configure an Azure Firewall, and attach a shiny Route Table (UDR) to **Spoke A** so all traffic routes through the firewall. You initiate a connection from Spoke A to Spoke B.

The application in Spoke B receives the packet, processes it, and sends a reply. Yet on Spoke A, the user or client sits staring at a spinning wheel until it times out:

```text
curl: (28) Failed to connect to 10.2.0.4 port 80: Connection timed out

```

You check the firewall logs—the inbound packet was allowed! You check the destination server—the application is running! Why is the connection failing?

Because cloud firewalls are **stateful inspection engines**. If a firewall observes the initial outbound request (`SYN`) but never sees the corresponding return response (`SYN-ACK`), the session stays half-open and collapses.

In this lab, we will intentionally break the return path on a destination service to recreate an asymmetric routing failure on your Mac mini, inspect the packets live with `tcpdump`, and apply the enterprise fix.

---

### What Are We Simulating?

```
                      THE ASYMMETRIC ROUTING TRAP
                      
                     ┌──────────────────────────────────────────────┐
                     │            Hub VNet (10.0.0.0/24)            │
                     │                                              │
                     │   ┌──────────────────────────────────────┐   │
                     │   │     Central NVA / Azure Firewall     │   │
                     │   │         (hub-firewall-nva)           │   │
                     │   │                                      │   │
                     │   │   [State Table: SYN_SENT]            │   │
                     │   │   "Waiting for SYN-ACK..."           │   │
                     │   │   Never receives return packet!      │   │
                     │   │   Connection times out.              │   │
                     │   └───────▲──────────────────────────────┘   │
                     │           │ (Hop 1)                          │
                     └───────────┼──────────────────────────────────┘
                                 │
                   Forward Path  │                     Broken Return Path
                   via UDR       │                     (Direct / Bypassing NVA)
                                 │
       ┌─────────────────────────┴────┐            ┌──────────────────────────────┐
       │   Spoke Compute (10.1.0.0/24)│            │    Spoke Data (10.2.0.0/24)  │
       │                              │            │                              │
       │   ┌──────────────────────┐   │            │   ┌──────────────────────┐   │
       │   │   spoke-web-app      │   │            │   │   spoke-db-service   │   │
       │   │   (10.1.0.4)         │   │            │   │   (10.2.0.4)         │   │
       │   │                      │   │            │   │                      │   │
       │   │   Sends: TCP SYN     │───┼────────────┼──►│   Receives: TCP SYN  │   │
       │   │   Awaits: SYN-ACK    │   │            │   │   Sends: TCP SYN-ACK │   │
       │   │   (HANGS FOREVER)    │   │            │   │   (To default gateway│   │
       │   │                      │   │            │   │    instead of NVA!)  │   │
       │   └──────────────────────┘   │            │   └──────────┬───────────┘   │
       │                              │            │              │ (Hop 2)       │
       │  UDR to 10.2.0.0/24 -> NVA   │            │              ▼               │
       └──────────────────────────────┘            └────── X BYPASS DROP X ───────┘

```

---

### Core Concepts Explained

#### 1. The TCP Three-Way Handshake

Before any HTTP payload, SQL query, or API call moves over a TCP socket, the client and server must establish a connection via a 3-step sequence:

1. **Client $\rightarrow$ Server (`SYN`):** "Let's open a connection."
2. **Server $\rightarrow$ Client (`SYN-ACK`):** "I acknowledge your request and agree to connect."
3. **Client $\rightarrow$ Server (`ACK`):** "Acknowledged, connection established."

```
Client                              Firewall                              Server
  │                                    │                                    │
  ├─── TCP SYN (Seq=0) ───────────────►├─── TCP SYN (Seq=0) ───────────────►│
  │    [Forward Path via NVA]          │    (Firewall logs: SYN_SENT)       │
  │                                    │                                    │
  │                                    │    [Broken Return Path]            │
  │◄── TCP SYN-ACK (Seq=0, Ack=1) ─────┴────────────────────────────────────┤
  │    (Bypasses Firewall! Sent via direct bridge/peering gateway)          │
  │                                                                         │
  ├─── TCP ACK (Seq=1, Ack=1) ────────► (DROPPED by Firewall!)              │
  │                                     Firewall: "I never saw a SYN-ACK    │
  │                                     for this flow, this is an invalid   │
  │                                     out-of-state packet!"               │

```

#### 2. Stateful Firewalls Demand Flow Symmetry

Azure Firewall and third-party NVAs are **stateful**. They do not treat incoming and outgoing packets in isolation:

* When the `SYN` packet passes through the firewall, a connection tracking entry is opened in state `SYN_SENT`.
* The firewall **expects** the corresponding `SYN-ACK` packet to return through the *exact same firewall instance* to validate the sequence numbers and transition the connection to `ESTABLISHED`.
* If Spoke B has a missing UDR, Spoke B sends the return packet out of its own default gateway, bypassing the firewall.
* Result: The firewall never transitions the session to `ESTABLISHED`, drops subsequent client packets, or the client drops the uncoordinated return packet due to mismatched socket states. The connection hangs until it hits an application-level timeout.

---

### Step-by-Step Hands-on Lab

#### Step 1: Set Up Your Project Directory

Open Terminal on your Mac mini and create a dedicated workspace:

```bash
mkdir -p ~/azure-labs/lab4-4-asymmetric-routing
cd ~/azure-labs/lab4-4-asymmetric-routing

```

---

#### Step 2: Create the Lab Definition (`docker-compose.yml`)

Create a `docker-compose.yml` file.

To demonstrate asymmetric routing cleanly:

1. `spoke-compute` (`10.1.0.0/24`) and `spoke-data` (`10.2.0.0/24`) are created as isolated subnets.
2. `hub-firewall-nva` bridges both networks, running stateful connection tracking with `iptables` and packet forwarding enabled.
3. `spoke-web-app` runs a client workload.
4. `spoke-db-service` runs a web API/database service listening on port 80.

```yaml
services:
  # ==========================================================
  # Central Hub: Stateful NVA / Azure Firewall
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
      spoke-compute:
        ipv4_address: 10.1.0.254
      spoke-data:
        ipv4_address: 10.2.0.254

  # ==========================================================
  # Spoke 1: Compute Workload (Client)
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
  # Spoke 2: Data Workload (Target Service on Port 80)
  # ==========================================================
  spoke-db-service:
    image: nicolaka/netshoot:latest
    container_name: spoke-db-service
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    command: >
      sh -c "while true; do echo -e 'HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n[DATA SUCCESS] Connected to database backend!' | nc -l -p 80 -q 1; done"
    networks:
      spoke-data:
        ipv4_address: 10.2.0.4

networks:
  spoke-compute:
    driver: bridge
    ipam:
      config:
        - subnet: 10.1.0.0/24
          gateway: 10.1.0.1

  spoke-data:
    driver: bridge
    ipam:
      config:
        - subnet: 10.2.0.0/24
          gateway: 10.2.0.1

```

---

#### Step 3: Launch the Stack

Start up the containers:

```bash
docker compose up -d

```

Verify that all three containers are healthy:

```bash
docker compose ps

```

---

#### Step 4: Program the Forward UDR (Spoke Compute $\rightarrow$ NVA)

Configure the User-Defined Route on **Spoke Compute**: Tell `spoke-web-app` that all traffic bound for Spoke Data (`10.2.0.0/24`) must go through the Hub Firewall (`10.1.0.254`):

```bash
docker exec -it spoke-web-app ip route add 10.2.0.0/24 via 10.1.0.254

```

Verify the route is active on `spoke-web-app`:

```bash
docker exec -it spoke-web-app ip route show

```

*Expected Output:*

```text
default via 10.1.0.1 dev eth0 
10.1.0.0/24 dev eth0 proto kernel scope link src 10.1.0.4 
10.2.0.0/24 via 10.1.0.254 dev eth0

```

---

#### Step 5: Program Stateful Filtering on the Hub NVA

Access the firewall container and set up strict stateful connection tracking:

```bash
docker exec -it hub-firewall-nva bash -c "
  # Flush existing rules
  iptables -F FORWARD

  # 1. Allow stateful return packets ONLY IF the connection is established/related
  iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

  # 2. Allow outbound NEW connections from Spoke Compute to Spoke Data
  iptables -A FORWARD -s 10.1.0.0/24 -d 10.2.0.0/24 -p tcp --dport 80 -m conntrack --ctstate NEW -j ACCEPT

  # 3. Drop all other forwarded traffic
  iptables -A FORWARD -j DROP
"

```

---

### Step 6: Triggering the Asymmetric Routing Outage

Notice what we **have not** done: We intentionally did **not** add a UDR on `spoke-db-service` pointing back to `10.2.0.254`!

`spoke-db-service` only has its default route pointing to the Docker gateway `10.2.0.1`.

#### Experiment 1: The Outage (Client Timeout)

From `spoke-web-app`, attempt to make an HTTP request to `spoke-db-service` (`10.2.0.4:80`):

```bash
docker exec -it spoke-web-app curl -i -s --connect-timeout 4 http://10.2.0.4

```

*Expected Output:*

```text
curl: (28) Failed to connect to 10.2.0.4 port 80 after 4001 ms: Couldn't connect to server

```

The connection hung and timed out!

---

#### Experiment 2: The Forensic Analysis with `tcpdump`

Let's see where the packets went. We will run three simultaneous captures across the components to trace the packet path.

1. **Terminal 2 (Inspect the Destination Server):**
Open a new terminal window and watch what arrives at `spoke-db-service`:

```bash
docker exec -it spoke-db-service tcpdump -n -i eth0 "port 80"

```

2. **Terminal 3 (Inspect the Hub Firewall):**
Open a third terminal window and watch the traffic traversing `hub-firewall-nva`:

```bash
docker exec -it hub-firewall-nva tcpdump -n -i any "port 80"

```

3. **Terminal 1 (The Client):**
Trigger the connection again from `spoke-web-app`:

```bash
docker exec -it spoke-web-app curl -s --connect-timeout 4 http://10.2.0.4

```

---

#### Interpreting the Capture Traces

Look at **Terminal 2 (`spoke-db-service`)**:

```text
10:15:02.100 IP 10.1.0.4.49210 > 10.2.0.4.80: Flags [S], seq 382910291, ...
10:15:02.101 IP 10.2.0.4.80 > 10.1.0.4.49210: Flags [S.], seq 982103811, ack 382910292, ...
10:15:03.105 IP 10.2.0.4.80 > 10.1.0.4.49210: Flags [S.], seq 982103811, ack 382910292, ... (retransmission)
10:15:05.110 IP 10.2.0.4.80 > 10.1.0.4.49210: Flags [S.], seq 982103811, ack 382910292, ... (retransmission)

```

Look at **Terminal 3 (`hub-firewall-nva`)**:

```text
10:15:02.100 IP 10.1.0.4.49210 > 10.2.0.4.80: Flags [S], seq 382910291, ... (Forwarded)
# Notice: ZERO packets returning! No [S.] flag is ever seen by the NVA!

```

Check the connection tracking state table inside `hub-firewall-nva`:

```bash
docker exec -it hub-firewall-nva conntrack -L -p tcp --dport 80

```

*Output:*

```text
tcp      6 118 SYN_SENT src=10.1.0.4 dst=10.2.0.4 sport=49210 dport=80 [UNREPLIED] src=10.2.0.4 dst=10.1.0.4 sport=80 dport=49210

```

**The Diagnosis:**

* **Forward leg:** `spoke-web-app` sent a `SYN` packet to the firewall (`10.1.0.254`). The firewall forwarded it to `spoke-db-service` and marked the connection state as `SYN_SENT [UNREPLIED]`.
* **The Return leg failed:** `spoke-db-service` generated a valid `SYN-ACK` packet. But because it did **not** have a UDR directing traffic for `10.1.0.0/24` back to the firewall (`10.2.0.254`), it dumped the packet onto its default bridge gateway (`10.2.0.1`).
* **The Result:** The firewall never saw the `SYN-ACK`. The session remained dead in `SYN_SENT`. The server retransmitted its `SYN-ACK` repeatedly until `spoke-web-app` gave up and timed out.

Press `Ctrl+C` in Terminal 2 and Terminal 3 to stop both packet captures.

---

### Step 7: The Fix — Restoring Route Symmetry

To eliminate asymmetric routing in Azure, every spoke subnet must have a corresponding route entry in its Route Table directing return traffic back through the firewall.

1. Add the missing User-Defined Route on **Spoke Data**:

```bash
docker exec -it spoke-db-service ip route add 10.1.0.0/24 via 10.2.0.254

```

2. Verify the route table on `spoke-db-service`:

```bash
docker exec -it spoke-db-service ip route show

```

*Expected Output:*

```text
default via 10.2.0.1 dev eth0 
10.1.0.0/24 via 10.2.0.254 dev eth0 
10.2.0.0/24 dev eth0 proto kernel scope link src 10.2.0.4

```

3. Now, re-test the connection from `spoke-web-app`:

```bash
docker exec -it spoke-web-app curl -s http://10.2.0.4

```

*Expected Output:*

```text
[DATA SUCCESS] Connected to database backend!

```

4. Verify the state table inside `hub-firewall-nva`:

```bash
docker exec -it hub-firewall-nva conntrack -L -p tcp --dport 80

```

*Output:*

```text
tcp      6 431999 ESTABLISHED src=10.1.0.4 dst=10.2.0.4 sport=49210 dport=80 src=10.2.0.4 dst=10.1.0.4 sport=80 dport=49210 [ASSURED]

```

The firewall observed both directions of the handshake, marked the stream as `ESTABLISHED [ASSURED]`, and allowed data packets to transfer cleanly.

---

### Step 8: Clean Up

Once the team has seen the tcpdump comparison, tear down the lab environment:

```bash
docker compose down

```

---

### Key Enterprise Takeaways for the Team

* **Stateful Firewalls Require Symmetry:** A stateful packet filter will drop traffic if it only sees one half of the TCP conversation. Both the outbound path and the return path must traverse the same firewall instance.
* **The "One-Sided UDR" Trap:** In Azure, configuring a Route Table on the frontend subnet is only half the job. If the backend database or API subnet does not have a route pointing back to the firewall, asymmetric routing will silently kill the connection.
* **Diagnostic Rule of Thumb:** If `tcpdump` on the destination shows incoming `SYN` packets and outgoing `SYN-ACK` packets, but the client keeps timing out, you are dealing with asymmetric routing. Check the return route table immediately.