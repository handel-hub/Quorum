# Quorum: A Distributed Key-Value Database in Node.js

Project specification · Working name: Quorum (rename freely) · Target: 8 weeks, slack up to 10 · Node.js ES modules (`.mjs`), JSDoc types checked by `tsc`

## 1. Purpose and vision

Quorum is a small distributed database built from scratch: a Raft-replicated, strongly consistent key-value store on top of its own LSM-tree storage engine, with a custom binary protocol, a small query language, live change streams, and a deterministic failure-testing harness. It is built with almost no dependencies so that every layer between a client's `PUT` and a byte on disk is code you wrote and can explain.

The project has three purposes at once:

1. **Learn** the hard parts of distributed storage by building them from primary sources (the Raft paper, the LSM-tree and LevelDB designs, linearizability theory).
2. **Showcase**: produce an artifact whose correctness is demonstrated by evidence (simulation, fault injection, a linearizability checker), not merely claimed.
3. **Use**: end with something small and reliable enough to run for real, such as a config store, a lock or leader-election service, or a metadata store for your other systems.

## 2. Goals and success criteria

| ID | Goal | Evidence it is met |
| --- | --- | --- |
| G1 | **Correct under failure.** Node crashes, restarts, partitions, and message loss, delay, duplication, and reordering never violate linearizability or lose an acknowledged write. | Simulator runs thousands of seeded schedules with zero invariant violations; the linearizability checker accepts every recorded history. |
| G2 | **Durable.** An acknowledged write has been fsynced on a majority of nodes. | Crash-recovery tests that kill the process at every step of WAL append, memtable flush, compaction, and manifest update. |
| G3 | **Understandable.** Narrow interfaces between layers, documented invariants. | Each Raft and storage invariant maps to at least one named test. Architecture doc exists. |
| G4 | **Observable.** An operator can see leader, term, log indexes, replication lag, and compaction state. | Metrics endpoint and live dashboard. |
| G5 | **Measured performance.** Honest benchmarks with stated methodology. | Reproducible `bench/` scripts and a results write-up. |
| G6 | **Primary-source implementation.** Each major algorithm is built from the original description. | `docs/paper-notes.md` mapping paper sections to code. |

**Definition of done for the whole project:** a 3-node cluster that passes the full chaos suite, has a working REPL, client library, watch streams, and dashboard, ships with the architecture write-up and benchmark report, and can be started with one command.

## 3. Non-goals (explicitly not required)

These are deliberately out of scope. Doing them would dilute the project or hide the interesting problems behind plumbing.

- **SQL, joins, secondary indexes, schemas.** The data model is ordered bytes to bytes. (Secondary indexes appear only as a stretch goal.)
- **Multi-datacenter or geo-replication.** One failure domain, low-latency network assumed.
- **Byzantine fault tolerance.** Nodes crash or are slow; they do not lie.
- **Authentication, authorization, multi-tenancy, encryption at rest.** Optional shared-secret handshake and TLS are stretch items only.
- **Automatic sharding and rebalancing** in the core scope (a stretch goal covers range sharding).
- **Production-grade hardening**: hot upgrades, backup tooling, rolling schema migration, operator runbooks.
- **Native addons or WASM** for speed. Pure JavaScript on Node's standard library.
- **Competing with RocksDB or etcd on throughput.** Performance must be reasonable and measured, not record-setting.
- **Browser clients.** The client is a Node library and a CLI.
- **Windows support.** Linux and macOS only; fsync and directory-sync semantics differ enough to be a project of their own.

## 4. Users and use cases

| User | What they do |
| --- | --- |
| **You (builder)** | Learn, debug, extend; run chaos experiments. |
| **Application developer** | Use the client library for config, feature flags, small metadata, distributed locks. |
| **Operator** | Start a cluster, watch it on the dashboard, kill nodes, observe recovery. |
| **Reviewer or interviewer** | Read the write-up, watch the kill-the-leader demo, inspect the simulator and checker. |

**Reference workloads** used to shape requirements and benchmarks:

1. Config store: few keys, read-heavy, needs watches for live reload.
2. Lock and lease service: CAS-heavy, small values, needs linearizability above all.
3. Metadata catalog: millions of small keys, prefix scans, moderate writes.
4. Write-heavy ingest: sequential keys, large batches, stresses compaction.

## 5. System overview

```
  Client / REPL / Dashboard
          |  (client protocol: framed binary over TCP)
  +-------v--------------------------------------------+
  |  Node process                                      |
  |  Query layer   lexer > parser > planner > ops      |
  |  Client server  sessions, routing, watch hub       |
  |  Raft module    election, replication, snapshots   |
  |  KV state machine  apply, MVCC revisions, dedupe   |
  |  LSM storage engine  WAL, memtable, SSTables       |
  |  Platform interfaces: Clock, Transport, Disk       |
  +-------^--------------------------------------------+
          |  (peer protocol: Raft RPCs over TCP)
     Other nodes
```

**Request path for a write:** client frame arrives, the query layer compiles it to an operation, the leader appends it to the Raft log (fsync), replicates it to followers, commits on majority, applies it to the state machine (LSM), and responds. Followers redirect clients to the leader with a hint.

**Key architectural decision:** the Raft log is the source of truth. The state machine records its last-applied index atomically with each memtable flush, so recovery replays Raft log entries after that index rather than relying on a second independent WAL for applied data. This avoids double-writing every mutation, but it couples the two components, so it must be documented and tested explicitly. A simpler fallback (keep both WALs) is acceptable for the first working version.

## 6. Functional requirements

Priority keywords: **MUST** (required for done), **SHOULD** (expected unless there is a reason), **MAY** (optional, listed so you know it was considered).

### 6.1 LSM storage engine

| ID | Requirement |
| --- | --- |
| SE-1 | MUST store an ordered mapping from byte keys to byte values, ordered lexicographically by unsigned bytes. |
| SE-2 | MUST append every mutation to a write-ahead log before applying it to the memtable. Records carry length, type, sequence number, payload, and a CRC32 checksum. |
| SE-3 | MUST support configurable fsync policy: `always`, `group` (batched group commit), and `none` (tests only). Default is `group`. |
| SE-4 | MUST handle torn writes: on recovery, stop at the first invalid or truncated WAL record and truncate the file there. |
| SE-5 | MUST use an in-memory sorted memtable (skip list or balanced tree), frozen at a size threshold (default 4 MiB) and flushed to an SSTable in the background while a new memtable accepts writes. |
| SE-6 | MUST write immutable SSTables: data blocks (about 4 KiB), an index block, a bloom filter block, and a footer with magic number and offsets. Each block carries its own checksum. |
| SE-7 | MUST keep a manifest recording which SSTables belong to which level. Updates are atomic via write-temp, fsync, rename, then fsync of the directory. |
| SE-8 | MUST implement the read path: memtable, immutable memtables, L0 files newest first, then one candidate file per deeper level via binary search on key ranges. Bloom filters skip files that cannot contain the key. |
| SE-9 | MUST implement leveled compaction with configurable level size ratio, running without blocking the event loop (chunked async work or `worker_threads`). |
| SE-10 | MUST write tombstones for deletes and drop them only when compacting into the bottom level or when no older version of the key can exist beneath. |
| SE-11 | MUST use internal keys of the form user key plus sequence number plus type, giving MVCC. Snapshots are sequence numbers; old versions below the oldest live snapshot are garbage collected. |
| SE-12 | MUST provide iterators: a merging iterator across all sources, honoring snapshots, supporting forward range scans. |
| SE-13 | MUST recover on startup from manifest then WAL, and delete orphaned files left by interrupted flushes or compactions. |
| SE-14 | MUST apply write stalls or slowdowns when L0 file count or memtable backlog exceeds thresholds (backpressure rather than unbounded growth). |
| SE-15 | MUST export a consistent point-in-time snapshot as a stream (for Raft InstallSnapshot) and import one atomically. |
| SE-16 | SHOULD include an LRU block cache with a configurable byte budget. |
| SE-17 | SHOULD support reverse range scans. |
| SE-18 | MAY support block compression through `node:zlib`, prefix-compressed keys in blocks, and size-tiered compaction as an alternative strategy. |

### 6.2 Raft consensus

| ID | Requirement |
| --- | --- |
| RF-1 | MUST implement leader election with randomized election timeouts and terms. `currentTerm` and `votedFor` are persisted and fsynced before any response is sent. |
| RF-2 | MUST implement AppendEntries with the log-consistency check and the fast backtracking optimization (conflict term and index hints). |
| RF-3 | MUST commit only by counting replicas for entries of the current term; a new leader appends a no-op entry to commit prior entries safely. |
| RF-4 | MUST persist the log in append-only segment files with checksums, and fsync before acknowledging AppendEntries or counting the leader's own copy toward commit. |
| RF-5 | MUST implement log compaction via snapshots and chunked, resumable InstallSnapshot for slow or new followers. |
| RF-6 | MUST implement linearizable reads via ReadIndex: leader confirms leadership with a heartbeat round, then serves the read once the state machine has applied up to the recorded commit index. |
| RF-7 | MUST implement client sessions (client ID plus per-client sequence number) with a dedupe table in the state machine so retried writes execute at most once. Sessions expire after a configured idle time. |
| RF-8 | MUST run a stable static cluster configuration (3 or 5 nodes) supplied at startup. |
| RF-9 | SHOULD implement PreVote and CheckQuorum so partitioned nodes do not disrupt a healthy cluster and stale leaders step down. |
| RF-10 | SHOULD batch and pipeline AppendEntries and coalesce concurrent client proposals into one log append. |
| RF-11 | MAY implement lease-based reads (documenting the clock-drift assumption), bounded-staleness follower reads, and leadership transfer. |
| RF-12 | MAY implement membership changes: single-server changes first, joint consensus as the advanced option. |

### 6.3 Data model and state machine

| ID | Requirement |
| --- | --- |
| DM-1 | MUST enforce limits: key up to 4 KiB, value up to 1 MiB (configurable), with clear errors beyond them. |
| DM-2 | MUST support operations `PUT`, `GET`, `DELETE`, `SCAN` (range or prefix, with limit), `CAS` (compare on revision or value), and `TXN` (guarded multi-operation atomic batch committed in a single Raft entry). |
| DM-3 | MUST maintain a global monotonic revision (the Raft log index of the last mutating entry) and, per key, `createRevision`, `modRevision`, and `version`. |
| DM-4 | MUST make the state machine deterministic: no wall-clock reads, no randomness, identical results on every replica. |
| DM-5 | MUST support reads at a specified past revision within the retention window (built on MVCC). |
| DM-6 | SHOULD provide a manual and automatic history compaction up to a revision. |
| DM-7 | MAY support TTL keys and leases (leader-driven expiry replicated through the log). |

### 6.4 Wire protocol and networking

| ID | Requirement |
| --- | --- |
| NP-1 | MUST use length-prefixed binary frames over `node:net` with a header: magic, protocol version, message type, flags, request ID, payload length. |
| NP-2 | MUST use a custom binary codec (varints, length-prefixed bytes), not JSON, on the data path. |
| NP-3 | MUST multiplex many in-flight requests per connection, matched by request ID, with correct handling of partial frames and frames split across TCP chunks. |
| NP-4 | MUST support streaming responses for scans and watches. |
| NP-5 | MUST respect backpressure: honor `write()` return values and `drain`, bound per-connection buffers, and disconnect or cancel slow consumers rather than grow memory. |
| NP-6 | MUST enforce a maximum frame size and reject malformed frames without crashing the process. |
| NP-7 | MUST apply timeouts, TCP keepalive, and client reconnect with exponential backoff and jitter. |
| NP-8 | MUST separate the client listener from the peer (Raft) listener and negotiate protocol version in a handshake. |
| NP-9 | MUST return a leader hint from non-leaders so clients can redirect. |
| NP-10 | MAY support optional TLS via `node:tls` and a shared-secret handshake. |

### 6.5 Query language

A small, line-oriented language compiled to storage operations. Illustrative grammar:

```
statement := put | get | delete | scan | cas | txn | watch | admin
put       := 'PUT' key value
get       := 'GET' key ['AT' revision]
delete    := 'DEL' key
scan      := 'SCAN' ('PREFIX' key | 'RANGE' key key) ['LIMIT' n] ['REVERSE']
cas       := 'CAS' key 'IF' ('REV' n | 'VALUE' value | 'ABSENT') 'SET' value
txn       := 'TXN' ['IF' guard {'AND' guard}] stmt {';' stmt} 'END'
watch     := 'WATCH' ('PREFIX' key | key) ['FROM' revision]
admin     := 'STATUS' | 'SHOW' 'CLUSTER' | 'COMPACT' revision
literal   := quoted-string | 0xHEX | number
```

| ID | Requirement |
| --- | --- |
| QL-1 | MUST implement a hand-written lexer and parser (recursive descent with a Pratt parser for guard expressions) producing a typed AST. |
| QL-2 | MUST report errors with line and column and a caret-style excerpt; the parser must recover enough to report more than the first error in multi-statement input. |
| QL-3 | MUST include a validator (limits, types, mutually exclusive options) and a planner that selects point lookup, range scan, or transaction; computes prefix bounds (the prefix successor); and pushes limits down into scans. |
| QL-4 | MUST ship a REPL with history, multi-line input, and table and JSON output modes. |
| QL-5 | MUST fuzz-test the parser (random and mutated input never crash or hang). |
| QL-6 | MAY provide tab completion, a script runner, and an `EXPLAIN` statement that prints the plan. |

### 6.6 Watch streams

| ID | Requirement |
| --- | --- |
| WT-1 | MUST deliver events (put, delete) for a key or prefix in strictly increasing revision order, with no gaps and no duplicates after resume. |
| WT-2 | MUST allow resuming from a revision after disconnect. If that revision is older than the compaction horizon, return a distinct error stating the oldest available revision. |
| WT-3 | MUST support multiple concurrent watchers per connection with independent cancel. |
| WT-4 | MUST bound per-watcher buffers and cancel slow watchers with an explicit error. |
| WT-5 | MUST send periodic progress notifications so clients can distinguish an idle stream from a dead one. |
| WT-6 | SHOULD allow watches on followers by having the follower stream committed, applied events with a stated freshness guarantee. |
| WT-7 | MAY include the previous value in each event. |

### 6.7 Client library and CLI

- MUST provide a promise-based Node client with connection pooling, automatic leader discovery and redirect, retry with sessions for idempotence, request deadlines, and async-iterator APIs for scans and watches.
- MUST provide a CLI (`quorumctl`) for one-off commands, the REPL, and cluster status.
- SHOULD expose typed JSDoc definitions so editors and `tsc --checkJs` consumers get full types.

### 6.8 Observability and dashboard

- MUST emit structured JSON-line logs including node ID, term, and log index on every Raft state change.
- MUST expose a status endpoint with: role, term, leader, commit and applied indexes, per-follower match index and lag, LSM level sizes, compaction backlog, WAL size, watcher count.
- MUST include counters and latency histograms for client requests, Raft RPCs, fsync, and compaction.
- MUST provide a dashboard (single static page served by `node:http`, live updates through Server-Sent Events) showing the cluster as a live graph: leader, terms, log indexes, replication lag, and recent events. It must visibly show re-election after the leader is killed.
- MAY provide request tracing across nodes.

### 6.9 Verification and chaos tooling

| ID | Requirement |
| --- | --- |
| VT-1 | MUST ship a deterministic simulator: a virtual clock, a simulated network (delay, drop, duplicate, reorder, partition, heal), a simulated disk (crash losing unsynced writes, torn writes, fsync failures), and a seeded PRNG. Any failing run is reproduced exactly from its seed. |
| VT-2 | MUST check Raft safety invariants continuously in simulation: election safety, leader append-only, log matching, leader completeness, state machine safety. |
| VT-3 | MUST implement a history recorder and a linearizability checker (WGL-style search with memoization) using a per-key register and CAS model. |
| VT-4 | MUST include model-based property tests of the storage engine against an in-memory reference (`Map` plus sorted keys) using `fast-check`, including injected crashes. |
| VT-5 | MUST include a real-process chaos harness: spawn nodes as child processes, `SIGKILL`, `SIGSTOP` and `SIGCONT` them, and partition them through a TCP proxy, while a workload records history for the checker. |
| VT-6 | SHOULD shrink failing simulator schedules to a minimal reproduction. |
| VT-7 | SHOULD deliberately inject known Raft bugs (mutation testing, such as committing old-term entries) to prove the harness detects them. |

## 7. Non-functional requirements

### 7.1 Safety and consistency

- **Consistency model:** linearizable reads and writes for all operations served through the leader; documented, bounded-staleness behavior for any follower reads.
- **Durability:** an acknowledged write is fsynced on a majority. With `2f+1` nodes, the cluster tolerates `f` simultaneous crash failures without data loss.
- **Availability:** the cluster serves requests while a majority is alive and connected; it refuses (not guesses) otherwise.
- **Integrity:** every on-disk structure is checksummed; corruption is detected and the node fail-stops rather than serve bad data.

### 7.2 Performance (starting targets, to be calibrated)

These are initial targets for a 3-node cluster on one developer machine with SSDs and 128-byte values. Revise them after the first benchmark; the point is to measure and explain, not to hit arbitrary numbers.

| Metric | Starting target |
| --- | --- |
| Sustained writes, group commit | at least 5,000 per second |
| Write latency p50 / p99 | under 10 ms / under 50 ms |
| Linearizable reads (ReadIndex) | at least 20,000 per second |
| Single-node engine writes (batch sync) | at least 50,000 per second |
| Leader failure to new leader serving | under 2 seconds |
| Node restart recovery with 1 GiB of data | under 30 seconds |
| Memory | bounded by configured memtable and cache budgets, no unbounded queues |

Benchmarks must state hardware, Node version, fsync policy, value size, concurrency, and warm-up, and report percentiles, not averages.

### 7.3 Failure model

**Tolerated:** crash-stop with restart, process pauses, network partitions, message loss, duplication, delay, and reordering, power loss (unsynced data lost, writes torn at block granularity), slow disks.

**Detected and fail-stop:** checksum mismatches (bit rot), manifest inconsistencies, unsupported protocol versions.

**Not tolerated (documented):** Byzantine behavior, permanent loss of a majority, clocks drifting beyond configured bounds (matters only for optional lease reads), disks that lie about fsync.

### 7.4 Code quality and tooling conventions

- `.mjs` everywhere, ES modules only, no CommonJS.
- JSDoc annotations on all exported functions and data structures; `tsconfig.json` with `allowJs`, `checkJs`, `strict`, and `noEmit`; `tsc` passes in CI with zero errors.
- Test runner is `node:test`; the only dev dependency expected is `fast-check`. Zero runtime dependencies.
- Stay on current Node LTS; use `node:fs/promises`, `node:net`, `node:http`, `node:worker_threads`, `node:zlib`, `node:crypto`, and `node:perf_hooks`.
- Every module depends only on injected interfaces (see section 8), never on global time, `net`, or `fs` directly, outside the platform adapter layer.
- Each invariant in this document has a named test; each paper-derived rule has a code comment citing its section.

## 8. Key interfaces and formats

### 8.1 Injectable platform interfaces

The single most important structural decision: all nondeterminism enters through three interfaces so the same Raft and storage code runs in production and in the simulator.

```js
/** @typedef {{ now(): number, setTimer(ms: number, fn: () => void): { cancel(): void } }} Clock */
/** @typedef {{ send(to: string, msg: Uint8Array): void, onMessage(fn: (from: string, msg: Uint8Array) => void): void }} Transport */
/** @typedef {{ open(path: string, mode: string): Promise<FileHandle>, rename(a: string, b: string): Promise<void>, syncDir(path: string): Promise<void>, readdir(path: string): Promise<string[]>, unlink(path: string): Promise<void> }} Disk */
```

### 8.2 On-disk formats (to be finalized in the design doc)

- **WAL and Raft log record:** `[crc32:u32][length:u32][type:u8][payload...]`; segments rotate at a size limit; recovery stops at the first bad record.
- **SSTable:** data blocks, index block (last key per block to offset), bloom block, footer with magic, version, and block handles; every block followed by a checksum.
- **Manifest:** append-only log of version edits (add file, remove file, set level, set last sequence, set last applied Raft index), with periodic rewrite and an atomic pointer file (`CURRENT`).
- **Snapshot stream:** sorted key, value, and revision triples with a header (last included index and term) and trailing checksum.

### 8.3 Safety-critical orderings

These rules must be written down, enforced by code structure, and tested:

1. Persist `currentTerm` and `votedFor` before replying to any vote request or term-changing RPC.
2. Fsync log entries before acknowledging AppendEntries.
3. The leader counts its own entry toward commit only after its own fsync.
4. Reply to a client only after the entry is committed and applied.
5. Write and fsync a new SSTable and then update the manifest; never delete inputs until the manifest edit is durable.
6. Install a received snapshot atomically; a partial snapshot is never visible.

## 9. Testing strategy

| Layer | Techniques |
| --- | --- |
| Storage engine | Unit tests per component; model-based property tests; crash-at-every-step tests; corruption injection (flip bits, truncate files). |
| Raft | Scripted scenario tests (the figures in the Raft paper as tests); simulator with invariant checking across many seeds; deliberately buggy variants that must be caught. |
| Protocol | Frame fuzzing; split-and-coalesced chunk tests; slow-consumer and backpressure tests. |
| Query language | Golden tests for parse and error output; fuzzing; planner property tests (prefix bounds equal brute-force scans). |
| Watches | Ordering and no-gap property tests under reconnect, leader change, and compaction. |
| System | Real-process chaos runs with the linearizability checker; soak tests of several hours with periodic faults. |

**CI gate:** type check, unit tests, a fixed set of simulator seeds, and a short chaos run. A longer simulator sweep runs on demand and nightly, and any failing seed is added to a regression list permanently.

## 10. Milestones and acceptance criteria

| Weeks | Milestone | Acceptance criteria |
| --- | --- | --- |
| 1-2 | **Storage engine** | Passes model-based tests with injected crashes; flush, leveled compaction, tombstones, bloom filters, and recovery work; benchmark of single-node engine recorded. |
| 3-4 | **Raft core** | 3-node cluster elects a leader, replicates, survives leader kill; snapshots and InstallSnapshot work; simulator exists with network faults; Raft invariants checked. |
| 5 | **Protocol and query layer** | Client talks to the cluster over the binary protocol; REPL runs all statements; parser fuzzing in CI; client sessions give at-most-once writes. |
| 6 | **Read path and watches** | ReadIndex reads verified linearizable; watches resume from revision across failover; slow-watcher cancellation works. |
| 7 | **Verification** | Disk fault simulation; linearizability checker over recorded histories; real-process chaos harness; at least one injected bug caught. |
| 8 | **Tooling and write-up** | Dashboard live; benchmarks published; architecture document and paper-notes complete; demo script (kill the leader live). |
| 9-10 | **Slack** | Overrun absorption, hardening, chosen stretch goals. |

**Scope-cut order if time runs short** (cut from the top): dashboard polish, query-language niceties, follower watches, reverse scans, block cache, PreVote. Never cut: durability tests, the simulator, the linearizability checker.

## 11. Optional and stretch features

Listed so you can pick deliberately, not drift into them.

| Feature | What it adds | Rough cost |
| --- | --- | --- |
| Single-server membership change | Live add and remove of nodes | 3-4 days |
| Joint consensus | The full Raft paper algorithm | 1 week |
| Lease reads | Fast reads, clock-drift analysis | 2-3 days |
| TTL keys and leases | Expiring keys, lock-service ergonomics | 3-4 days |
| MVCC transactions with snapshot isolation | Read-write transactions across keys | 1-2 weeks |
| Secondary indexes | Index maintenance inside transactions | 1 week |
| Range sharding across Raft groups | Horizontal scale, router, split | 2+ weeks |
| Block compression and prefix-compressed blocks | Smaller disk footprint | 2-3 days |
| Optional TLS and shared-secret auth | Basic transport security | 2 days |
| Distributed tracing | Cross-node request visibility | 2-3 days |
| Deterministic schedule shrinking | Minimal failing reproductions | 3-5 days |
| Pluggable storage engine interface | Swap in an in-memory or B-tree engine | 3 days |

## 12. Risks and mitigations

| Risk | Why it matters | Mitigation |
| --- | --- | --- |
| Raft subtle bugs | Distributed bugs surface only under rare interleavings. | Build the simulator before the Raft code is finished; invariant checks run on every step. |
| Fsync semantics misunderstood | Silent durability loss that tests rarely reveal. | Write the ordering rules (section 8.3) first; simulate lost unsynced writes; test directory sync on rename. |
| Event-loop starvation | Compaction or big scans stall heartbeats and trigger spurious elections. | Chunk work, use `worker_threads`, measure event-loop delay via `perf_hooks`. |
| Scope creep | Eight weeks is tight for this much surface. | Follow the cut order; keep stretch goals out until acceptance criteria are met. |
| Linearizability checker blow-up | Search is exponential on long histories. | Per-key partitioning, memoization, bounded history lengths, short high-contention workloads. |
| Performance disappointing | Pure JS and fsync costs. | Treat targets as measurements; profile with `--cpu-prof`; group commit and batching first. |
| Over-coupling log and state machine | Applied-index atomicity is easy to get wrong. | Start with the double-WAL fallback; migrate once tests exist. |

## 13. Proposed repository layout

```
quorum/
  package.json  tsconfig.json  README.md
  src/
    platform/   clock.mjs  transport-tcp.mjs  disk-node.mjs
    storage/    wal.mjs  memtable.mjs  sstable.mjs  bloom.mjs
                manifest.mjs  compaction.mjs  iterator.mjs  engine.mjs
    raft/       node.mjs  log.mjs  snapshot.mjs  readindex.mjs  sessions.mjs
    kv/         state-machine.mjs  mvcc.mjs  watch-hub.mjs
    net/        frame.mjs  codec.mjs  server.mjs  peer.mjs
    query/      lexer.mjs  parser.mjs  ast.mjs  planner.mjs
    client/     client.mjs  pool.mjs
    cli/        quorumctl.mjs  repl.mjs
    dashboard/  server.mjs  index.html
    sim/        sim-clock.mjs  sim-net.mjs  sim-disk.mjs  invariants.mjs
    verify/     history.mjs  linearizability.mjs  chaos-runner.mjs
  test/  bench/  docs/
    architecture.md  paper-notes.md  on-disk-formats.md  invariants.md
```

## 14. Deliverables checklist

- Source tree meeting the conventions in section 7.4, type-checking clean.
- Passing CI: unit, property, fuzz, simulator seeds, short chaos run.
- Architecture document, on-disk format document, invariants catalog, paper notes.
- Benchmark report with methodology and graphs.
- Dashboard and a scripted live demo.
- A write-up of bugs the harness found in your own code, since this is the strongest portfolio evidence.

## 15. Decisions to make in the first week

1. Memtable structure: skip list or balanced tree?
2. Single Raft log as source of truth, or double-WAL fallback for version one?
3. Revision numbering: Raft log index, or a separate counter?
4. Cluster size for demos and CI: 3 only, or 3 and 5?
5. Group commit window: fixed delay, or adaptive based on pending writes?
6. Binary codec style: hand-rolled tagged fields, or a minimal schema compiler?
7. Single process per node with workers, or separate processes for compaction?

## 16. Glossary

- **Linearizability:** every operation appears to take effect atomically at one point between its call and return, consistent with real-time order.
- **LSM tree:** log-structured merge tree; writes buffered in memory and flushed as sorted immutable files that are merged in the background.
- **MVCC:** multi-version concurrency control; keeping multiple versions of each key identified by sequence numbers.
- **ReadIndex:** Raft read technique where the leader confirms it is still leader before serving a read at its commit index.
- **Group commit:** batching multiple writes behind one fsync.
- **Tombstone:** a delete marker that shadows older versions until compaction removes them.
- **Torn write:** a write partially persisted at power loss.

## 17. Recommended references

- Ongaro and Ousterhout, *In Search of an Understandable Consensus Algorithm* (the Raft paper), and Ongaro's PhD thesis for membership changes, snapshots, and client interaction.
- O'Neil et al., *The Log-Structured Merge-Tree*, and the LevelDB and RocksDB design documents.
- Herlihy and Wing, *Linearizability: A Correctness Condition for Concurrent Objects*; Wing and Gong, and Lowe, on checking algorithms; the Porcupine and Jepsen write-ups.
- Kleppmann, *Designing Data-Intensive Applications*; Petrov, *Database Internals*.
- The etcd and TiKV Raft implementations and the FoundationDB simulation-testing talks, for design ideas on deterministic testing.
