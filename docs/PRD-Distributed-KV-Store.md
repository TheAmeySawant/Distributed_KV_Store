# PRD: Distributed Key-Value Store
**Phase 1 — Systems & Infrastructure Track**

## 1. Overview
**Problem:** Standard CRUD/database projects don't differentiate candidates in a saturated market. This project proves ground-up understanding of how a database engine is built — storage, persistence, concurrency, replication — not just how to query one.

**Product:** A custom in-memory key-value store in Rust with:
- Append-Only Log (AOL) persistence
- Multi-node replication over TCP
- Concurrent-safe in-memory state
- A simple wire protocol for client commands

## 2. Goals
- Working KV store with GET/SET/DELETE semantics
- Durability via AOL (crash recovery replays the log)
- At least 2-node replication (primary → replica)
- Concurrent client connections without data races
- Portfolio-ready artifact: repo, README, benchmarks, architecture diagram

## 3. Non-Goals (v1)
- Sharding / partitioning
- Consensus protocols (Raft/Paxos) — single-primary replication only
- Range queries, transactions, secondary indexes
- Production-grade auth/TLS

## 4. Functional Requirements

| ID | Requirement |
|----|-------------|
| FR1 | Server accepts concurrent TCP connections from multiple clients |
| FR2 | Support commands: `SET key value`, `GET key`, `DEL key`, `PING` |
| FR3 | Every write (SET/DEL) is appended to an on-disk log before ACK |
| FR4 | On startup, server replays the AOL to rebuild in-memory state |
| FR5 | Primary streams writes to connected replica(s) in real time |
| FR6 | Replica applies writes in received order; read-only until promoted |
| FR7 | Malformed commands return a structured error, never a crash |

## 5. Architecture

```
Client(s) ──TCP──▶ Listener (Tokio) ──▶ Command Parser
                                              │
                                              ▼
                                     Shared State (Arc<Mutex<HashMap>>)
                                              │
                                    ┌─────────┴─────────┐
                                    ▼                   ▼
                              AOL Writer          Replication Stream
                              (fsync per write)   (TCP to replica)
```

- **Listener**: Tokio TCP listener, one task per connection
- **Parser**: byte stream → command enum (framing: newline-delimited or length-prefixed)
- **State**: `Arc<Mutex<HashMap<String, Bytes>>>` shared across async tasks
- **AOL**: sequential append writer, fsync'd per write (or batched — document the trade-off)
- **Replication**: primary forwards each committed write to replica socket(s); replica applies locally

## 6. Core Engineering Mechanics
- **Byte-stream parsing**: raw TCP bytes → structured commands (akin to a compiler's macro-processing pass); handle partial reads and frame boundaries.
- **Concurrency safety**: `Arc<Mutex<...>>` (or `RwLock` for read-heavy loads) to share state across tasks without data races; document lock-contention trade-offs.
- **Persistence ordering**: write must hit the AOL before the client gets an ACK (durability guarantee).
- **Replication semantics**: at-least-once delivery to replica; replica tolerates reconnects (full resync acceptable for v1).

## 7. Tech Stack
- Rust (std)
- Tokio (async runtime)
- `bytes` crate
- TCP sockets (`TcpListener` / `TcpStream`)
- Serde (command / log entry encoding)

## 8. Milestones
1. **M1 — Single-node store**: TCP server, GET/SET/DEL, no persistence
2. **M2 — AOL persistence**: durable writes, crash-safe restart via replay
3. **M3 — Replication**: primary streams to one replica; replica serves reads
4. **M4 — Hardening**: malformed-input handling, benchmarks, README + architecture doc

## 9. Success Metrics
- **Correctness**: passes scripted tests (concurrent clients, crash-recovery replay, replica consistency)
- **Performance**: documented throughput/latency under concurrent load (no fixed target for v1)
- **Portfolio signal**: reviewer understands storage engine, concurrency model, and replication design from the README in under 10 minutes

## 10. Testing & Validation
- **Unit**: parser (valid/malformed commands), AOL read/write, replay correctness
- **Integration**: multi-client concurrent SET/GET, kill-and-restart recovery, primary/replica consistency after N writes
- **Manual**: netcat/telnet smoke test against the running server

## 11. Deliverables
- GitHub repo, clean commit history
- README: setup, protocol spec, design decisions, trade-offs
- Architecture diagram (Section 5)
- Short benchmark write-up (numbers + method)
