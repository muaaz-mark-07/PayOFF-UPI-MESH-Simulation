# PayOFF UPI MESH Simulation

[![Java 17](https://img.shields.io/badge/Java-17%2B-ED8B00?logo=openjdk&logoColor=white)](https://www.oracle.com/java/)
[![Spring Boot 3.3.5](https://img.shields.io/badge/Spring%20Boot-3.3.5-6DB33F?logo=springboot&logoColor=white)](https://spring.io/projects/spring-boot)
[![Maven](https://img.shields.io/badge/Build-Maven-C71A36?logo=apachemaven&logoColor=white)](https://maven.apache.org/)

A Java and Spring Boot demonstration of deferred payment settlement through a simulated offline, phone-to-phone mesh. It models encrypted payment packets moving between virtual devices and being submitted to a backend when a simulated bridge device has internet access.


## Contents

- [Overview](#overview)
- [Problem statement](#problem-statement)
- [Features](#features)
- [Technology stack](#technology-stack)
- [Architecture](#architecture)
- [Project structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Installation and local run](#installation-and-local-run)
- [API reference](#api-reference)
- [Tests](#tests)
- [Known limitations](#known-limitations)
- [Potential improvements](#potential-improvements)
- [Attribution and licensing](#attribution-and-licensing)

## Overview

The application demonstrates one possible way to defer settlement when a sender's device has no internet connection. A simulated sender creates a payment instruction, encrypts it for the backend, and adds it to a virtual mesh. Virtual devices exchange packets during simulated gossip rounds. When a device marked as an internet bridge is flushed, the backend receives its packets, checks duplicates and freshness, decrypts accepted instructions, and updates a sample account ledger.

All components run in one Spring Boot process. The sender, mesh devices, and bridge are software simulations; no mobile application, Bluetooth radio, bank integration, or real UPI network is included.

## Problem statement

Intermittent connectivity makes it difficult for a device to submit a payment immediately. This project explores the technical stages of a deferred-settlement model: carrying an opaque payment payload across intermediary devices, accepting a later delivery, and preventing the same delivered payload from being processed repeatedly.

It does not establish that the sender has spendable funds while offline. The backend decides whether to debit the sender only after the packet reaches it.

## Features

- Hybrid payload encryption using RSA-OAEP with SHA-256 and AES-256-GCM.
- A simulated mesh of virtual devices with packet forwarding and a hop time-to-live.
- A simulated bridge that submits held packets to the backend.
- Ciphertext-hash-based, JVM-local duplicate suppression.
- A 24-hour payment-instruction freshness window.
- Transactional updates to seeded sample accounts and a transaction ledger in H2.
- A browser dashboard and REST endpoints for operating the demonstration.
- Tests covering encryption/decryption, modified ciphertext rejection, and concurrent duplicate delivery.

## Technology stack

| Area | Technology |
|---|---|
| Language and runtime | Java 17 |
| Application framework | Spring Boot 3.3.5 |
| HTTP and dashboard | Spring Web MVC, embedded Tomcat, Thymeleaf |
| Persistence | Spring Data JPA, Hibernate, H2 in-memory database |
| JSON | Jackson |
| Build | Maven Wrapper |
| Tests | Spring Boot Test, JUnit 5 |

## Architecture

```text
Browser dashboard
       │ HTTP
       ▼
DashboardController and ApiController
       │
       ├── DemoService creates a payment instruction
       │        └── HybridCryptoService encrypts it for the server
       │
       ├── MeshSimulatorService exchanges packets between VirtualDevices
       │        └── Simulated bridge submits packets to /api/bridge/ingest
       │
       └── BridgeIngestionService
                ├── hashes ciphertext and claims it in IdempotencyService
                ├── decrypts and checks instruction freshness
                └── SettlementService updates accounts and transaction ledger
                            │
                            ▼
                     H2 in-memory database
```

The encrypted payload uses a fresh AES key per instruction. AES-GCM encrypts and authenticates the payload; RSA-OAEP protects the AES key so that only the server's private key can decrypt it. The bridge service hashes the ciphertext and uses an atomic `ConcurrentHashMap.putIfAbsent` operation to claim it before decryption. Successful settlement runs in a database transaction.

The public-key pair and idempotency cache are held in memory. The key pair is regenerated at each application start, and the H2 database is configured as in-memory with a create/drop lifecycle.

## Project structure

```text
.
├── .gitignore
├── .mvn/
│   └── wrapper/
│       └── maven-wrapper.properties
├── mvnw
├── mvnw.cmd
├── pom.xml
├── README.md
└── src/
    ├── main/
    │   ├── java/com/demo/upimesh/
    │   │   ├── UpiMeshApplication.java
    │   │   ├── config/
    │   │   │   └── AppConfig.java
    │   │   ├── controller/
    │   │   │   ├── ApiController.java
    │   │   │   └── DashboardController.java
    │   │   ├── crypto/
    │   │   │   ├── HybridCryptoService.java
    │   │   │   └── ServerKeyHolder.java
    │   │   ├── model/
    │   │   │   ├── Account.java
    │   │   │   ├── AccountRepository.java
    │   │   │   ├── MeshPacket.java
    │   │   │   ├── PaymentInstruction.java
    │   │   │   ├── Transaction.java
    │   │   │   └── TransactionRepository.java
    │   │   └── service/
    │   │       ├── BridgeIngestionService.java
    │   │       ├── DemoService.java
    │   │       ├── IdempotencyService.java
    │   │       ├── MeshSimulatorService.java
    │   │       ├── SettlementService.java
    │   │       └── VirtualDevice.java
    │   ├── resources/
    │   │   ├── application.properties
    │   │   └── templates/
    │   │       └── dashboard.html
    │   └── test/
    │       └── java/com/demo/upimesh/
    │           └── IdempotencyConcurrencyTest.java
```

## Prerequisites

- JDK 17 or later, available on `PATH` or configured through `JAVA_HOME`.
- Internet access for the first build so the Maven Wrapper can download Maven and dependencies.
- A free local port 8080. The configured port is controlled by the existing `server.port` property.

The included Maven Wrapper means a separate Maven installation is not required.

## Installation and local run

1. Open a terminal in the project directory.
2. Start the application with the wrapper for your operating system:

   **Windows PowerShell**

   ```powershell
   .\mvnw.cmd spring-boot:run
   ```

   **macOS or Linux**

   ```bash
   ./mvnw spring-boot:run
   ```

3. When Spring Boot reports that the application has started, open [http://localhost:8080](http://localhost:8080).
4. Use the dashboard to create a sample packet, run mesh gossip, and flush bridge-held packets.
5. Stop the application with `Ctrl+C` in the terminal.

The first run downloads Maven and project dependencies. The sample accounts and ledger are stored in the in-memory H2 database and are not retained after the application stops. The server's RSA key pair is also regenerated on startup.

## API reference

The application listens on port `8080` by default. Requests and responses use JSON unless noted otherwise.

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/` | Serve the dashboard. |
| `GET` | `/api/server-key` | Return the current server public key and encryption scheme metadata. |
| `POST` | `/api/demo/send` | Create an encrypted sample payment and inject it into a virtual device. |
| `GET` | `/api/mesh/state` | Return virtual-device packet counts and idempotency-cache size. |
| `POST` | `/api/mesh/gossip` | Run one simulated packet-exchange round. |
| `POST` | `/api/mesh/flush` | Submit packets held by virtual devices marked as internet bridges. |
| `POST` | `/api/mesh/reset` | Clear virtual-device packets and the local idempotency cache. |
| `POST` | `/api/bridge/ingest` | Submit one packet to the backend ingestion pipeline. |
| `GET` | `/api/accounts` | Return all sample accounts and balances. |
| `GET` | `/api/transactions` | Return the latest 20 ledger records. |
| `GET` | `/h2-console` | Open the development H2 console. |

### Create a simulated payment

`POST /api/demo/send`

```json
{
  "senderVpa": "alice@demo",
  "receiverVpa": "bob@demo",
  "amount": 25.00,
  "pin": "demo-only",
  "ttl": 5,
  "startDevice": "phone-alice"
}
```

The `ttl` and `startDevice` fields are optional; the defaults are `5` and `phone-alice`. The response contains the generated `packetId`, a ciphertext preview, the TTL, and the device receiving the packet. The supplied PIN is hashed as part of this demo flow; it is not checked against a bank credential.

### Submit a packet to the bridge endpoint

`POST /api/bridge/ingest`

Optional headers:

| Header | Default | Description |
|---|---|---|
| `X-Bridge-Node-Id` | `unknown` | Identifier recorded for the submitting bridge. |
| `X-Hop-Count` | `0` | Hop count recorded with the ledger entry. |

Example request shape:

```json
{
  "packetId": "packet-identifier",
  "ttl": 2,
  "createdAt": 1730000000000,
  "ciphertext": "base64-encoded-encrypted-payload"
}
```

The ciphertext must be a valid payload generated for the current server key; the placeholder above is not a usable packet. A response includes an `outcome` (`SETTLED`, `DUPLICATE_DROPPED`, or `INVALID`), a packet hash, an optional reason, and an optional transaction ID. Note that the current ingestion response reports `SETTLED` when the settlement service returns a ledger record, even if insufficient funds cause that record's own status to be `REJECTED`; check the transaction status for that distinction.

### H2 console

Visit [http://localhost:8080/h2-console](http://localhost:8080/h2-console) while the app is running and connect with:

- JDBC URL: `jdbc:h2:mem:upimesh`
- User name: `sa`
- Password: leave blank

The console is enabled for local demonstration only.

## Tests

Run the test suite from the project directory:

**Windows PowerShell**

```powershell
.\mvnw.cmd test
```

**macOS or Linux**

```bash
./mvnw test
```

The existing `IdempotencyConcurrencyTest` covers encryption/decryption round-trip, rejection of modified ciphertext, and concurrent delivery of one packet by three simulated bridges. No test result is implied by these instructions.

## Known limitations

- The mesh is a deterministic software model: each virtual device is treated as being in range of every other device during a gossip round. It does not use Bluetooth, BLE, Wi-Fi Direct, or real phones.
- The default simulated network has one internet-connected bridge. Concurrent delivery is exercised by a test, not by multiple internet bridges in the default dashboard scenario.
- Offline senders cannot prove that funds are available. Competing offline payments can therefore reach settlement in an order where later payments are rejected for insufficient funds.
- The demo uses seeded accounts rather than bank accounts. It does not connect to UPI, NPCI, a bank, or payment rails, and it does not verify a real UPI PIN.
- There is no bridge authentication, user authentication, authorization, or rate limiting. The H2 web console is enabled without a password.
- Idempotency state is local to one JVM and is not shared across instances. The in-memory database and generated server key are lost or replaced on restart.
- The freshness timestamp is part of the encrypted instruction, but there is no real device signature or trusted hardware-backed offline balance mechanism.
- The implementation is intended for demonstration and learning, not handling real money or sensitive payment data.

## Potential improvements

These are possible directions for a separately scoped future implementation; none are included in this repository's current behavior:

- Add a real mobile client and evaluate platform-specific offline connectivity and background execution constraints.
- Replace in-memory persistence and deduplication with durable, shared services and define recovery behavior.
- Design bridge authentication, user authorization, key lifecycle, and operational monitoring before any deployment.
- Integrate with a regulated payment provider and a trusted offline-balance model rather than treating a deferred instruction as a completed payment.
- Add API-level integration tests for the dashboard workflow, bridge responses, freshness boundaries, and persistence behavior.
