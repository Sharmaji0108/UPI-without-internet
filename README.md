# UPI Offline Mesh — Demo Backend

A Spring Boot proof of concept that explores how encrypted payment instructions could travel across a simulated Bluetooth-style mesh when devices do not have internet access, then reach a backend through a connected bridge device.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Transaction Flow](#transaction-flow)
- [Security Design](#security-design)
- [Technology Stack](#technology-stack)
- [Project Structure](#project-structure)
- [API Reference](#api-reference)
- [Demo Accounts](#demo-accounts)
- [Run Locally](#run-locally)
- [Data Model](#data-model)
- [Limitations and Future Work](#limitations-and-future-work)
- [Disclaimer](#disclaimer)

## Overview

Traditional digital payments generally require connectivity to authorize and process a transaction. This project demonstrates a prototype flow in which a sender creates an encrypted payment instruction, virtual devices relay the packet, and a bridge device submits it to a Spring Boot backend when connectivity is available.

The backend checks duplicate delivery, decrypts the instruction, validates its timestamp, and attempts to update a simulated account ledger.

## Features

- Simulated mesh devices and packet gossip/forwarding
- Time-to-live (TTL) value to limit packet forwarding
- Hybrid encryption: AES-GCM for payment data and RSA-OAEP for the AES key
- SHA-256 ciphertext hash used as an idempotency key
- Duplicate packet rejection using a concurrent in-memory cache
- Scheduled cleanup of expired idempotency-cache entries
- Timestamp freshness checks to reject stale or far-future instructions
- Simulated account balances and transactional debit/credit processing
- Optimistic locking on account records
- H2 database dependency and Spring Data JPA persistence
- Web dashboard using Spring MVC and Thymeleaf

## Architecture

```text
Sender simulation
      |
      | PaymentInstruction
      v
HybridCryptoService
  AES-256-GCM encrypts instruction
  RSA-OAEP encrypts AES key
      |
      v
MeshPacket (ciphertext + packet ID + TTL)
      |
      v
MeshSimulatorService
  VirtualDevice instances gossip packets
      |
      v
Virtual bridge device (simulated internet)
      |
      | POST /api/bridge/ingest
      v
ApiController
      |
      v
BridgeIngestionService
  1. Hash ciphertext
  2. Claim idempotency key
  3. Decrypt and authenticate ciphertext
  4. Check timestamp freshness
      |
      v
SettlementService (@Transactional)
  Validate sender/receiver and amount
  Debit sender + credit receiver
  Save transaction record
      |
      v
H2 / JPA ledger
```

## Transaction Flow

1. The demo creates a payment instruction containing sender VPA, receiver VPA, amount, a nonce, a PIN hash, and a timestamp.
2. `HybridCryptoService` serializes and encrypts the instruction. It generates a fresh AES key, encrypts the JSON using AES-GCM, and encrypts the AES key using the server's RSA public key.
3. The encrypted data is wrapped in a `MeshPacket`.
4. `MeshSimulatorService` passes packets between virtual devices, reducing TTL on forwarded copies.
5. A virtual bridge device holds a copy and the mesh flush operation submits packets to the ingestion pipeline.
6. `BridgeIngestionService` hashes the ciphertext and uses `IdempotencyService` to reject duplicate delivery.
7. The backend decrypts the packet and checks the instruction timestamp.
8. `SettlementService` validates the simulated accounts and amount. If the sender has enough balance, it debits the sender, credits the receiver, and records a `SETTLED` transaction. If funds are insufficient, it records a `REJECTED` transaction.
9. The dashboard/API can be used to inspect simulated accounts and recent transactions.

## Security Design

### Hybrid encryption

- **AES-256-GCM** encrypts the payment instruction and detects ciphertext tampering through its authentication tag.
- **RSA-OAEP with SHA-256** encrypts the one-time AES key using the server's public key.
- The server's private key is used for decryption by the demo's `ServerKeyHolder`.

### Duplicate delivery / idempotency

The SHA-256 hash of the ciphertext is used as the idempotency key. `ConcurrentHashMap.putIfAbsent` makes the in-memory claim atomic within one JVM. The transaction entity also declares a unique constraint on `packetHash` as a database-level defense.

### Freshness and concurrency

The ingestion service rejects old packets and packets dated more than five minutes into the future. Account entities use JPA `@Version` optimistic locking, and settlement runs in a database transaction.

## Technology Stack

| Technology | Purpose |
|---|---|
| Java 17 | Application language |
| Spring Boot | Application framework |
| Spring Web MVC | REST endpoints and web controller |
| Spring Data JPA | Persistence and repository abstraction |
| H2 | Database dependency for local/demo use |
| Thymeleaf | Server-rendered dashboard |
| Maven | Build and dependency management |
| Java Cryptography APIs | AES-GCM, RSA-OAEP, SHA-256 |
| JUnit / Spring Boot test dependencies | Application tests |

The exact dependency versions are controlled by the project's `pom.xml`. Confirm the configured Spring Boot version and dependency compatibility in your local build before publishing.

## Project Structure

```text
src/main/java/com/project/upi_mesh/
├── Application.java
├── config/
│   └── AppConfig.java
├── controller/
│   ├── ApiController.java
│   └── DashboardController.java
├── crypto/
│   ├── HybridCryptoService.java
│   └── ServerKeyHolder.java
├── model/
│   ├── Account.java
│   ├── AccountRepository.java
│   ├── MeshPacket.java
│   ├── PaymentInstruction.java
│   ├── Transaction.java
│   └── TransactionRepository.java
└── service/
    ├── BridgeIngestionService.java
    ├── DemoService.java
    ├── IdempotencyService.java
    ├── MeshSimulatorService.java
    └── VirtualDevice.java
```

## API Reference

The following routes are defined in the uploaded `ApiController.java`. The controller uses the `/api` base path.

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/mesh/gossip` | Run one simulated packet-gossip round |
| `POST` | `/api/mesh/flush` | Submit packets held by virtual bridge devices to the ingestion pipeline |
| `POST` | `/api/mesh/reset` | Clear virtual-device packet state and the idempotency cache |
| `POST` | `/api/bridge/ingest` | Ingest a mesh packet |
| `GET` | `/api/accounts` | List simulated accounts |
| `GET` | `/api/transactions` | List the latest 20 transactions |

### `POST /api/bridge/ingest`

Request body: a `MeshPacket` JSON object, including its `packetId`, `ttl`, `createdAt`, and `ciphertext` fields.

Optional headers:

- `X-Bridge-Node-Id` — bridge identifier; defaults to `unknown`
- `X-Hop-Count` — number of hops; defaults to `0`

The response contains an outcome such as `SETTLED`, `DUPLICATE_DROPPED`, or `INVALID`, along with the packet hash and relevant transaction/reason fields.

### Dashboard

The MVC controller maps `/` to the Thymeleaf view named `dashboard`. Ensure the corresponding template is included under `src/main/resources/templates/dashboard.html` in the complete project.

## Demo Accounts

The startup seeding service creates these accounts when the account table is empty:

| VPA | Holder | Initial balance |
|---|---|---:|
| `alice@demo` | Alice | ₹5,000.00 |
| `bob@demo` | Bob | ₹1,000.00 |
| `carol@demo` | Carol | ₹2,500.00 |
| `dave@demo` | Dave | ₹500.00 |

These are dummy accounts for local demonstration only.

## Run Locally

### Prerequisites

- JDK 17
- Maven (or the Maven Wrapper, if included)
- Eclipse IDE or another Java IDE

### Run from Eclipse

1. Import the project as an **Existing Maven Project**.
2. Confirm the project JRE is Java 17.
3. In the project, right-click `pom.xml` and choose **Maven → Update Project**.
4. Wait for dependencies to download.
5. Run `Application.java` using **Run As → Java Application** or **Spring Boot App** if the Spring Tools plugin is installed.
6. Open `http://localhost:8080/` for the dashboard, if the template is present.

### Run from a terminal

If Maven is installed:

```bash
mvn clean test
mvn spring-boot:run
```

If the project includes the Maven Wrapper, use `mvnw.cmd` on Windows or `./mvnw` on macOS/Linux.

> The uploaded source archive did not include `src/main/resources`, the dashboard template, or test source files. Make sure to include those from your complete Eclipse project. Also check `application.properties` for the intended database configuration before running.

## Data Model

### `Account`

- `vpa` — primary key
- `holderName` — account holder name
- `balance` — decimal balance
- `version` — optimistic-locking version

### `Transaction`

- `id` — generated primary key
- `packetHash` — unique SHA-256 hash of ciphertext
- `senderVpa`, `receiverVpa` — sender and receiver identifiers
- `amount` — transaction amount
- `signedAt`, `settledAt` — instruction and settlement timestamps
- `bridgeNodeId` — bridge that delivered the packet
- `hopCount` — recorded hop count
- `status` — `SETTLED` or `REJECTED`

## Limitations and Future Work

- **No real Bluetooth implementation:** the mesh uses in-memory virtual devices; no BLE discovery, GATT transfer, or physical phone-to-phone communication is implemented.
- **No real UPI/bank integration:** balances and settlement are simulated locally.
- **Offline double-spend risk:** disconnected devices cannot reliably know the latest account balance. A production design needs trusted authorization, spending limits, reconciliation, and bank-side controls.
- **In-memory idempotency:** the cache is local to one process and is not shared between application instances. A distributed store and robust database constraints would be needed for production.
- **Key lifecycle:** the demo key holder should be replaced by secure key provisioning, rotation, and protected private-key storage.
- **Authentication/authorization:** the exposed demo endpoints should not be treated as production-secure APIs.
- **Replay protection:** timestamps help, but production systems need a fully specified trust model and stronger request authorization.
- **Validation and error handling:** production code needs comprehensive input validation, consistent error responses, observability, and tests for concurrency and failure cases.

Potential next steps include a real Android BLE prototype, distributed idempotency, stronger signing/authorization, reconciliation workflows, API security, and automated integration tests.

## Disclaimer

This repository is for learning and demonstration. It is not affiliated with NPCI, does not connect to UPI rails, and must not be used to process real payments.
