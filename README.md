# Unnati

> **An offline-first synchronization platform that combines event-driven processing, idempotent request handling, and transactional consistency to reliably synchronize mobile data across unreliable networks.**

![Go](https://img.shields.io/badge/Go-1.24-00ADD8?logo=go)
![Fiber](https://img.shields.io/badge/Fiber-Web%20Framework-00B894)
![Redis](https://img.shields.io/badge/Redis-7.x-DC382D?logo=redis)
![Apache Kafka](https://img.shields.io/badge/Apache-Kafka-black)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker)
![License](https://img.shields.io/badge/License-MIT-green)

---

## Table of Contents

- [Overview](#overview)
- [Application User Interface](#application-user-interface)
- [Tech Stack](#tech-stack)
- [System Architecture](#system-architecture)
- [Engineering Decisions](#engineering-decisions)
- [License](#license)
- [Author](#author)

---

# Overview

Unnati is an offline-first synchronization platform designed for mobile commerce applications operating in environments with unreliable or intermittent internet connectivity. Instead of relying on synchronous database writes, the platform adopts an event-driven architecture that decouples request ingestion from persistence, allowing mobile devices to synchronize data reliably even under unstable network conditions.

Traditional mobile applications frequently resend the same requests whenever acknowledgements are delayed or connections drop unexpectedly. Under poor network conditions, this often results in duplicate writes, inconsistent product states, race conditions, and unnecessary database load. Unnati addresses these challenges through a layered synchronization pipeline consisting of authentication, idempotency protection, binary serialization, asynchronous event streaming, and transactional conflict resolution.

Incoming synchronization requests are transmitted as compact Protocol Buffer (Protobuf) binary payloads to minimize bandwidth consumption on constrained mobile networks. Before any payload reaches durable storage, it passes through a Redis-backed Idempotency Shield that leverages atomic **SETNX** operations to detect and eliminate duplicate retry requests in constant time.

Rather than writing directly to the database, validated payloads are immediately published to Apache Kafka. This allows the API Gateway to acknowledge requests within milliseconds while background worker processes asynchronously consume events and perform database synchronization. Each worker executes ACID-compliant PostgreSQL transactions using row-level locking together with Optimistic Concurrency Control (OCC), ensuring that delayed or out-of-order events never overwrite newer product versions.

The overall architecture emphasizes reliability, fault tolerance, and horizontal scalability, enabling the synchronization layer to absorb sudden traffic spikes, tolerate unstable mobile connectivity, and maintain strict data consistency without sacrificing throughput or responsiveness.

### Core Engineering Objectives

- Build an offline-first synchronization platform resilient to intermittent mobile connectivity.
- Achieve low-latency request ingestion through an asynchronous event-driven architecture.
- Prevent duplicate synchronization requests using Redis-backed atomic idempotency locks.
- Minimize mobile bandwidth consumption through compact Protobuf binary serialization.
- Decouple API ingestion from database persistence using Apache Kafka event streaming.
- Guarantee transactional consistency using PostgreSQL row-level locking and Optimistic Concurrency Control.
- Enable independent horizontal scaling of API Gateways and worker services.
- Ensure reliable synchronization under concurrent update workloads and network instability.

### Core Features

- High-performance Fiber API Gateway
- Offline-first synchronization workflow
- JWT-based authentication middleware
- Redis-powered Idempotency Shield
- Atomic SETNX duplicate request protection
- Protocol Buffer binary payload ingestion
- Apache Kafka asynchronous event streaming
- Horizontally scalable worker consumers
- PostgreSQL transactional synchronization
- Optimistic Concurrency Control (OCC)
- Row-level locking using "FOR UPDATE"
- Secure password hashing with bcrypt
- Dockerized local infrastructure
- Event-driven distributed architecture

---

# Application User Interface

<table>
  <tr>
    <td align="center">
      <img width="649" alt="unnati" src="https://github.com/user-attachments/assets/b8c25617-6e55-46cd-9ff3-2489ad79f576" />
    </td>
  </tr>
  <tr>
    <td align="center">
      <img width="712" alt="unnati1" src="https://github.com/user-attachments/assets/58a9a0fd-34b2-458f-89a6-38e49d7a54fb" />
    </td>
  </tr>
</table>

---

# Tech Stack

| Layer | Technologies | Purpose |
|-------|--------------|---------|
| **API Gateway** | Go, Fiber | High-performance request routing, Protobuf ingestion, authentication, and request validation |
| **Authentication** | JWT, bcrypt | Stateless seller authentication and secure password hashing |
| **Caching & Idempotency** | Redis | Duplicate request detection through atomic SETNX operations |
| **Message Broker** | Apache Kafka | Durable asynchronous event streaming between gateway and workers |
| **Core Worker** | Go, kafka-go | Background event processing and database synchronization |
| **Database** | PostgreSQL | ACID-compliant persistence, row-level locking, optimistic concurrency control |
| **Serialization** | Protocol Buffers (Protobuf) | Compact binary payload serialization for bandwidth-efficient synchronization |
| **Concurrency Control** | PostgreSQL Transactions | Version validation, conflict resolution, row locking |
| **Infrastructure** | Docker, Docker Compose | Containerized deployment and local development environment |

---

# System Architecture

<p align="center">
  <img src="https://github.com/user-attachments/assets/ff029ef3-4167-4d99-beb5-b5039fa17fe1"
       alt="High Level Design"
       width="100%">
</p>

---
# Engineering Decisions

The architecture of **Unnati** is designed around **offline-first reliability**, **event-driven scalability**, and **transactional consistency**. Every architectural decision prioritizes handling unreliable mobile networks while guaranteeing that synchronized data remains correct, even under duplicate requests, concurrent updates, or delayed event delivery.

| Design Decision | Rationale |
|----------------|-----------|
| **Why an Offline-First Architecture?** | Mobile applications frequently operate in environments with unstable or intermittent internet connectivity. An offline-first synchronization model allows users to continue working while data is safely synchronized once connectivity is restored. |
| **Why Go Fiber?** | Fiber is built on top of `fasthttp`, providing extremely low memory overhead and high request throughput. It enables the gateway to ingest thousands of concurrent synchronization requests with minimal latency. |
| **Why Protocol Buffers instead of JSON?** | Binary Protobuf serialization significantly reduces payload size compared to JSON, minimizing bandwidth usage and serialization overhead for mobile devices operating on slow or unreliable networks. |
| **Why JWT Authentication?** | Stateless JWT authentication eliminates repeated database lookups for every synchronization request while securely identifying sellers before data enters the synchronization pipeline. |
| **Why Redis as an Idempotency Shield?** | Mobile applications often resend requests whenever acknowledgements are delayed. Redis uses atomic `SETNX` operations to detect duplicate requests in constant time, preventing unnecessary Kafka events and duplicate database writes. |
| **Why Atomic SETNX?** | `SETNX` guarantees that only the first synchronization request with a given idempotency key is accepted. Any subsequent retries are safely discarded without affecting downstream systems. |
| **Why Apache Kafka?** | Kafka decouples API ingestion from persistence, allowing the gateway to immediately acknowledge synchronization requests while worker services process events asynchronously. This prevents API threads from blocking on database operations during traffic spikes. |
| **Why an Event-Driven Architecture?** | Asynchronous event streaming improves fault tolerance, absorbs burst traffic, and allows gateway services and worker services to scale independently without creating database bottlenecks. |
| **Why Dedicated Worker Services?** | Background workers isolate expensive synchronization logic from user-facing APIs, improving responsiveness while allowing worker replicas to scale horizontally under increased synchronization workloads. |
| **Why PostgreSQL?** | PostgreSQL provides strong ACID guarantees, transactional consistency, row-level locking, and mature concurrency control mechanisms that are essential for reliable synchronization systems. |
| **Why Optimistic Concurrency Control (OCC)?** | Kafka provides at-least-once delivery, meaning delayed messages may arrive after newer updates. Version checking ensures that stale synchronization events are ignored, preventing outdated data from overwriting the latest product state. |
| **Why Row-Level `FOR UPDATE` Locks?** | Multiple worker instances may attempt to synchronize the same product simultaneously. Row-level locks serialize concurrent updates and eliminate race conditions while maintaining transactional integrity. |
| **Why Transaction-Based Database Updates?** | Every synchronization operation executes within a PostgreSQL transaction, ensuring that partial failures never leave product records in an inconsistent state. |
| **Why Stateless Gateway Services?** | Keeping API gateways stateless allows them to scale horizontally behind a load balancer while Redis and Kafka maintain synchronization state externally. |
| **Why Docker Compose?** | Docker Compose provides a reproducible local development environment by orchestrating PostgreSQL, Redis, Kafka, and Zookeeper with minimal setup overhead. |

---

# License

Distributed under the **MIT License**.

See the `LICENSE` file for more information.

---

# Author

**Aryan Jaiswal**

- GitHub: https://github.com/AryanJais1224
- LinkedIn: https://www.linkedin.com/in/aryan-jaiswal-618965256/

---
