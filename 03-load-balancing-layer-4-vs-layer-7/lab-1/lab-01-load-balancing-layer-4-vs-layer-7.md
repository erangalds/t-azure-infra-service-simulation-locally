# Lab 3: Load Balancing — Layer 4 (Azure Load Balancer) vs. Layer 7 (Application Gateway)

Welcome to Lab 3! Now that we have subnets locked down and understand how security rules filter packets, let's explore how outside users actually reach our workloads: **Load Balancers**.

In Azure, one of the first design choices an enterprise architect makes is picking the right ingress service. You’ll usually see two heavyweights in every landing zone:

1. **Azure Load Balancer (Layer 4)**: Lightning fast, dumb as a rock, operates down at the transport layer.
2. **Azure Application Gateway (Layer 7)**: Smart, inspects every HTTP header, terminates SSL, routes based on URL paths, and can run a Web Application Firewall (WAF).

Choosing the wrong one leads to awkward architectural hurdles—like trying to do cookie-based session sticking or URL path rewrites on an Azure Load Balancer (which is mathematically impossible).

In this lab, we'll configure both on your Mac mini side by side. We’ll use **HAProxy** in pure TCP mode to emulate the Layer 4 **Azure Load Balancer**, and **Nginx** to emulate the Layer 7 **Application Gateway**.

---

### What Are We Simulating?

```
                     [Incoming Client Requests]
                                 │
         ┌───────────────────────┴───────────────────────┐
         │                                               │
         ▼ (Port 8080)                                   ▼ (Port 9090)
 ┌───────────────────────────────┐               ┌───────────────────────────────┐
 │ Layer 4: Azure Load Balancer  │               │  Layer 7: Application Gateway │
 │ (Simulated with HAProxy)      │               │  (Simulated with Nginx)       │
 │                               │               │                               │
 │ - Only looks at IP & Port     │               │ - Inspects HTTP URI path      │
 │ - Blind TCP distribution      │               │ - Path-based URL routing:     │
 │ - Round-Robin across pool     │               │     • /api/* ──► Backend API  │
 │ - Oblivious to HTTP paths     │               │     • /*     ──► Web Nodes    │
 └───────────────┬───────────────┘               └───────────────┬───────────────┘
                 │                                               │
                 ├───────────────────────┬───────────────────────┤
                 ▼                       ▼                       ▼
        ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
        │   be-web-01     │     │   be-web-02     │     │   be-api-01     │
        │  (10.10.2.11)   │     │  (10.10.2.12)   │     │  (10.10.2.20)   │
        │   Port 80       │     │   Port 80       │     │   Port 80       │
        │ [Web Node 1]    │     │ [Web Node 2]    │     │ [API Service]   │
        └─────────────────┘     └─────────────────┘     └─────────────────┘

```

---

### Core Concepts Explained

#### 1. Layer 4: The Flow Distributor (Azure Load Balancer)

* **What it looks at:** Only the TCP/UDP headers—specifically the **5-tuple** (Source IP, Source Port, Destination IP, Destination Port, Protocol).
* **How it works:** It does not terminate the TCP connection. It acts as a super-fast pass-through forwarder. It has no idea whether the payload inside is HTTP, gRPC, raw database queries, or Minecraft packets.
* **Health Probes:** It periodically pings a port (or does a basic TCP handshake). If a backend host doesn't answer, it pulls it from the rotation.
* **Limitation:** It cannot see URL paths (`/api` vs `/checkout`), cannot inspect or inject HTTP cookies, and cannot terminate TLS certificates directly.

#### 2. Layer 7: The Intelligent Reverse Proxy (Azure Application Gateway)

* **What it looks at:** Full application layer payloads (HTTP/HTTPS headers, URIs, query parameters, cookies, request bodies).
* **How it works:** It **terminates** the client’s TCP connection, reads the full HTTP request into memory, applies routing logic, and opens a *new* upstream TCP connection to the appropriate backend pool.
* **Key Capabilities:**
* **Path-Based Routing:** Send requests matching `/api/*` to a dedicated API microservice cluster, and all default traffic `/*` to frontend static web nodes.
* **SSL/TLS Termination:** Decrypt HTTPS traffic at the gateway so your internal backend servers only have to speak unencrypted HTTP.
* **Cookie-Based Session Affinity:** Injects a sticky cookie so a user's browser always hits the same backend server for stateful applications.



---

### Step-by-Step Hands-on Lab

#### Step 1: Set Up Your Project Directory

Open Terminal on your Mac mini and create a dedicated workspace:

```bash
mkdir -p ~/azure-labs/lab3-load-balancing
cd ~/azure-labs/lab3-load-balancing

```

---

#### Step 2: Create the HAProxy (Layer 4) Configuration

Create a file named `haproxy.cfg`.

We'll configure HAProxy in **`mode tcp`** to strip away any HTTP intelligence. It will balance raw TCP connections across `be-web-01` and `be-web-02` using round-robin, just like an Azure Basic/Standard Load Balancer:

```haproxy
global
    log stdout format raw local0

defaults
    log     global
    mode    tcp
    timeout connect 5000ms
    timeout client  50000ms
    timeout server  50000ms

# Front-facing listener (Emulating Azure Load Balancer Frontend IP + Port)
frontend l4_inbound
    bind *:8080
    default_backend l4_web_pool

# Backend Pool (Emulating Azure Backend Address Pool + Health Probe)
backend l4_web_pool
    balance roundrobin
    mode tcp
    # TCP health check on port 80
    option tcp-check
    server web01 10.10.2.11:80 check
    server web02 10.10.2.12:80 check

```

---

#### Step 3: Create the Nginx (Layer 7) Configuration

Create a file named `nginx.conf`.

We'll configure Nginx to act as an Application Gateway with path-based routing rules:

* Traffic to `/api/` or `/api/*` routes to `be-api-01`.
* All other traffic (`/`) routes to the `be-web-pool` (`be-web-01` and `be-web-02`).

```nginx
events { worker_connections 1024; }

http {
    # Upstream pool for general web traffic
    upstream web_backend_pool {
        server 10.10.2.11:80;
        server 10.10.2.12:80;
    }

    # Upstream pool for dedicated API services
    upstream api_backend_pool {
        server 10.10.2.20:80;
    }

    server {
        listen 9090;

        # Layer 7 Path Rule: Route all /api requests to the API service
        location /api/ {
            proxy_pass http://api_backend_pool/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        # Layer 7 Default Rule: Route root requests to Web pool
        location / {
            proxy_pass http://web_backend_pool;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}

```

---

#### Step 4: Create the Lab Definition (`docker-compose.yml`)

Create `docker-compose.yml` to assemble the five services:

* 2 Web Nodes (`be-web-01`, `be-web-02`)
* 1 API Node (`be-api-01`)
* 1 Layer 4 Load Balancer (`l4-azure-lb` listening on host port 8080)
* 1 Layer 7 Application Gateway (`l7-app-gateway` listening on host port 9090)

```yaml
services:
  # ==========================================================
  # Layer 4: Azure Load Balancer Emulation (HAProxy)
  # ==========================================================
  l4-azure-lb:
    image: haproxy:alpine
    container_name: l4-azure-lb
    ports:
      - "8080:8080"
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    networks:
      vnet-backend:
        ipv4_address: 10.10.2.4

  # ==========================================================
  # Layer 7: Azure Application Gateway Emulation (Nginx)
  # ==========================================================
  l7-app-gateway:
    image: nginx:alpine
    container_name: l7-app-gateway
    ports:
      - "9090:9090"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      vnet-backend:
        ipv4_address: 10.10.2.5

  # ==========================================================
  # Backend Pool Workloads
  # ==========================================================
  be-web-01:
    image: nicolaka/netshoot:latest
    container_name: be-web-01
    stdin_open: true
    tty: true
    command: >
      sh -c "while true; do echo -e 'HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n[Web Node 01] Hello from instance 1' | nc -l -p 80 -q 1; done"
    networks:
      vnet-backend:
        ipv4_address: 10.10.2.11

  be-web-02:
    image: nicolaka/netshoot:latest
    container_name: be-web-02
    stdin_open: true
    tty: true
    command: >
      sh -c "while true; do echo -e 'HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n[Web Node 02] Hello from instance 2' | nc -l -p 80 -q 1; done"
    networks:
      vnet-backend:
        ipv4_address: 10.10.2.12

  be-api-01:
    image: nicolaka/netshoot:latest
    container_name: be-api-01
    stdin_open: true
    tty: true
    command: >
      sh -c "while true; do echo -e 'HTTP/1.1 200 OK\r\nContent-Type: application/json\r\n\r\n{\"service\": \"orders-api\", \"status\": \"healthy\", \"version\": \"v1.0\"}' | nc -l -p 80 -q 1; done"
    networks:
      vnet-backend:
        ipv4_address: 10.10.2.20

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

Verify that all five containers are up and running:

```bash
docker compose ps

```

---

### Step 6: Verification & Experiments

#### Experiment 1: Testing Layer 4 Round-Robin TCP Balancing

Let's hit port `8080` (our simulated Azure Load Balancer).

Make four consecutive requests using `curl`:

```bash
curl -s http://localhost:8080
curl -s http://localhost:8080
curl -s http://localhost:8080
curl -s http://localhost:8080

```

*Expected Output:*

```text
[Web Node 01] Hello from instance 1
[Web Node 02] Hello from instance 2
[Web Node 01] Hello from instance 1
[Web Node 02] Hello from instance 2

```

**What happened:** HAProxy simply balanced raw TCP streams back and forth across the backend pool. It didn't look at HTTP methods, URLs, or headers.

---

#### Experiment 2: The Layer 4 Limitation — Can It Route `/api`?

What happens if a client asks the Layer 4 Load Balancer for `/api`?

```bash
curl -s http://localhost:8080/api

```

*Expected Output:*

```text
[Web Node 01] Hello from instance 1

```

**The big takeaway:** It still hits `be-web-01`! The Layer 4 load balancer **cannot read the URI string `/api**`. It only saw a TCP handshake on port 8080 and forwarded the bytes to whichever web node was next in line. The fact that the client wanted the API service was completely invisible to it.

---

#### Experiment 3: Testing Layer 7 Path-Based Routing (App Gateway)

Now let's hit port `9090` (our simulated Azure Application Gateway).

1. First, request the root path `/*`:

```bash
curl -s http://localhost:9090/

```

*Output:*

```text
[Web Node 01] Hello from instance 1

```

2. Run it again:

```bash
curl -s http://localhost:9090/

```

*Output:*

```text
[Web Node 02] Hello from instance 2

```

It round-robins between our two web frontends, just as expected.

3. Now, send a request to the `/api/` path:

```bash
curl -s http://localhost:9090/api/

```

*Output:*

```json
{"service": "orders-api", "status": "healthy", "version": "v1.0"}

```

**What happened:** The Application Gateway paused the incoming packet, terminated the TCP stream, inspected the HTTP `GET /api/ HTTP/1.1` line, checked its path map, and forwarded the request strictly to `be-api-01` (`10.10.2.20`). The web nodes were never touched!

---

#### Experiment 4: Observing Health Probe Failures

In Azure, both services maintain health probes. What happens when a backend instance crashes?

Let's stop `be-web-01`:

```bash
docker stop be-web-01

```

Wait about 5–10 seconds for the health checks to notice the failure.

1. Test the Layer 4 Load Balancer:

```bash
curl -s http://localhost:8080
curl -s http://localhost:8080

```

*Result:* Both requests will consistently return `[Web Node 02]`. HAProxy's TCP check failed on `10.10.2.11`, so it removed it from the active rotation without dropping client connections.

2. Test the Layer 7 Application Gateway:

```bash
curl -s http://localhost:9090/
curl -s http://localhost:9090/

```

*Result:* It also cleanly directs 100% of web traffic to `be-web-02`, while `/api/` traffic continues uninterrupted to `be-api-01`.

Restart the container to bring it back into service:

```bash
docker start be-web-01

```

---

### Step 7: Clean Up

Once your team finishes testing, shut down the containers and isolated bridge network:

```bash
docker compose down

```

---

### Quick Comparison Summary for the Team

| Feature | Azure Load Balancer (Layer 4) | Azure Application Gateway (Layer 7) |
| --- | --- | --- |
| **OSI Layer** | Layer 4 (Transport: TCP / UDP) | Layer 7 (Application: HTTP / HTTPS / WebSocket) |
| **Routing Decisions Based On** | IP 5-tuple hash | URL Path, Host header, Cookies, HTTP Query |
| **TCP Termination** | No (Pass-through / Direct Server Return) | Yes (Terminates client connection, proxies to backend) |
| **SSL / TLS Offloading** | No (Must decrypt on the VM/App) | Yes (Gateway decrypts certificate at the edge) |
| **WAF Protection** | No | Yes (OWASP rules for SQLi, XSS prevention) |
| **Latency & Cost** | Ultra-low latency, very inexpensive | Higher processing overhead, higher hourly cost |
| **Typical Azure Use Case** | High-throughput TCP pipelines, internal VM balancing | Microservices, Web Apps, path-routed REST APIs |