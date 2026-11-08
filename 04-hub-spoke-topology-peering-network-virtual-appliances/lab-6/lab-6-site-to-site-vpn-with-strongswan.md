# Lab 4.6: Site-to-Site IPSec VPN with StrongSwan — Cryptographic Tunneling & ESP Inspection

Welcome to Lab 4.6! In Lab 4.5, we simulated the routing behavior of Gateway Transit. However, real enterprise connections to Azure don't travel across bare private wires—they traverse the untrusted public internet encapsulated inside an **encrypted Site-to-Site (S2S) IPSec VPN tunnel**.

When setting up an Azure VPN Gateway, you configure cryptographic policies:

* **IKEv2 (Internet Key Exchange v2)**: The control-plane handshake that authenticates both ends via a Pre-Shared Key (PSK) and negotiates encryption parameters.
* **IPSec Security Associations (SA)**: The agreed-upon cryptographic keys, hashing algorithms, and lifetimes.
* **ESP (Encapsulating Security Payload - IP Protocol 50)**: The data-plane protocol that encrypts and seals the actual customer traffic inside an unreadable outer wrapper.

In this lab, we will deploy **StrongSwan** (the standard Linux IPSec daemon used across enterprise appliances) on both sides of a simulated public internet transit bridge. Your team will observe the IKEv2 security negotiation, inspect live `ESP` packets with `tcpdump`, and prove that internal packets cannot be read on the public wire without the shared secret.

---

### What Are We Simulating?

```
 ┌───────────────────────────────────┐               ┌───────────────────────────────────┐
 │   Corporate On-Premises Branch    │               │       Azure Hub & Spoke VNet      │
 │        (192.168.1.0/24)           │               │          (10.1.0.0/24)            │
 │                                   │               │                                   │
 │  ┌─────────────────────────────┐  │               │  ┌─────────────────────────────┐  │
 │  │ onprem-payroll-srv          │  │               │  │ spoke-app-workload          │  │
 │  │ (192.168.1.4)               │  │               │  │ (10.1.0.4)                  │  │
 │  │ Plaintext HTTP: 8443        │  │               │  │ Plaintext Client            │  │
 │  └──────────────┬──────────────┘  │               │  └──────────────┬──────────────┘  │
 │                 │                 │               │                 │                 │
 │  ┌──────────────▼──────────────┐  │               │  ┌──────────────▼──────────────┐  │
 │  │ onprem-vpn-gw (StrongSwan)  │  │               │  │ hub-vpn-gw (StrongSwan)     │  │
 │  │ Internal IP: 192.168.1.254  │  │               │  │ Internal IP: 10.1.0.254     │  │
 │  │ Public WAN:  198.51.100.10  │  │               │  │ Public WAN:  198.51.100.20  │  │
 └──┴──────────────┬──────────────┴──┘               └──┴──────────────┬──────────────┴──┘
                   │                                                   │
                   └───────────────────────┬───────────────────────────┘
                                           │
                         ┌─────────────────┴─────────────────┐
                         │   Simulated Public Internet WAN   │
                         │         (198.51.100.0/24)         │
                         │                                   │
                         │   Live tcpdump inspection:        │
                         │   • IKEv2 Negotiation (UDP 500)   │
                         │   • ESP Packets (IP Protocol 50)  │
                         │   • ZERO plaintext HTTP visible!  │
                         └───────────────────────────────────┘

```

---

### Core Concepts Explained

#### 1. The Two Phases of IPSec

1. **IKEv2 Phase 1 (IKE SA):** The two VPN gateways authenticate each other using a Pre-Shared Key (`psk123!`), establish an authenticated Diffie-Hellman channel, and agree on encryption algorithms (e.g., `aes256-sha256-modp2048`).
2. **IKEv2 Phase 2 (CHILD SA):** The gateways negotiate the cryptographic parameters specifically for encrypting user traffic between the two private subnets (`192.168.1.0/24` $\leftrightarrow$ `10.1.0.0/24`).

#### 2. ESP (Encapsulating Security Payload - IP Protocol 50)

Once the tunnel is active, packets leaving `spoke-app-workload` (`10.1.0.4`) heading to `onprem-payroll-srv` (`192.168.1.4`) do **not** travel as raw TCP packets on the WAN:

* The gateway encapsulates the original packet (headers and data) into an encrypted ciphertext payload.
* It wraps this ciphertext in a new **ESP header (Protocol 50)** with outer IP headers:
* **Source IP:** `198.51.100.20` (Azure Gateway Public IP)
* **Destination IP:** `198.51.100.10` (On-Prem Gateway Public IP)


* Anyone tapping the public internet only sees `ESP` packets with arbitrary hex values. Neither the internal IP addresses nor the payload are visible.

---

### Step-by-Step Hands-on Lab

#### Step 1: Set Up Your Project Directory

Open Terminal on your Mac mini and create a dedicated workspace:

```bash
mkdir -p ~/azure-labs/lab4-6-ipsec-vpn
cd ~/azure-labs/lab4-6-ipsec-vpn

```

---

#### Step 2: Create StrongSwan Gateway Configurations

We need to configure two configuration files for each VPN gateway:

1. `swanctl.conf`: Defines the IKEv2 connection, cryptographic suites, and local/remote subnets.
2. The shared secret (`psk`): Authenticates the gateways.

##### 1. On-Premises Gateway Configuration

Create a file named `onprem-swanctl.conf`:

```ini
connections {
   azure-s2s {
      local_addrs  = 198.51.100.10
      remote_addrs = 198.51.100.20
      version = 2
      proposals = aes256-sha256-modp2048

      local {
         auth = psk
         id = 198.51.100.10
      }
      remote {
         auth = psk
         id = 198.51.100.20
      }
      children {
         net-to-net {
            local_ts  = 192.168.1.0/24
            remote_ts = 10.1.0.0/24
            esp_proposals = aes256-sha256-modp2048
            start_action = trap
         }
      }
   }
}

secrets {
   ike-azure {
      id = 198.51.100.20
      secret = "AzureCloudSecretKey2026!"
   }
}

```

##### 2. Azure Hub Gateway Configuration

Create a file named `hub-swanctl.conf`:

```ini
connections {
   onprem-s2s {
      local_addrs  = 198.51.100.20
      remote_addrs = 198.51.100.10
      version = 2
      proposals = aes256-sha256-modp2048

      local {
         auth = psk
         id = 198.51.100.20
      }
      remote {
         auth = psk
         id = 198.51.100.10
      }
      children {
         net-to-net {
            local_ts  = 10.1.0.0/24
            remote_ts = 192.168.1.0/24
            esp_proposals = aes256-sha256-modp2048
            start_action = trap
         }
      }
   }
}

secrets {
   ike-onprem {
      id = 198.51.100.10
      secret = "AzureCloudSecretKey2026!"
   }
}

```

---

#### Step 3: Create the Lab Definition (`docker-compose.yml`)

Create `docker-compose.yml` to define our 4 services across 3 isolated networks:

* `wan-internet` (`198.51.100.0/24`): The untrusted public transit line.
* `onprem-net` (`192.168.1.0/24`): The private on-premise network.
* `spoke-net` (`10.1.0.0/24`): The private cloud network.

Notice that the VPN gateways have Linux `NET_ADMIN` privileges and kernel IP forwarding enabled.

```yaml
services:
  # ==========================================================
  # On-Premises StrongSwan VPN Gateway
  # ==========================================================
  onprem-vpn-gw:
    image: nicolaka/netshoot:latest
    container_name: onprem-vpn-gw
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    sysctls:
      - net.ipv4.ip_forward=1
    volumes:
      - ./onprem-swanctl.conf:/etc/swanctl/swanctl.conf:ro
    command: >
      sh -c "charon-systemd & sleep 2 && swanctl --load-all && tail -f /dev/null"
    networks:
      wan-internet:
        ipv4_address: 198.51.100.10
      onprem-net:
        ipv4_address: 192.168.1.254

  # ==========================================================
  # Azure Hub StrongSwan VPN Gateway
  # ==========================================================
  hub-vpn-gw:
    image: nicolaka/netshoot:latest
    container_name: hub-vpn-gw
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    sysctls:
      - net.ipv4.ip_forward=1
    volumes:
      - ./hub-swanctl.conf:/etc/swanctl/swanctl.conf:ro
    command: >
      sh -c "charon-systemd & sleep 2 && swanctl --load-all && tail -f /dev/null"
    networks:
      wan-internet:
        ipv4_address: 198.51.100.20
      spoke-net:
        ipv4_address: 10.1.0.254

  # ==========================================================
  # On-Premises Payroll Workload (Port 8443)
  # ==========================================================
  onprem-payroll-srv:
    image: nicolaka/netshoot:latest
    container_name: onprem-payroll-srv
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    command: >
      sh -c "while true; do echo -e 'HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n[CONFIDENTIAL PAYROLL] Employee Salaries: Verified Encrypted Payload' | nc -l -p 8443 -q 1; done"
    networks:
      onprem-net:
        ipv4_address: 192.168.1.4

  # ==========================================================
  # Azure Cloud Workload (Client)
  # ==========================================================
  spoke-app-workload:
    image: nicolaka/netshoot:latest
    container_name: spoke-app-workload
    cap_add:
      - NET_ADMIN
    stdin_open: true
    tty: true
    networks:
      spoke-net:
        ipv4_address: 10.1.0.4

networks:
  wan-internet:
    driver: bridge
    ipam:
      config:
        - subnet: 198.51.100.0/24
          gateway: 198.51.100.1

  onprem-net:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 192.168.1.0/24

  spoke-net:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 10.1.0.0/24

```

---

#### Step 4: Spin Up the Stack

Launch the environment:

```bash
docker compose up -d

```

Verify that all 4 containers are running:

```bash
docker compose ps

```

---

#### Step 5: Configure Subnet Routes Pointing to Gateways

Each endpoint needs to know that traffic for the opposite network should head to its local VPN gateway:

1. On `spoke-app-workload`: Route `192.168.1.0/24` to `10.1.0.254`:

```bash
docker exec -it spoke-app-workload ip route add 192.168.1.0/24 via 10.1.0.254

```

2. On `onprem-payroll-srv`: Route `10.1.0.0/24` to `192.168.1.254`:

```bash
docker exec -it onprem-payroll-srv ip route add 10.1.0.0/24 via 192.168.1.254

```

---

### Step 6: Verification & Experiments

#### Experiment 1: Initiating the IKEv2 Handshake & Verifying Security Associations

Let's manually bring up the connection from the Azure Hub Gateway and inspect the active cryptographic Security Associations (SAs).

1. Trigger the tunnel connection from `hub-vpn-gw`:

```bash
docker exec -it hub-vpn-gw swanctl --initiate --child net-to-net

```

*Expected output:*

```text
[IKE] initiating IKE_SA onprem-s2s[1] to 198.51.100.10
[IKE] establishing CHILD_SA net-to-net{1}
[IKE] IKE_SA onprem-s2s[1] established between 198.51.100.20[198.51.100.20]...198.51.100.10[198.51.100.10]
[IKE] CHILD_SA net-to-net{1} established with SPIs c12a34b5_i d56e78f9_o and TS 10.1.0.0/24 === 192.168.1.0/24
connection 'net-to-net' established successfully

```

2. Inspect the established SAs:

```bash
docker exec -it hub-vpn-gw swanctl --list-sas

```

*Expected output:*
Notice the negotiated cipher suites:

* **IKE Proposal:** `AES_CBC_256 / HMAC_SHA2_256 / MODP_2048`
* **ESP Child SA:** `AES_CBC_256 / HMAC_SHA2_256`
* **Traffic Selectors (TS):** `10.1.0.0/24 === 192.168.1.0/24`

This matches what you see in the Azure Portal under **VPN Connection -> Security Association**.

---

#### Experiment 2: The Core Test — Live ESP Packet Sniffing on the Public WAN

Now we will prove that unencrypted application traffic never touches the simulated public internet.

1. Open a **second terminal window** on your Mac mini and attach to `hub-vpn-gw` to monitor the public WAN interface (`eth0` on `198.51.100.0/24`):

```bash
docker exec -it hub-vpn-gw tcpdump -n -i eth0 -X

```

2. In your **first terminal window**, send an HTTP request from `spoke-app-workload` across the tunnel to `onprem-payroll-srv`:

```bash
docker exec -it spoke-app-workload curl -s http://192.168.1.4:8443

```

*Expected output in Terminal 1:*

```text
[CONFIDENTIAL PAYROLL] Employee Salaries: Verified Encrypted Payload

```

The application call succeeded!

3. Now inspect **Terminal 2 (the tcpdump output on the public WAN)**:

```text
11:20:14.521892 IP 198.51.100.20 > 198.51.100.10: ESP(spi=0xd56e78f9,seq=1), length 164
        0x0000:  4500 00a4 1234 0000 4032 5b2a c633 6414  E...@2[*.3d.
        0x0010:  c633 640a d56e 78f9 0000 0001 9a8b 7c6d  .3d..nx.......|m
        0x0020:  e123 4567 89ab cdef 0123 4567 89ab cdef  .#Eg.....#Eg....
        0x0030:  a1b2 c3d4 e5f6 0718 293a 4b5c 6d7e 8f90  ........):K\m~..

```

**What this proves:**

* The protocol indicator is `ESP` (`IP Protocol 50`).
* Notice the hex and ASCII dump on the right: You see **random ciphertext**, not the plaintext string `[CONFIDENTIAL PAYROLL]`!
* You also do **not** see the source IP `10.1.0.4` or destination IP `192.168.1.4`. The entire IP frame was encrypted and hidden behind the gateway's public IP headers (`198.51.100.20` $\rightarrow$ `198.51.100.10`).

Press `Ctrl+C` in Terminal 2 to stop `tcpdump`.

---

#### Experiment 3: Sniffing the Internal Private Interface

To contrast with the public WAN trace, let's sniff on the internal interface where traffic hasn't been encrypted yet.

1. In Terminal 2, run `tcpdump` on the internal private interface (`eth1` on `10.1.0.0/24`):

```bash
docker exec -it hub-vpn-gw tcpdump -n -i eth1 -A "port 8443"

```

2. In Terminal 1, run the curl command again:

```bash
docker exec -it spoke-app-workload curl -s http://192.168.1.4:8443

```

3. Look at Terminal 2:

```text
IP 192.168.1.4.8443 > 10.1.0.4.48291: Flags [P.], seq 1:115, ack 82 ...
....[CONFIDENTIAL PAYROLL] Employee Salaries: Verified Encrypted Payload

```

On the internal VNet interface (`eth1`), the traffic is plaintext. The moment it leaves `eth0` bound for on-prem, the IPSec kernel module intercepts the packet, encrypts it with the negotiated SA keys, and emits it as ESP.

Press `Ctrl+C` in Terminal 2.

---

### Step 7: Clean Up

When you've finished the exercises, shut down the lab environment:

```bash
docker compose down

```

---

### Key Takeaways for the Team

* **Control Plane vs. Data Plane:** IKEv2 (UDP 500/4500) only sets up the tunnel. The actual user data is carried over **ESP (IP Protocol 50)**.
* **Firewall Awareness:** If an on-premise perimeter firewall blocks IP Protocol 50 or UDP 500/4500, the IPSec tunnel will either fail to form or drop all application payloads.
* **Traffic Selectors:** Both sides must agree on the local and remote subnets (`10.1.0.0/24` $\leftrightarrow$ `192.168.1.0/24`). A mismatch in traffic selectors is one of the most common reasons an Azure S2S VPN reports `Connected` while traffic fails to pass.