# 🗺️ `[Project Plan]: Distributed Key-Value Store`

A ground-up implementation of a distributed key-value store in Go, built incrementally.
Each phase is a working, tested milestone before moving to the next.

---

# 🧭 Guiding Principles

- **Don't design for distribution first.** Get each layer solid before adding complexity.
- **A phase isn't done until it has tests and you understand it cold.**
- **Document decisions as you make them** — why you chose something, what you considered and rejected.

---

# 🔵 `Phase 1/` → Single-Node KV Store over TCP

The foundation. A server that accepts TCP connections, speaks a simple text protocol, and stores data in memory.

### ✅ Milestones
- [ ] TCP server that accepts concurrent connections
- [ ] Simple text protocol: `SET key value`, `GET key`, `DEL key`
- [ ] Thread-safe in-memory store using `sync.RWMutex`
- [ ] Graceful shutdown on signal
- [ ] You can connect with `telnet` or `nc` and use it manually

### 🧰 Key Go concepts
- `net.Listener`, `net.Conn`
- Goroutines per connection
- `sync.RWMutex` for safe concurrent map access
- `bufio.Scanner` for reading lines off a TCP connection

---

# 🔵 `Phase 2/` → Persistence (Write-Ahead Log)

If the server restarts, data should survive. Every write gets appended to a log file before being acknowledged. On startup, replay the log to reconstruct state.

### ✅ Milestones
- [ ] Append-only log file for every write operation
- [ ] Server replays log on startup to restore state
- [ ] Log rotation or compaction strategy (basic — can revisit later)
- [ ] Crash recovery test: kill the server mid-write, verify consistency on restart

### 🧰 Key Go concepts
- File I/O with `os.OpenFile`
- `fsync` and why it matters for durability
- Encoding log entries (start simple: one line per entry, same format as your protocol)

---

# 🔵 `Phase 3/` → Client Library

Write a Go package that wraps the TCP protocol and exposes a clean API: `client.Set(key, value)`, `client.Get(key)`, `client.Del(key)`.

### ✅ Milestones
- [ ] Client package with `Set`, `Get`, `Del`
- [ ] Connection management (single persistent connection to start)
- [ ] Error handling for dropped connections
- [ ] Integration tests written against a real server using the client

### 🧰 Key Go concepts
- Package design and exported API surface
- `net.Dial`, connection reuse
- Error wrapping with `fmt.Errorf` and `errors.Is`

### 📝 Notes
> This phase will expose rough edges in your protocol. Fix them here before moving on — changing the protocol gets expensive later.

---

# 🔵 `Phase 4/` → Replication (Leader-Follower)

Add a second node. The leader streams writes to the follower. The follower applies them in order. Now you have redundancy.

### ✅ Milestones
- [ ] Follower node that connects to the leader on startup
- [ ] Leader streams write operations to followers after applying locally
- [ ] Follower rejects writes directly (reads only)
- [ ] Handle follower falling behind (backpressure or buffering)
- [ ] Handle leader crash: what happens to in-flight writes?

### 🧰 Key Go concepts
- Goroutines for replication stream
- Channels for decoupling write application from replication
- `context` for managing goroutine lifecycle

### ❓ Questions to answer before moving on
- What consistency guarantee does a client get? (Is an ack from the leader enough, or must the follower confirm?)
- What happens to the follower's state if it restarts and has missed writes?

### 📓 Decision log
> *Document the consistency model you chose and why.*

---

# 🔵 `Phase 5/` → Raft Consensus

Replace naive replication with Raft — a well-defined protocol for distributed consensus. Start with `hashicorp/raft` to understand the interface and verify correctness, then optionally implement your own.

### ✅ Milestones
- [ ] Integrate `hashicorp/raft` with your storage backend as the FSM (finite state machine)
- [ ] 3-node cluster with leader election working
- [ ] Writes rejected when cluster has no quorum
- [ ] Leader failover: kill the leader, verify a new one is elected, writes resume
- [ ] Reads from followers with bounded staleness

### 🧩 Optional stretch goal
- [ ] Implement your own Raft from the paper: [In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf)

### 🧰 Key Go concepts
- Finite state machine interface design
- gRPC or raw TCP for inter-node communication
- Testing distributed systems (introduce artificial delays, partition nodes)

### 📚 Resources
- [The Raft paper](https://raft.github.io/raft.pdf) — read this before starting
- [hashicorp/raft](https://github.com/hashicorp/raft)
- [Eli Bendersky's Raft series](https://eli.thegreenplace.net/2020/implementing-raft-part-0-introduction/) — excellent walkthrough for rolling your own

---

# 🔵 `Phase 6/` → Snapshotting & Log Compaction

The write-ahead log grows forever. Periodically snapshot the full state and truncate the log so startup time stays bounded.

### ✅ Milestones
- [ ] Snapshot current state to disk on a schedule or size threshold
- [ ] On startup, load latest snapshot then replay only log entries after it
- [ ] Integrate snapshots with Raft (Raft has built-in snapshot support)

---

## 🚀 Stretch Goals

Things to explore once the core is solid:

- **Partitioning / sharding** — distribute different key ranges across different nodes
- **TTL support** — keys that expire after a duration
- **Pub/Sub** — subscribe to changes on a key
- **HTTP API** — expose the store over REST in addition to TCP
- **Benchmarking** — measure throughput and latency, profile with `pprof`
- **TLS** — encrypt inter-node and client communication

---

## 📖 Reference Reading

- [Designing Data-Intensive Applications](https://dataintensive.net/) — chapters on replication and consensus are directly relevant
- [The Raft paper](https://raft.github.io/raft.pdf)
- [etcd architecture](https://etcd.io/docs/v3.5/learning/architecture/)
- [Build Your Own Redis](https://build-your-own.org/) — good protocol and networking intuition

---