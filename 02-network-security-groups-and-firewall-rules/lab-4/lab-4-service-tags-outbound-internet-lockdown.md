# Lab 2.4: Service Tags & Outbound Internet Lockdown — Taming the Default Outbound Rule

Welcome to Lab 2.4! In this lab, we dive into one of the most common security compliance requirements in enterprise Azure deployments: **locking down outbound internet egress while keeping platform dependencies alive**.

Every Azure Network Security Group (NSG) comes out of the box with three default outbound rules that cannot be deleted:

1. `AllowVnetOutBound` (Priority 65000)
2. **`AllowInternetOutBound` (Priority 65001)** — *Allows all workloads to dial any public IP address on the internet!*
3. `DenyAllOutBound` (Priority 65500)

When security auditors notice that every VM or App Service can talk freely to the public internet, teams rush to add a custom rule: **Priority 4000: Deny All Outbound Traffic**.

Then things immediately break:

* Operating system clocks drift and break TLS certificates because **Network Time Protocol (NTP)** is blocked.
* Package updates (`apt`, `apk`, `dnf`) fail.
* VMs drop out of Azure management because VM agent telemetry to Microsoft endpoints is cut off.

To solve this, Azure provides **Service Tags**. A Service Tag is a Microsoft-managed alias representing an underlying, dynamically updated block of IP prefixes (like `AzureCloud`, `Storage`, `Sql`, or `AzureActiveDirectory`). Instead of hunting down thousands of public IP blocks, you write security rules targeting the tag.

In this lab, we will build an outbound firewall on your Mac mini that locks down general internet access, observe the broken core services, and then use Linux `ipset` to emulate Azure Service Tags to restore essential platform communication safely.

---

### What Are We Simulating?

```
                      [Internal Subnet: 10.10.1.0/24]
 ┌──────────────────────────────────────────────────────────────────────┐
 │                                                                      │
 │                      ┌──────────────────────┐                        │
 │                      │    app-workload      │                        │
 │                      │    (10.10.1.4)       │                        │
 │                      │                      │                        │
 │                      │  • Needs NTP Clock   │                        │
 │                      │  • Needs Storage API │                        │
 │                      │  • Block Data Exfil  │                        │
 │                      └──────────┬───────────┘                        │
 └─────────────────────────────────┼────────────────────────────────────┘
                                   │
                                   ▼
 ┌──────────────────────────────────────────────────────────────────────┐
 │             Central Router / Firewall (nsg-egress-firewall)          │
 │                                                                      │
 │   Emulated Azure Service Tags via ipset:                             │
 │   • tag_ntp_servers      ──► [ 162.159.200.1, 162.159.200.123 ]      │
 │   • tag_azure_storage    ──► [ 20.150.0.0/16 ]                       │
 │                                                                      │
 │   Outbound NSG Rule Order:                                           │
 │   - Priority 100: Allow UDP 123 to match-set tag_ntp_servers         │
 │   - Priority 200: Allow TCP 443 to match-set tag_azure_storage       │
 │   - Priority 4000: Custom DenyAllOutbound (0.0.0.0/0 -> DROP)        │
 └─────────────────────────────────┬────────────────────────────────────┘
                                   │
                                   ▼
                       [Public Internet / Destinations]
         ┌─────────────────────────┼─────────────────────────┐
         ▼                         ▼                         ▼
 ┌───────────────┐         ┌───────────────┐         ┌───────────────┐
 │ Cloud NTP Pool│         │ Azure Storage │         │ Untrusted Web │
 │  (UDP 123)    │         │  (TCP 443)    │         │  (Data Exfil) │
 │  ✅ ALLOWED   │         │  ✅ ALLOWED   │         │  ❌ BLOCKED   │
 └───────────────┘         └───────────────┘         └───────────────┘

```

---

### Core Concepts Explained

#### 1. Why Not Just Block Port 80 and 443?

Data exfiltration doesn't just happen over web ports. Malicious actors or compromised dependencies can tunnel data over DNS (UDP 53), raw TCP sockets, or custom ports. A true zero-trust outbound posture requires blocking `0.0.0.0/0` outbound by default.

#### 2. What Service Tags Actually Represent Under the Hood

When you specify `Storage` or `AzureCloud.EastUS` in an Azure NSG, Microsoft's control plane translates that tag into a dynamic routing/filtering access list of thousands of CIDR blocks. Microsoft automatically updates these IP lists weekly as new data centers and subnets go live, without requiring changes to your NSG rules.

#### 3. Emulating Service Tags with `ipset`

Linux `ipset` can store both individual IPs and entire CIDR blocks (`hash:net`). By binding `iptables` rules to named sets, we replicate the exact behavior of Azure NSG rules referencing Service Tags:

```bash
# Concept equivalent:
Azure NSG: Allow Outbound Destination: ServiceTag "Storage" on Port 443
Linux:     iptables -A FORWARD -m set --match-set tag_azure_storage dst -p tcp --dport 443 -j ACCEPT

```

---

### Step-by-Step Hands-on Lab

#### Step 1: Project Setup

Open Terminal on your Mac mini and create a dedicated workspace:

```bash
mkdir -p ~/azure-labs/lab2-4-service-tags
cd ~/azure-labs/lab2-4-service-tags

```

---

#### Step 2: Create the Lab Definition (`docker-compose.yml`)

Create a `docker-compose.yml` file.

Notice the topology layout:

* `vnet-workload` (`10.10.1.0/24`) is set to **`internal: true`**. The workload container has no direct bridge to the host internet.
* `hub-egress` is a standard bridge with normal host internet connectivity.
* `nsg-egress-firewall` bridges both subnets with Linux `NET_ADMIN` capabilities and kernel IP forwarding enabled.

```yaml
services:
  # ==========================================================
  # Outbound Firewall Router (Simulating Azure NSG Egress Engine)
  # ==========================================================
  nsg-egress-firewall:
    image: nicolaka/netshoot:latest
    container_name: nsg-egress-firewall
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    sysctls:
      - net.ipv4.ip_forward=1
    networks:
      hub-egress:
        ipv4_address: 10.0.0.4
      vnet-workload:
        ipv4_address: 10.10.1.254

  # ==========================================================
  # Application Workload (Internal Subnet)
  # ==========================================================
  app-workload:
    image: nicolaka/netshoot:latest
    container_name: app-workload
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    networks:
      vnet-workload:
        ipv4_address: 10.10.1.4

networks:
  # Hub Egress Network (Has internet access via Docker host NAT)
  hub-egress:
    driver: bridge
    ipam:
      config:
        - subnet: 10.0.0.0/24
          gateway: 10.0.0.1

  # Workload Network (Isolated: internal=true blocks direct internet access)
  vnet-workload:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 10.10.1.0/24

```

---

#### Step 3: Launch the Stack

Start up the lab:

```bash
docker compose up -d

```

Verify that both containers are running:

```bash
docker compose ps

```

---

#### Step 4: Configure the Workload Default Route

Point the workload's default route to the firewall interface (`10.10.1.254`):

```bash
docker exec -it app-workload ip route add default via 10.10.1.254

```

Verify the route is active on `app-workload`:

```bash
docker exec -it app-workload ip route show

```

*Expected Output:*

```text
default via 10.10.1.254 dev eth0 
10.10.1.0/24 dev eth0 proto kernel scope link src 10.10.1.4

```

---

#### Step 5: Phase 1 — The Naive Outbound Lockdown (The Outage Simulation)

First, let's configure the firewall the way inexperienced administrators do: they turn on NAT, but apply a blanket `DenyAllOutBound` rule with no service exceptions.

1. Access the firewall container:

```bash
docker exec -it nsg-egress-firewall bash

```

2. Flush rules and enable basic SNAT masquerading on the outbound internet interface (`eth0`):

```bash
iptables -F FORWARD
iptables -t nat -F

# Allow stateful return packets
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Azure SNAT: Masquerade outgoing packets leaving via eth0
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# ------------------------------------------------------------------------------
# Priority 4000: Custom DenyAllOutBound (Drop all outbound internet traffic)
# ------------------------------------------------------------------------------
iptables -A FORWARD -s 10.10.1.0/24 -j DROP

```

Exit the firewall container:

```bash
exit

```

---

### Step 6: Verification & Experiments

#### Experiment 1: Observing the Outage Caused by Blanket Lockdown

Let's see what happens to basic operational health inside `app-workload`.

1. **Attempt NTP Time Synchronization:**
Operating systems rely on NTP (UDP port 123) to keep their clocks in sync. Run an NTP query using `sntp` or `chrony` against a public time server:

```bash
docker exec -it app-workload sntp pool.ntp.org

```

*Result:* Times out! The client cannot sync its system clock.

2. **Attempt an Outbound HTTPS Request (e.g., Calling an External API):**

```bash
docker exec -it app-workload curl -s --connect-timeout 3 https://1.1.1.1

```

*Result:*

```text
curl: (28) Failed to connect to 1.1.1.1 port 443: Connection timed out

```

The blanket drop succeeded in blocking unwanted internet egress, but broke core platform functionality.

---

#### Step 7: Phase 2 — Implementing Emulated Azure Service Tags

Now we will use `ipset` on the firewall to construct two Service Tags:

1. **`tag_ntp_servers`**: Resolves to public pool NTP servers (`162.159.200.1` and `162.159.200.123` - Cloudflare time servers).
2. **`tag_azure_storage`**: Emulating an Azure Storage region prefix block (e.g., `20.150.0.0/16` or mock target).
3. Jump back into `nsg-egress-firewall`:

```bash
docker exec -it nsg-egress-firewall bash

```

2. Create the `ipset` groups:

```bash
# Create Service Tag set for NTP (host IPs)
ipset create tag_ntp_servers hash:ip

# Create Service Tag set for Azure Storage (CIDR network blocks)
ipset create tag_azure_storage hash:net

# Populate tag_ntp_servers with Cloudflare NTP IPs
ipset add tag_ntp_servers 162.159.200.1
ipset add tag_ntp_servers 162.159.200.123

# Populate tag_azure_storage with an approved destination block (using 1.1.1.0/24 for demonstration)
ipset add tag_azure_storage 1.1.1.0/24

```

3. Insert high-priority allow rules **before** the Priority 4000 Deny rule:

```bash
# Priority 100: Allow NTP (UDP 123) to destinations in tag_ntp_servers
iptables -I FORWARD 2 -s 10.10.1.0/24 -p udp --dport 123 -m set --match-set tag_ntp_servers dst -m conntrack --ctstate NEW -j ACCEPT

# Priority 200: Allow HTTPS (TCP 443) to destinations in tag_azure_storage
iptables -I FORWARD 3 -s 10.10.1.0/24 -p tcp --dport 443 -m set --match-set tag_azure_storage dst -m conntrack --ctstate NEW -j ACCEPT

```

4. Inspect the updated rule table:

```bash
iptables -L FORWARD -v -n --line-numbers

```

*Expected layout:*

```text
num   pkts bytes target     prot opt in     out     source          destination         
1        0     0 ACCEPT     all  --  *      *       0.0.0.0/0       0.0.0.0/0      ctstate RELATED,ESTABLISHED
2        0     0 ACCEPT     udp  --  *      *       10.10.1.0/24    0.0.0.0/0      udp dpt:123 match-set tag_ntp_servers dst ctstate NEW
3        0     0 ACCEPT     tcp  --  *      *       10.10.1.0/24    0.0.0.0/0      tcp dpt:443 match-set tag_azure_storage dst ctstate NEW
4        0     0 DROP       all  --  *      *       10.10.1.0/24    0.0.0.0/0           

```

Exit the firewall:

```bash
exit

```

---

#### Step 8: Re-testing with Service Tag Exceptions

#### Experiment 2: Verifying Time Sync Recovery

Test NTP synchronization from `app-workload` against one of the allowed IPs in `tag_ntp_servers`:

```bash
docker exec -it app-workload sntp 162.159.200.1

```

*Expected Output:*

```text
+0.00... +/- 0.01... 162.159.200.1

```

The query succeeded! UDP port 123 reached the time server through the firewall rule matching `tag_ntp_servers`.

---

#### Experiment 3: Verifying Approved Cloud Service Access

Test HTTPS access to `1.1.1.1`, which belongs to the prefix block in `tag_azure_storage`:

```bash
docker exec -it app-workload curl -s -I --connect-timeout 3 https://1.1.1.1

```

*Expected Output:*

```text
HTTP/2 200 
...

```

The connection was permitted through the rule matching `tag_azure_storage`.

---

#### Experiment 4: Verifying Data Exfiltration Lockdown

Now attempt to reach an unapproved external destination (for example, Google Public DNS `8.8.8.8` or an arbitrary public web server):

```bash
docker exec -it app-workload curl -s --connect-timeout 3 https://8.8.8.8

```

*Expected Output:*

```text
curl: (28) Failed to connect to 8.8.8.8 port 443: Connection timed out

```

Check the packet counters on the firewall:

```bash
docker exec -it nsg-egress-firewall iptables -L FORWARD -v -n

```

Notice the packet hit counter on rule 4 (`DROP`) incremented!

The firewall verified:

* Destination was not in `tag_ntp_servers`.
* Destination was not in `tag_azure_storage`.
* Packet dropped at Priority 4000. Data exfiltration blocked!

---

### Step 9: Clean Up

When you've finished the exercises, tear down the lab environment:

```bash
docker compose down

```

---

### Key Enterprise Takeaways for the Team

* **Never Deploy a Blind `DenyAllOutBound`:** If you add an outbound deny rule without exception rules, you will immediately break time sync (NTP), package managers, and Azure platform management agents.
* **Service Tags Minimize Rule Sprawl:** Instead of maintaining hundreds of dynamic IP addresses, Azure Service Tags abstract cloud endpoints into managed labels (`Storage`, `Sql`, `AzureKeyVault`, `AzureActiveDirectory`).
* **Regional Service Tags:** When tighter scoping is required, Azure provides regional tags (e.g., `Storage.EastUS` or `Sql.WestEurope`) so workloads can only access PaaS services within their home region, preventing egress across cloud boundaries.