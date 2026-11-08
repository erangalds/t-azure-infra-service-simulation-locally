# Lab 4.2: Forced Tunneling & Centralized Egress via Hub NVA

Welcome to Lab 4.2! In Lab 4.1, we saw how to route traffic between two spokes across a central Network Virtual Appliance (NVA). Now, we’re going to tackle one of the strictest governance requirements in enterprise cloud security: **Forced Tunneling**.

In an enterprise landing zone, internal workloads (VMs, container clusters, internal App Services) must **never** be allowed to talk directly to the public internet. If every subnet has its own direct internet pipe, you have zero centralized visibility into data exfiltration, malware downloads, or rogue outbound API calls.

To prevent this, security architects attach a Route Table (UDR) to every spoke subnet containing a **`0.0.0.0/0` (Default Route)** pointing to the central **Hub Firewall** as the `Virtual Appliance` next hop. All outbound traffic is funneled through the firewall, inspected, and then egressed to the internet through the firewall's public IP using **Source Network Address Translation (SNAT)**.

In this lab, we'll build this architecture on your Mac mini, completely stripping direct internet access from our spoke and forcing it to route out through our Hub NVA.

---

### What Are We Simulating?

```
                                  [Public Internet]
                                         ▲
                                         │ (Only Hub has public access)
                                         │
                 ┌──────────────────────────────────────────────┐
                 │            Hub VNet (10.0.0.0/24)            │
                 │                                              │
                 │   ┌──────────────────────────────────────┐   │
                 │   │     Central NVA / Azure Firewall     │   │
                 │   │         (hub-firewall-nva)           │   │
                 │   │                                      │   │
                 │   │   • External SNAT (MASQUERADE)       │   │
                 │   │   • Outbound FQDN/IP Filtering       │   │
                 │   │   • tcpdump Egress Inspection        │   │
                 │   └──────────────────▲───────────────────┘   │
                 │                      │ (10.1.0.254)          │
                 └──────────────────────┼───────────────────────┘
                                        │
                    FORCED TUNNEL (UDR: 0.0.0.0/0 -> NVA)
                                        │
                 ┌──────────────────────┴───────────────────────┐
                 │         Spoke Compute (10.1.0.0/24)          │
                 │         (internal: true - NO DIRECT EGRESS)  │
                 │                                              │
                 │   ┌──────────────────────────────────────┐   │
                 │   │     spoke-web-app (10.1.0.4)         │   │
                 │   │     All internet traffic forced      │   │
                 │   │     through the Hub NVA              │   │
                 │   └──────────────────────────────────────┘   │
                 └──────────────────────────────────────────────┘

```

---

### Core Concepts Explained

#### 1. Azure System Default Routes vs. Forced Tunneling

By default, Azure injects hidden system routes into every subnet. One of the default routes is:

* **Address Prefix:** `0.0.0.0/0`
* **Next Hop Type:** `Internet`

This means any VM with a public IP (or using Azure's default outbound access) can hit the internet directly without passing through a security inspection engine.

To lock this down, we create a **User-Defined Route (UDR)**:

* **Address Prefix:** `0.0.0.0/0`
* **Next Hop Type:** `Virtual Appliance`
* **Next Hop IP:** `10.0.0.4` (Internal IP of Azure Firewall)

Because a UDR with `0.0.0.0/0` matches the default system route, Azure’s **route precedence engine** prefers the user-defined route over the system route. Direct internet access is severed, and all outbound packets are forced into the firewall interface.

#### 2. Source Network Address Translation (SNAT) on Egress

When `spoke-web-app` (`10.1.0.4`) sends a packet destined for `1.1.1.1`, the Hub Firewall receives it. But the public internet does not know what private IP `10.1.0.4` is (RFC 1918).

The NVA performs **SNAT**:

1. Rewrites the packet's source IP from `10.1.0.4` to its own external IP.
2. Forwards the packet to the destination.
3. When the response returns, the NVA maps it back to `10.1.0.4` using its connection tracking state table.

To the outside world, **100% of outbound enterprise traffic originates from the Azure Firewall's single public IP**.

---

### Step-by-Step Hands-on Lab

#### Step 1: Set Up Your Project Directory

Open Terminal on your Mac mini and navigate to your labs folder:

```bash
mkdir -p ~/azure-labs/lab4-2-forced-tunneling
cd ~/azure-labs/lab4-2-forced-tunneling

```

---

#### Step 2: Create the Lab Definition (`docker-compose.yml`)

Create a `docker-compose.yml` file.

Notice key configuration details:

* `spoke-compute` is marked as **`internal: true`**. This tells the Docker engine to completely disable NAT and host routing on that bridge. Containers inside this subnet **cannot** reach the Mac host or the internet directly.
* `hub-vnet` is a standard bridge with normal internet egress enabled.
* `hub-firewall-nva` bridges both networks and runs with `sysctls: net.ipv4.ip_forward=1` and `cap_add: NET_ADMIN`.

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
      spoke-compute:
        ipv4_address: 10.1.0.254

  # ==========================================================
  # Spoke Workload (Strictly Internal Subnet)
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

networks:
  # Hub Network (Has external egress to internet via host NAT)
  hub-vnet:
    driver: bridge
    ipam:
      config:
        - subnet: 10.0.0.0/24
          gateway: 10.0.0.1

  # Spoke Network (Isolated: internal=true blocks direct internet egress)
  spoke-compute:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 10.1.0.0/24

```

---

#### Step 3: Launch the Stack

Start the lab in detached mode:

```bash
docker compose up -d

```

Verify both containers are running:

```bash
docker compose ps

```

---

### Step 4: Verification & Experiments

#### Experiment 1: Verify the Direct Egress Blackout

Because `spoke-compute` is configured as an internal isolated network, it has no default gateway pointing to the internet.

1. Inspect the routing table inside `spoke-web-app`:

```bash
docker exec -it spoke-web-app ip route show

```

*Expected output:*

```text
10.1.0.0/24 dev eth0 proto kernel scope link src 10.1.0.4

```

*Notice there is no `default via ...` route!*

2. Try to hit a public internet IP (like Cloudflare DNS `1.1.1.1`):

```bash
docker exec -it spoke-web-app ping -c 2 -W 2 1.1.1.1

```

*Expected output:*

```text
ping: connect: Network is unreachable

```

The workload is completely dark to the outside world. Direct egress is physically blocked at the subnet boundary.

---

#### Experiment 2: Configure the Forced Tunneling Route (`0.0.0.0/0 -> NVA`)

In Azure, you associate a Route Table with a rule:

* **Prefix:** `0.0.0.0/0`
* **Next Hop:** `10.1.0.254` (Hub Firewall internal interface)

Let's inject that default route inside `spoke-web-app`:

```bash
docker exec -it spoke-web-app ip route add default via 10.1.0.254

```

Verify the route is active:

```bash
docker exec -it spoke-web-app ip route show

```

*Expected output:*

```text
default via 10.1.0.254 dev eth0 
10.1.0.0/24 dev eth0 proto kernel scope link src 10.1.0.4

```

Now, try pinging `1.1.1.1` again:

```bash
docker exec -it spoke-web-app ping -c 2 -W 2 1.1.1.1

```

*Result:* It still fails (or times out)!

**Why?** The packet successfully arrives at `hub-firewall-nva`, but the firewall hasn't been configured with NAT (SNAT) yet, and the public internet doesn't know how to route back to private IP `10.1.0.4`.

---

#### Experiment 3: Configure SNAT and Forwarding on the Hub NVA

To act as an Azure Firewall, our NVA must:

1. Accept forwarded packets from the spoke subnet.
2. Masquerade (SNAT) them using the Hub's outgoing interface IP (`eth0`).
3. Access the shell of `hub-firewall-nva`:

```bash
docker exec -it hub-firewall-nva bash

```

2. Configure `iptables` forwarding and SNAT rules:

```bash
# Flush existing forwarding and NAT rules
iptables -F FORWARD
iptables -t nat -F

# 1. Stateful connection tracking: Allow return packets back to spoke
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 2. Allow forwarded traffic originating from Spoke Compute out to the internet
iptables -A FORWARD -s 10.1.0.0/24 -i eth1 -o eth0 -j ACCEPT

# 3. Azure SNAT: Masquerade outgoing packets with the Hub NVA's egress IP
iptables -t nat -A POSTROUTING -s 10.1.0.0/24 -o eth0 -j MASQUERADE

```

3. Exit the firewall container:

```bash
exit

```

---

#### Experiment 4: Verify Forced Egress and Live Packet Inspection

Now let's test internet connectivity from the isolated spoke and watch the packets traverse the firewall.

1. Open a **second terminal window** on your Mac mini and attach to `hub-firewall-nva` to run a live packet capture on outbound internet traffic:

```bash
docker exec -it hub-firewall-nva tcpdump -n -i eth0 "host 1.1.1.1 or port 80"

```

2. In your **first terminal window**, ping `1.1.1.1` from `spoke-web-app`:

```bash
docker exec -it spoke-web-app ping -c 3 1.1.1.1

```

*Result:* Replies succeed with low latency!

3. Look at the output in **Terminal 2 (the tcpdump window)**:

```text
IP 10.0.0.4 > 1.1.1.1: ICMP echo request, id 1, seq 1, length 64
IP 1.1.1.1 > 10.0.0.4: ICMP echo reply, id 1, seq 1, length 64

```

**Notice the Source IP in the tcpdump output:**
Even though the ping originated from `spoke-web-app` (`10.1.0.4`), the packet hitting the wire shows **`10.0.0.4`** (the Hub NVA)!

The Hub Firewall completely masked the spoke's identity, providing uniform egress.

Press `Ctrl+C` in Terminal 2 to stop `tcpdump`.

---

#### Experiment 5: Simulating Azure Firewall Outbound FQDN / IP Filtering

The primary reason security teams force traffic through a central hub is to enforce **Outbound Application Rules** (e.g., "Spoke workloads may only download packages from approved mirrors, blocking all other external endpoints").

Let's simulate an Azure Firewall rule that **allows HTTP calls to `icanhazip.com` (for public IP checks)**, but **blocks access to any other public IP**.

1. Identify the current IP of `icanhazip.com`:

```bash
docker exec -it hub-firewall-nva drill icanhazip.com | grep -A 1 "ANSWER SECTION"

```

*(Suppose the IP resolved is `104.18.27.120` or similar).*

2. Enter the firewall container and update the policy:

```bash
docker exec -it hub-firewall-nva bash

```

3. Lock down outbound forwarding to permit only that specific destination IP:

```bash
# Flush forwarding rules
iptables -F FORWARD

# Keep stateful return traffic open
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# ALLOW Rule: Outbound HTTP to icanhazip.com (replace with resolved IP if different)
iptables -A FORWARD -s 10.1.0.0/24 -p tcp -d 104.18.0.0/16 --dport 80 -m conntrack --ctstate NEW -j ACCEPT

# DENY Rule: Drop all other outbound internet traffic from the spoke
iptables -A FORWARD -s 10.1.0.0/24 -j DROP

```

4. Exit the firewall container:

```bash
exit

```

5. Test the approved destination from `spoke-web-app`:

```bash
docker exec -it spoke-web-app curl -s --connect-timeout 3 http://icanhazip.com

```

*Result:* Returns the public IP of your Mac's internet connection. The request was permitted through the NVA!

6. Test an unapproved destination (like Google DNS or raw IP `1.1.1.1`):

```bash
docker exec -it spoke-web-app curl -s --connect-timeout 3 http://1.1.1.1

```

*Result:*

```text
curl: (28) Failed to connect to 1.1.1.1 port 80 after 3000 ms: Couldn't connect to server

```

The Hub Firewall caught the packet, detected that it was not on the approved destination list, and dropped it immediately.

---

### Step 5: Clean Up

When you've finished the exercises, shut down the lab environment:

```bash
docker compose down

```

---

### Key Enterprise Takeaways for the Team

* **Default Route Override:** In Azure, adding a UDR with `0.0.0.0/0` pointing to a `Virtual Appliance` overrides Azure's default system route to the internet, enforcing forced tunneling.
* **Visibility & Compliance:** Centralizing egress through a single NVA enables compliance logging, threat intelligence feeds, and granular FQDN/domain whitelisting across all corporate spokes.
* **SNAT Port Exhaustion Awareness:** Because every spoke workload shares the firewall's public IP for outbound sessions, high-volume outbound architectures (like high-throughput microservices calling external SaaS APIs) must monitor Azure Firewall SNAT port metrics to avoid connection drops.