# System Design Summary: Idempotency

Summarized from [AlgoMaster.io](https://algomaster.io/learn/system-design/idempotency)

## Overview
**Idempotency** is a fundamental principle in distributed systems. It ensures that performing an operation multiple times has the same effect as performing it once. This is critical for building reliable systems that can handle network failures and retries without corrupting data.



## 1. The Mathematical Definition
An operation $f(x)$ is idempotent if:
$$f(f(x)) = f(x)$$
In software terms, if you send the same "Pay $50" request ten times, the user should only be charged $50 once.

## 2. Idempotency in HTTP Methods

| Method | Idempotent | Description |
| :--- | :--- | :--- |
| **GET** | **Yes** | Used to fetch data. Repeated calls do not change the state of the resource. |
| **PUT** | **Yes** | Replaces a resource. Updating the same field to the same value repeatedly results in the same state. |
| **DELETE** | **Yes** | Deleting an object once is the same as deleting it multiple times (it remains gone). |
| **POST** | **No** | Typically used to create resources. Sending a "Create Order" POST twice usually creates two orders. |

## 3. How to Implement Idempotency (The Idempotency Key)
The standard way to make non-idempotent operations safe is by using a **Unique Idempotency Key**.

1. **Key Generation:** The client generates a unique ID (like a UUID) for the transaction.
2. **Key Submission:** The client sends this key in the HTTP header (e.g., `Idempotency-Key: 123e4567-e89b...`).
3. **Server Validation:**
   - The server checks if the key exists in a cache (like Redis).
   - **If it's new:** The server processes the request and stores the result linked to that key.
   - **If it's a duplicate:** The server skips the logic and returns the previously stored result.



## 4. Key Benefits
* **Prevents Double Charging:** Essential for payment gateways (Stripe, PayPal).
* **Network Resilience:** Allows clients to safely retry failed requests due to timeouts.
* **Consistency:** Ensures that "at-least-once" delivery in message queues doesn't result in duplicate data processing.


# Example:
```python
from fastapi import FastAPI, Header, HTTPException
import redis
import uuid

app = FastAPI()
# Connecting to Redis to store processed request keys
r = redis.Redis(host='localhost', port=6379, decode_responses=True)

@app.post("/create-order")
def create_order(order_data: dict, idempotency_key: str = Header(None)):
    if not idempotency_key:
        raise HTTPException(status_code=400, detail="Idempotency-Key header is missing")

    # 1. Check if the key already exists in Redis
    existing_response = r.get(idempotency_key)
    
    if existing_response:
        # If found, return the cached response immediately
        return {"status": "success", "message": "Duplicate request detected", "data": existing_response}

    # 2. Simulate business logic (e.g., saving to a database)
    order_id = str(uuid.uuid4())
    processed_data = f"Order {order_id} created for {order_data.get('item')}"

    # 3. Store the result in Redis with an expiration (e.g., 24 hours)
    # This prevents the same request from being processed again
    r.setex(idempotency_key, 86400, processed_data)

    return {"status": "success", "message": "Order created", "data": processed_data}
```
---