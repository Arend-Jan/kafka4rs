# kafka4rs

> **Pure‑Rust client for Apache Kafka 4.0+** – drop‑in replacement for `librdkafka`, powered by Tokio and zero‑copy buffers.

---

| CI                                             | Crate | Docs |
| ---------------------------------------------- | ----- | ---- |
| [![CI](https://github.com/Arend-Jan/kafka4rs/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/Arend-Jan/kafka4rs/actions/workflows/ci.yml) | [![crates.io](https://img.shields.io/crates/v/kafka4rs.svg)](https://crates.io/crates/kafka4rs) | [![docs.rs](https://docs.rs/kafka4rs/badge.svg)](https://docs.rs/kafka4rs) |

---

## ✨ Key Features

* **Kafka 4.0 protocol** – supports the new KRaft‑only clusters, consumer rebalance v2 (KIP‑848), idempotent producer by default and duration‑based offset resets (KIP‑1106).
* **Async‑first** – built on Tokio; one runtime thread can drive hundreds of broker sockets.
* **Zero‑copy performance** – `bytes::Bytes` pools, gather‑write and adaptive batching; targets ≥ librdkafka throughput.
* **Secure by default** – opt‑in TLS (Rustls) and SASL mechanisms (PLAIN, SCRAM, OAUTHBEARER).
* **Feature‑gated crates** – compile only what you need (e.g. producer‑only binary without TLS or Zstd keeps size tiny).
* **Native Rust safety** – no C FFI or `unsafe` in the hot path; memory‑safe networking from day one.

---

## 🚀 Quick Start

Add the dependency (until crates.io release use the Git branch):

```toml
[dependencies]
kafka4rs = { git = "https://github.com/Arend-Jan/kafka4rs", features = ["producer"] }
```

Minimal Producer example:

```rust
use kafka4rs::{ClientConfig, Producer};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // 1. Build a shared client configuration
    let client_cfg = ClientConfig::new()
        .bootstrap_servers("localhost:9092")
        .build()?;

    // 2. Create a producer – idempotence enabled by default
    let producer = Producer::new(client_cfg)?;

    // 3. Fire‑and‑forget (future resolves on ACK)
    producer.send("demo-topic", "hello‑world").await?;
    producer.flush().await?;
    Ok(())
}
```

Minimal Consumer example (new rebalance protocol):

```rust
use kafka4rs::{ClientConfig, Consumer, OffsetReset};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let cfg = ClientConfig::new()
        .bootstrap_servers("localhost:9092")
        .group_id("demo-group")
        .auto_offset_reset(OffsetReset::Earliest)
        .build()?;

    let mut consumer = Consumer::subscribe(cfg, ["demo-topic"])?;

    loop {
        for record in consumer.poll(std::time::Duration::from_secs(1)).await? {
            println!("{} => {}", record.partition, String::from_utf8_lossy(&record.value));
        }
    }
}
```

---

## 🗂 Workspace Layout

```
.
├─ kafka-protocol/   # auto‑generated wire structs & enc/dec traits
├─ kafka-io/         # framed TCP/TLS transport
├─ kafka-core/       # Client, connection manager, metadata cache
├─ kafka-producer/   # High‑level async Producer API
├─ kafka-consumer/   # High‑level async Consumer (groups)
├─ kafka-admin/      # (later) Admin client
└─ docs/             # project docs → architecture.md, ADRs, diagrams
```

---

## 🛣 Roadmap

| Milestone                                  | Status         | Notes                                                |
| ------------------------------------------ | -------------- | ---------------------------------------------------- |
| Core networking + simple producer/consumer | 🔧 in progress | see [`docs/architecture.md`](./docs/architecture.md) |
| Consumer groups (KIP‑848)                  | ⏳              |                                                      |
| Batching, compression, idempotence         | ⏳              |                                                      |
| TLS & SASL authentication                  | ⏳              |                                                      |
| Transactions / EOS                         | ⏳              |                                                      |
| Admin & metrics                            | ⏳              |                                                      |

Detailed backlog lives in **`docs/architecture.md`** and GitHub Issues.

---

## 🤝 Contributing

1. Fork & clone the repo.
2. `cargo test` – all tests must pass.
3. Follow the coding style (`cargo fmt`, `clippy --all-targets`).
4. Open a PR; the CI pipeline will run integration tests against a Kafka 4.0 docker‑compose cluster.
5. Feedback is welcome – design discussions → GitHub Discussions.

---

## 📜 License

Dual‑licensed under **MIT** or **Apache‑2.0** – choose either at your discretion.

---

> *Happy streaming – the Rust‑way!*
