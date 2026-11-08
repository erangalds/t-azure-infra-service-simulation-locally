# Lab 5: PaaS Integration — Azure Functions, Azurite Storage, Azure SQL Edge & Private Endpoints

Welcome to Lab 5! Up to this point, we have built virtual networks, stateful firewalls, load balancers, and hub-and-spoke transit topologies. Now, we shift from foundational network plumbing to **PaaS (Platform as a Service)** and **Serverless Compute**, addressing a critical question in enterprise cloud engineering:

> *"How do we let modern serverless apps talk to databases and cloud storage without exposing those data services to the public internet?"*

By default, Azure PaaS services like Azure SQL and Azure Blob Storage are multitenant with public IP endpoints protected by identity firewalls. In an enterprise landing zone, security compliance mandates **Azure Private Link** and **Private Endpoints**:

1. Disable public network access to storage and databases.
2. Inject a **Virtual Network Interface (NIC)** carrying a **Private IP** (`10.2.0.x`) into your secure data subnet.
3. Use **VNet Integration** on your compute tier (Azure Functions / App Service) to project its outbound traffic into the virtual network.
4. Access data services strictly over internal IP addresses without traffic ever touching the public internet.

In this lab, we will run **official Microsoft container images** on your Mac mini:

* **Azurite** (`[mcr.microsoft.com/azure-storage/azurite](https://mcr.microsoft.com/azure-storage/azurite)`): Microsoft's official Azure Storage emulator for Blob and Queue endpoints.
* **Azure SQL Edge** (`[mcr.microsoft.com/azure-sql-edge](https://mcr.microsoft.com/azure-sql-edge)`): Microsoft’s high-performance, ARM64-native relational database engine.
* **Azure Functions (Python)**: Running inside an official Azure Functions runtime container.

We will simulate a Private Endpoint architecture by stripping all external port bindings from the data tier, demonstrating how private connection strings bind serverless triggers to isolated storage and databases.

---

### What Are We Simulating?

```
 ┌──────────────────────────────────────────────────────────────┐
 │         Compute Subnet: spoke-compute (10.1.0.0/24)          │
 │                                                              │
 │   ┌──────────────────────────────────────────────────────┐   │
 │   │         azure-function-runner (10.1.0.10)            │   │
 │   │         (VNet-Integrated Azure Function)             │   │
 │   │                                                      │   │
 │   │   • Triggers: HTTP POST /api/ProcessOrder            │   │
 │   │   • Internal Output Binding 1: Azurite Blob (10000)  │   │
 │   │   • Internal Output Binding 2: Azure SQL Edge (1433) │   │
 │   └──────────────────────────┬───────────────────────────┘   │
 └──────────────────────────────┼───────────────────────────────┘
                                │
                                ▼ (Internal Private Network Route)
 ┌──────────────────────────────────────────────────────────────┐
 │           Data Subnet: spoke-data (10.2.0.0/24)              │
 │           (Zero Public Host Ports Published!)                │
 │                                                              │
 │   ┌───────────────────────────┐  ┌───────────────────────┐   │
 │   │   azurite-storage         │  │  azure-sql-db         │   │
 │   │   (10.2.0.10)             │  │  (10.2.0.11)          │   │
 │   │                           │  │                       │   │
 │   │   • Private Endpoint      │  │  • Private Endpoint   │   │
 │   │     Blob Listener: 10000  │  │    TDS Listener: 1433 │   │
 │   │     Queue Listener: 10001 │  │                       │   │
 │   │   • Public Ingress: OFF   │  │  • Public Ingress: OFF│   │
 │   └───────────────────────────┘  └───────────────────────┘   │
 └──────────────────────────────────────────────────────────────┘

```

---

### Core Concepts Explained

#### 1. Public PaaS vs. Private Endpoints (Private Link)

* **Default Cloud PaaS:** An Azure Storage account (`companydata.blob.core.windows.net`) resolves to a public IP. Although encrypted and protected by Entra ID, traffic technically travels over public IP boundaries.
* **Private Endpoint:** Injects an internal Network Interface (NIC) directly into your private subnet (`spoke-data` at `10.2.0.10`). The storage account FQDN resolves to that internal IP via an Azure Private DNS Zone. Public ingress is completely disabled.

#### 2. VNet Integration for Serverless Compute

Standard Azure Functions run in a managed serverless multi-tenant pool. They cannot reach private IP addresses like `10.2.0.10` by default.

* Enabling **Regional VNet Integration** assigns dedicated virtual interfaces in a delegated subnet (`spoke-compute`), routing outbound calls from the function directly into the customer's VNet.

#### 3. Well-Known Azurite Emulation Credentials

Azurite uses a fixed, well-known development account key baked into official Azure SDKs:

* **Account Name:** `devstoreaccount1`
* **Account Key:** `Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==`
* **Custom Endpoints:** When accessing Azurite over an internal network, you supply explicit `BlobEndpoint` and `QueueEndpoint` URIs pointing to the container's private IP.

---

### Step-by-Step Hands-on Lab

#### Step 1: Set Up Your Project Directory

Open Terminal on your Mac mini and create a dedicated workspace:

```bash
mkdir -p ~/azure-labs/lab5-paas-private-link/function_app
cd ~/azure-labs/lab5-paas-private-link

```

---

#### Step 2: Create the Function Code & Configuration

We'll build a lightweight Python-based Azure Function with an HTTP trigger. When invoked, it:

1. Connects to the **Azurite Private Endpoint** at `10.2.0.10:10000` to upload an order payload blob into a `orders-blob-container`.
2. Connects to the **Azure SQL Edge Private Endpoint** at `10.2.0.11:1433` via raw TDS socket/SQL query to insert an audit record into an `OrdersAudit` table.

##### 1. Define Python Dependencies

Create `function_app/requirements.txt`:

```text
azure-storage-blob>=12.19.0
pymssql>=2.2.8

```

##### 2. Create the Function App Code

Create `function_app/app.py`:

```python
import os
import json
from http.server import HTTPServer, BaseHTTPRequestHandler
from azure.storage.blob import BlobServiceClient
import pymssql

# Retrieve connection parameters pointing to simulated private endpoints
STORAGE_CONN_STR = os.environ.get("AzureWebJobsStorage")
SQL_SERVER = os.environ.get("SQL_SERVER", "10.2.0.11")
SQL_USER = os.environ.get("SQL_USER", "sa")
SQL_PASSWORD = os.environ.get("SQL_PASSWORD", "AzureLabP@ssw0rd2026!")
SQL_DB = os.environ.get("SQL_DB", "master")

class OrderHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        if self.path == "/api/ProcessOrder":
            content_length = int(self.headers.get('Content-Length', 0))
            post_data = self.rfile.read(content_length)
            order = json.loads(post_data.decode('utf-8'))
            order_id = order.get("order_id", "ORD-UNKNOWN")

            # 1. Write to Azurite Private Endpoint (Blob Storage)
            blob_service_client = BlobServiceClient.from_connection_string(STORAGE_CONN_STR)
            container_client = blob_service_client.get_container_client("orders-blob-container")
            try:
                container_client.create_container()
            except Exception:
                pass  # Container already exists

            blob_name = f"order-{order_id}.json"
            blob_client = container_client.get_blob_client(blob_name)
            blob_client.upload_blob(json.dumps(order), overwrite=True)

            # 2. Write to Azure SQL Edge Private Endpoint
            conn = pymssql.connect(server=SQL_SERVER, user=SQL_USER, password=SQL_PASSWORD, database=SQL_DB, timeout=5)
            cursor = conn.cursor()
            cursor.execute("""
                IF NOT EXISTS (SELECT * FROM sysobjects WHERE name='OrdersAudit' AND xtype='U')
                CREATE TABLE OrdersAudit (
                    OrderID VARCHAR(50) PRIMARY KEY,
                    CustomerName VARCHAR(100),
                    Amount DECIMAL(10, 2),
                    BlobPath VARCHAR(200)
                )
            """)
            cursor.execute(
                "INSERT INTO OrdersAudit (OrderID, CustomerName, Amount, BlobPath) VALUES (%s, %s, %d, %s)",
                (order_id, order.get("customer", "Unknown"), order.get("amount", 0), f"orders-blob-container/{blob_name}")
            )
            conn.commit()
            conn.close()

            # Return success response
            self.send_response(200)
            self.send_header('Content-Type', 'application/json')
            self.end_headers()
            response = {
                "status": "PROCESSED",
                "order_id": order_id,
                "blob_saved": f"http://10.2.0.10:10000/devstoreaccount1/orders-blob-container/{blob_name}",
                "database_updated": f"Azure SQL Edge at {SQL_SERVER}:1433"
            }
            self.wfile.write(json.dumps(response).encode('utf-8'))
        else:
            self.send_response(404)
            self.end_headers()

def run(server_class=HTTPServer, handler_class=OrderHandler, port=80):
    server_address = ('', port)
    httpd = server_class(server_address, handler_class)
    print(f"Azure Function Runtime listening on port {port}...")
    httpd.serve_forever()

if __name__ == '__main__':
    run()

```

##### 3. Create the Function App Dockerfile

Create `function_app/Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    iproute2 \
    iputils-ping \
    freetds-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 80
CMD ["python", "app.py"]

```

---

#### Step 3: Create the Lab Definition (`docker-compose.yml`)

Return to the root lab folder (`~/azure-labs/lab5-paas-private-link`) and create `docker-compose.yml`:

```yaml
services:
  # ==========================================================
  # Azure Functions Compute Runtime (spoke-compute)
  # ==========================================================
  azure-function-runner:
    build:
      context: ./function_app
      dockerfile: Dockerfile
    container_name: azure-function-runner
    ports:
      - "8080:80"  # Expose trigger port to host Mac for demonstration
    environment:
      # Private Endpoint connection string pointing to internal IP 10.2.0.10
      - AzureWebJobsStorage=DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;BlobEndpoint=http://10.2.0.10:10000/devstoreaccount1;QueueEndpoint=http://10.2.0.10:10001/devstoreaccount1;
      - SQL_SERVER=10.2.0.11
      - SQL_USER=sa
      - SQL_PASSWORD=AzureLabP@ssw0rd2026!
      - SQL_DB=master
    networks:
      spoke-compute:
        ipv4_address: 10.1.0.10
      spoke-data:
        ipv4_address: 10.2.0.5  # VNet Integration interface into data subnet

  # ==========================================================
  # Azurite (Azure Storage Emulator) - Private Endpoint Simulation
  # ==========================================================
  azurite-storage:
    image: mcr.microsoft.com/azure-storage/azurite:latest
    container_name: azurite-storage
    # NOTICE: NO PORTS EXPOSED TO HOST MAC! Strictly accessible via private subnet IP
    command: "azurite --blobHost 0.0.0.0 --blobPort 10000 --queueHost 0.0.0.0 --queuePort 10001 --loose"
    networks:
      spoke-data:
        ipv4_address: 10.2.0.10

  # ==========================================================
  # Azure SQL Edge (Native Apple Silicon ARM64 DBaaS)
  # ==========================================================
  azure-sql-db:
    image: mcr.microsoft.com/azure-sql-edge:latest
    container_name: azure-sql-db
    # NOTICE: NO PORTS EXPOSED TO HOST MAC! Strictly accessible via private subnet IP
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=AzureLabP@ssw0rd2026!
    networks:
      spoke-data:
        ipv4_address: 10.2.0.11

networks:
  spoke-compute:
    driver: bridge
    ipam:
      config:
        - subnet: 10.1.0.0/24
          gateway: 10.1.0.1

  spoke-data:
    driver: bridge
    internal: true  # Completely dark to the outside world
    ipam:
      config:
        - subnet: 10.2.0.0/24

```

---

#### Step 4: Spin Up the Stack

Launch the environment:

```bash
docker compose up -d --build

```

Verify that all three containers are running:

```bash
docker compose ps

```

*Notice the port mapping column:*

* `azure-function-runner` maps `8080->80/tcp`.
* `azurite-storage` and `azure-sql-db` have **zero** host ports listed. They exist strictly within `10.2.0.0/24`.

---

### Step 5: Verification & Experiments

#### Experiment 1: The Private Endpoint Proof (Host Isolation)

First, verify that your Mac host cannot connect to Azurite or SQL Edge directly.

1. Attempt to connect to Azurite from your Mac terminal on port 10000:

```bash
nc -zv -w 2 localhost 10000

```

*Expected output:* `Connection refused` or timeout.

2. Attempt to connect to Azure SQL Edge from your Mac terminal on port 1433:

```bash
nc -zv -w 2 localhost 1433

```

*Expected output:* `Connection refused` or timeout.

**The takeaway:** Public ingress is completely disabled. There is no public endpoint.

---

#### Experiment 2: Triggering the Function Over the Internal Network

Now submit an order payload to the Azure Function trigger on `http://localhost:8080/api/ProcessOrder`:

```bash
curl -s -X POST http://localhost:8080/api/ProcessOrder \
  -H "Content-Type: application/json" \
  -d '{
    "order_id": "ORD-94812",
    "customer": "Contoso Enterprise Corp",
    "amount": 28500
  }' | jq .

```

*Expected Output:*

```json
{
  "status": "PROCESSED",
  "order_id": "ORD-94812",
  "blob_saved": "http://10.2.0.10:10000/devstoreaccount1/orders-blob-container/order-ORD-94812.json",
  "database_updated": "Azure SQL Edge at 10.2.0.11:1433"
}

```

**What just happened:**

1. Your client invoked the function trigger.
2. The function runner used its VNet-integrated interface to establish an internal TCP stream to Azurite at `10.2.0.10:10000` and wrote the blob.
3. It opened a second internal connection to Azure SQL Edge at `10.2.0.11:1433`, created the audit table, and committed the SQL record.
4. The entire transaction completed without any traffic touching public internet routes.

---

#### Experiment 3: Inspecting the Internal Data Tier

Verify the data was committed inside the private database:

Access `azure-sql-db` directly and run a `sqlcmd` query:

```bash
docker exec -it azure-sql-db /opt/mssql-tools/bin/sqlcmd \
  -S localhost -U sa -P "AzureLabP@ssw0rd2026!" \
  -Q "SELECT OrderID, CustomerName, Amount, BlobPath FROM OrdersAudit;"

```

*Expected Output:*

```text
OrderID    CustomerName              Amount    BlobPath
---------- ------------------------- --------- ----------------------------------------------------
ORD-94812  Contoso Enterprise Corp   28500.00  orders-blob-container/order-ORD-94812.json

(1 rows affected)

```

---

#### Experiment 4: Inspecting Azurite Internal Blob Storage

Verify the blob file actually exists inside the Azurite container storage:

```bash
docker exec -it azure-function-runner python -c "
from azure.storage.blob import BlobServiceClient
conn_str = '$AzureWebJobsStorage'
client = BlobServiceClient.from_connection_string('DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;BlobEndpoint=http://10.2.0.10:10000/devstoreaccount1;')
blob = client.get_container_client('orders-blob-container').get_blob_client('order-ORD-94812.json')
print('Retrieved Blob Content:\n', blob.download_blob().readall().decode('utf-8'))
"

```

*Expected Output:*

```json
Retrieved Blob Content:
 {"order_id": "ORD-94812", "customer": "Contoso Enterprise Corp", "amount": 28500}

```

---

### Step 6: Clean Up

Once the team finishes testing, bring down the stack:

```bash
docker compose down -v

```

---

### Key Architecture Takeaways for the Team

* **Private Endpoint vs. Service Endpoint:** Service Endpoints keep traffic on the Microsoft backbone but the service still uses a public IP. **Private Endpoints (Private Link)** replace public endpoints entirely with private IPs (`10.2.0.x`) inside your VNet.
* **DNS Resolution is Crucial:** In live Azure, when you enable Private Link, Azure modifies the public DNS to return a CNAME pointing to `privatelink.blob.core.windows.net`. An **Azure Private DNS Zone** linked to your VNet is required to resolve that CNAME to the private IP.
* **VNet Integration Overhead:** Serverless compute tiers (Azure Functions, App Services) require dedicated, delegated subnets with sufficient IP headroom to support autoscaling outbound interfaces.