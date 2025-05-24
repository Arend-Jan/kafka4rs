# Kafka4rs

---

## 1  Vision & Scope

* **Goal**  Deliver a *pure-Rust* Apache Kafka 4.0 client—Producer, Consumer (new rebalance protocol), and later Admin/Transactions—that can replace `librdkafka` while matching or exceeding its performance, reliability, and security.
* **Non-Goals (for v1.0)**  Mirror every legacy broker quirk (Produce v0-v2, Fetch v0-v3), embedded ZooKeeper interactions, or exotic SASL mechanisms beyond the roadmap.

---

## 2  Guiding Principles

| Principle                   | Rationale                                                      | Concrete Technique                                           |
| --------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------ |
| **Layered separation**      | Isolate change; keep public API stable while protocol evolves. | Cargo workspace → one crate per concern.                     |
| **Async-first**             | One runtime thread can multiplex hundreds of broker sockets.   | Tokio, `select!`, bounded MPSC.                              |
| **Zero-copy paths**         | Hit ≥ librdkafka throughput with lower CPU.                    | `bytes::Bytes( Mut)` pools, gather-write (`write_vectored`). |
| **Fail-fast, typed errors** | Let callers decide retry vs. abort.                            | `thiserror` + `#[non_exhaustive] enum KafkaError`.           |
| **Opt-in heavyweight deps** | Edge/WASM users shouldn’t drag in Zstd or TLS.                 | Cargo features: `tls`, `zstd`, `sasl-scram`, …               |
| **Observability built-in**  | Production diagnostics without code patch.                     | `metrics` facade, structured logs with correlation-id.       |

---

## 3  High-Level Component Model

```
 ┌─────────────────┐ ┌────────────────────┐
 │  kafka-producer │ │   kafka-consumer   │  ← public async APIs
 └───────┬─────────┘ └──────────┬─────────┘
         │                      │
 ┌───────▼──────────────────────▼───────────────┐
 │             kafka-core (Client)              │
 │  • Connection Manager  • Metadata Cache      │
 │  • Group Coordinator   • Retry & Back-off    │
 └───────┬──────────────────────┬───────────────┘
         │                      │
   ┌─────▼──────┐        ┌──────▼─────┐
   │ BrokerTask │ …  N   │ BrokerTask │        (one per broker)
   └─────┬──────┘        └──────┬─────┘
         │ framed bytes         │
         ▼                      ▼
   ┌──────────────┐      ┌──────────────┐
   │   Transport  │ … N  │   Transport  │      (TCP or TLS)
   └──────────────┘      └──────────────┘
```

* **kafka-protocol** – auto-generated structs/enums for every Kafka request/response, plus `Encode`/`Decode` traits.
* **kafka-io** – `Transport<T>` (generic over `TcpStream` or `TlsStream`) implements framed, length-delimited I/O.
* **BrokerTask** – single async task per broker; owns socket, correlation-id map, and outbound request queue.
* **Client** – routes requests, holds shared state, enforces back-pressure, and spawns BrokerTasks on demand.
* **Producer / Consumer** – façade crates that translate high-level semantics (batching, partitioning, groups) into low-level requests.

---

## 4  Concurrency & Back-Pressure

* **Single writer rule** per broker achieved via a dedicated task; avoids cross-thread locks on every write.
* **Bounded MPSC channels** between APIs and core implement natural back-pressure. When the internal queue is full, `send()` or `poll()` simply await capacity.
* **Metadata fan-out** – updates published through a `watch::Sender`; subscribers (producer, consumer) pick up changes without polling.

---

## 5  Error Taxonomy

```rust
enum KafkaError {
    Network(io::Error),          // retriable
    Server { code: i16, retriable: bool },
    Timeout,
    Auth(AuthError),             // fatal
    Protocol(&'static str),      // fatal, indicates bug or incompatible broker
}
```

* Retriable errors are **handled internally** (with back-off); fatal ones bubble to the caller immediately.
* Mapping strictly follows the official error-code table, keeping semantics consistent with Java & C clients.

---

## 6  Extensibility & Feature Flags

```toml
[features]
default       = ["producer", "consumer"]
producer      = []
consumer      = []
admin         = ["kafka-admin"]
tls           = ["rustls", "webpki-roots"]
sasl-plain    = []
sasl-scram    = ["sha2"]
zstd          = ["zstd-safe"]
snappy        = ["snap"]
```

* **Compile-time pay-only-for-what-you-use** approach keeps binaries lean for IoT / FaaS deployments.
* Feature gates also isolate licence boundaries (e.g., OpenSSL vs. Rustls).

---

## 7  Security Posture

* Memory-safe language ⇒ eliminates classic overflow/UAF vulnerabilities in networking path.
* TLS default when `ssl://` bootstrap URL provided; Rustls chosen to avoid C dependencies.
* **Red-action** of secrets in logs; passwords & tokens zeroed after handshake.
* Planned support: SASL PLAIN → SCRAM → OAUTHBEARER; Kerberos optional.

---

## 8  Testing Strategy

| Layer                    | Key Tests                                             | Tooling             |
| ------------------------ | ----------------------------------------------------- | ------------------- |
| Protocol encode/decode   | Golden fixtures vs. Java client hex dumps             | `insta`, `proptest` |
| Connection & retry       | Dockerised 4.0 cluster, Toxiproxy network cuts        | `testcontainers`    |
| Consumer group rebalance | Multi-consumer integration harness                    | `cargo nextest` CI  |
| Idempotence & EOS        | Produce under broker restart; assert exactly-one copy | custom soak         |
| Security                 | TLS handshake fuzzing; invalid cert paths             | `cargo fuzz`        |

---

## 9  Roadmap Alignment

* The crates map 1-to-1 with the implementation milestones:

  * **M-1** ➜ `kafka-protocol`, `kafka-io`, `kafka-core` skeleton, `Producer` (no batching).
  * **M-2** ➜ `Consumer` + Group Coordinator.
  * **M-3** ➜ Batching, Compression, Idempotence.
  * **M-4** ➜ TLS & SASL feature flags.
  * **M-5/6** ➜ Transactions & Admin.

Continuous delivery: each crate tagged on `main` after passing full CI, enabling early adopters to depend on the subset they need.

---

## 10  Glossary

| Term              | Meaning in this project                                      |
| ----------------- | ------------------------------------------------------------ |
| **BrokerTask**    | Tokio task managing one broker connection.                   |
| **Transport**     | Generic framed I/O over TCP or TLS.                          |
| **MetadataCache** | Thread-safe snapshot of brokers, topics, partitions.         |
| **Batcher**       | Producer component that aggregates records per partition.    |
| **RebEvent**      | Broadcast message signalling assign/revoke during rebalance. |

---

### Contributors, please…

1. Read this brief *before* diving into code.
2. Keep public APIs minimal, ergonomic, and **immutable** once released.
3. Favour clarity over micro-optimisation until profiling proves the need.

*Questions or proposed deviations?*  Open a design-discussion issue tagged **`architecture`** so we keep the blueprint and implementation in sync.
