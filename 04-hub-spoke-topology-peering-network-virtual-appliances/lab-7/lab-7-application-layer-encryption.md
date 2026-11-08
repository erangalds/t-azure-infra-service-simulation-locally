# Lab 4.7: Application-Layer Encryption — End-to-End TLS & The Firewall Blind Spot

Welcome to Lab 4.7! In Lab 4.6, we looked at **network-layer encryption** using Site-to-Site IPSec tunnels (ESP). That protected traffic across the public WAN between two gateways. But once inside the VNet, the packets were plaintext.

Now let’s look at the other side of the zero-trust coin: **application-layer encryption (TLS/HTTPS)**.

In a modern enterprise zero-trust model, organizations mandate **End-to-End Encryption in Transit (E2E TLS)**. Workloads encrypt data directly at the application layer using TLS certificates before putting bytes on the wire.

This provides security, but it creates a major operational challenge for cloud architects: **The Firewall Blind Spot**.

* If a packet is encrypted end-to-end with TLS, standard Layer 4 and Layer 7 firewalls (including Azure Firewall Standard) **cannot inspect the payload, headers, or request URI**.
* To the firewall, it is just an opaque stream of encrypted TLS records on port 8443.
* Unless an enterprise deploys an **SSL/TLS Forward Proxy** with certificate interception (like Azure Firewall Premium with TLS Termination / Private CA inspection), the firewall is completely blind to data exfiltration or malicious commands inside the session.

In this lab, we will generate a self-signed TLS certificate, wrap our mock payroll service in an **Nginx HTTPS engine**, route traffic across our Hub NVA, and use `tcpdump` to verify why the firewall cannot see what is inside the payload.

---

### What Are We Simulating?

```
 ┌──────────────────────────────────────────────────────────────┐
 │         Spoke Compute / Client Subnet (10.1.0.0/24)          │
 │                                                              │
 │   ┌──────────────────────────────────────────────────────┐   │
 │   │               spoke-app-workload (10.1.0.4)          │   │
 │   │                                                      │   │
 │   │   • Initiates TLS 1.3 Handshake                      │   │
 │   │   • Encrypts HTTP request with Server Public Key     │   │
 │   └──────────────────────────┬───────────────────────────┘   │
 └──────────────────────────────┼───────────────────────────────┘
                                │
                                ▼ (Encrypted TLS Stream on TCP 8443)
 ┌──────────────────────────────────────────────────────────────┐
 │                    Hub VNet (10.0.0.0/24)                    │
 │                                                              │
 │   ┌──────────────────────────────────────────────────────┐   │
 │   │         Central Hub NVA / Firewall (10.0.0.4)        │   │
 │   │                 (hub-firewall-nva)                   │   │
 │   │                                                      │   │
 │   │   The Firewall Blind Spot:                           │   │
 │   │   • Sees TCP 8443 3-way handshake                    │   │
 │   │   • Sees TLS ClientHello / ServerHello               │   │
 │   │   • CANNOT read HTTP headers or URI!                 │   │
 │   │   • CANNOT read confidential payroll payload!        │   │
 │   └──────────────────────────┬───────────────────────────┘   │
 └──────────────────────────────┼───────────────────────────────┘
                                │
                                ▼ (Encrypted TLS Stream on TCP 8443)
 ┌──────────────────────────────────────────────────────────────┐
 │             On-Prem / Data Subnet (10.2.0.0/24)              │
 │                                                              │
 │   ┌──────────────────────────────────────────────────────┐   │
 │   │               onprem-payroll-tls (10.2.0.4)          │   │
 │   │                                                      │   │
 │   │   • Listens on HTTPS (Port 8443)                     │   │
 │   │   • Holds Private Key to decrypt TLS stream          │   │
 │   │   • Returns sensitive payload over encrypted socket  │   │
 │   └──────────────────────────────────────────────────────┘   │
 └──────────────────────────────────────────────────────────────┘

```

---

### Core Concepts Explained

#### 1. Transport Layer Security (TLS 1.3)

When `spoke-app-workload` connects to `[https://10.2.0.4:8443](https://10.2.0.4:8443)`:

1. **TCP Handshake:** Establishes Layer 4 connectivity (SYN $\rightarrow$ SYN-ACK $\rightarrow$ ACK).
2. **TLS Handshake:**
* **ClientHello:** Client offers supported cipher suites.
* **ServerHello + Certificate:** Server sends its public certificate.
* **Key Exchange:** Both sides establish ephemeral symmetric session keys (e.g., using ECDHE).


3. **Application Data:** All subsequent HTTP verbs (`GET /payroll/salaries`), response headers, and JSON payloads are encrypted with the symmetric session key.

#### 2. Network-Layer (IPSec) vs. Application-Layer (TLS)

| Attribute | Network-Layer (Lab 4.6 IPSec) | Application-Layer (Lab 4.7 TLS) |
| --- | --- | --- |
| **Where Encryption Happens** | At the router / VPN Gateway NIC | Inside the application runtime / web server |
| **Protected Scope** | Only between the two gateways | **End-to-End** (from client container to server container) |
| **Payload Visibility in VNet** | Plaintext inside the VNet subnets | **Fully encrypted everywhere**, even across internal subnets |
| **Firewall Inspection** | Firewall decrypts traffic automatically | Firewall **blind** unless acting as an explicit TLS proxy with CA certificates |

---

### Step-by-Step Hands-on Lab

#### Step 1: Set Up Your Project Directory

Open Terminal on your Mac mini and create a dedicated workspace:

```bash
mkdir -p ~/azure-labs/lab4-7-tls-encryption
cd ~/azure-labs/lab4-7-tls-encryption

```

---

#### Step 2: Generate the Self-Signed TLS Certificate

Let’s generate an X.509 certificate and private key using OpenSSL.

Run this command in your terminal:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout server.key \
  -out server.crt \
  -subj "/C=US/ST=Cloud/L=Azure/O=Enterprise/CN=payroll.internal"

```

Verify that both `server.key` (the private key) and `server.crt` (the public certificate) exist in your directory:

```bash
ls -l server.*

```

---

#### Step 3: Create the Nginx HTTPS Configuration

Create a configuration file named `nginx-tls.conf`.

This instructs Nginx to terminate SSL/TLS on port 8443 and return a mock confidential payroll JSON document:

```nginx
events { worker_connections 1024; }

http {
    server {
        listen 8443 ssl;
        server_name payroll.internal;

        ssl_certificate     /etc/nginx/certs/server.crt;
        ssl_certificate_key /etc/nginx/certs/server.key;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        location /api/salaries {
            default_type application/json;
            return 200 '{"department": "Executive", "payroll_records": [{"emp_id": 101, "name": "Alice", "salary": 185000}, {"emp_id": 102, "name": "Bob", "salary": 165000}], "status": "CONFIDENTIAL_PAYROLL"}';
        }

        location / {
            default_type text/plain;
            return 200 "HTTPS Payroll Server Active. Query /api/salaries for payroll records.\n";
        }
    }
}

```

---

#### Step 4: Create the Lab Definition (`docker-compose.yml`)

Create `docker-compose.yml` to define our 3 services across 3 isolated subnets:

* `spoke-compute` (`10.1.0.0/24`): The client container.
* `hub-vnet` (`10.0.0.0/24`): The central inspection router/NVA.
* `spoke-data` (`10.2.0.0/24`): The HTTPS backend server.

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
      spoke-data:
        ipv4_address: 10.2.0.254

  # ==========================================================
  # Spoke 1: Client Workload
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
  # Spoke 2: Confidential Payroll Service with TLS
  # ==========================================================
  onprem-payroll-tls:
    image: nginx:alpine
    container_name: onprem-payroll-tls
    volumes:
      - ./nginx-tls.conf:/etc/nginx/nginx.conf:ro
      - ./server.crt:/etc/nginx/certs/server.crt:ro
      - ./server.key:/etc/nginx/certs/server.key:ro
    networks:
      spoke-data:
        ipv4_address: 10.2.0.4

networks:
  hub-vnet:
    driver: bridge
    ipam:
      config:
        - subnet: 10.0.0.0/24
          gateway: 10.0.0.1

  spoke-compute:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 10.1.0.0/24

  spoke-data:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 10.2.0.0/24

```

---

#### Step 5: Spin Up the Stack

Launch the environment:

```bash
docker compose up -d

```

Verify that all three containers are running:

```bash
docker compose ps

```

---

#### Step 6: Configure User-Defined Routes (UDRs)

Just like in our previous Hub-and-Spoke labs, tell the spokes to route traffic across the Hub NVA:

1. Configure `spoke-app-workload` to route traffic for `10.2.0.0/24` via `10.1.0.254`:

```bash
docker exec -it spoke-app-workload ip route add 10.2.0.0/24 via 10.1.0.254

```

2. Enable routing on `onprem-payroll-tls` by giving it a return route to `10.1.0.0/24` via `10.2.0.254`:

```bash
docker exec -it onprem-payroll-tls ip route add 10.1.0.0/24 via 10.2.0.254

```

---

### Step 7: Verification & Experiments

#### Experiment 1: Verifying the Encrypted Application Call

From `spoke-app-workload`, make a secure API call over HTTPS to fetch the payroll data.

> **Note on `-k` / `--insecure`:** Because we generated a self-signed certificate, we pass `-k` so curl trusts our local certificate.

```bash
docker exec -it spoke-app-workload curl -k -s https://10.2.0.4:8443/api/salaries

```

*Expected Output:*

```json
{"department": "Executive", "payroll_records": [{"emp_id": 101, "name": "Alice", "salary": 185000}, {"emp_id": 102, "name": "Bob", "salary": 165000}], "status": "CONFIDENTIAL_PAYROLL"}

```

The client established a TLS session, queried the API, and received the confidential records.

---

#### Experiment 2: Inspecting the Traffic on the Hub Firewall (`tcpdump`)

Now let's see what the central Hub Firewall sees while this transaction is happening.

1. Open a **second terminal window** on your Mac mini and attach to `hub-firewall-nva` to sniff packets traversing port 8443 with ASCII payload decoding enabled (`-A`):

```bash
docker exec -it hub-firewall-nva tcpdump -n -i any -A "port 8443"

```

2. In your **first terminal window**, run the curl command again:

```bash
docker exec -it spoke-app-workload curl -k -s https://10.2.0.4:8443/api/salaries

```

3. Look closely at **Terminal 2 (the tcpdump trace on the firewall)**:

```text
11:35:10.100 IP 10.1.0.4.49218 > 10.2.0.4.8443: Flags [S], seq 182910291 ...
11:35:10.101 IP 10.2.0.4.8443 > 10.1.0.4.49218: Flags [S.], seq 918273918 ...
11:35:10.103 IP 10.1.0.4.49218 > 10.2.0.4.8443: Flags [P.], seq 1:518 ...
.....E.....@.@.0...
...d....payroll.internal...............
.....+.-...3...0..
11:35:10.108 IP 10.2.0.4.8443 > 10.1.0.4.49218: Flags [P.], seq 1:1240 ...
.....E..L..@.@.....
.....34......q.w.e.r.t.y....#$%.^&*()_+......

```

**What this proves:**

* In the initial `ClientHello` packet, the firewall can see the Server Name Indication (SNI) string `payroll.internal`.
* But look at the data packets: **You see pure encrypted binary garbage!**
* Search the capture trace: You will **NOT** see the HTTP URI `/api/salaries`, nor will you see the employee names `Alice` or `Bob`, nor their salaries.
* The firewall forwarded the packets strictly as raw Layer 4 TCP streams. It is **blind** to the contents of the application layer.

Press `Ctrl+C` in Terminal 2 to stop `tcpdump`.

---

#### Experiment 3: Can the Firewall Block Specific HTTP URIs Without TLS Inspection?

Let’s demonstrate why cloud security architects require **TLS Inspection (Azure Firewall Premium)** to enforce URL-level policies.

Suppose company policy states:

> *"Employees may view the general payroll portal (`GET /`), but access to `/api/salaries` is strictly blocked at the network perimeter."*

Try configuring a standard firewall rule on `hub-firewall-nva`:

* A standard firewall operates at **Layers 3 and 4** (IP and Port) or can only inspect the unencrypted **SNI header** (Domain name).
* Because the HTTP path `/api/salaries` is encrypted inside the TLS stream, the firewall **cannot see the URL path**.
* Its only choices are:
1. Block port 8443 entirely (which breaks the whole service).
2. Allow port 8443 (which allows access to `/api/salaries`).



This illustrates why standard firewalls cannot perform Layer 7 URL filtering on HTTPS traffic without **TLS Termination / SSL Forward Proxying**.

---

#### Experiment 4: The Enterprise Solution — How Azure Firewall Premium Fixes This

Walk through this concept with your team:

```
[Client] ──(TLS Session 1)──► [ Azure Firewall Premium ] ──(TLS Session 2)──► [ Backend Server ]
                              (Terminates TLS with CA)
                              (Inspects /api/salaries)
                              (Re-encrypts to Backend)

```

1. **Enterprise CA (Certificate Authority):** A private enterprise root CA is installed on all corporate client machines and stored in **Azure Key Vault**.
2. **Intermediate Decryption:** Azure Firewall Premium generates a dynamic certificate signed by that trusted CA on the fly.
3. **Inspection:** The firewall terminates the client's TLS connection, inspects the plaintext HTTP request, checks for malware or restricted paths (`/api/salaries`), and if approved, opens a *new* TLS connection to the destination server.

---

### Step 8: Clean Up

Once the team finishes testing, tear down the containers and isolated bridge networks:

```bash
docker compose down

```

---

### Key Takeaways for the Team

* **End-to-End Encryption Protects Against Snooping:** Even if an attacker gains root access to the central routing appliance (Hub NVA), they cannot read confidential application payloads protected by TLS.
* **The Inspection Dilemma:** When applications encrypt traffic with TLS, standard Layer 4 firewalls can only inspect the IP 5-tuple and the Server Name Indication (SNI). They cannot inspect URL paths, HTTP query parameters, or payloads.
* **TLS Termination Requirements:** To enforce deep packet inspection or URL filtering on encrypted sessions, cloud landing zones must deploy appliances capable of TLS termination and re-encryption (such as Azure Firewall Premium or an Application Gateway with WAF).