# Lab 5.4: Event-Driven Serverless — Azurite Storage Queues & Poison Message Dead-Lettering

Welcome to Lab 5.4! In synchronous microservices, when a frontend web app processes an order, it makes a direct HTTP call to a backend database or payment API. If that backend slows down or drops connections, the frontend threads lock up, HTTP connection pools exhaust, and the entire system cascades into a complete outage.

In enterprise cloud architecture, systems decouple using **asynchronous message queues**.

1. **The Producer:** The frontend application pushes a lightweight message into an **Azure Storage Queue** and returns an immediate `202 Accepted` to the client.
2. **The Consumer:** An **Azure Function (`QueueTrigger`)** or background worker pulls messages off the queue, processes the business logic, and deletes the message upon successful completion.

However, decoupling introduces a critical resilience challenge: **Poison Messages (The Message of Death)**.

* What happens if a client pushes a corrupted JSON payload, an order with a negative price, or an unparseable binary string?
* A naive worker crashes, fails to acknowledge the message, and puts the message back on the queue.
* The message reappears after its **visibility timeout**, gets pulled by another worker, crashes it, and repeats in an infinite retry loop that burns compute resources and halts the queue.

Azure Functions and Azure Storage solve this natively with **automatic retry policies and Poison Queues**:

* Each message maintains a **`DequeueCount`** attribute.
* If a message fails processing `maxDequeueCount` times (by default, **5 times**), the runtime automatically shunts the offending message into a dead-letter queue: **`<queue-name>-poison`**.
* The main queue continues moving at full speed, while site reliability engineers (SREs) can inspect and alert on the dead-letter queue.

In this lab, we will use **Azurite's Queue Service** on `spoke-data` (`10.2.0.10:10001`), build an asynchronous Python worker simulating an Azure Functions QueueTrigger with automatic poison handling, and observe how bad messages are quarantined without halting the pipeline.

---

### What Are We Simulating?

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │                      Spoke Compute (10.1.0.0/24)                       │
 │                                                                        │
 │   ┌──────────────────────────┐         ┌───────────────────────────┐   │
 │   │   order-producer         │         │   queue-worker-service    │   │
 │   │   (10.1.0.15)            │         │   (10.1.0.25)             │   │
 │   │                          │         │                           │   │
 │   │   • Pushes valid orders  │         │   Simulates QueueTrigger: │   │
 │   │   • Pushes poison order  │         │   • Dequeue message       │   │
 │   │     (corrupted JSON)     │         │   • Inspect DequeueCount  │   │
 │   └────────────┬─────────────┘         │   • If count >= 5:        │   │
 │                │                       │     Dead-letter to poison!│   │
 │                │                       └─────────────┬─────────────┘   │
 └────────────────┼─────────────────────────────────────┼─────────────────┘
                  │                                     │
                  │   Internal Queue Calls (TCP 10001)  │
                  └──────────────────┬──────────────────┘
                                     │
                                     ▼
 ┌────────────────────────────────────────────────────────────────────────┐
 │                 Spoke Data Subnet (10.2.0.0/24)                        │
 │                                                                        │
 │   ┌────────────────────────────────────────────────────────────────┐   │
 │   │  azurite-queue-storage (10.2.0.10:10001)                       │   │
 │   │                                                                │   │
 │   │   Queue 1: "orders-incoming"                                   │   │
 │   │   • Valid orders: processed and deleted immediately            │   │
 │   │   • Corrupt order: retried 5 times, then removed               │   │
 │   │                                                                │   │
 │   │   Queue 2: "orders-incoming-poison"                            │   │
 │   │   • Holds dead-lettered message with error metadata            │   │
 │   └────────────────────────────────────────────────────────────────┘   │
 └────────────────────────────────────────────────────────────────────────┘

```

---

### Core Concepts Explained

#### 1. Visibility Timeout & Peek-Lock Pattern

When an Azure Function or worker dequeues a message from an Azure Storage Queue:

* The message is **not** deleted immediately.
* It becomes **invisible** to other workers for a configured duration (e.g., 5 seconds in this lab, default 30 seconds in Azure).
* If the worker finishes successfully, it calls `delete_message(message_id, pop_receipt)`.
* If the worker crashes or raises an unhandled exception before calling delete, the visibility timeout expires. The message pops back up on the queue for another worker to retry.

#### 2. The `DequeueCount` Metadata Property

Every time a message is fetched from the queue, Azure increments its hidden metadata property: `DequeueCount`.

* Attempt 1: `DequeueCount = 1`
* Attempt 2: `DequeueCount = 2`
* ...
* Attempt 5: `DequeueCount = 5`

If `DequeueCount >= maxDequeueCount`:
The worker logs the poison event, copies the failed payload along with diagnostic headers into `<original-queue-name>-poison`, and deletes it from the active processing queue.

---

### Step-by-Step Hands-on Lab

#### Step 1: Set Up Your Project Directory

Open Terminal on your Mac mini and create a dedicated workspace:

```bash
mkdir -p ~/azure-labs/lab5-4-queue-poison/worker_app
cd ~/azure-labs/lab5-4-queue-poison

```

---

#### Step 2: Build the Resilient Queue Worker

We will write a Python worker that mirrors how the Azure Functions Host handles QueueTriggers under the hood.

##### 1. Define Dependencies

Create `worker_app/requirements.txt`:

```text
azure-storage-queue>=12.9.0

```

##### 2. Create the Queue Worker Code

Create `worker_app/worker.py`:

```python
import os
import time
import json
import logging
from azure.storage.queue import QueueClient, BinaryBase64EncodePolicy, BinaryBase64DecodePolicy

logging.basicConfig(level=logging.INFO, format='%(asctime)s [%(levelname)s] %(message)s')

CONN_STR = os.environ.get("STORAGE_CONN_STR")
MAIN_QUEUE_NAME = "orders-incoming"
POISON_QUEUE_NAME = f"{MAIN_QUEUE_NAME}-poison"
MAX_DEQUEUE_COUNT = 5
VISIBILITY_TIMEOUT = 3  # Keep short (3 seconds) for quick lab demonstration

def init_queues():
    main_client = QueueClient.from_connection_string(
        CONN_STR,
        MAIN_QUEUE_NAME,
        message_encode_policy=BinaryBase64EncodePolicy(),
        message_decode_policy=BinaryBase64DecodePolicy()
    )
    poison_client = QueueClient.from_connection_string(
        CONN_STR,
        POISON_QUEUE_NAME,
        message_encode_policy=BinaryBase64EncodePolicy(),
        message_decode_policy=BinaryBase64DecodePolicy()
    )
    
    # Ensure queues exist
    for client, name in [(main_client, MAIN_QUEUE_NAME), (poison_client, POISON_QUEUE_NAME)]:
        try:
            client.create_queue()
            logging.info(f"Initialized queue: {name}")
        except Exception:
            pass
            
    return main_client, poison_client

def process_order(content_str):
    """Business logic that validates and executes an incoming order"""
    data = json.loads(content_str)
    
    # Validation rules: order must have positive amount and customer name
    if "amount" not in data or data["amount"] <= 0:
        raise ValueError("Invalid order: amount must be greater than 0")
    if "customer" not in data or not data["customer"].strip():
        raise ValueError("Invalid order: missing customer name")
        
    logging.info(f"==> [SUCCESS] Order {data.get('order_id')} for {data['customer']} (${data['amount']}) processed!")

def main():
    logging.info("Starting Azure Functions QueueTrigger worker daemon...")
    main_client, poison_client = init_queues()
    
    while True:
        # Receive up to 1 message with visibility timeout
        messages = main_client.receive_messages(messages_per_page=1, visibility_timeout=VISIBILITY_TIMEOUT)
        
        message_found = False
        for msg in messages:
            message_found = True
            raw_content = msg.content.decode('utf-8')
            logging.info(f"Received Message ID: {msg.id[:8]}... | Attempt {msg.dequeue_count} of {MAX_DEQUEUE_COUNT}")
            
            # Check for poison message threshold
            if msg.dequeue_count >= MAX_DEQUEUE_COUNT:
                logging.error(f"[POISON DETECTED] Message {msg.id[:8]} exceeded {MAX_DEQUEUE_COUNT} attempts. Dead-lettering to {POISON_QUEUE_NAME}!")
                
                # Move to poison queue with diagnostic audit trail
                poison_payload = {
                    "original_message_id": msg.id,
                    "failed_attempts": msg.dequeue_count,
                    "dead_letter_time": time.strftime("%Y-%m-%dT%H:%M:%SZ"),
                    "raw_payload": raw_content
                }
                poison_client.send_message(json.dumps(poison_payload).encode('utf-8'))
                
                # Delete from main queue to unblock the line
                main_client.delete_message(msg.id, msg.pop_receipt)
                logging.info(f"Message {msg.id[:8]} removed from {MAIN_QUEUE_NAME} and quarantined.")
                continue

            try:
                # Attempt processing
                process_order(raw_content)
                # Success: Acknowledge and delete message
                main_client.delete_message(msg.id, msg.pop_receipt)
                logging.info(f"Message {msg.id[:8]} deleted from {MAIN_QUEUE_NAME}.")
            except Exception as e:
                logging.warning(f"Processing failed for Message {msg.id[:8]}: {e}. Will retry in {VISIBILITY_TIMEOUT}s...")
                # Do NOT delete message; visibility timeout will elapse and re-surface it
                
        if not message_found:
            time.sleep(1)

if __name__ == "__main__":
    main()

```

##### 3. Create the Worker Dockerfile

Create `worker_app/Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    iproute2 \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY worker.py .

CMD ["python", "-u", "worker.py"]

```

---

#### Step 3: Create the Lab Definition (`docker-compose.yml`)

Return to the root directory (`~/azure-labs/lab5-4-queue-poison`) and create `docker-compose.yml`:

```yaml
services:
  # ==========================================================
  # Azure Storage Emulator: Azurite Queue Service (spoke-data)
  # ==========================================================
  azurite-queue:
    image: mcr.microsoft.com/azure-storage/azurite:latest
    container_name: azurite-queue
    # Internal private IP only - no host port publishing!
    command: "azurite --queueHost 0.0.0.0 --queuePort 10001 --loose"
    networks:
      spoke-data:
        ipv4_address: 10.2.0.10

  # ==========================================================
  # Serverless Consumer: QueueTrigger Worker (spoke-compute)
  # ==========================================================
  queue-worker-service:
    build:
      context: ./worker_app
      dockerfile: Dockerfile
    container_name: queue-worker-service
    environment:
      - STORAGE_CONN_STR=DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;QueueEndpoint=http://10.2.0.10:10001/devstoreaccount1;
    networks:
      spoke-compute:
        ipv4_address: 10.1.0.25

  # ==========================================================
  # Producer Container: Simulates Frontend Apps Publishing Orders
  # ==========================================================
  order-producer:
    image: nicolaka/netshoot:latest
    container_name: order-producer
    stdin_open: true
    tty: true
    cap_add:
      - NET_ADMIN
    networks:
      spoke-compute:
        ipv4_address: 10.1.0.15

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

```

---

#### Step 4: Spin Up the Stack

Launch the environment:

```bash
docker compose up -d --build

```

Verify that all three containers are healthy:

```bash
docker compose ps

```

---

#### Step 5: Configure Internal Private Network Routing

Configure the routing path so `order-producer` and `queue-worker-service` can route directly to the private data subnet (`10.2.0.0/24`) using the Docker bridge gateway:

```bash
docker exec -it order-producer ip route add 10.2.0.0/24 via 10.1.0.1
docker exec -it queue-worker-service ip route add 10.2.0.0/24 via 10.1.0.1

```

Verify that `order-producer` can reach Azurite's Queue listener on port 10001:

```bash
docker exec -it order-producer nc -zv -w 2 10.2.0.10 10001

```

*Expected Output:*

```text
10.2.0.10 10001 (scp-config) open

```

---

### Step 6: Verification & Experiments

#### Experiment 1: The Happy Path — Processing a Valid Message

Let's post a valid order into the `orders-incoming` queue from `order-producer`.

1. Open a **second terminal window** on your Mac mini and stream the logs of `queue-worker-service`:

```bash
docker logs -f queue-worker-service

```

2. In your **first terminal window**, use a quick inline Python command inside `order-producer` to publish a valid order:

```bash
docker exec -it order-producer python -c "
import json
from azure.storage.queue import QueueClient, BinaryBase64EncodePolicy

conn_str = 'DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;QueueEndpoint=http://10.2.0.10:10001/devstoreaccount1;'
client = QueueClient.from_connection_string(conn_str, 'orders-incoming', message_encode_policy=BinaryBase64EncodePolicy())

valid_order = {'order_id': 'ORD-1001', 'customer': 'Acme Global', 'amount': 1500}
client.send_message(json.dumps(valid_order).encode('utf-8'))
print('Valid order sent!')
"

```

3. Look at the output in **Terminal 2 (worker logs)**:

```text
[INFO] Received Message ID: a1b2c3d4... | Attempt 1 of 5
[INFO] ==> [SUCCESS] Order ORD-1001 for Acme Global ($1500) processed!
[INFO] Message a1b2c3d4... deleted from orders-incoming.

```

**Observation:**

* `DequeueCount` was `1`.
* The order was successfully processed.
* The message was acknowledged and deleted immediately from the queue.

---

#### Experiment 2: The Poison Message Disaster (The Chaos Test)

Now let's submit a **poison message** containing corrupted, invalid data that deliberately crashes the worker logic (e.g., negative amount and malformed fields).

In your first terminal window, send the poisoned order:

```bash
docker exec -it order-producer python -c "
import json
from azure.storage.queue import QueueClient, BinaryBase64EncodePolicy

conn_str = 'DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;QueueEndpoint=http://10.2.0.10:10001/devstoreaccount1;'
client = QueueClient.from_connection_string(conn_str, 'orders-incoming', message_encode_policy=BinaryBase64EncodePolicy())

# Corrupted payload: negative amount and missing customer field
poison_order = {'order_id': 'ORD-POISON-999', 'amount': -500}
client.send_message(json.dumps(poison_order).encode('utf-8'))
print('Poison order sent!')
"

```

Now, watch **Terminal 2 (worker logs)** in real-time as the failure unfolds:

```text
[INFO] Received Message ID: f9e8d7c6... | Attempt 1 of 5
[WARNING] Processing failed for Message f9e8d7c6...: Invalid order: amount must be greater than 0. Will retry in 3s...

[INFO] Received Message ID: f9e8d7c6... | Attempt 2 of 5
[WARNING] Processing failed for Message f9e8d7c6...: Invalid order: amount must be greater than 0. Will retry in 3s...

[INFO] Received Message ID: f9e8d7c6... | Attempt 3 of 5
[WARNING] Processing failed for Message f9e8d7c6...: Invalid order: amount must be greater than 0. Will retry in 3s...

[INFO] Received Message ID: f9e8d7c6... | Attempt 4 of 5
[WARNING] Processing failed for Message f9e8d7c6...: Invalid order: amount must be greater than 0. Will retry in 3s...

[INFO] Received Message ID: f9e8d7c6... | Attempt 5 of 5
[ERROR] [POISON DETECTED] Message f9e8d7c6... exceeded 5 attempts. Dead-lettering to orders-incoming-poison!
[INFO] Message f9e8d7c6... removed from orders-incoming and quarantined.

```

**What happened:**

1. Each time processing raised a `ValueError`, the worker did **not** delete the message.
2. The message stayed in the queue, became invisible for 3 seconds, and then resurfaced with an incremented `DequeueCount`.
3. When `DequeueCount` reached `5`, the runtime triggered the dead-letter policy:
* It copied the failing message into `orders-incoming-poison`.
* It deleted the message from `orders-incoming`.
* The active queue was unblocked and ready for subsequent orders!



---

#### Experiment 3: Verifying Quarantine & Inspecting the Poison Queue

Let's verify that the main queue is completely empty, while the dead-letter queue holds the bad message for SRE triage.

Run an inspection script from `order-producer`:

```bash
docker exec -it order-producer python -c "
import json
from azure.storage.queue import QueueClient, BinaryBase64DecodePolicy

conn_str = 'DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;QueueEndpoint=http://10.2.0.10:10001/devstoreaccount1;'

main_client = QueueClient.from_connection_string(conn_str, 'orders-incoming', message_decode_policy=BinaryBase64DecodePolicy())
poison_client = QueueClient.from_connection_string(conn_str, 'orders-incoming-poison', message_decode_policy=BinaryBase64DecodePolicy())

print('--- Active Queue (orders-incoming) ---')
main_props = main_client.get_queue_properties()
print(f'Approximate Message Count: {main_props.approximate_message_count}')

print('\n--- Dead-Letter Queue (orders-incoming-poison) ---')
poison_props = poison_client.get_queue_properties()
print(f'Approximate Message Count: {poison_props.approximate_message_count}')

messages = poison_client.peek_messages(max_messages=1)
for m in messages:
    print('Quarantined Payload Content:')
    print(json.dumps(json.loads(m.content.decode('utf-8')), indent=2))
"

```

*Expected Output:*

```text
--- Active Queue (orders-incoming) ---
Approximate Message Count: 0

--- Dead-Letter Queue (orders-incoming-poison) ---
Approximate Message Count: 1
Quarantined Payload Content:
{
  "original_message_id": "f9e8d7c6-...",
  "failed_attempts": 5,
  "dead_letter_time": "2026-11-08T10:35:12Z",
  "raw_payload": "{\"order_id\": \"ORD-POISON-999\", \"amount\": -500}"
}

```

The corrupted message is quarantined with full audit context, ready for engineers to review without having crashed the production consumer!

---

#### Experiment 4: Pipeline Resilience (Subsequent Messages Still Flow)

To prove the pipeline did not freeze during the poison message lifecycle, send a new valid order:

```bash
docker exec -it order-producer python -c "
import json
from azure.storage.queue import QueueClient, BinaryBase64EncodePolicy

conn_str = 'DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;QueueEndpoint=http://10.2.0.10:10001/devstoreaccount1;'
client = QueueClient.from_connection_string(conn_str, 'orders-incoming', message_encode_policy=BinaryBase64EncodePolicy())

order = {'order_id': 'ORD-1002', 'customer': 'Fabrikam Inc', 'amount': 4200}
client.send_message(json.dumps(order).encode('utf-8'))
"

```

Check **Terminal 2 (worker logs)**:

```text
[INFO] Received Message ID: 12345678... | Attempt 1 of 5
[INFO] ==> [SUCCESS] Order ORD-1002 for Fabrikam Inc ($4200) processed!
[INFO] Message 12345678... deleted from orders-incoming.

```

The queue recovered smoothly with zero downtime and zero human intervention.

---

### Step 7: Clean Up

When you've finished testing, stop and clean up the containers:

```bash
docker compose down -v

```

---

### Key Enterprise Takeaways for the Team

* **Asynchronous Decoupling Saves Systems:** Putting an Azure Storage Queue or Service Bus queue in front of databases protects systems from traffic surges and rate-limiting.
* **The `maxDequeueCount` Parameter:** In Azure Functions, `host.json` configures queue behavior:
```json
{
  "version": "2.0",
  "extensions": {
    "queues": {
      "maxPollingInterval": "00:00:02",
      "visibilityTimeout": "00:00:30",
      "maxDequeueCount": 5
    }
  }
}

```


* **Alerting on the Poison Queue:** SRE teams should configure an **Azure Monitor Metric Alert** on `orders-incoming-poison` where `ApproximateMessageCount > 0` to trigger a Slack or PagerDuty alert whenever a bad message is quarantined.
