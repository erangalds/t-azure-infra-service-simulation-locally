# Lab 5.3: Azure Files, Managed SMB Shares & The Port 445 ISP Block

Welcome to Lab 5.3! In previous labs, we explored object storage (Blob) and relational engines (SQL). Now we look at another core storage primitive in Azure: **Azure Files (Server Message Block / SMB 3.0)**.

Many enterprise applications—such as legacy content management systems (Wordpress, Drupal), multi-instance web farms, batch processing workers, and stateful container workloads (like Azure Kubernetes Service)—require a **shared POSIX/SMB filesystem** mounted concurrently across dozens of compute instances.

However, Azure Files introduces two significant networking traps for cloud engineers:

1. **The ISP Port 445 Blackout:** SMB protocol operates over **TCP port 445**. Because port 445 was historically targeted by worms (WannaCry, Conficker) and SMB relay attacks, **virtually all residential ISPs, mobile data carriers, and hotel Wi-Fi networks block outbound port 445 at their edge gateways**. If an engineer tries to mount Azure Files directly from their laptop over the public internet, the connection hangs and fails silently.
2. **Concurrent Multi-Mount Consistency:** Unlike block storage (Azure Managed Disks), which can typically only attach to a single VM at a time, Azure Files provides multi-writer capability (`ReadWriteMany` / RWX). Workloads must maintain reliable file-locking without data corruption.

To bypass public ISP blocks and meet enterprise compliance standards, Azure Files must be secured behind **Azure Private Endpoints** or accessed via an **IPSec VPN / ExpressRoute** terminating in the VNet.

In this lab, we will deploy a **Samba SMB 3.0 server** inside our private `spoke-data` subnet (`10.2.0.20`), simulate an external ISP port 445 filter, mount the share concurrently across multiple web nodes in `spoke-compute`, and observe real-time multi-writer consistency.

---

### What Are We Simulating?

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │           External / Untrusted WAN (Developer Remote Client)           │
 │                                                                        │
 │   ┌────────────────────────────────────────────────────────────────┐   │
 │   │  developer-laptop (Host / External Network)                    │   │
 │   │  • Attempts: mount -t cifs //203.0.113.10/shared-data          │   │
 │   │  • Result: BLOCKED BY ISP OUTBOUND PORT 445 FILTER!            │   │
 │   └───────────────────────────────┬────────────────────────────────┘   │
 └───────────────────────────────────┼────────────────────────────────────┘
                                     │
                                     ▼ (TCP Port 445 Drop Simulation)
 ┌────────────────────────────────────────────────────────────────────────┐
 │               Spoke Compute Subnet (10.1.0.0/24)                       │
 │               (Access via Private Link / Internal Routing)             │
 │                                                                        │
 │   ┌───────────────────────────┐      ┌───────────────────────────┐     │
 │   │  web-node-01 (10.1.0.11)  │      │  web-node-02 (10.1.0.12)  │     │
 │   │                           │      │                           │     │
 │   │  • Mounts /mnt/azurefiles │      │  • Mounts /mnt/azurefiles │     │
 │   │  • Writes: session-101.txt│      │  • Reads: session-101.txt │     │
 │   │  • Append log stream      │      │  • Appends lock updates   │     │
 │   └─────────────┬─────────────┘      └─────────────┬─────────────┘     │
 └─────────────────┼──────────────────────────────────┼───────────────────┘
                   │                                  │
                   └─────────────────┬────────────────┘
                                     │ (Internal TCP 445 Allowed)
                                     ▼
 ┌────────────────────────────────────────────────────────────────────────┐
 │                 Spoke Data Subnet (10.2.0.0/24)                        │
 │                                                                        │
 │   ┌────────────────────────────────────────────────────────────────┐   │
 │   │  azure-files-smb (Samba 3.0 Container)                         │   │
 │   │  Internal IP: 10.2.0.20                                        │   │
 │   │  Exported Share: //10.2.0.20/shared-data                       │   │
 │   │  Zero host ports published; accessible only via Private IP!    │   │
 │   └────────────────────────────────────────────────────────────────┘   │
 └────────────────────────────────────────────────────────────────────────┘

```

---

### Core Concepts Explained

#### 1. Azure Files Storage Tiers & Protocol Support

* **SMB 3.0/3.1.1:** The standard protocol for Windows and Linux instances. Azure Files requires SMB 3.x with **encryption in transit** enabled. If a client attempts to negotiate unencrypted SMB 2.x, Azure rejects the session.
* **NFS 4.1:** Supported on Premium Azure Files (SSD). NFS mounts require strict VNet integration and do **not** support public endpoints at all—private routing is mandatory.

#### 2. The ISP Port 445 Block Explained

TCP port 445 is the direct host port for SMB over IP. ISPs block outbound port 445 to prevent residential machines from inadvertently leaking NTLM credentials or participating in SMB-based lateral botnets.

* When mounting Azure Files over the public internet (`<storageaccount>.file.core.windows.net:445`), the connection gets dropped by the ISP edge router.
* **The Azure Fix:** Deploy a **Private Endpoint** for the `file` sub-resource. Because traffic to `10.2.0.20` stays within the Azure private VNet (or traverses an IPSec VPN on UDP 500/ESP 50), the ISP never inspects or blocks the encapsulated port 445 traffic.

#### 3. Concurrent Multi-Mount (`ReadWriteMany`)

Unlike an Azure Managed Disk (`/dev/sdX`), which can only attach to one VM at a time (unless configured as a shared clustered disk with SCSI-3 PR), SMB shares support hundreds of concurrent read-write clients using POSIX-compatible file locking (`fcntl` / `flock`).

---

### Step-by-Step Hands-on Lab

#### Step 1: Project Setup

Open Terminal on your Mac mini and create a dedicated workspace:

```bash
mkdir -p ~/azure-labs/lab5-3-azure-files
cd ~/azure-labs/lab5-3-azure-files

```

---

#### Step 2: Create the Lab Definition (`docker-compose.yml`)

Create a `docker-compose.yml` file.

Notice the architectural details:

* `azure-files-smb` uses an Alpine-based Samba container running inside `spoke-data` (`10.2.0.20`).
* We do **not** expose port 445 to the Mac host (`-p 445:445`). It exists exclusively on the private bridge.
* Two web nodes (`web-node-01` and `web-node-02`) are deployed inside `spoke-compute` (`10.1.0.0/24`). Both have `cap_add: [SYS_ADMIN, DAC_READ_SEARCH]` so the Linux kernel can mount CIFS/SMB filesystems.
* `blocked-isp-client` simulates an offsite developer machine attempting to reach the storage over the public internet, with a firewall rule dropping port 445.

```yaml
services:
  # ==========================================================
  # Azure Files Emulation: Samba SMB 3.0 (spoke-data)
  # ==========================================================
  azure-files-smb:
    image: dperson/samba:latest
    container_name: azure-files-smb
    environment:
      - USERID=1000
      - GROUPID=1000
    # Share definition: -s "share_name;path;browseable;readonly;guest;users;admins"
    command: >
      -u "azureuser;AzureStorageP@ss2026!"
      -s "shared-data;/share/data;yes;no;no;azureuser;azureuser"
      -p
    networks:
      spoke-data:
        ipv4_address: 10.2.0.20

  # ==========================================================
  # Web Compute Node 1 (spoke-compute)
  # ==========================================================
  web-node-01:
    image: nicolaka/netshoot:latest
    container_name: web-node-01
    cap_add:
      - SYS_ADMIN
      - DAC_READ_SEARCH
      - NET_ADMIN
    stdin_open: true
    tty: true
    networks:
      spoke-compute:
        ipv4_address: 10.1.0.11

  # ==========================================================
  # Web Compute Node 2 (spoke-compute)
  # ==========================================================
  web-node-02:
    image: nicolaka/netshoot:latest
    container_name: web-node-02
    cap_add:
      - SYS_ADMIN
      - DAC_READ_SEARCH
      - NET_ADMIN
    stdin_open: true
    tty: true
    networks:
      spoke-compute:
        ipv4_address: 10.1.0.12

  # ==========================================================
  # External Remote Developer Client (Simulating ISP Blocking)
  # ==========================================================
  blocked-isp-client:
    image: nicolaka/netshoot:latest
    container_name: blocked-isp-client
    cap_add:
      - NET_ADMIN
      - SYS_ADMIN
    stdin_open: true
    tty: true
    networks:
      external-wan:
        ipv4_address: 203.0.113.50

networks:
  spoke-compute:
    driver: bridge
    ipam:
      config:
        - subnet: 10.1.0.0/24
          gateway: 10.1.0.1

  spoke-data:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 10.2.0.0/24

  external-wan:
    driver: bridge
    ipam:
      config:
        - subnet: 203.0.113.0/24
          gateway: 203.0.113.1

```

---

#### Step 3: Spin Up the Stack

Launch the environment:

```bash
docker compose up -d

```

Verify that all containers are running:

```bash
docker compose ps

```

---

#### Step 4: Configure Internal Private Routing

To simulate Private Link connectivity, enable private routing between `spoke-compute` and `spoke-data` using the Docker network bridge gateway:

```bash
docker exec -it web-node-01 ip route add 10.2.0.0/24 via 10.1.0.1
docker exec -it web-node-02 ip route add 10.2.0.0/24 via 10.1.0.1

```

Verify that `web-node-01` can reach the SMB server's private IP on port 445:

```bash
docker exec -it web-node-01 nc -zv -w 2 10.2.0.20 445

```

*Expected Output:*

```text
10.2.0.20 445 (microsoft-ds) open

```

---

### Step 5: Verification & Experiments

#### Experiment 1: The ISP Port 445 Block Simulation

First, let's observe what happens when a developer tries to mount an Azure Files SMB share from an offsite network where the ISP filters port 445.

1. Access the `blocked-isp-client` container:

```bash
docker exec -it blocked-isp-client bash

```

2. Simulate an ISP edge router dropping outbound TCP 445 packets:

```bash
iptables -A OUTPUT -p tcp --dport 445 -j DROP

```

3. Attempt to verify port connectivity or mount the storage endpoint:

```bash
nc -zv -w 3 10.2.0.20 445

```

*Expected Output:*

```text
nc: connect to 10.2.0.20 port 445 (tcp) timed out: Operation timed out

```

4. Attempt to run the standard Linux `mount -t cifs` command:

```bash
mkdir -p /mnt/azurefiles
mount -t cifs //10.2.0.20/shared-data /mnt/azurefiles \
  -o username=azureuser,password=AzureStorageP@ss2026!,vers=3.0

```

*Expected Output:*

```text
mount error(110): Connection timed out
Refer to the mount.cifs(8) manual page (e.g. man mount.cifs) and kernel log messages (dmesg)

```

Exit the client:

```bash
exit

```

**The Takeaway for the Team:** When users complain that "Azure Files works fine from our Azure VMs, but our home developers cannot mount it," **do not debug storage permissions**. It is almost certainly an ISP port 445 block. The only solutions are:

* Connect via an Azure Point-to-Site (P2S) VPN.
* Access files through an Azure Private Endpoint from an internal VM or Bastion host.
* Transition to Azure Blob Storage over HTTPS (port 443).

---

#### Experiment 2: Mounting Azure Files Concurrently Across Private Web Nodes

Now let's mount the shared SMB filesystem inside our private compute tier across both web nodes simultaneously.

1. On **`web-node-01`**: Create the mount directory and mount the SMB share:

```bash
docker exec -it web-node-01 bash -c "
  mkdir -p /mnt/azurefiles
  mount -t cifs //10.2.0.20/shared-data /mnt/azurefiles \
    -o username=azureuser,password=AzureStorageP@ss2026!,vers=3.0,sec=ntlmssp
"

```

2. On **`web-node-02`**: Create the mount directory and mount the exact same share:

```bash
docker exec -it web-node-02 bash -c "
  mkdir -p /mnt/azurefiles
  mount -t cifs //10.2.0.20/shared-data /mnt/azurefiles \
    -o username=azureuser,password=AzureStorageP@ss2026!,vers=3.0,sec=ntlmssp
"

```

3. Verify that both containers have the filesystem mounted:

```bash
docker exec -it web-node-01 df -h /mnt/azurefiles
docker exec -it web-node-02 df -h /mnt/azurefiles

```

*Expected Output:*
Both output listings will show `//10.2.0.20/shared-data` mounted on `/mnt/azurefiles` with shared disk space.

---

#### Experiment 3: Verifying Multi-Writer Consistency & Shared State

Now we test real-time synchronization between the two independent compute instances.

1. From `web-node-01`, simulate a user session or image asset being written to the shared filesystem:

```bash
docker exec -it web-node-01 bash -c "
  echo '{\"session_id\": \"USR-9021\", \"cart_total\": 450.00, \"node\": \"web-node-01\"}' > /mnt/azurefiles/user-session.json
"

```

2. Immediately read that file from `web-node-02`:

```bash
docker exec -it web-node-02 cat /mnt/azurefiles/user-session.json

```

*Expected Output:*

```json
{"session_id": "USR-9021", "cart_total": 450.00, "node": "web-node-01"}

```

3. From `web-node-02`, append data to a shared audit log:

```bash
docker exec -it web-node-02 bash -c "
  echo '[LOG - web-node-02] Transaction processed at 15:30:00' >> /mnt/azurefiles/audit.log
"

```

4. Inspect the log file from `web-node-01`:

```bash
docker exec -it web-node-01 cat /mnt/azurefiles/audit.log

```

*Expected Output:*

```text
[LOG - web-node-02] Transaction processed at 15:30:00

```

Both instances share the same consistent data view without requiring a custom replication daemon, file sync utility, or rsync cron job.

---

#### Experiment 4: Multi-Node File Locking & Leases

In enterprise workloads, if two instances write to the same file simultaneously without coordination, data corruption can occur. Azure Files implements **SMB File Leases and Byte-Range Locks**.

Let's test file locking using Python’s native `fcntl` module across both nodes:

1. Open a **second terminal window** and run a script on `web-node-01` that acquires an exclusive lock (`LOCK_EX`) on a shared lockfile for 10 seconds:

```bash
docker exec -it web-node-01 python -c "
import fcntl, time
with open('/mnt/azurefiles/database.lock', 'w') as f:
    print('[Node 1] Acquiring exclusive lock...')
    fcntl.flock(f, fcntl.LOCK_EX)
    print('[Node 1] Lock acquired! Holding for 10 seconds...')
    time.sleep(10)
    fcntl.flock(f, fcntl.LOCK_UN)
    print('[Node 1] Lock released.')
"

```

2. In your **first terminal window**, immediately attempt to acquire the lock from `web-node-02`:

```bash
docker exec -it web-node-02 python -c "
import fcntl, time
with open('/mnt/azurefiles/database.lock', 'w') as f:
    print('[Node 2] Attempting to acquire lock with non-blocking check...')
    try:
        fcntl.flock(f, fcntl.LOCK_EX | fcntl.LOCK_NB)
        print('[Node 2] SUCCESS: Acquired lock.')
    except BlockingIOError:
        print('[Node 2] BLOCKED: File is currently locked by another node (web-node-01)!')
"

```

*Expected Output in Terminal 1:*

```text
[Node 2] Attempting to acquire lock with non-blocking check...
[Node 2] BLOCKED: File is currently locked by another node (web-node-01)!

```

Once the 10 seconds elapse on Node 1, Node 2 can successfully acquire the lock.

This proves that SMB file locking primitives are respected across separate compute containers over the private network.

---

### Step 6: Clean Up

Once the team finishes testing, unmount the filesystems and tear down the environment:

```bash
# Unmount CIFS shares
docker exec -it web-node-01 umount /mnt/azurefiles
docker exec -it web-node-02 umount /mnt/azurefiles

# Stop containers
docker compose down -v

```

---

### Key Architecture Takeaways for the Team

* **Blob Storage vs. Azure Files:**
* **Azure Blob Storage:** REST API-based, object storage, unlimited scale, works over port 443 (HTTPS), ideal for cloud-native apps.
* **Azure Files:** Filesystem-based (SMB/NFS), mountable directly as a local directory (`/mnt/azurefiles`), supports concurrent POSIX/SMB file locking, ideal for legacy migrations and shared compute pools.


* **The Port 445 Constraint:** Public access to Azure Files over SMB requires outbound port 445, which is blocked by almost every consumer and mobile ISP. For offsite clients or external branches, **Private Endpoints combined with VPN or ExpressRoute are mandatory**.
* **SMB Encryption in Transit:** Azure Files rejects connections that do not support SMB 3.x encryption unless explicitly toggled off in the storage account settings. Always enforce SMB 3.1.1 encryption for data-in-transit compliance.