# HookPulse

> High-throughput, fault-tolerant webhook delivery and ingestion engine built with FastAPI, Redis, and PostgreSQL.

---

## 01. Overview

**HookPulse** is a resilient, asynchronous webhook dispatching engine designed to handle high-frequency event ingestion and guarantee at-least-once delivery to external consumer endpoints.

* **Primary Stack:** Python 3.11+, FastAPI, Redis, PostgreSQL, HTTPX, Docker
* **Core Patterns:** Non-blocking async ingestion, Exponential Backoff with Jitter, Per-Domain Circuit Breaker, HMAC-SHA256 Signatures, Dead-Letter Queue (DLQ).

---

## 02. Why I Built This

Sending webhooks looks simple on paper: `requests.post(url, payload)`. But in production, third-party endpoints are unpredictable—they drop connections, time out, or throw 500 errors. 

A naive synchronous implementation causes:
* **Thread Starvation:** A few slow customer servers block backend threads, freezing the entire API.
* **Silent Data Loss:** Network hiccups cause failed deliveries to be lost forever.
* **Noisy Neighbors:** One crashing destination consumes all retry resources, starving healthy destinations.

---

## 03. Problems Solved

| Problem | Naive Implementation | HookPulse Solution |
| :--- | :--- | :--- |
| **API Latency Spikes** | Ingestion blocks while awaiting remote HTTP response | Decoupled ingestion returns `202 Accepted` in $<5\text{ms}$; work offloaded to Redis. |
| **Flaky Endpoints** | Dropped payload on first failure | At-least-once delivery via **Exponential Backoff with Full Jitter** to prevent thundering herds. |
| **Poison Endpoints** | Endless retries starve system connections | **Circuit Breaker** automatically trips OPEN after consecutive failures, isolating the domain. |
| **Payload Forgery** | Receiver cannot verify sender identity | Cryptographic **HMAC-SHA256** signature in `X-HookPulse-Signature` header. |
| **Terminal Failures** | Failures disappear into server logs | **Dead-Letter Queue (DLQ)** with full attempt audit history and one-click replay. |

---

## 04. Architecture & Tech Stack

```
                     STAGE 1: FAST INGESTION (< 5ms)
                     ───────────────────────────────
[ Client / Producer ] ──► [ FastAPI API ]
                                 │
                                 ├──► 1. Validate payload schema (Pydantic v2)
                                 ├──► 2. Persist event to PostgreSQL ("pending")
                                 ├──► 3. Push event task to Redis Stream/Queue
                                 └──► 4. Return HTTP 202 Accepted immediately

                     STAGE 2: ASYNCHRONOUS WORKER DISPATCH
                     ─────────────────────────────────────
                                [ Redis Queue ]
                                       │
                                       ▼
                       [ Asyncio Worker Pool (HTTPX) ]
                                       │
                  ┌────────────────────┴────────────────────┐
                  ▼                                         ▼
         [ HTTP 2xx Response ]                    [ 5xx Error / Timeout ]
                  │                                         │
                  ▼                                         ▼
        Mark "delivered" in DB                   Exponential Backoff + Jitter
                                                            │
                                                            ▼ (Max Retries Exceeded)
                                                  [ Dead-Letter Queue (DLQ) ]
                                                  - Mark "failed" in DB
                                                  - Circuit Breaker trips for host
```

* **FastAPI:** Non-blocking async endpoints for ingestion and administration.
* **Redis:** In-memory event dispatch queue and circuit-breaker state cache.
* **PostgreSQL (SQLAlchemy 2.0 / asyncpg):** Persistent relational ledger for endpoints, events, and delivery attempts.
* **HTTPX (AsyncClient):** High-efficiency async HTTP client with connection pooling and strict socket timeouts.
* **Docker & Compose:** Fully containerized multi-service deployment.

---

<!-- ## 05. Quick Start & API Documentation

### Run Locally (Docker Compose)

```bash
# 1. Clone repository
git clone https://github.com/your-username/hookpulse.git
cd hookpulse

# 2. Start services (API, Worker, Redis, PostgreSQL)
docker-compose up -d

# 3. Access Swagger UI docs
open http://localhost:8000/docs
```

### Core API Endpoints

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `POST` | `/api/v1/endpoints` | Register a new target webhook destination URL & secret. |
| `POST` | `/api/v1/events` | Ingest event payload. Enqueues delivery and returns `202 Accepted`. |
| `GET` | `/api/v1/events/{id}/deliveries` | Inspect chronological delivery attempts, HTTP status codes, and latency. |
| `POST` | `/api/v1/dlq/{id}/replay` | Manually re-queue a dead-lettered event for dispatch. |

---

## 06. Engineering Decisions & Testing

* **Asyncio + HTTPX over Heavy Task Runners:**  
  Webhook delivery requires strict per-socket timeout configurations, connection pool reuse, and fine-grained circuit breaking. Building a focused async worker runtime avoids the operational overhead and blind defaults of generic task frameworks.
* **Exponential Backoff with Full Jitter:**  
  Retries scale exponentially ($\text{delay} = 2^{\text{attempt}} \pm \text{jitter}$). Random jitter prevents thousands of retrying tasks from hitting a recovered server at the exact same millisecond (*thundering herd*).
* **At-Least-Once Delivery Contract:**  
  Events are acknowledged only after a verified `2xx` HTTP response from the receiver. In all other scenarios, events transition through retry schedules or DLQ retention.
* **Testing Strategy:**  
  * **Unit & Integration:** Automated Pytest suite covering HMAC generation/validation, backoff calculations, and circuit breaker state transitions.
  * **Load Testing:** Locust scripts simulating concurrent event ingestion to measure throughput and p95/p99 latency.

---

## 07. Limitations & Scope Boundaries

* **At-Least-Once Semantics:** Because network requests can fail after the recipient processes a payload but before the ACK is received, consumer endpoints must implement idempotency.
* **Single-Cluster Scope:** Designed and optimized for single-cluster deployments with Redis and PostgreSQL; multi-region active-active federation is outside current scope.
* **Payload Transformation:** HookPulse is strictly a delivery and ingestion engine—it does not perform arbitrary payload mutation or ETL mapping. -->