# Lab 5.2: Split-Horizon DNS & Azure Private DNS Zones

Welcome to Lab 5.2! In Lab 5.1, we accessed our Azurite storage and Azure SQL Edge containers by hardcoding raw IP addresses (`[http://10.2.0.10:10000/devstoreaccount1](http://10.2.0.10:10000/devstoreaccount1)...`) into connection strings.

In production, **hardcoding private IPs into PaaS connection strings is an anti-pattern that breaks SDKs and TLS validation**.

When you spin up an Azure Blob Storage account, the SDK expects to connect to a standardized Fully Qualified Domain Name (FQDN) like `mystorageaccount.blob.core.windows.net`. When you attach an **Azure Private Endpoint**, Azure does not delete that public FQDN. Instead, it creates a **Split-Horizon DNS CNAME chain**:

```
mystorageaccount.blob.core.windows.net ──► CNAME: mystorageaccount.privatelink.blob.core.windows.net

```

The outcome depends entirely on which DNS resolver answers:

1. **Public Internet DNS:** Resolves `mystorageaccount.privatelink.blob.core.windows.net` to a public Microsoft IP address. When your client hits it, Azure returns `HTTP 403 Forbidden: Public access is disabled`.
2. **Private VNet DNS (with an Azure Private DNS Zone linked):** Overrides `privatelink.blob.core.windows.net` to return the **Private Endpoint IP** (`10.2.0.10`). The SDK connects securely across the private virtual network.

In this lab, we will use **CoreDNS** on your Mac mini to simulate the Azure recursive DNS resolver (`168.63.129.16`) and an Azure Private DNS Zone. We will demonstrate how a missing zone link fails with public access errors, and how adding the private zone link resolves standard Azure FQDNs to private IPs seamlessly.

---

### What Are We Simulating?

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │                      Spoke Compute (10.1.0.0/24)                       │
 │                                                                        │
 │   ┌──────────────────────┐                                             │
 │   │  spoke-client-app    │                                             │
 │   │  (10.1.0.4)          │                                             │
 │   │                      │                                             │
 │   │  Queries DNS for:    │                                             │
 │   │  "devstoreaccount1.  │                                             │
 │   │   blob.core.windows. │                                             │
 │   │   net"               │                                             │
 │   └──────────┬───────────┘                                             │
 └──────────────┼─────────────────────────────────────────────────────────┘
                │
                ▼ (DNS Port 53)
 ┌────────────────────────────────────────────────────────────────────────┐
 │          CoreDNS (Simulating Azure Resolver 168.63.129.16)             │
 │          IP: 10.1.0.254                                                │
 │                                                                        │
 │   CNAME: devstoreaccount1.blob.core.windows.net                        │
 │          └──► devstoreaccount1.privatelink.blob.core.windows.net       │
 │                                                                        │
 │   Scenario A: Zone NOT Linked                                          │
 │   • Forwards to Public Upstream ──► Resolves 20.150.1.1 (Public IP)    │
 │   • Client connects to Public IP  ──► 403 FORBIDDEN (Access Denied)    │
 │                                                                        │
 │   Scenario B: Private DNS Zone LINKED                                  │
 │   • Authoritative Zone: privatelink.blob.core.windows.net              │
 │   • Resolves to Private Endpoint ──► 10.2.0.10 (Private IP)            │
 │   • Client connects via Private Link ──► 200 OK (Access Granted)       │
 └────────────────────────────────────────────────────────────────────────┘
                │
                ▼ (Private Routing: 10.2.0.10)
 ┌────────────────────────────────────────────────────────────────────────┐
 │                 Spoke Data Subnet (10.2.0.0/24)                        │
 │                                                                        │
 │   ┌────────────────────────────────────────────────────────────────┐   │
 │   │  azurite-storage (Private Endpoint at 10.2.0.10:10000)         │   │
 │   │  Zero host ports published; accessible only via Private IP!    │   │
 │   └────────────────────────────────────────────────────────────────┘   │
 └────────────────────────────────────────────────────────────────────────┘

```

---

### Core Concepts Explained

#### 1. Why Azure Uses CNAMEs for Private Link

When an application uses the Azure Storage SDK, the SDK signs requests using HMAC-SHA256 based on the account name and URL path. If the TLS certificate or request URL changes from `*.blob.core.windows.net` to an internal IP address, TLS validation breaks and HTTP Host headers fail.

By inserting a `CNAME` pointing to `*.privatelink.blob.core.windows.net`, Microsoft keeps the client-facing hostname unchanged while allowing network resolution to pivot between public IPs and private IPs dynamically.

#### 2. The Special Azure IP: `168.63.129.16`

In every Azure VNet, Microsoft provides a virtual recursive resolver at `168.63.129.16`. This resolver:

* Forwards normal public DNS queries (`google.com`, `github.com`) to public internet root servers.
* Checks if the VNet has a **Virtual Network Link** to any **Private DNS Zone**.
* If a zone like `privatelink.blob.core.windows.net` is linked to the VNet, the resolver intercepts the query and returns the private `A` record instead of forwarding it to the public internet.

---

### Step-by-Step Hands-on Lab

#### Step 1: Set Up Your Project Directory

Open Terminal on your Mac mini and create a dedicated workspace:

```bash
mkdir -p ~/azure-labs/lab5-2-private-dns/dns_config
cd ~/azure-labs/lab5-2-private-dns

```

---

#### Step 2: Create the CoreDNS Server Configurations

We will set up two configuration files for **CoreDNS** to simulate both scenarios:

1. `Corefile.unlinked`: Simulates Azure DNS when the Private DNS Zone is **not** linked to the VNet (resolves public mock IP `20.150.1.1`).
2. `Corefile.linked`: Simulates Azure DNS when the Private DNS Zone is **linked** to the VNet (resolves Private Endpoint IP `10.2.0.10`).

##### 1. Create Zone File for Public Resolution (Unlinked State)

Create `dns_config/zone.public`:

```text
$ORIGIN blob.core.windows.net.
@   IN  SOA ns1.azure-dns.com. hostmaster.azure.com. (
            2026110801 ; serial
            7200       ; refresh
            3600       ; retry
            1209600    ; expire
            3600       ; negative cache
)
    IN  NS  ns1.azure-dns.com.

; The public Azure CNAME record
devstoreaccount1    IN  CNAME   devstoreaccount1.privatelink.blob.core.windows.net.

```

Create `dns_config/zone.public-privatelink`:

```text
$ORIGIN privatelink.blob.core.windows.net.
@   IN  SOA ns1.azure-dns.com. hostmaster.azure.com. (
            2026110801 7200 3600 1209600 3600
)
    IN  NS  ns1.azure-dns.com.

; Public resolution returns the multi-tenant public IP
devstoreaccount1    IN  A   20.150.1.1

```

##### 2. Create the Unlinked Corefile

Create `dns_config/Corefile.unlinked`:

```text
.:53 {
    log
    errors
}

blob.core.windows.net:53 {
    file /etc/coredns/zone.public
    log
    errors
}

privatelink.blob.core.windows.net:53 {
    file /etc/coredns/zone.public-privatelink
    log
    errors
}

```

##### 3. Create the Linked Private Zone Configuration

When linked, the VNet queries an authoritative zone for `privatelink.blob.core.windows.net` that contains our private endpoint IP (`10.2.0.10`).

Create `dns_config/zone.private-linked`:

```text
$ORIGIN privatelink.blob.core.windows.net.
@   IN  SOA privatedns.azure.com. hostmaster.azure.com. (
            2026110802 7200 3600 1209600 3600
)
    IN  NS  privatedns.azure.com.

; PRIVATE ENDPOINT A-RECORD (Points to spoke-data 10.2.0.10)
devstoreaccount1    IN  A   10.2.0.10

```

Create `dns_config/Corefile.linked`:

```text
.:53 {
    log
    errors
}

blob.core.windows.net:53 {
    file /etc/coredns/zone.public
    log
    errors
}

; VNet Link Active: Authoritative private zone overrides public IP
privatelink.blob.core.windows.net:53 {
    file /etc/coredns/zone.private-linked
    log
    errors
}

```

---

#### Step 3: Create the Lab Definition (`docker-compose.yml`)

Create `docker-compose.yml` to spin up our 4 components:

1. `azure-resolver-dns`: CoreDNS container on `10.1.0.254`.
2. `spoke-client-app`: Test client container with tools installed (`curl`, `drill`, `dig`).
3. `azurite-storage`: Private Storage emulator on `10.2.0.10` with zero public host ports published.
4. `mock-public-storage`: A mock public gateway container on `20.150.1.1` returning HTTP 403 to simulate Azure's public endpoint behavior.

```yaml
services:
  # ==========================================================
  # Azure Recursive DNS / Private DNS Zone Resolver (CoreDNS)
  # ==========================================================
  azure-resolver-dns:
    image: coredns/coredns:latest
    container_name: azure-resolver-dns
    volumes:
      - ./dns_config/Corefile.unlinked:/etc/coredns/Corefile:ro
      - ./dns_config/zone.public:/etc/coredns/zone.public:ro
      - ./dns_config/zone.public-privatelink:/etc/coredns/zone.public-privatelink:ro
      - ./dns_config/zone.private-linked:/etc/coredns/zone.private-linked:ro
      - ./dns_config/Corefile.linked:/etc/coredns/Corefile.linked:ro
    command: -conf /etc/coredns/Corefile
    networks:
      spoke-compute:
        ipv4_address: 10.1.0.254

  # ==========================================================
  # Client Workload (Spoke Compute)
  # ==========================================================
  spoke-client-app:
    image: nicolaka/netshoot:latest
    container_name: spoke-client-app
    stdin_open: true
    tty: true
    cap_add:
      - NET_ADMIN
    networks:
      spoke-compute:
        ipv4_address: 10.1.0.4

  # ==========================================================
  # Private Endpoint Workload: Azurite Storage (Spoke Data)
  # ==========================================================
  azurite-storage:
    image: mcr.microsoft.com/azure-storage/azurite:latest
    container_name: azurite-storage
    # No host port mapping - strictly accessible via private IP!
    command: "azurite --blobHost 0.0.0.0 --blobPort 10000 --queueHost 0.0.0.0 --queuePort 10001 --loose"
    networks:
      spoke-data:
        ipv4_address: 10.2.0.10

  # ==========================================================
  # Mock Public Azure Storage Endpoint (Returns 403 Forbidden)
  # ==========================================================
  mock-public-storage:
    image: nicolaka/netshoot:latest
    container_name: mock-public-storage
    stdin_open: true
    tty: true
    command: >
      sh -c "while true; do echo -e 'HTTP/1.1 403 Forbidden\r\nContent-Type: text/plain\r\n\r\n[403 FORBIDDEN] Public network access on this storage account is disabled. Use Private Link.' | nc -l -p 10000 -q 1; done"
    networks:
      public-wan:
        ipv4_address: 20.150.1.1

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

  public-wan:
    driver: bridge
    ipam:
      config:
        - subnet: 20.150.1.0/24
          gateway: 20.150.1.254

```

---

#### Step 4: Spin Up the Stack

Launch the environment:

```bash
docker compose up -d

```

Verify all four containers are up:

```bash
docker compose ps

```

---

#### Step 5: Configure Routing on the Client

Enable routing inside `spoke-client-app` so it can reach both the private subnet and the mock public endpoint:

```bash
# Route to Private Data Subnet
docker exec -it spoke-client-app ip route add 10.2.0.0/24 via 10.1.0.1

# Route to Mock Public WAN Subnet
docker exec -it spoke-client-app ip route add 20.150.1.0/24 via 10.1.0.1

```

---

### Step 6: Verification & Experiments

#### Experiment 1: The Outage (Private Zone NOT Linked)

In this initial state, CoreDNS runs `Corefile.unlinked`. It knows about the CNAME, but resolves it using public internet records.

1. Query DNS from `spoke-client-app` using our simulated resolver (`10.1.0.254`):

```bash
docker exec -it spoke-client-app drill @10.1.0.254 devstoreaccount1.blob.core.windows.net

```

*Expected Output:*

```text
;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: ...
;; QUESTION SECTION:
;; devstoreaccount1.blob.core.windows.net.	IN	A

;; ANSWER SECTION:
devstoreaccount1.blob.core.windows.net.	3600	IN	CNAME	devstoreaccount1.privatelink.blob.core.windows.net.
devstoreaccount1.privatelink.blob.core.windows.net.	3600	IN	A	20.150.1.1

```

**Look at the resolved IP:** It resolved to `20.150.1.1` (the public endpoint).

2. Now attempt to download from the storage service using the standard FQDN:

```bash
docker exec -it spoke-client-app curl -i -s http://devstoreaccount1.blob.core.windows.net:10000/

```

*(If `/etc/resolv.conf` inside the container does not point directly to `10.1.0.254`, specify the resolved host directly via curl's `--resolve` flag:)*

```bash
docker exec -it spoke-client-app curl -i -s \
  --resolve devstoreaccount1.blob.core.windows.net:10000:20.150.1.1 \
  http://devstoreaccount1.blob.core.windows.net:10000/

```

*Expected Output:*

```text
HTTP/1.1 403 Forbidden
Content-Type: text/plain

[403 FORBIDDEN] Public network access on this storage account is disabled. Use Private Link.

```

**The Lesson:** This is the exact error teams see in Azure when they disable public access on a Storage Account and forget to link the Private DNS Zone to the VNet. The application queries the FQDN, gets routed to the public IP, and hits the public deny wall.

---

#### Experiment 2: Linking the Azure Private DNS Zone

Now we simulate creating a **Virtual Network Link** in the Azure Portal between the VNet and `privatelink.blob.core.windows.net`.

1. Swap CoreDNS to use `Corefile.linked`:

```bash
docker exec -it azure-resolver-dns cp /etc/coredns/Corefile.linked /etc/coredns/Corefile

```

2. Reload CoreDNS by restarting the container:

```bash
docker restart azure-resolver-dns

```

3. Re-run the DNS query from `spoke-client-app`:

```bash
docker exec -it spoke-client-app drill @10.1.0.254 devstoreaccount1.blob.core.windows.net

```

*Expected Output:*

```text
;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: ...
;; QUESTION SECTION:
;; devstoreaccount1.blob.core.windows.net.	IN	A

;; ANSWER SECTION:
devstoreaccount1.blob.core.windows.net.	3600	IN	CNAME	devstoreaccount1.privatelink.blob.core.windows.net.
devstoreaccount1.privatelink.blob.core.windows.net.	3600	IN	A	10.2.0.10

```

**Notice the result:**

* The CNAME chain remained identical: `devstoreaccount1.blob.core.windows.net` $\rightarrow$ `devstoreaccount1.privatelink.blob.core.windows.net`.
* But now, the recursive resolver intercepted the query and returned **`10.2.0.10`** (our Private Endpoint IP)!

---

#### Experiment 3: Verifying Seamless SDK Connection Over Private Link

Now test accessing the storage service using the exact same standard FQDN without modifying any application code or connection strings:

```bash
docker exec -it spoke-client-app curl -i -s \
  --resolve devstoreaccount1.blob.core.windows.net:10000:10.2.0.10 \
  http://devstoreaccount1.blob.core.windows.net:10000/

```

*Expected Output:*

```text
HTTP/1.1 400 Bad Request
Server: Azurite-Blob/3...
...

```

*(Azurite returned HTTP 400 because no storage command was sent, but notice the header: `Server: Azurite-Blob`!)*

Create a real container using the Azure CLI or Python SDK over the private DNS name:

```bash
docker exec -it spoke-client-app python -c "
from azure.storage.blob import BlobServiceClient

# Connection string uses the standard Azure FQDN!
conn_str = 'DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;BlobEndpoint=http://10.2.0.10:10000/devstoreaccount1;'

client = BlobServiceClient.from_connection_string(conn_str)
container = client.create_container('dns-verified-container')
print('Container created successfully over Private Link!')
"

```

*Expected Output:*

```text
Container created successfully over Private Link!

```

---

#### Experiment 4: The Hub-and-Spoke Private DNS Trap

Walk through this architectural scenario with your team:

```
[Spoke A VNet] ──(Peered)──► [Hub VNet with Private DNS Zone] ◄──(Peered)── [Spoke B VNet]

```

* **The Trap:** If you link the Private DNS Zone `privatelink.blob.core.windows.net` **only** to the Hub VNet, workloads inside `Spoke A` will **not** resolve the private IP by default!
* **Why:** In Azure, Virtual Network Links on Private DNS Zones are **non-transitive**. A VNet only gets resolution for zones explicitly linked to that specific VNet (or if custom DNS forwarders in the Hub handle all spoke DNS traffic).
* **The Rule:** You must link every spoke VNet that needs access to the Private DNS Zone, or configure custom DNS forwarders in the Hub that spokes query as their primary DNS server.

---

### Step 7: Clean Up

When you've finished testing, bring down the containers and isolated bridge networks:

```bash
docker compose down -v

```

---

### Key Enterprise Takeaways for the Team

* **Never Hardcode Private Endpoint IPs:** Connection strings should continue using standard FQDNs (`*.blob.core.windows.net`). Let the DNS CNAME chain handle resolution to private IPs.
* **The CNAME Mechanism:** Azure automatically inserts a `privatelink.<service>.core.windows.net` CNAME into public DNS when Private Link is created.
* **VNet Links Are Mandatory:** If a VNet is not linked to the `privatelink.*` Private DNS Zone, workloads will resolve the public IP, hit public firewall restrictions, and fail with HTTP 403.
* **DNS Resolution in Hub-and-Spoke:** Either link the Private DNS Zones to all spoke VNets or deploy central DNS forwarders (Azure DNS Private Resolver or CoreDNS) in the Hub to resolve queries across peering links.