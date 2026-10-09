# The Engineering Question & Architectural Strategy

### The Question
> **How do you engineer an asynchronous, zero-data-loss event engine that ingests payloads in sub-5ms, protects workers from hostile/failing endpoints, and guarantees delivery without relying on proprietary cloud lock-in?**

---

### HookPulse's 4-Part Architectural Strategy

1. **Sub-5ms Ingestion (Decoupling):**  
   The API validates payloads, writes a durable `pending` record to PostgreSQL, enqueues the task in Redis, and immediately returns `202 Accepted`—producers never wait on external network latency.

2. **Worker Protection (Isolation):**  
   Outbound dispatchers use `httpx.AsyncClient` with strict socket timeouts and **per-domain Circuit Breakers** that trip `OPEN` on consecutive failures, preventing crashing endpoints from exhausting the worker pool.

3. **Guaranteed Delivery (Zero Data Loss):**  
   Temporary network drops trigger **Exponential Backoff with Full Jitter**. Hard failures are routed to a **Dead-Letter Queue (DLQ)** with full audit history, allowing manual or automated replays.

4. **Zero Cloud Lock-in (Portability):**  
   The system relies solely on open-source distributed primitives (FastAPI, Redis, PostgreSQL, Docker), avoiding proprietary cloud queues (SQS/EventBridge) and costly SaaS lock-in.
