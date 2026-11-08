# Lab 3.2: Advanced Ingress — Health Probes, Sticky Sessions & Multi-Site Routing

Welcome to Lab 3.2! In Lab 3, we compared raw Layer 4 TCP balancing against Layer 7 URL-path routing. Now we’re tackling the production issues that cause silent outages and user-session bugs:

1. **The "Zombie Server" Trap:** Why a TCP health probe thinks a crashed web application throwing `HTTP 500` errors is completely fine.
2. **Cookie-Based Session Affinity (Sticky Sessions):** How to keep a user pinned to the same backend node without relying on source IP hashing.
3. **Multi-Site Hosting:** Running multiple domain names (`portal.local` vs. `billing.local`) through a single public IP and port.

We will simulate this locally on your Mac mini by pairing **HAProxy (L4)** with **Nginx (L7)** against a set of backend application services.

---

### What Are We Simulating?

```
                     Incoming Requests (Port 8080 & Port 9090)
                                        │
        ┌───────────────────────────────┴───────────────────────────────┐
        │                                                               │
        ▼ (Port 8080)                                                   ▼ (Port 9090)
┌───────────────────────────────┐                               ┌───────────────────────────────┐
│ Layer 4: Azure Load Balancer  │                               │ Layer 7: Application Gateway  │
│ (HAProxy TCP Mode)            │                               │ (Nginx Reverse Proxy)         │
│                               │                               │                               │
│ • Probe: Pure TCP Handshake   │                               │ • Probe: GET /healthz == 200  │
│ • Zombie Node 01 throws 500:  │                               │ • Zombie Node 01 throws 500:  │
│   LB says "Port open! ROUTE!" │                               │   AppGW pulls node instantly! │
│ • No cookie support           │                               │ • Sticky Cookie: SERVERID     │
│ • No host-header inspection   │                               │ • Host: portal vs billing     │
└───────────────┬───────────────┘                               └───────────────┬───────────────┘
                │                                                               │
                ├───────────────────────┬───────────────────────┬───────────────┤
                ▼                       ▼                       ▼               ▼
       ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐ ┌─────────────────┐
       │   app-zombie    │     │   app-healthy   │     │   app-portal    │ │   app-billing   │
       │  (10.10.2.11)   │     │  (10.10.2.12)   │     │  (10.10.2.21)   │ │  (10.10.2.22)   │
       │ Port 80: 500 Err│     │ Port 80: 200 OK │     │ Port 80: Portal │ │ Port 80: Billing│
       │ TCP Port: OPEN  │     │ TCP Port: OPEN  │     │                 │ │                 │
       └─────────────────┘     └─────────────────┘     └─────────────────┘ └─────────────────┘

```

---

### Core Concepts Explained

#### 1. TCP Probes vs. HTTP Status Code Probes

* **Layer 4 Azure Load Balancer:** Its default health probe initiates a TCP 3-way handshake (`SYN` $\rightarrow$ `SYN-ACK` $\rightarrow$ `ACK`). As long as the operating system kernel accepts the socket, the load balancer considers the node healthy—even if the web server process inside is returning `HTTP 500 Internal Server Error` or a database connection pool is exhausted.
* **Layer 7 Azure Application Gateway:** Sends an active HTTP request (`GET /healthz`) and inspects the HTTP response status code (requiring a range like `200-399`). If a node returns 500, the Gateway pulls it from the backend pool immediately.

#### 2. Session Affinity (Sticky Sessions)

* **Layer 4 Limitations:** Because an Azure Load Balancer cannot read HTTP headers, it can only hash the **Source IP** (2-tuple or 3-tuple). If hundreds of corporate users sit behind an egress NAT gateway or proxy, they all share one public IP and get dumped onto a single overloaded server.
* **Layer 7 Cookie Affinity:** An Application Gateway inspects or injects an HTTP cookie (e.g., `ApplicationGatewayAffinity`). As long as the client returns that cookie, the Gateway routes them back to the exact same backend server, preserving in-memory user sessions.

#### 3. Multi-Site Hosting (Host Header Routing)

* Without an L7 gateway, running 5 distinct websites requires 5 public IP addresses or non-standard ports.
* With an Application Gateway, a single listener on port 80/443 reads the HTTP `Host:` header (`portal.company.com` vs. `billing.company.com`) and routes to completely distinct backend pools behind a single public IP.

---

### Step-by-Step Hands-on Lab

#### Step 1: Project Setup

Open Terminal on your Mac mini and create a dedicated workspace:

```bash
mkdir -p ~/azure-labs/lab3-2-advanced-ingress
cd ~/azure-labs/lab3-2-advanced-ingress

```

---

#### Step 2: Create the HAProxy (Layer 4) Configuration

Create a file named `haproxy.cfg`.

This configures HAProxy in pure TCP mode. Its health probe only verifies that port 80 is listening via `option tcp-check`:

```haproxy
global
    log stdout format raw local0

defaults
    log     global
    mode    tcp
    timeout connect 3000ms
    timeout client  30000ms
    timeout server  30000ms

frontend l4_inbound
    bind *:8080
    default_backend l4_pool

backend l4_pool
    balance roundrobin
    mode tcp
    # TCP-only health probe: Only tests if the port accepts connections
    option tcp-check
    server zombie  10.10.2.11:80 check inter 2000ms rise 2 fall 2
    server healthy 10.10.2.12:80 check inter 2000ms rise 2 fall 2

```

---

#### Step 3: Create the Nginx (Layer 7) Configuration

Create a file named `nginx.conf`.

This configures:

1. **Multi-Site Listeners:** Inspecting the `Host` header to separate `portal.local` and `billing.local`.
2. **Cookie-Based Sticky Sessions:** Sticking requests carrying the `SERVERID` cookie.
3. **HTTP Health Fallback:** Pulling failed nodes out of rotation.

```nginx
events { worker_connections 1024; }

http {
    # -------------------------------------------------------------
    # Pool with Sticky Sessions (Cookie-Based Affinity)
    # -------------------------------------------------------------
    upstream sticky_app_pool {
        # Pins client to backend based on the SERVERID cookie value
        hash $cookie_SERVERID consistent;
        server 10.10.2.11:80 max_fails=2 fail_timeout=5s;
        server 10.10.2.12:80 max_fails=2 fail_timeout=5s;
    }

    # -------------------------------------------------------------
    # Upstream Pools for Multi-Site Hosting
    # -------------------------------------------------------------
    upstream portal_pool {
        server 10.10.2.21:80;
    }

    upstream billing_pool {
        server 10.10.2.22:80;
    }

    # =============================================================
    # Listener A: General Workload with Session Affinity (Port 9090)
    # =============================================================
    server {
        listen 9090;
        server_name app.local default_server;

        location / {
            proxy_pass http://sticky_app_pool;
            proxy_next_upstream error timeout http_500 http_502 http_503;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }

    # =============================================================
    # Listener B: Multi-Site - Portal Application (Port 9090)
    # =============================================================
    server {
        listen 9090;
        server_name portal.local;

        location / {
            proxy_pass http://portal_pool;
            proxy_set_header Host $host;
        }
    }

    # =============================================================
    # Listener C: Multi-Site - Billing Application (Port 9090)
    # =============================================================
    server {
        listen 9090;
        server_name billing.local;

        location / {
            proxy_pass http://billing_pool;
            proxy_set_header Host $host;
        }
    }
}

```

---

#### Step 4: Create the Lab Definition (`docker-compose.yml`)

Create `docker-compose.yml` to define our 6 containers:

```yaml
services:
  # ==========================================================
  # Layer 4 Load Balancer (HAProxy)
  # ==========================================================
  l4-lb:
    image: haproxy:alpine
    container_name: l4-lb
    ports:
      - "8080:8080"
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    networks:
      vnet-backend:
        ipv4_address: 10.10.2.4

  # ==========================================================
  # Layer 7 Application Gateway (Nginx)
  # ==========================================================
  l7-appgw:
    image: nginx:alpine
    container_name: l7-appgw
    ports:
      - "9090:9090"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      vnet-backend:
        ipv4_address: 10.10.2.5

  # ==========================================================
  # Workload 1: The "Zombie Server" (Port 80 open, throws HTTP 500)
  # ==========================================================
  app-zombie:
    image: nicolaka/netshoot:latest
    container_name: app-zombie
    stdin_open: true
    tty: true
    command: >
      sh -c "while true; do echo -e 'HTTP/1.1 500 Internal Server Error\r\nContent-Type: text/plain\r\n\r\n[CRITICAL FAILURE] Database pool exhausted on Node 1' | nc -l -p 80 -q 1; done"
    networks:
      vnet-backend:
        ipv4_address: 10.10.2.11

  # ==========================================================
  # Workload 2: Healthy Server (Port 80 open, returns HTTP 200)
  # ==========================================================
  app-healthy:
    image: nicolaka/netshoot:latest
    container_name: app-healthy
    stdin_open: true
    tty: true
    command: >
      sh -c "while true; do echo -e 'HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n[HEALTHY] Web Node 2 responding cleanly' | nc -l -p 80 -q 1; done"
    networks:
      vnet-backend:
        ipv4_address: 10.10.2.12

  # ==========================================================
  # Workload 3: Multi-Site Portal App
  # ==========================================================
  app-portal:
    image: nicolaka/netshoot:latest
    container_name: app-portal
    stdin_open: true
    tty: true
    command: >
      sh -c "while true; do echo -e 'HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n[PORTAL SERVICE] Welcome to the Enterprise Portal' | nc -l -p 80 -q 1; done"
    networks:
      vnet-backend:
        ipv4_address: 10.10.2.21

  # ==========================================================
  # Workload 4: Multi-Site Billing App
  # ==========================================================
  app-billing:
    image: nicolaka/netshoot:latest
    container_name: app-billing
    stdin_open: true
    tty: true
    command: >
      sh -c "while true; do echo -e 'HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n[BILLING SERVICE] Welcome to Payment Processing' | nc -l -p 80 -q 1; done"
    networks:
      vnet-backend:
        ipv4_address: 10.10.2.22

networks:
  vnet-backend:
    driver: bridge
    ipam:
      config:
        - subnet: 10.10.2.0/24
          gateway: 10.10.2.1

```

---

#### Step 5: Spin Up the Stack

Launch the environment:

```bash
docker compose up -d

```

Verify that all 6 containers are running:

```bash
docker compose ps

```

---

### Step 6: Verification & Experiments

#### Experiment 1: The "Zombie Server" Trap (L4 vs. L7 Health Probes)

Let's test what happens when an application crashes internally and begins returning `HTTP 500`, while the operating system socket remains open.

1. Send requests to the **Layer 4 Load Balancer** (`http://localhost:8080`):

```bash
curl -i -s http://localhost:8080
curl -i -s http://localhost:8080

```

*Expected Result:*

* **Request 1:** `HTTP/1.1 500 Internal Server Error` $\rightarrow$ `[CRITICAL FAILURE] Database pool exhausted on Node 1`
* **Request 2:** `HTTP/1.1 200 OK` $\rightarrow$ `[HEALTHY] Web Node 2 responding cleanly`

**Why this happened:** The L4 Load Balancer's health check only verifies that TCP port 80 completes a handshake. Because the Linux kernel on `app-zombie` accepts TCP connections, the L4 balancer marks it healthy and routes half of your users directly into an HTTP 500 crash.

2. Now send requests to the **Layer 7 Application Gateway** (`http://localhost:9090`):

```bash
curl -i -s http://localhost:9090
curl -i -s http://localhost:9090

```

*Expected Result:*
Both requests return:

```text
HTTP/1.1 200 OK
[HEALTHY] Web Node 2 responding cleanly

```

**Why this happened:** The Layer 7 Gateway evaluates HTTP protocol health. Through `proxy_next_upstream http_500`, the moment Node 1 responds with an HTTP 500, the Gateway catches the application-level failure, marks Node 1 dead, and automatically retries the user on the healthy node.

---

#### Experiment 2: Cookie-Based Session Affinity (Sticky Sessions)

Now let's verify how an Application Gateway keeps stateful user sessions pinned to a single backend host.

In our Nginx setup, requests carrying a specific cookie are consistently routed using hash affinity.

1. Send three requests without a cookie header:

```bash
curl -s http://localhost:9090
curl -s http://localhost:9090

```

Notice how traffic balances across available healthy backends.

2. Now send multiple requests with a specific `SERVERID` session cookie:

```bash
curl -s --cookie "SERVERID=node2_session" http://localhost:9090
curl -s --cookie "SERVERID=node2_session" http://localhost:9090
curl -s --cookie "SERVERID=node2_session" http://localhost:9090

```

*Expected Result:* Every single request hits the exact same backend instance.

**Takeaway for the team:** In Azure Application Gateway, enabling **Cookie-Based Affinity** automatically injects an `ApplicationGatewayAffinity` cookie into the user's browser. As long as that cookie is present, subsequent requests stay pinned to that specific backend instance, ensuring carts and local login sessions don't get lost.

---

#### Experiment 3: Multi-Site Hosting on a Single Frontend Port

Let's see how an Application Gateway hosts multiple different web domains on one IP address and port (`9090`).

1. Send an HTTP request specifying the `portal.local` domain in the `Host:` header:

```bash
curl -s -H "Host: portal.local" http://localhost:9090

```

*Expected Output:*

```text
[PORTAL SERVICE] Welcome to the Enterprise Portal

```

2. Send an HTTP request to the exact same IP and port, but specify `billing.local`:

```bash
curl -s -H "Host: billing.local" http://localhost:9090

```

*Expected Output:*

```text
[BILLING SERVICE] Welcome to Payment Processing

```

3. Send a request with no recognized host header (or a default IP):

```bash
curl -s http://localhost:9090

```

*Expected Output:* Falls back to the default general application pool.

**Takeaway for the team:** An Azure Application Gateway can bind multiple **Multi-site Listeners** to a single frontend IP. By inspecting the HTTP `Host:` header (or Server Name Indication / SNI in TLS), it routes different company domain names to separate backend pools without requiring multiple public IP addresses.

---

### Step 7: Clean Up

When you've finished the experiments, shut down the stack:

```bash
docker compose down

```

---

### Key Comparison Summary for the Team

| Scenario | Azure Load Balancer (Layer 4) | Azure Application Gateway (Layer 7) |
| --- | --- | --- |
| **Backend returns HTTP 500** | Considers node healthy (TCP port is open). Continues sending broken traffic. | Detects application error via HTTP probe (`200-399`). Pulls node out of rotation. |
| **User Session Persistence** | Source IP hash only (2-tuple/3-tuple). Fails behind corporate NAT/proxies. | Cookie-based session affinity (`ApplicationGatewayAffinity`). Pins individual browser sessions. |
| **Hosting Multiple Domains** | Requires a distinct Public Frontend IP per domain. | Single Public IP hosts multiple domains using Host-header routing & SNI. |