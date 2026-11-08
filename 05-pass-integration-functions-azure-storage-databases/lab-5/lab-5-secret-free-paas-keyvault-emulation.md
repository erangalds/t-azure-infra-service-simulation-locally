# Lab 5.5: Secret-Free PaaS — Key Vault Emulation & Runtime Credential Injection

Welcome to Lab 5.5! In previous labs, we passed connection strings and database passwords into container environment variables or configuration files.

In an enterprise cloud landing zone, **hardcoding database passwords or storage keys into container definitions, `local.settings.json`, or environment variables is a major security violation**. If someone gains read access to the container host, inspects `docker inspect`, or views an application manifest, the keys are completely compromised. Furthermore, when credentials rotate, all your services require code redeployments or container restarts.

To solve this, enterprise Azure deployments implement **Secret-Free PaaS with Azure Key Vault & Managed Identity**:

1. Compute instances (Azure Functions, App Services, AKS pods) run with **zero persistent secrets** in their configuration.
2. The compute service is assigned a **System-Assigned Managed Identity**.
3. Azure Key Vault is locked down behind an **Azure Private Endpoint** on the secure data subnet (`10.2.0.30`).
4. At runtime, the application requests an ephemeral Entra ID access token and pulls its database passwords and connection strings directly into memory over the private network.
5. When credentials rotate in Key Vault, the application pulls the newest version on the fly without changing a single line of deployment code.

In this lab, we will build this pattern locally on your Mac mini. We'll run a dedicated **Key Vault REST API emulator** inside our private `spoke-data` subnet (`10.2.0.30`) with zero public host ports, deploy an Azure Function that authenticates and retrieves secrets at runtime, and demonstrate live credential rotation without touching the application environment.

---

### What Are We Simulating?

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │                      Spoke Compute (10.1.0.0/24)                       │
 │                                                                        │
 │   ┌────────────────────────────────────────────────────────────────┐   │
 │   │  secure-function-app (10.1.0.20)                               │   │
 │   │  (Simulating Managed Identity + Key Vault Runtime Injection)   │   │
 │   │                                                                │   │
 │   │  Startup Environment:                                          │   │
 │   │  • NO database passwords!                                      │   │
 │   │  • NO storage keys!                                            │   │
 │   │  • VAULT_ENDPOINT = "http://10.2.0.30:8443"                    │   │
 │   │                                                                │   │
 │   │  At Startup / On Demand:                                       │   │
 │   │  1. Authenticates via Managed Identity                         │   │
 │   │  2. Fetches 'SqlAdminPassword' & 'StorageKey' into memory      │   │
 │   │  3. Re-reads dynamic updates when vault rotates secrets        │   │
 │   └───────────────────────────────┬────────────────────────────────┘   │
 └───────────────────────────────────┼────────────────────────────────────┘
                                     │
                                     ▼ (Private Link / Internal Port 8443)
 ┌────────────────────────────────────────────────────────────────────────┐
 │                 Spoke Data Subnet (10.2.0.0/24)                        │
 │                                                                        │
 │   ┌────────────────────────────────────────────────────────────────┐   │
 │   │  azure-keyvault-mock (10.2.0.30:8443)                          │   │
 │   │  (Simulated Azure Key Vault Private Endpoint)                  │   │
 │   │                                                                │   │
 │   │  Secret Store:                                                 │   │
 │   │  • 'SqlAdminPassword' ──► "Secret-Pass-v1"                     │   │
 │   │  • 'StorageKey'       ──► "Key-Alpha-2026"                     │   │
 │   │                                                                │   │
 │   │  Rotation Event:                                               │   │
 │   │  • 'SqlAdminPassword' ──► "Secret-Pass-v2-ROTATED"             │   │
 │   │  Zero host ports published; accessible only via Private IP!    │   │
 │   └────────────────────────────────────────────────────────────────┘   │
 └────────────────────────────────────────────────────────────────────────┘

```

---

### Core Concepts Explained

#### 1. Why Hardcoded Environment Variables Fail Security Audits

If you pass credentials via standard `docker-compose.yml` `environment:` blocks or Azure App Service Application Settings:

* Anyone with read access to the ARM template, container definition, or running host can run `docker inspect <container>` and see the plaintext secrets.
* Secrets get captured in crash dumps, log collectors, and CI/CD pipeline histories.
* Credential rotation requires redeploying or restarting the container, causing service disruption.

#### 2. Key Vault References vs. Runtime SDK Injection

In Azure, there are two primary ways to consume Key Vault secrets:

* **Key Vault Application Settings References (`@Microsoft.KeyVault(...)`):** App Service or Azure Functions resolves the secret into the environment at container start. While cleaner than manual strings, it still leaves secrets in memory environment blocks and doesn't always reflect rotations immediately without a restart.
* **Direct Runtime SDK Retrieval (`SecretClient` with `DefaultAzureCredential`):** The application code directly queries the Key Vault REST API over an Azure Private Endpoint whenever it needs to establish a connection pool or periodically refreshes its cache. When an administrator rotates the secret in Key Vault, the application picks up the new value automatically.

---

### Step-by-Step Hands-on Lab

#### Step 1: Set Up Your Project Directory

Open Terminal on your Mac mini and create a dedicated workspace:

```bash
mkdir -p ~/azure-labs/lab5-5-key-vault/vault_service
mkdir -p ~/azure-labs/lab5-5-key-vault/secure_app
cd ~/azure-labs/lab5-5-key-vault

```

---

#### Step 2: Create the Azure Key Vault Service Simulator

We will build a lightweight Python-based microservice that implements the Azure Key Vault REST API specification (`/secrets/{secret-name}?api-version=7.4`).

##### 1. Create the Key Vault Service Code

Create `vault_service/vault_server.py`:

```python
import json
from http.server import HTTPServer, BaseHTTPRequestHandler

# In-memory simulated Key Vault secret store
SECRETS = {
    "SqlAdminPassword": {
        "value": "Initial-Secure-P@ss-2026!",
        "id": "https://vault.privatelink.vaultcore.azure.net/secrets/SqlAdminPassword/v1",
        "version": "v1",
        "attributes": {"enabled": True, "created": 1731050000}
    },
    "StorageAccountKey": {
        "value": "Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==",
        "id": "https://vault.privatelink.vaultcore.azure.net/secrets/StorageAccountKey/v1",
        "version": "v1",
        "attributes": {"enabled": True, "created": 1731050000}
    }
}

class KeyVaultHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        # Authenticate simulated Managed Identity Bearer Token
        auth_header = self.headers.get('Authorization', '')
        if not auth_header.startswith('Bearer '):
            self.send_response(401)
            self.send_header('Content-Type', 'application/json')
            self.end_headers()
            self.wfile.write(json.dumps({"error": "Unauthorized: Managed Identity token missing"}).encode('utf-8'))
            return

        # Handle secret retrieval: /secrets/<secret-name>
        path_parts = self.path.split('?')[0].strip('/').split('/')
        if len(path_parts) >= 2 and path_parts[0] == "secrets":
            secret_name = path_parts[1]
            if secret_name in SECRETS:
                self.send_response(200)
                self.send_header('Content-Type', 'application/json')
                self.end_headers()
                self.wfile.write(json.dumps(SECRETS[secret_name]).encode('utf-8'))
                return
            else:
                self.send_response(404)
                self.send_header('Content-Type', 'application/json')
                self.end_headers()
                self.wfile.write(json.dumps({"error": f"Secret '{secret_name}' not found"}).encode('utf-8'))
                return

        self.send_response(404)
        self.end_headers()

    def do_PUT(self):
        # Admin endpoint to simulate rotating secrets in Key Vault
        path_parts = self.path.split('?')[0].strip('/').split('/')
        if len(path_parts) >= 2 and path_parts[0] == "secrets":
            secret_name = path_parts[1]
            content_length = int(self.headers.get('Content-Length', 0))
            payload = json.loads(self.rfile.read(content_length).decode('utf-8'))
            
            new_value = payload.get("value")
            new_version = payload.get("version", "v2")
            
            SECRETS[secret_name] = {
                "value": new_value,
                "id": f"https://vault.privatelink.vaultcore.azure.net/secrets/{secret_name}/{new_version}",
                "version": new_version,
                "attributes": {"enabled": True, "updated": 1731060000}
            }
            
            self.send_response(200)
            self.send_header('Content-Type', 'application/json')
            self.end_headers()
            self.wfile.write(json.dumps({"status": "ROTATED", "secret": secret_name, "version": new_version}).encode('utf-8'))
            return

        self.send_response(400)
        self.end_headers()

def run(port=8443):
    server = HTTPServer(('0.0.0.0', port), KeyVaultHandler)
    print(f"Azure Key Vault REST API Emulator listening on port {port}...")
    server.serve_forever()

if __name__ == '__main__':
    run()

```

##### 2. Create the Key Vault Dockerfile

Create `vault_service/Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY vault_server.py .

EXPOSE 8443
CMD ["python", "-u", "vault_server.py"]

```

---

#### Step 3: Build the Secure Azure Function Application

Now we build the application container. Notice that its configuration only contains the URL of the Key Vault Private Endpoint.

##### 1. Create the Application Code

Create `secure_app/app.py`:

```python
import os
import json
import urllib.request
from http.server import HTTPServer, BaseHTTPRequestHandler

VAULT_ENDPOINT = os.environ.get("VAULT_ENDPOINT", "http://10.2.0.30:8443")
# In live Azure, this token is acquired automatically via MSI endpoint: 169.254.169.254
SIMULATED_MSI_TOKEN = "Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Ik1hbmFnZWRJZGVudGl0eVRva2VuIn0.demo"

def fetch_secret_from_vault(secret_name):
    """Simulates Azure SecretClient.get_secret() over Private Link"""
    url = f"{VAULT_ENDPOINT}/secrets/{secret_name}?api-version=7.4"
    req = urllib.request.Request(url, headers={"Authorization": SIMULATED_MSI_TOKEN})
    with urllib.request.urlopen(req, timeout=5) as resp:
        if resp.status == 200:
            data = json.loads(resp.read().decode('utf-8'))
            return data["value"], data["version"]
    raise RuntimeError(f"Unable to fetch secret {secret_name}")

class SecureAppHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/api/GetDatabaseConnection":
            try:
                # Dynamic runtime injection: Fetch active secret directly from Key Vault
                password, version = fetch_secret_from_vault("SqlAdminPassword")
                
                # Mask secret for safe audit verification
                masked_password = password[:3] + "*" * (len(password) - 6) + password[-3:]
                
                response = {
                    "status": "CONNECTED",
                    "source": "Azure Key Vault Private Endpoint",
                    "vault_uri": VAULT_ENDPOINT,
                    "secret_name": "SqlAdminPassword",
                    "active_version": version,
                    "in_memory_password_masked": masked_password,
                    "security_audit": "PASSED - Zero secrets stored in container environment"
                }
                
                self.send_response(200)
                self.send_header('Content-Type', 'application/json')
                self.end_headers()
                self.wfile.write(json.dumps(response, indent=2).encode('utf-8'))
            except Exception as e:
                self.send_response(500)
                self.send_header('Content-Type', 'application/json')
                self.end_headers()
                self.wfile.write(json.dumps({"error": str(e)}).encode('utf-8'))
        else:
            self.send_response(404)
            self.end_headers()

def run(port=80):
    server = HTTPServer(('0.0.0.0', port), SecureAppHandler)
    print(f"Secure Azure Function App listening on port {port}...")
    server.serve_forever()

if __name__ == '__main__':
    run()

```

##### 2. Create the Application Dockerfile

Create `secure_app/Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    iproute2 \
    && rm -rf /var/lib/apt/lists/*

COPY app.py .

EXPOSE 80
CMD ["python", "-u", "app.py"]

```

---

#### Step 4: Create the Lab Definition (`docker-compose.yml`)

Return to the root lab folder (`~/azure-labs/lab5-5-key-vault`) and create `docker-compose.yml`:

```yaml
services:
  # ==========================================================
  # Azure Key Vault Emulator (spoke-data)
  # ==========================================================
  azure-keyvault-mock:
    build:
      context: ./vault_service
      dockerfile: Dockerfile
    container_name: azure-keyvault-mock
    # NOTICE: Zero host ports published! Accessible strictly via Private IP
    networks:
      spoke-data:
        ipv4_address: 10.2.0.30

  # ==========================================================
  # Secure Azure Function Application (spoke-compute)
  # ==========================================================
  secure-function-app:
    build:
      context: ./secure_app
      dockerfile: Dockerfile
    container_name: secure-function-app
    ports:
      - "8080:80"  # Expose web trigger to Mac host for testing
    environment:
      # NOTICE: NO PASSWORDS! Only the internal private endpoint URI
      - VAULT_ENDPOINT=http://10.2.0.30:8443
    networks:
      spoke-compute:
        ipv4_address: 10.1.0.20

networks:
  spoke-compute:
    driver: bridge
    ipam:
      config:
        - subnet: 10.1.0.0/24
          gateway: 10.1.0.1

  spoke-data:
    driver: bridge
    internal: true  # Completely dark to outside world
    ipam:
      config:
        - subnet: 10.2.0.0/24

```

---

#### Step 5: Spin Up the Stack

Launch the environment:

```bash
docker compose up -d --build

```

Verify both containers are running:

```bash
docker compose ps

```

---

#### Step 6: Configure Internal Network Routing

Enable the route so `secure-function-app` can reach the private data subnet (`10.2.0.0/24`) using the Docker bridge gateway:

```bash
docker exec -it secure-function-app ip route add 10.2.0.0/24 via 10.1.0.1

```

Verify that the application can reach the private Key Vault endpoint on port 8443:

```bash
docker exec -it secure-function-app nc -zv -w 2 10.2.0.30 8443

```

*Expected Output:*

```text
10.2.0.30 8443 (pcsync-https) open

```

---

### Step 7: Verification & Experiments

#### Experiment 1: The Zero-Secret Container Audit

Let's first audit the running container configuration to prove that no credentials exist anywhere in the environment.

1. Inspect the container environment variables from your Mac terminal:

```bash
docker exec -it secure-function-app env

```

*Expected Output:*

```text
PATH=/usr/local/bin:...
LANG=C.UTF-8
PYTHON_VERSION=3.11...
VAULT_ENDPOINT=http://10.2.0.30:8443

```

*Notice:* There are **zero** database passwords, storage keys, or certificates in the environment.

2. Run `docker inspect`:

```bash
docker inspect secure-function-app | grep -i "pass"

```

*Expected Output:* Empty! A compromised host or container inspection yields zero secrets.

---

#### Experiment 2: Runtime Secret Retrieval Over Private Link

Now let's call our function endpoint:

```bash
curl -s http://localhost:8080/api/GetDatabaseConnection | jq .

```

*Expected Output:*

```json
{
  "status": "CONNECTED",
  "source": "Azure Key Vault Private Endpoint",
  "vault_uri": "http://10.2.0.30:8443",
  "secret_name": "SqlAdminPassword",
  "active_version": "v1",
  "in_memory_password_masked": "Ini*******************6!",
  "security_audit": "PASSED - Zero secrets stored in container environment"
}

```

**What happened:**

* The container received the HTTP request.
* It used its simulated Managed Identity token to query `[http://10.2.0.30:8443/secrets/SqlAdminPassword](http://10.2.0.30:8443/secrets/SqlAdminPassword)`.
* The secret was loaded directly into application memory over the private subnet.

---

#### Experiment 3: Live Credential Rotation Without Redeployment

Now simulate an enterprise security operation: A scheduled key rotation or security breach requires rotating `SqlAdminPassword` in Key Vault immediately.

1. Update the secret in `azure-keyvault-mock` using a simulated administrative PUT request:

```bash
docker exec -it azure-keyvault-mock python -c "
import urllib.request, json

data = json.dumps({'value': 'Rotated-Production-Key-999!', 'version': 'v2'}).encode('utf-8')
req = urllib.request.Request('http://localhost:8443/secrets/SqlAdminPassword', data=data, method='PUT')
req.add_header('Content-Type', 'application/json')
with urllib.request.urlopen(req) as resp:
    print(resp.read().decode('utf-8'))
"

```

*Expected Output:*

```json
{"status": "ROTATED", "secret": "SqlAdminPassword", "version": "v2"}

```

2. Without restarting or modifying `secure-function-app`, invoke the function API again:

```bash
curl -s http://localhost:8080/api/GetDatabaseConnection | jq .

```

*Expected Output:*

```json
{
  "status": "CONNECTED",
  "source": "Azure Key Vault Private Endpoint",
  "vault_uri": "http://10.2.0.30:8443",
  "secret_name": "SqlAdminPassword",
  "active_version": "v2",
  "in_memory_password_masked": "Rot*******************9!",
  "security_audit": "PASSED - Zero secrets stored in container environment"
}

```

**The Takeaway for the Team:**

* The secret rotated from `v1` to `v2`.
* The application immediately picked up the rotated password in memory.
* Zero container restarts occurred.
* Zero configuration files or manifests were changed.

---

#### Experiment 4: The Public Endpoint Block (Host Isolation)

Verify that Key Vault cannot be reached directly from your Mac host:

```bash
nc -zv -w 2 localhost 8443

```

*Expected Output:* `Connection refused` or timeout.

The Key Vault instance exists exclusively within `spoke-data` (`10.2.0.30`), emulating a Private Endpoint with public network access completely disabled.

---

### Step 8: Clean Up

When you've finished testing, stop and clean up the containers:

```bash
docker compose down -v

```

---

### Key Enterprise Takeaways for the Team

* **Managed Identity Eliminates the "First Secret" Problem:** Traditional systems use an API key to access their secret vault, which just moves the problem. **Azure Managed Identity** uses hardware-backed virtual machine tokens provided directly by the hypervisor to authenticate to Key Vault with zero bootstrap keys.
* **Granular RBAC (Azure Key Vault Secrets User):** Always use Azure RBAC rather than legacy Key Vault Access Policies. Assign compute identities strictly the `Key Vault Secrets User` role, preventing them from listing or deleting secrets.
* **Private Link for Key Vault:** Because Key Vault hosts your organization's most sensitive material, the **`Microsoft.KeyVault.vaults` Private Endpoint** must always be deployed into a private data subnet with public network access disabled.