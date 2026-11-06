# Quorum

**A distributed, strongly consistent key-value database, built from scratch in Node.js.**

Quorum is a Raft-replicated key-value store sitting on its own LSM-tree storage engine, with a custom binary protocol, a small query language, live change streams, and a deterministic failure-testing harness. It has zero runtime dependencies: every layer between a client's `PUT` and a byte on disk is in this repository.

> **Status: pre-alpha, under active development.**
> Sections marked _(planned)_ describe target behaviour from the [project specification](docs/spec.md), not what runs today. The [roadmap](#roadmap) tracks what is done.

---

## Why this exists

Quorum has three purposes:

1. **Learn** the hard parts of distributed storage by building them from primary sources (the Raft paper, LSM-tree and LevelDB designs, linearizability theory).
2. **Demonstrate correctness with evidence**, not claims: a deterministic simulator, fault injection, and a linearizability checker run against the system's own histories.
3. **Be useful**: small and reliable enough to run for real as a config store, a lock and leader-election service, or a metadata store.

## Features _(planned)_

| Area               | What you get                                                                                                                                   |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **Consensus**      | Raft with leader election, log replication, snapshots, PreVote and CheckQuorum, linearizable reads via ReadIndex, exactly-once client sessions |
| **Storage**        | LSM-tree engine: WAL, memtable, SSTables with bloom filters, leveled compaction, MVCC revisions, crash recovery                                |
| **Data model**     | Ordered byte keys to byte values; `PUT`, `GET`, `DEL`, `SCAN`, `CAS`, guarded multi-key `TXN`; reads at a past revision                        |
| **Protocol**       | Length-prefixed binary framing over TCP, multiplexed requests, streaming responses, backpressure                                               |
| **Query language** | Hand-written lexer, parser, and planner; REPL with history and multi-line input                                                                |
| **Watches**        | Ordered, gap-free change streams on a key or prefix, resumable from a revision                                                                 |
| **Observability**  | Structured logs, metrics, a status endpoint, and a live cluster dashboard                                                                      |
| **Verification**   | Seeded deterministic simulator, Raft invariant checks, linearizability checker, real-process chaos harness                                     |

## Guarantees _(planned)_

- **Linearizable** reads and writes for everything served through the leader.
- **Durable**: an acknowledged write has been fsynced on a majority of nodes. A cluster of `2f+1` nodes tolerates `f` simultaneous crashes without data loss.
- **Available** while a majority is alive and connected. Otherwise Quorum refuses requests rather than guessing.
- **Integrity**: every on-disk structure is checksummed. On corruption a node fail-stops instead of serving bad data.

**Failure model:** crash-stop with restart, process pauses, network partitions, message loss, duplication, delay and reordering, power loss (unsynced writes lost, torn writes). Byzantine faults, permanent loss of a majority, and disks that lie about fsync are out of scope.

## Architecture

```
  Client / REPL / Dashboard
          |  client protocol: framed binary over TCP
  +-------v--------------------------------------------+
  |  Quorum node                                       |
  |  Query layer     lexer > parser > planner > ops    |
  |  Client server   sessions, routing, watch hub      |
  |  Raft module     election, replication, snapshots  |
  |  KV state machine  apply, MVCC revisions, dedupe   |
  |  LSM storage engine  WAL, memtable, SSTables       |
  |  Platform interfaces: Clock, Transport, Disk       |
  +-------^--------------------------------------------+
          |  peer protocol: Raft RPCs over TCP
     Other nodes
```

All nondeterminism (time, network, disk) enters through three injectable interfaces. The same Raft and storage code runs in production and inside the simulator, which is what makes any failing run reproducible from a seed.

See [`docs/architecture.md`](docs/architecture.md) for the full design.

## Requirements

- Node.js, current LTS
- Linux or macOS
- No native addons, no runtime dependencies

## Quick start _(planned)_

Install and type-check:

```sh
git clone <repo-url> quorum
cd quorum
npm install
npm run typecheck
```

Start a local 3-node cluster:

```sh
npm run cluster:local
# starts n1, n2, n3 on 127.0.0.1 with data under ./data/
```

Or start nodes individually:

```sh
quorumd --config ./quorum.n1.config.mjs
quorumd --config ./quorum.n2.config.mjs
quorumd --config ./quorum.n3.config.mjs
```

Talk to it:

```sh
quorumctl --endpoints 127.0.0.1:7001,127.0.0.1:7002,127.0.0.1:7003
```

### Node configuration

```js
// quorum.n1.config.mjs
/** @type {import('quorum/config').NodeConfig} */
export default {
  nodeId: "n1",
  dataDir: "./data/n1",
  clientListen: "127.0.0.1:7001",
  peerListen: "127.0.0.1:8001",
  peers: {
    n1: "127.0.0.1:8001",
    n2: "127.0.0.1:8002",
    n3: "127.0.0.1:8003",
  },
  fsync: "group", // 'always' | 'group' | 'none' (tests only)
  memtableBytes: 4 * 1024 * 1024,
  maxValueBytes: 1024 * 1024,
};
```

## Query language _(planned)_

```
PUT 'config/feature-x' 'on'
GET 'config/feature-x'
GET 'config/feature-x' AT 1042
DEL 'config/feature-x'

SCAN PREFIX 'config/' LIMIT 50
SCAN RANGE 'a' 'm' REVERSE

CAS 'lock/job-42' IF ABSENT SET 'worker-1'

TXN IF REV 'balance/a' = 7 AND ABSENT 'lock/a'
  PUT 'balance/a' '90';
  PUT 'balance/b' '110'
END

WATCH PREFIX 'config/' FROM 1042

STATUS
SHOW CLUSTER
COMPACT 1000
```

The full grammar and error format are in [`docs/query-language.md`](docs/query-language.md).

## Client library _(planned)_

```js
import { QuorumClient } from "quorum/client";

const client = await QuorumClient.connect({
  endpoints: ["127.0.0.1:7001", "127.0.0.1:7002", "127.0.0.1:7003"],
});

await client.put("config/feature-x", "on");

const { value, modRevision } = await client.get("config/feature-x");

const acquired = await client.cas("lock/job-42", {
  ifAbsent: true,
  set: "worker-1",
});

for await (const entry of client.scan({ prefix: "config/", limit: 50 })) {
  console.log(entry.key, entry.value);
}

for await (const event of client.watch({
  prefix: "config/",
  fromRevision: modRevision,
})) {
  console.log(event.type, event.key, event.revision);
}

await client.close();
```

The client discovers the leader, follows redirects, retries safely using sessions (writes execute at most once), enforces request deadlines, and ships with JSDoc types so `tsc --checkJs` consumers get full type information.

## Testing and verification _(planned)_

Quorum's claim to correctness is evidence, so verification is a first-class part of the repository.

```sh
npm test                    # unit and property tests (node:test, fast-check)
npm run typecheck           # tsc --checkJs, strict, zero errors
npm run sim -- --seeds 1000 # deterministic simulator across 1000 seeds
npm run sim -- --seed 4812  # replay one failing schedule exactly
npm run chaos               # real processes: SIGKILL, SIGSTOP, TCP-proxy partitions
npm run bench               # benchmark suite
```

| Layer          | Techniques                                                                                                                                                                                                                             |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Storage engine | Model-based property tests against an in-memory reference, crash-at-every-step tests, bit-flip and truncation injection                                                                                                                |
| Raft           | Scenario tests from the Raft paper, simulator with continuous invariant checks (election safety, leader append-only, log matching, leader completeness, state machine safety), deliberately buggy variants that the harness must catch |
| Protocol       | Frame fuzzing, split and coalesced chunk tests, slow-consumer and backpressure tests                                                                                                                                                   |
| Query language | Golden tests, parser fuzzing, planner property tests                                                                                                                                                                                   |
| System         | Recorded histories checked for linearizability, multi-hour soak tests with periodic faults                                                                                                                                             |

Any failing simulator seed is added to a permanent regression list.

## Benchmarks

Not yet measured. Initial targets for a 3-node cluster on a single developer machine (SSD, 128-byte values) are tracked in the spec and will be replaced by measured results, with hardware, Node version, fsync policy, concurrency, and percentile latencies, in [`docs/benchmarks.md`](docs/benchmarks.md).

## Project layout

```
quorum/
  src/
    platform/   clock, TCP transport, disk adapters
    storage/    wal, memtable, sstable, bloom, manifest, compaction, iterator, engine
    raft/       node, log, snapshot, readindex, sessions
    kv/         state machine, mvcc, watch hub
    net/        frame, codec, server, peer
    query/      lexer, parser, ast, planner
    client/     client, connection pool
    cli/        quorumd, quorumctl, repl
    dashboard/  server and static page
    sim/        simulated clock, network, disk, invariants
    verify/     history recorder, linearizability checker, chaos runner
  test/  bench/  docs/
```

## Conventions

- ES modules only; every source file is `.mjs`.
- JSDoc on all exported functions and data structures; `tsc` runs with `allowJs`, `checkJs`, `strict`, and `noEmit` and must pass with zero errors.
- Tests use `node:test`. The only expected dev dependency is `fast-check`.
- Modules depend on the injected `Clock`, `Transport`, and `Disk` interfaces, never on global time, `net`, or `fs`, outside the `platform/` adapters.
- Every invariant in the docs has a named test; every paper-derived rule has a comment citing its section.

## Roadmap

- [ ] **Storage engine**: WAL, memtable, SSTables, leveled compaction, recovery, model-based tests
- [ ] **Raft core**: election, replication, snapshots, simulator with network faults
- [ ] **Protocol and query layer**: binary protocol, client sessions, REPL, parser fuzzing
- [ ] **Read path and watches**: ReadIndex reads, resumable watches
- [ ] **Verification**: disk fault simulation, linearizability checker, real-process chaos harness
- [ ] **Tooling and write-up**: dashboard, benchmarks, architecture document, live failover demo

Stretch goals: membership changes, lease reads, TTL keys and leases, MVCC snapshot-isolation transactions, secondary indexes, range sharding across Raft groups, optional TLS.

## Non-goals

SQL, joins, and schemas; multi-datacenter replication; Byzantine fault tolerance; authentication and multi-tenancy; automatic sharding in the core scope; Windows support; competing with RocksDB or etcd on raw throughput.

## Documentation

- [`docs/spec.md`](docs/spec.md): full project specification and requirements
- [`docs/architecture.md`](docs/architecture.md): design and rationale
- [`docs/invariants.md`](docs/invariants.md): safety invariants and the tests that enforce them
- [`docs/on-disk-formats.md`](docs/on-disk-formats.md): WAL, SSTable, manifest, and snapshot formats
- [`docs/paper-notes.md`](docs/paper-notes.md): mapping from paper sections to code

## References

- Ongaro and Ousterhout, _In Search of an Understandable Consensus Algorithm_ (the Raft paper), and Ongaro's PhD thesis
- O'Neil et al., _The Log-Structured Merge-Tree_; the LevelDB and RocksDB design documents
- Herlihy and Wing, _Linearizability: A Correctness Condition for Concurrent Objects_
- Kleppmann, _Designing Data-Intensive Applications_; Petrov, _Database Internals_

## License

Not yet chosen. Add a `LICENSE` file before publishing.
