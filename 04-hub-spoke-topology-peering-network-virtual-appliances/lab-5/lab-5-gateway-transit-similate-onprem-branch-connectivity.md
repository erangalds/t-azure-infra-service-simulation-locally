# Lab 4.5: Gateway Transit — Simulating On-Premises Hybrid Connectivity

Welcome to Lab 4.5! In enterprise cloud adoption, virtually no company runs completely isolated in the cloud from day one. You almost always have existing corporate data centers, office branches, or co-location facilities that need secure, private access to cloud workloads.

Building individual IPSec VPN tunnels or ExpressRoute circuits from on-premises to every single spoke VNet is expensive, unmanageable, and hits platform limits fast. Instead, Azure uses **Gateway Transit**:

1. You provision **one** Virtual Network Gateway (VPN or ExpressRoute) inside a dedicated `GatewaySubnet` in your **Hub VNet**.
2. When you peer a Spoke VNet to the Hub, you check **"Use the remote virtual network's gateway"** (on the Spoke) and **"Allow gateway transit"** (on the Hub).
3. The Hub dynamically shares its gateway connectivity with the Spokes, enabling on-prem systems to reach Spoke workloads without creating separate VPN tunnels.

In this lab, we'll introduce a fourth isolated network representing your **On-Premises Corporate Data Center (`192.168.1.0/24`)**, connect it strictly to the Hub Gateway, and demonstrate how Spoke workloads communicate across the hybrid bridge using Gateway Transit.

---

### What Are We Simulating?

```
                     ┌──────────────────────────────────────────────┐
                     │            Hub VNet (10.0.0.0/24)            │
                     │                                              │
                     │   ┌──────────────────────────────────────┐   │
                     │   │    Hub VPN Gateway (GatewaySubnet)   │   │
                     │   │         (hub-vpn-gateway)            │   │
                     │   │                                      │   │
                     │   │   • Allow Gateway Transit = Enabled  │   │
                     │   │   • Routes between On-Prem & Spokes  │   │
                     │   └──────▲───────────────────────▲───────┘   │
                     │          │                       │           │
                     └──────────┼───────────────────────┼───────────┘
                                │                       │
            IPSec / ExpressRoute│                       │ Peering Link
            (Simulated Tunnel)  │                       │ (Use Remote Gateway)
                                │                       │
 ┌──────────────────────────────┴─────┐   ┌─────────────┴────────────────┐
 │   On-Prem Branch (192.168.1.0/24)  │   │   Spoke Compute (10.1.0.0/24)│
 │                                    │   │                              │
 │  ┌──────────────────────────────┐  │   │  ┌────────────────────────┐  │
 │  │ onprem-payroll-srv           │  │   │  │ spoke-app-workload     │  │
 │  │ (192.168.1.4)                │◄─┼───┼──┤ (10.1.0.4)             │  │
 │  │ Port 8443: On-Prem Payroll   │  │   │  │                        │  │
 │  └──────────────────────────────┘  │   │  └────────────────────────┘  │
 │                                    │   │                              │
 │ Default GW -> Hub (192.168.1.254)  │   │ Remote GW -> Hub (10.1.0.254)│
 └────────────────────────────────────┘   └──────────────────────────────┘

```

---

### Core Concepts Explained

#### 1. The Gateway Transit Flags in Azure

When configuring peering in Azure, Gateway Transit relies on two reciprocal toggles:

* **On the Hub VNet Peering:** **"Allow gateway transit"** tells Azure that this VNet is permitted to share its VPN or ExpressRoute Gateway with the peered spoke.
* **On the Spoke VNet Peering:** **"Use the remote virtual network's gateway"** tells Azure to accept routes learned by the Hub Gateway and install them as system routes on the spoke's route table.

If either toggle is omitted, on-prem traffic stops dead at the Hub.

#### 2. Cross-Premises Route Propagation

When Gateway Transit is enabled:

* The Hub Gateway learns on-premises CIDRs (e.g., `192.168.1.0/24`) via BGP or local network gateway definitions.
* Azure's software-defined network automatically injects those routes into the Spoke subnets with next hop `Virtual network gateway`.
* The Spoke workload sends traffic destined for `192.168.1.0/24` directly to the Hub Gateway interface without needing a manual public internet path.

---

### Step-by-Step Hands-on Lab

#### Step 1: Set Up Your Project Directory

Open Terminal on your Mac mini and navigate to your labs workspace:

```bash
mkdir -p ~/azure-labs/lab4-5-gateway-transit
cd ~/azure-labs/lab4-5-gateway-transit

```

---

#### Step 2: Create the Lab Definition (`docker-compose.yml`)

Create a `docker-compose.yml` file.

Notice how the topology is separated:

* `onprem-branch` uses a typical on-premise RFC 1918 range (`192.168.1.0/24`) set to `internal: true`.
* `spoke-compute` uses the cloud range (`10.1.0.0/24`) set to `internal: true`.
* `hub-vnet` (`10.0.0.0/24`) connects both domains through our simulated `hub-vpn-gateway` container with packet forwarding enabled.

```yaml
services:
  # ==========================================================
  # Central Hub: VPN Gateway / GatewaySubnet Emulation
  # ==========================================================
  hub-vpn-gateway:
    image: nicolaka/netshoot:latest
    container_name: hub-vpn-gateway
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
      onprem-branch:
        ipv4_address: 192.168.1.254

  # ==========================================================
  # Spoke 1: Cloud Application Workload
  # ==========================================================
  spoke-app-workload:
    image: nicolaka/netshoot:latest
    container_name: spoke-app-workload
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    networks:
      spoke-compute:
        ipv4_address: 10.1.0.4

  # ==========================================================
  # On-Premises: Legacy Corporate Server (Payroll App)
  # ==========================================================
  onprem-payroll-srv:
    image: nicolaka/netshoot:latest
    container_name: onprem-payroll-srv
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    command: >
      sh -c "while true; do echo -e 'HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n[ON-PREM CORP DATA] Payroll records reached via Gateway Transit!' | nc -l -p 8443 -q 1; done"
    networks:
      onprem-branch:
        ipv4_address: 192.168.1.4

networks:
  # Azure Hub VNet
  hub-vnet:
    driver: bridge
    ipam:
      config:
        - subnet: 10.0.0.0/24
          gateway: 10.0.0.1

  # Azure Spoke VNet (Isolated)
  spoke-compute:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 10.1.0.0/24

  # Corporate On-Premises Branch (Isolated Data Center)
  onprem-branch:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 192.168.1.0/24

```

---

#### Step 3: Launch the Stack

Spin up the lab:

```bash
docker compose up -d

```

Verify that all three containers are healthy:

```bash
docker compose ps

```

---

#### Step 4: Verification & Experiments

#### Experiment 1: The Initial Disconnect (No Transit Route)

Before enabling gateway transit, check if the cloud app can reach the on-prem server directly:

```bash
docker exec -it spoke-app-workload ping -c 2 -W 2 192.168.1.4

```

*Expected Output:*

```text
ping: connect: Network is unreachable

```

Check the route table inside `spoke-app-workload`:

```bash
docker exec -it spoke-app-workload ip route show

```

*Output:*

```text
10.1.0.0/24 dev eth0 proto kernel scope link src 10.1.0.4

```

`spoke-app-workload` only knows about its own local subnet. It has zero awareness of the on-premises `192.168.1.0/24` network.

---

#### Experiment 2: Simulating "Use Remote Gateway" & "Allow Gateway Transit"

In Azure, when you enable Gateway Transit on the peering, Azure automatically injects the on-prem route (`192.168.1.0/24`) into the Spoke subnet, and injects the Spoke route (`10.1.0.0/24`) into the on-prem router via BGP.

Let's simulate this dynamic route propagation:

1. Configure the Spoke to route on-premises traffic (`192.168.1.0/24`) via the Hub Gateway interface (`10.1.0.254`):

```bash
docker exec -it spoke-app-workload ip route add 192.168.1.0/24 via 10.1.0.254

```

2. Configure the On-Premises server to route cloud traffic (`10.1.0.0/24`) via the Hub Gateway interface (`192.168.1.254`):

```bash
docker exec -it onprem-payroll-srv ip route add 10.1.0.0/24 via 192.168.1.254

```

3. Verify the updated route table on `spoke-app-workload`:

```bash
docker exec -it spoke-app-workload ip route show

```

*Expected Output:*

```text
10.1.0.0/24 dev eth0 proto kernel scope link src 10.1.0.4 
192.168.1.0/24 via 10.1.0.254 dev eth0

```

---

#### Experiment 3: Live Verification with `tcpdump`

Let's verify that traffic from the cloud spoke successfully traverses the Hub Gateway to access the legacy corporate service on port 8443.

1. Open a **second terminal window** on your Mac mini and attach to `hub-vpn-gateway` to monitor packets crossing between interfaces:

```bash
docker exec -it hub-vpn-gateway tcpdump -n -i any "port 8443"

```

2. In your **first terminal window**, make an HTTP request to the on-premise service from `spoke-app-workload`:

```bash
docker exec -it spoke-app-workload curl -s http://192.168.1.4:8443

```

*Expected output in Terminal 1:*

```text
[ON-PREM CORP DATA] Payroll records reached via Gateway Transit!

```

3. Look at the output in **Terminal 2 (the tcpdump capture window)**:

```text
IP 10.1.0.4.52184 > 192.168.1.4.8443: Flags [S], seq 19283719, ...
IP 192.168.1.4.8443 > 10.1.0.4.52184: Flags [S.], seq 98127391, ack 19283720, ...
IP 10.1.0.4.52184 > 192.168.1.4.8443: Flags [P.], seq 1:82, ack 1, ...
IP 192.168.1.4.8443 > 10.1.0.4.52184: Flags [P.], seq 1:126, ack 82, ...

```

**What this confirms:**

* The cloud workload sent packets across the VNet peering boundary into the Hub Gateway.
* The Hub Gateway routed the packets across the simulated hybrid tunnel into the on-premise data center.
* The on-premises payroll server replied through the gateway seamlessly without needing its own direct connection to the spoke.

Press `Ctrl+C` in Terminal 2 to stop `tcpdump`.

---

#### Experiment 4: The Peering Gotcha — Only ONE Gateway Allowed per VNet

A common architectural mistake in Azure is attempting to deploy a VPN gateway inside a spoke VNet *while* enabling "Use Remote Gateway".

* **The Rule:** An Azure VNet can **either** have its own Virtual Network Gateway **or** use a remote gateway via peering. It can **never** do both simultaneously.
* If a Spoke VNet already contains a `GatewaySubnet` with an active gateway, Azure disables the "Use Remote Gateway" option on the peering link until the local gateway is deleted.

---

### Step 5: Clean Up

When you've finished the exercises, shut down the lab environment:

```bash
docker compose down

```

---

### Key Hybrid Architecture Takeaways for the Team

* **Gateway Transit Scales Bandwidth & Cost:** Instead of paying for VPN/ExpressRoute gateways in every spoke, you deploy one centralized gateway in the Hub and share it across dozens of spokes.
* **Reciprocal Configuration:** Both flags must be set. The Hub must enable **"Allow gateway transit"**, and the Spoke must enable **"Use the remote virtual network's gateway"**.
* **Zero Hairpinning to On-Prem:** Traffic between Spoke A and Spoke B in the cloud does not need to traverse back to on-premise routers—it remains entirely within the Azure backbone.