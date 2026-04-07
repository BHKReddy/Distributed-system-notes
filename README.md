# Distributed Systems — Deep Dive

Everything beyond the Educative lesson — CAP theorem, PACELC, linearizability, CRDTs, Paxos, Raft, TrueTime, and how real systems like Cassandra, Spanner, DynamoDB, MongoDB, and Kafka apply these concepts.

## Contents

- [CAP Theorem + PACELC](#cap-theorem--pacelc)
- [Linearizability vs Serializability](#linearizability-vs-serializability)
- [Clocks: Lamport → Vector → HLC](#clocks-lamport-→-vector-→-hlc)
- [CRDTs](#crdts)
- [Paxos + Raft](#paxos--raft)
- [How Real Systems Apply These Concepts](#how-real-systems-apply-these-concepts)
- [Two-Phase Commit (2PC) + Failure Modes](#two-phase-commit-2pc--failure-modes)

---

## 01 — Foundation

### CAP Theorem + PACELC

The deeper "why" behind every consistency trade-off.

**CAP theorem (Brewer, 2000)**

In the presence of a network **Partition**, a distributed system can guarantee either **Consistency** or **Availability** — but not both simultaneously.

### CAP categories

- **CP systems** — choose Consistency over Availability during a partition.
  - When nodes can't communicate, the system refuses to respond rather than return potentially stale data.
  - Examples: HBase, Zookeeper, etcd, Redis (cluster mode), MongoDB (default).
  - Use when correctness is non-negotiable — banking, leader election, configuration.

- **AP systems** — choose Availability over Consistency during a partition.
  - Every node continues responding, but different nodes may return different (stale) values.
  - Examples: Cassandra, DynamoDB (default), Riak, CouchDB.
  - Use when uptime is critical and stale reads are acceptable — social media, DNS, shopping carts.

- **CA systems** — Consistent AND Available, but no partition tolerance.
  - This only works on a single node or a perfectly reliable network. In practice, network partitions always happen, so pure CA distributed systems don't exist.
  - Examples: Single-node PostgreSQL, single-node MySQL (not distributed).
  - Note: Brewer clarified that "CA" in distributed systems is essentially theoretical.

### PACELC theorem (Abadi, 2012)

CAP only covers the partition case. PACELC extends it:

- **If Partition** → choose Consistency or Availability.
- **Else** (normal operation) → choose Latency or Consistency.

Most real systems trade latency for consistency even without partitions.

| System         | Partition behaviour  | Normal behaviour                    | Classification |
| -------------- | -------------------- | ----------------------------------- | -------------- |
| DynamoDB       | Prefers Availability | Prefers Latency (async replication) | PA/EL          |
| Cassandra      | Prefers Availability | Tunable (ONE→ALL)                   | PA/EL          |
| Google Spanner | Prefers Consistency  | Prefers Consistency (TrueTime)      | PC/EC          |
| Zookeeper      | Prefers Consistency  | Prefers Consistency                 | PC/EC          |
| MongoDB        | Prefers Consistency  | Prefers Latency                     | PC/EL          |

---

## 02 — Consistency Precision

### Linearizability vs Serializability

Two terms often confused — they mean very different things.

#### Linearizability

- A **single-object** guarantee.
- Every operation appears to take effect atomically at some point between its invocation and completion.
- The system behaves as if there's one copy of the data.
- Also called "atomic consistency" or "strong consistency."
- **Key property:** real-time ordering is preserved. If operation A completes before operation B starts, B must see A's result.

#### Serializability

- A **multi-object / transaction** guarantee.
- Concurrent transactions produce a result equivalent to some serial execution.
- Does NOT require real-time ordering — a serializable execution might look like transactions ran in the past.
- **Key property:** used in databases (SQL ISOLATION LEVEL SERIALIZABLE). Weaker than linearizability in terms of time.

#### Strict Serializability

- Strict Serializability = Linearizability + Serializability.
- The strongest guarantee: transactions appear to execute serially AND in real-time order.
- Google Spanner provides this. Most systems provide one or the other, not both.

#### Why the confusion?

- Both terms use the word "consistency" in conversation.
- Linearizability is about individual reads/writes on one object.
- Serializability is about transactions across multiple objects.
- A bank transfer (debit account A, credit account B) needs serializability.
- Reading the latest balance needs linearizability.

#### Isolation levels (weakest → strongest)

| Level                | Prevents                     | Still allows                        | Real system           |
| -------------------- | ---------------------------- | ----------------------------------- | --------------------- |
| `READ UNCOMMITTED`   | Nothing                      | Dirty reads, phantoms, lost updates | Rarely used           |
| `READ COMMITTED`     | Dirty reads                  | Non-repeatable reads, phantoms      | PostgreSQL default    |
| `REPEATABLE READ`    | Dirty + non-repeatable reads | Phantom reads                       | MySQL InnoDB default  |
| `SNAPSHOT ISOLATION` | Most anomalies               | Write skew                          | CockroachDB, Spanner  |
| `SERIALIZABLE`       | All anomalies                | —                                   | PostgreSQL (explicit) |

---

## 03 — Time in Distributed Systems

### Clocks: Lamport → Vector → HLC

How distributed systems reason about time without a global clock.

#### Lamport Clocks (1978)

- Each process keeps a counter.
- On send: increment and attach.
- On receive: take `max(local, received) + 1`.
- Gives a partial ordering — if A → B then `LC(A) < LC(B)`.
- But `LC(A) < LC(B)` does NOT mean A → B.
- Can't detect concurrency.

#### Vector Clocks (Fidge/Mattern, 1988)

- Each process keeps a vector of counters (one per process).
- On send: increment own slot, attach full vector.
- On receive: element-wise max + increment.
- Now: A → B iff `VC(A) < VC(B)` component-wise.
- Concurrent if neither dominates.
- Detects concurrency — used in Dynamo, Riak.

#### Version Vectors

- Similar structure but track data versions, not event causality.
- Used to detect conflicts in replicated data.
- DynamoDB uses version vectors internally.
- Difference: vector clocks timestamp events; version vectors version data objects.

#### Hybrid Logical Clocks — HLC (Kulkarni et al., 2014)

- Combines physical time (wall clock) with logical time.
- Tracks maximum observed physical time + a logical counter for same-millisecond events.
- Gives causality tracking while staying close to real time.
- Used in CockroachDB and YugabyteDB.

#### TrueTime — Google Spanner (2012)

- Uses atomic clocks + GPS receivers in every datacenter.
- Exposes time as an interval `[earliest, latest]` with bounded uncertainty (~7ms).
- Spanner commits wait out this uncertainty before returning.
- Guarantees that if transaction A commits before B starts, `commit_ts(A) < commit_ts(B)`.
- Enables globally consistent snapshots.

#### HLC — how it works

```text
// Each node tracks: (physical_time, logical_counter)
// abbreviated as (pt, l)

On send message:
  pt = max(pt, wall_clock_now())
  l = l + 1
  attach (pt, l) to message

On receive message with timestamp (msg_pt, msg_l):
  if msg_pt == pt:
    l = max(l, msg_l) + 1
  else if msg_pt > pt:
    pt = msg_pt
    l = msg_l + 1
  else:
    l = l + 1

// Result: stays within a few ms of wall time,
// but still captures causality within the same ms
```

---

## 04 — Conflict-Free Data Structures

### CRDTs

Data structures that merge automatically — no coordination needed.

**The core idea**

- CRDTs (Conflict-free Replicated Data Types) are designed so any two replicas can be merged and the result is always the same, regardless of merge order.
- This makes eventual consistency correct by construction — no conflict resolution logic needed.

#### State-based CRDTs (CvRDTs)

- Nodes periodically exchange their full state.
- Merge function must be commutative, associative, and idempotent.
- Example: a G-Counter where each node tracks its own increment. Merge = element-wise max. Value = sum of all.

#### Operation-based CRDTs (CmRDTs)

- Nodes broadcast operations (not full state).
- Operations must be commutative.
- Smaller messages — send "increment" instead of full state.
- Requires reliable delivery (no lost ops).
- Example: collaborative text editors.

#### Common CRDT types

| Type         | Operations                 | Merge rule              | Used in                   |
| ------------ | -------------------------- | ----------------------- | ------------------------- |
| G-Counter    | increment                  | element-wise max, sum   | View counters, likes      |
| PN-Counter   | increment, decrement       | two G-Counters (P - N)  | Inventory, balance        |
| G-Set        | add                        | set union               | Tags, membership          |
| 2P-Set       | add, remove                | union of adds ∪ removes | Shopping cart (limited)   |
| OR-Set       | add, remove (any order)    | unique tags per add     | Riak, collaborative lists |
| LWW-Register | write with timestamp       | last-write-wins         | Profile fields            |
| RGA / LSEQ   | insert, delete at position | unique position IDs     | Google Docs, Figma        |

---

## 05 — Consensus Algorithms

### Paxos + Raft

How distributed nodes agree on a single value despite failures.

**The consensus problem**

Given N nodes, some of which may crash or be unreachable, reach agreement on a single value such that:

1. only proposed values are chosen,
2. only one value is chosen,
3. a node only learns a chosen value.

FLP impossibility theorem: no deterministic algorithm can guarantee consensus in an asynchronous system with even one crash failure. Paxos and Raft work around this with timeouts and probabilistic guarantees.

### Paxos — phase by phase

#### Phase 1a: Prepare

- Proposer sends `Prepare(n)` to all Acceptors.
- `n` is a proposal number (unique, higher than any seen).
- It asks: "Will you promise not to accept any proposal numbered less than n?"

**Insight:** The proposal number is like a ballot number in an election. Higher = more recent. The proposer doesn't send a value yet — just asks for a promise.

#### Phase 1b: Promise

- Acceptor replies `Promise(n, v_prev)` if `n` is the highest proposal number seen.
- It promises to ignore proposals numbered less than n.
- It also includes the highest-numbered accepted value so far, if any.

**Insight:** If any acceptor returns a previously accepted value, the proposer MUST use that value — not its own. This is how Paxos ensures consistency across failed rounds.

#### Phase 2a: Accept

- Once the proposer receives promises from a majority, it sends `Accept(n, v)`.
- `v` is either the proposer’s own value (if no prior value was returned) or the highest-numbered previously accepted value from the promises.

**Insight:** Majority quorums are the key. Any two majorities overlap by at least one node — so any two rounds of Paxos share at least one acceptor, preventing contradictory decisions.

#### Phase 2b: Accepted

- Each acceptor accepts `Accept(n, v)` if `n` is still the highest proposal number it has promised.
- It then notifies all Learners of the accepted value.

**Insight:** An acceptor can receive multiple Accept messages from competing proposers. It only accepts the one with the highest proposal number — this is the "last writer wins" of Paxos.

#### Learn

- A value is chosen when a majority of acceptors have accepted the same `(n, v)`.
- Learners can now act on this value.
- Multi-Paxos extends this to a log of values by electing a stable leader to skip Phase 1 for subsequent entries.

**Insight:** Paxos is notoriously hard to implement correctly. Raft was designed specifically to be easier to understand — same guarantees, more prescriptive structure.

### Raft — leader election simulator

- One leader at a time (per term).
- Leader replicates log entries to followers.
- Entry committed when majority acknowledges.
- Leader election via randomized timeouts — reduces split votes.
- Safety: a node only votes for a candidate whose log is at least as up-to-date as its own.

#### Paxos vs Raft

- **Paxos:** theoretical foundation. Flexible but underspecified — each implementation makes different choices. Used in Chubby (Google), Zookeeper (ZAB variant).
- **Raft:** designed for understandability. Prescribes leader election, log replication, and safety rules explicitly. Used in etcd, CockroachDB, TiKV, Consul.

---

## 06 — Real Systems

### How Real Systems Apply These Concepts

#### Cassandra — tunable consistency

Cassandra lets you choose consistency level per query. The magic number is `R + W > N`, which guarantees strong consistency when read quorum and write quorum overlap.

- `N` = replicas
- `W` = write replicas
- `R` = read replicas
- If `R + W > N`, strong consistency is guaranteed.
- Cost: higher latency, lower availability.
- If `R + W ≤ N`, eventual consistency only. A reader may not see the latest write, but latency and availability are better.

#### Google Spanner — TrueTime explained

- Spanner uses atomic clocks + GPS to bound clock uncertainty to ~7ms.
- Every transaction must wait out this uncertainty before committing.
- This pause is the price of global consistency.
- `TrueTime.now()` returns interval `[T_earliest, T_latest]`.
- On commit, Spanner waits until `T_latest` passes (`commit_wait`).
- Guarantee: if Txn A commits before Txn B starts, `ts(A) < ts(B)`.

This is the only production system providing external consistency (linearizability + serializability) at global scale. Other systems either relax consistency or require explicit coordination per transaction.

#### MongoDB — evolution of consistency

- Before v3.6 — eventual consistency only. Reads from secondaries could return stale data. No causal guarantees across sessions.
- v3.6 — causal consistency sessions. Uses cluster time (a hybrid logical clock) to ensure reads always see the writes from the same session in order.
- v4.0+ — multi-document transactions. Added ACID transactions across multiple documents and collections. Uses snapshot isolation. Combined with causal sessions gives near-serializable guarantees within a replica set.

---

## 07 — Distributed Transactions

### Two-Phase Commit (2PC) + Failure Modes

How distributed transactions work — and why they're fragile.

#### Phase 1: Prepare

- Coordinator sends PREPARE to all participants.
- Participants check if they can commit, lock resources, write to a redo log, and reply VOTE-YES or VOTE-NO.

**Insight:** Once a participant votes YES, it's locked in — it cannot unilaterally abort anymore. This is the "point of no return" for that participant.

#### Phase 2: Commit

- If all participants voted YES, the coordinator sends COMMIT.
- Each participant applies the transaction, releases locks, and acknowledges.
- If any voted NO, the coordinator sends ABORT and all participants roll back.

**Insight:** The coordinator must durably log its decision before sending — if it crashes after deciding but before all participants hear it, recovery must re-send the decision.

#### Failure: coordinator crash

- If the coordinator crashes after receiving all votes but before sending its decision, participants are blocked.
- They cannot commit or abort safely without the coordinator’s decision.

**Insight:** This is 2PC's fatal flaw. It is a blocking protocol — a coordinator crash can hold locks indefinitely.

#### Failure: participant crash

- If a participant crashes after voting YES but before receiving the decision, it must check its redo log on recovery.
- If it finds PREPARE, it asks the coordinator for the decision and then applies it.
- If it finds COMMIT, it commits. If it finds nothing, it can safely abort.

**Insight:** The redo log is essential. Without durable logging, recovery is impossible.

#### 3PC — the fix

- Three-Phase Commit adds a PRE-COMMIT phase.
- After all YES votes, the coordinator sends PRE-COMMIT before the actual COMMIT.
- If participants see PRE-COMMIT, they know a COMMIT is coming — they can safely commit even if the coordinator crashes.

**Insight:** 3PC works under crash failures but not under network partitions. A partitioned participant that received PRE-COMMIT and one that didn't can make conflicting decisions. In practice, 2PC + recovery is more commonly used than 3PC.

#### Modern alternatives to 2PC

- **Saga pattern:** Break a distributed transaction into a sequence of local transactions, each with a compensating action for rollback. No global locks — eventual consistency with explicit rollback logic. Used in microservices.
- **Percolator (Google):** Optimistic concurrency with timestamps. Read a snapshot, detect conflicts at commit time. Used in TiKV / TiDB.
- **Calvin:** Pre-order transactions before executing — avoids distributed locking entirely by making sequencing the only coordination step.

---

Distributed-systems deep dive · covers CAP, PACELC, linearizability, CRDTs, Paxos, Raft, TrueTime, 2PC · built with Claude
