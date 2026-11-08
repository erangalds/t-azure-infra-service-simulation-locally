An Azure Virtual Network (VNet) is an isolated Layer 3 boundary within Microsoft's software-defined network (SDN). When you provision subnets within that address space, network isolation and IP address allocation are governed by strict mathematical and platform rules.

This lab simulates how Azure allocates IP prefixes, handles default routing, and isolates subnets on macOS using custom Docker bridge networks.

---

### Conceptual Architecture

```
           Azure Virtual Network (VNet Address Space: 10.10.0.0/16)
 ┌───────────────────────────────────────┬───────────────────────────────────────┐
 │ Subnet: vnet-frontend (10.10.1.0/24)  │ Subnet: vnet-backend (10.10.2.0/24)   │
 │                                       │                                       │
 │  .0   Network Address (Reserved)      │  .0   Network Address (Reserved)      │
 │  .1   Default Gateway (Reserved)      │  .1   Default Gateway (Reserved)      │
 │  .2   Azure DNS       (Reserved)      │  .2   Azure DNS       (Reserved)      │
 │  .3   Azure DNS       (Reserved)      │  .3   Azure DNS       (Reserved)      │
 │  .4   [fe-app-01]     (Usable IP)     │  .4   [be-api-01]     (Usable IP)     │
 │  .5   [fe-app-02]     (Usable IP)     │  .255 Broadcast       (Reserved)      │
 │  .255 Broadcast       (Reserved)      │                                       │
 └───────────────────┬───────────────────┴───────────────────┬───────────────────┘
                     │                                       │
                     ▼                                       ▼
       ┌───────────────────────────┐           ┌───────────────────────────┐
       │ Docker Bridge: fe-bridge  │           │ Docker Bridge: be-bridge  │
       │ Gateway: 10.10.1.1        │           │ Gateway: 10.10.2.1        │
       └─────────────┬─────────────┘           └─────────────┬─────────────┘
                     │                                       │
                     └─────────────── X ─────────────────────┘
                       Default Isolated Layer 3 Boundary
                       (No routing between subnets yet)

```

---

### Core Concepts Explained

#### 1. CIDR Math and Usable Hosts

A Classless Inter-Domain Routing (CIDR) block is written as `A.B.C.D/n`, where `/n` denotes the number of fixed network bits in a 32-bit IPv4 address:

* Host bits = $32 - n$
* Total IP addresses in subnet = $2^{(32 - n)}$

For a `/24` subnet:

* Total addresses = $2^{(32 - 24)} = 2^8 = 256$.
* In standard RFC 1918 IPv4 networking, usable addresses = $256 - 2 = 254$ (excluding network `.0` and broadcast `.255`).

#### 2. Azure's 5 Reserved IP Addresses

In any Azure subnet, standard host capacity drops by **5 reserved addresses**, regardless of subnet size:

| IP Address | Role | Description |
| --- | --- | --- |
| `x.x.x.0` | **Network Address** | The wire identifier of the subnet. |
| `x.x.x.1` | **Default Gateway** | Managed gateway enabling internal routing and internet egress. |
| `x.x.x.2` | **Azure DNS** | Redundant internal recursive resolver mapped to `168.63.129.16`. |
| `x.x.x.3` | **Azure DNS** | Secondary internal resolver mapped to `168.63.129.16`. |
| `x.x.x.255` | **Broadcast Address** | Broadcast identifier (Azure SDN drops broadcast, but address is withheld). |

**Usable IPs formula in Azure:**


$$\text{Usable Hosts} = 2^{(32 - n)} - 5$$

For a `/24` subnet: $256 - 5 = 251\text{ usable host addresses}$. The first compute resource deployed into `10.10.1.0/24` will always receive **`10.10.1.4`**.

#### 3. Layer 3 Isolation

A subnet represents a single Layer 2 broadcast domain and Layer 3 IP range. Two different subnets cannot communicate without an intermediate Layer 3 routing mechanism (such as an IP router, an Azure System Route, or a software gateway).

---

### Step-by-Step Hands-on Lab

#### Step 1: Project Directory Setup

Open Terminal on your Mac mini and create a dedicated workspace:

```bash
mkdir -p ~/azure-labs/lab1-subnets
cd ~/azure-labs/lab1-subnets

```

#### Step 2: Create the Lab Definition (`docker-compose.yml`)

Create a `docker-compose.yml` file. This configures two distinct network bridges and assigns static IP addresses starting at `.4` to mirror Azure's reserved host offset:

```yaml
services:
  # Frontend Workload 1 (Emulating a Web Server / App Service instance)
  fe-app-01:
    image: nicolaka/netshoot:latest
    container_name: fe-app-01
    stdin_open: true
    tty: true
    networks:
      vnet-frontend:
        ipv4_address: 10.10.1.4

  # Frontend Workload 2 (Emulating a second Web Server in the same subnet)
  fe-app-02:
    image: nicolaka/netshoot:latest
    container_name: fe-app-02
    stdin_open: true
    tty: true
    networks:
      vnet-frontend:
        ipv4_address: 10.10.1.5

  # Backend Workload 1 (Emulating an internal API service)
  be-api-01:
    image: nicolaka/netshoot:latest
    container_name: be-api-01
    stdin_open: true
    tty: true
    networks:
      vnet-backend:
        ipv4_address: 10.10.2.4

networks:
  # Emulates Azure Subnet: vnet-frontend (10.10.1.0/24)
  vnet-frontend:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 10.10.1.0/24
          gateway: 10.10.1.1

  # Emulates Azure Subnet: vnet-backend (10.10.2.0/24)
  vnet-backend:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 10.10.2.0/24
          gateway: 10.10.2.1

```

> **Why `nicolaka/netshoot`?** This lightweight container image contains common network diagnostic utilities, including `ip`, `route`, `ping`, `traceroute`, `nc` (netcat), `drill`, `curl`, and `tcpdump`.

#### Step 3: Instantiate the Environment

Start the lab in detached mode:

```bash
docker compose up -d

```

Verify that all three containers are healthy and running:

```bash
docker compose ps

```

---

### Step 4: Verification and Experiments

#### Experiment 1: Inspect the Subnet Default Gateway and Route Table

Access the shell of `fe-app-01` and verify the IP assignment and internal routing table:

```bash
docker exec -it fe-app-01 ip addr show eth0

```

*Expected output:* You will see `inet 10.10.1.4/24`, matching the first usable IP address in an Azure subnet.

Next, inspect the kernel routing table:

```bash
docker exec -it fe-app-01 ip route show

```

*Expected output:*

```text
default via 10.10.1.1 dev eth0 
10.10.1.0/24 dev eth0 proto kernel scope link src 10.10.1.4

```

**Takeaway for the team:** Any packet heading to an IP within `10.10.1.0/24` is delivered directly over Layer 2 (ARP/local switch). Anything destined outside this subnet is forwarded to the default gateway at `10.10.1.1`.

---

#### Experiment 2: Intra-Subnet Communication (Within `vnet-frontend`)

Test direct local communication between `fe-app-01` (`10.10.1.4`) and `fe-app-02` (`10.10.1.5`):

```bash
docker exec -it fe-app-01 ping -c 3 10.10.1.5

```

*Result:* Success with sub-millisecond round-trip times.

Inspect the ARP cache on `fe-app-01`:

```bash
docker exec -it fe-app-01 ip neigh show

```

*Result:* You will see an active ARP entry mapping `10.10.1.5` directly to the virtual MAC address of `fe-app-02`. No routing gateway was touched.

---

#### Experiment 3: Inter-Subnet Boundary Isolation

Now attempt to communicate across the boundary from `fe-app-01` (`10.10.1.4`) to `be-api-01` on the backend subnet (`10.10.2.4`):

```bash
docker exec -it fe-app-01 ping -c 3 -W 2 10.10.2.4

```

*Result:* 100% packet loss (timeout).

Trace the packet path to identify where the traffic stops:

```bash
docker exec -it fe-app-01 traceroute -n -m 3 10.10.2.4

```

*Result:* The packet reaches the gateway `10.10.1.1`, but cannot traverse to `10.10.2.0/24`. The bridges operate as separate Layer 3 broadcast domains with no inter-bridge routing.

---

#### Experiment 4: Testing Layer 4 Port Boundaries with Netcat (`nc`)

In production Azure environments, a service can be active on a host while ICMP (`ping`) is disabled. Simulate an API service listening on TCP port 8080:

1. In your first terminal window, start a TCP listener inside `be-api-01`:

```bash
docker exec -it be-api-01 nc -l -v -p 8080

```

2. In a second terminal window, attempt to establish a TCP stream from `fe-app-01`:

```bash
docker exec -it fe-app-01 nc -zv -w 3 10.10.2.4 8080

```

*Result:* `nc: connect to 10.10.2.4 port 8080 (tcp) timed out: Operation timed out`.

**Takeaway for the team:** In a standard physical or virtualized network, separating resources into subnets enforces a hard boundary. For `fe-app-01` to reach `be-api-01`, you must explicitly route traffic through a gateway or router appliance (which we implement via NSGs and Route Tables in Labs 2 and 4).

---

### Step 5: Lab Clean-up

Once the team completes the verification exercises, tear down the containers and isolated bridge networks:

```bash
docker compose down

```